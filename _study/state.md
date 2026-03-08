# ステート管理（tfstate 構造・Remote Backend・Locking）

## 概要

Terraform の State（`terraform.tfstate`）は、実際のインフラとコード定義のマッピングを保持する JSON ファイル。
Terraform はこの State を参照して「現在の状態」を把握し、Plan で差分を計算する。

## tfstate の JSON 構造

```json
{
  "version": 4,
  "terraform_version": "1.5.0",
  "serial": 42,
  "lineage": "550e8400-e29b-41d4-a716-446655440000",
  "outputs": {
    "bucket_name": {
      "value": "my-bucket",
      "type": "string"
    }
  },
  "resources": [
    {
      "module": "module.network",
      "mode": "managed",
      "type": "aws_vpc",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "vpc-12345678",
            "cidr_block": "10.0.0.0/16"
          },
          "sensitive_attributes": [],
          "dependencies": ["aws_internet_gateway.main"]
        }
      ]
    }
  ]
}
```

## 主要フィールド

| フィールド | 説明 |
|-----------|------|
| `version` | State フォーマットのバージョン（現在 4） |
| `serial` | 変更のたびにインクリメントされるカウンター（競合検出に使用） |
| `lineage` | State の一意識別子 UUID（初回作成時に生成） |
| `mode` | `managed`（resource）か `data`（data source） |
| `sensitive_attributes` | 機密値のパス一覧（表示時にマスク） |

## Go の State 型構造

```go
// internal/states/state.go
type State struct {
    Modules map[string]*Module
}

type Module struct {
    Resources map[string]*Resource
    OutputValues map[string]*OutputValue
}

type Resource struct {
    Addr          addrs.AbsResource
    EachMode      EachMode  // NoEach, EachList, EachMap
    Instances     map[addrs.InstanceKey]*ResourceInstance
    ProviderConfig addrs.AbsProviderConfig
}

type ResourceInstance struct {
    Current  *ResourceInstanceObjectSrc
    Deposed  map[DeposedKey]*ResourceInstanceObjectSrc
}
```

## Deposed（廃棄待ち）インスタンス

`create_before_destroy = true` のリソース変更時に発生：

```
1. 新リソース作成 → Current に追加
2. 旧リソースを Deposed キューに移動
3. Apply 完了後に Deposed を削除
4. クラッシュ時は Deposed が残り、次回 apply で削除
```

## Remote Backend と State 保存

State はデフォルトでローカルの `terraform.tfstate` に保存。Remote Backend を使うことで共有・管理が可能。

```hcl
# S3 backend 設定例
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-northeast-1"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}
```

## State Locking

同時実行による State 破損を防ぐためのロック機構。

| Backend | ロック機構 |
|---------|-----------|
| S3 | DynamoDB テーブル（LockID カラム） |
| GCS | GCS オブジェクトロック |
| azurerm | Azure Blob リース |
| Terraform Cloud | API レベルのロック |
| local | ファイルロック（`.terraform.tfstate.lock.info`） |

```go
// internal/states/statemgr/locker.go
type Locker interface {
    Lock(info *LockInfo) (string, error)
    Unlock(id string) error
}

type LockInfo struct {
    ID        string
    Operation string  // "plan", "apply" 等
    Info      string
    Who        string  // ユーザー名@ホスト名
    Version   string
    Created   time.Time
    Path      string
}
```

## State の読み書きフロー

```
terraform apply:
  1. Backend.StateMgr() でマネージャー取得
  2. StateMgr.Lock() でロック取得
  3. StateMgr.RefreshState() でリモートから読み込み
  4. Apply 実行
  5. StateMgr.WriteState() でメモリ更新
  6. StateMgr.PersistState() でリモートに書き込み
  7. StateMgr.Unlock() でロック解放
```

## State の暗号化（Terraform 1.7+）

```hcl
terraform {
  encryption {
    key_provider "pbkdf2" "my_key" {
      passphrase = var.state_passphrase
    }
    method "aes_gcm" "my_method" {
      keys = key_provider.pbkdf2.my_key
    }
    state {
      method = method.aes_gcm.my_method
    }
  }
}
```

## 関連パッケージ

| パッケージ | 内容 |
|-----------|------|
| `internal/states` | State 型定義・操作 |
| `internal/states/statemgr` | State Manager インターフェース・実装 |
| `internal/backend` | Backend インターフェース定義 |
| `internal/backend/local` | ローカルバックエンド実装 |
