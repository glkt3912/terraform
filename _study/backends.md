# バックエンド設定（S3/GCS/Consul の仕組み）

## 概要

Terraform の Backend は State の保存先と操作（Plan/Apply の実行場所）を定義する。ローカルの `terraform.tfstate` に保存するデフォルトのローカルバックエンドから、
チーム開発向けのリモートバックエンドまで多数サポートする。

## バックエンドの2つの役割

1. **State Storage**: tfstate の保存・読み込み・ロック
2. **Operations**: Plan/Apply をどこで実行するか（ローカル or リモート）

```go
// internal/backend/backend.go
type Backend interface {
    // State Storage
    StateMgr(workspace string) (statemgr.Full, error)
    Workspaces() ([]string, error)
    DeleteWorkspace(name string, force bool) error
}

// Operations backend（Plan/Apply をリモートで実行する場合）
type Enhanced interface {
    Backend
    Operation(context.Context, *Operation) (*RunningOperation, error)
}
```

## S3 バックエンド

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "environments/prod/terraform.tfstate"
    region         = "ap-northeast-1"

    # ロック設定（DynamoDB）
    dynamodb_table = "terraform-state-lock"

    # 暗号化
    encrypt        = true
    kms_key_id     = "arn:aws:kms:ap-northeast-1:123456789:key/xxxxx"

    # 認証（環境変数 AWS_PROFILE 等でも可）
    profile        = "my-aws-profile"
  }
}
```

S3 バックエンドの内部動作：

```
State 読み込み:
  s3.GetObject(bucket, key) → JSON をデシリアライズ

State 書き込み:
  s3.PutObject(bucket, key, body)

ロック取得:
  dynamodb.PutItem(
    TableName: "terraform-state-lock",
    Item: { LockID: "bucket/key", Info: {...} },
    ConditionExpression: "attribute_not_exists(LockID)"
  )

ロック解放:
  dynamodb.DeleteItem(TableName, { LockID: "bucket/key" })
```

## GCS バックエンド

```hcl
terraform {
  backend "gcs" {
    bucket  = "my-terraform-state"
    prefix  = "terraform/state"

    # ロック: GCS オブジェクトロック（metadata に記録）
  }
}
```

GCS バックエンドのロック機構：

- GCS の Object Lock（正確には `x-goog-if-generation-match` ヘッダーを使用した楽観的ロック）
- `gs://bucket/prefix/default.tflock` オブジェクトの存在確認
- 世代番号を使って競合を検出

## Consul バックエンド

```hcl
terraform {
  backend "consul" {
    address = "consul.example.com:8500"
    scheme  = "https"
    path    = "terraform/myapp/prod"

    # アクセス制御
    access_token = var.consul_token

    # ロック: Consul Session + KV
    lock = true
  }
}
```

Consul バックエンドのロック機構：

```
1. consul.Session.Create() でセッション作成
2. consul.KV.Acquire(key, session_id) でロック取得（CAS 操作）
3. ロック取得失敗時は待機・リトライ
4. consul.Session.Destroy() でセッション削除（ロック自動解放）
```

## azurerm バックエンド

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate"
    storage_account_name = "mystorageaccount"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"

    # ロック: Azure Blob Lease
  }
}
```

## Terraform Cloud バックエンド

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "my-workspace"
    }
  }
}
```

- Plan/Apply を Terraform Cloud 上で実行（Enhanced backend）
- State の保存も Terraform Cloud
- API トークンで認証（`~/.terraform.d/credentials.tfrc.json`）

## バックエンド初期化フロー

```
terraform init:
  1. 設定の backend ブロックを読み込み
  2. 前回の backend 設定（.terraform/terraform.tfstate）と比較
  3. 変更がある場合: State の移行を提案
     - 現在の State を読み込み
     - 新 Backend に書き込み
     - 旧 Backend から削除
  4. .terraform/terraform.tfstate にバックエンド設定を記録
```

## Workspace のサポート

```bash
terraform workspace new staging
terraform workspace select prod
terraform workspace list
```

```go
// バックエンドが workspace に対応している場合
// State のパスが変わる（例: S3 の場合）
// env:/staging/terraform.tfstate
// env:/prod/terraform.tfstate
```

## ロック競合時の強制解除

```bash
# ロックが残った場合の強制解除
terraform force-unlock LOCK_ID
```

## 関連パッケージ

| パッケージ | 内容 |
|-----------|------|
| `internal/backend` | Backend インターフェース定義 |
| `internal/backend/local` | ローカルバックエンド（デフォルト） |
| `internal/backend/remote` | Terraform Cloud バックエンド |
| `internal/backend/init` | バックエンド初期化・登録マップ |
| `internal/cloud` | Terraform Cloud API クライアント |
| `website/docs/language/settings/backends/` | 各バックエンドの公式ドキュメント |
