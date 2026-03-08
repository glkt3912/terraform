# バックエンド設定（S3/GCS/Consul の仕組み）

## なぜ Remote Backend が必要なのか？

デフォルトのローカル State（`terraform.tfstate`）には根本的な問題がある。

| 問題 | 説明 |
|-----|------|
| **チーム共有できない** | ローカルファイルは git にコミットしにくい（機密値が含まれるため） |
| **同時実行できない** | 複数人が同時に apply すると State が壊れる |
| **CI/CD に載せにくい** | CI のエフェメラルな実行環境に State が残らない |

Remote Backend はこれらをすべて解決する。

> **State を git に入れてはいけない理由**: State には `sensitive = true` の値も平文で記録される。RDS のマスターパスワード、TLS 秘密鍵などが含まれることがあり、git 履歴に残ると取り返しがつかない。

---

## バックエンドの2つの役割

```mermaid
graph TD
    Backend[Backend]
    Backend --> Storage["① State Storage\nState の保存・読み込み・ロック"]
    Backend --> Operations["② Operations\nPlan/Apply をどこで実行するか"]
    Storage --> Local["ローカル実行\n（ほとんどの Backend）"]
    Operations --> Remote["リモート実行\n（Terraform Cloud のみ）"]
```

---

## S3 バックエンドの内部動作

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "environments/prod/terraform.tfstate"
    region         = "ap-northeast-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:..."
  }
}
```

```mermaid
sequenceDiagram
    participant Core
    participant S3
    participant DynamoDB

    Core->>DynamoDB: PutItem(LockID="bucket/key", Who="user@host")\nConditionExpression: attribute_not_exists(LockID)
    alt ロック取得成功
        DynamoDB-->>Core: OK
        Core->>S3: GetObject(bucket, key)
        S3-->>Core: terraform.tfstate の JSON
        Note over Core: Apply 実行
        Core->>S3: PutObject(bucket, key, new_state)
        Core->>DynamoDB: DeleteItem(LockID="bucket/key")
    else ロック取得失敗（既に誰かがロック中）
        DynamoDB-->>Core: ConditionalCheckFailedException
        Core-->>User: Error: Error acquiring the state lock
    end
```

**なぜ DynamoDB を使うのか？**

S3 はオブジェクトの条件付き書き込みが弱い（S3 の条件付きリクエストは 2024 年に追加されたが普及途上）。
DynamoDB の `ConditionExpression` は原子的な比較・書き込みを保証するため、分散ロックの実装に適している。

---

## GCS バックエンドのロック機構

```mermaid
sequenceDiagram
    participant Core
    participant GCS

    Core->>GCS: オブジェクト作成\n(gs://bucket/prefix/default.tflock)
    alt 世代番号 0（オブジェクトが存在しない）で成功
        GCS-->>Core: OK（ロック取得）
        Note over Core: Apply 実行
        Core->>GCS: ロックファイル削除
    else 既にオブジェクトが存在
        GCS-->>Core: 412 Precondition Failed
        Core-->>User: Error: Error acquiring the state lock
    end
```

`x-goog-if-generation-match: 0` ヘッダーにより、
オブジェクトが存在しない場合のみ作成を許可する楽観的ロックを実装している。

---

## Consul バックエンドのロック機構

```hcl
terraform {
  backend "consul" {
    address      = "consul.example.com:8500"
    path         = "terraform/myapp/prod"
    access_token = var.consul_token
    lock         = true
  }
}
```

```mermaid
sequenceDiagram
    participant Core
    participant Consul

    Core->>Consul: Session.Create(TTL="15s")
    Consul-->>Core: session_id
    Core->>Consul: KV.Acquire(key, session_id)\n（CAS 操作）
    alt 取得成功
        Consul-->>Core: true
        Note over Core,Consul: Session の TTL を定期更新（ハートビート）
        Note over Core: Apply 実行
        Core->>Consul: Session.Destroy(session_id)
    else 取得失敗
        Consul-->>Core: false
        Core-->>User: Error: Error acquiring the state lock
    end
```

Consul Session の TTL による自動解放:
Core がクラッシュしてもセッションが TTL 切れになればロックが自動解放される。

---

## Terraform Cloud バックエンド（推奨）

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

```mermaid
graph TD
    A[terraform plan / apply]
    A --> B[Terraform Cloud API\nに Plan/Apply をキューイング]
    B --> C[Terraform Cloud 上で\nリモート実行]
    C --> D[State を Terraform Cloud に保存]
    C --> E[実行ログをストリーミング]
    E --> F[ローカルターミナルに表示]
```

**Terraform Cloud が推奨される理由:**

- API レベルのロックで最も堅牢
- 実行ログの永続化・監査ログ
- Plan の UI レビュー・承認フロー
- State の暗号化・バージョン履歴

---

## バックエンド初期化フロー

```mermaid
flowchart TD
    A[terraform init] --> B[設定の backend ブロックを読み込み]
    B --> C{.terraform/terraform.tfstate の\n前回設定と比較}
    C -- 変更なし --> F[既存 Backend を使用]
    C -- 変更あり --> D[State の移行を提案]
    D --> E[現在の State を旧 Backend から読み込み\n新 Backend に書き込み]
    E --> F
    F --> G[.terraform/terraform.tfstate に\nバックエンド設定を記録]
```

---

## 運用・障害の観点

| シナリオ | 症状 | 対処 |
|---------|------|------|
| DynamoDB テーブルが存在しない | `NoSuchTable` エラー | DynamoDB テーブルを事前に作成。または Terraform で管理する |
| S3 バケットが存在しない | `NoSuchBucket` | backend 設定より先にバケットを作成する（bootstrap 問題） |
| ロックが残る | `Error acquiring the state lock` | `terraform force-unlock <LOCK_ID>` |
| backend 変更後の State 移行失敗 | State が壊れる | 変更前にバックアップ（`terraform state pull > backup.tfstate`） |
| CI での Provider インストール失敗 | `registry.terraform.io` に届かない | プロキシ設定または `HTTPS_PROXY` 環境変数を確認 |

> **監視指標**: S3 + DynamoDB 構成では、DynamoDB の `ConditionalCheckFailedRequests` メトリクスを CloudWatch で監視する。増加傾向があれば Apply の同時実行が増えているサインで、チームのワークフロー見直しが必要。

---

## 関連パッケージ

| パッケージ | 内容 |
|-----------|------|
| `internal/backend` | Backend インターフェース定義 |
| `internal/backend/local` | ローカルバックエンド（デフォルト） |
| `internal/backend/remote` | Terraform Cloud バックエンド |
| `internal/backend/init` | バックエンド初期化・登録マップ |
| `internal/cloud` | Terraform Cloud API クライアント |
