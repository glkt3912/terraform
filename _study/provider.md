# プロバイダープラグイン（gRPC プロトコル・Plugin Protocol）

## 概要

Terraform の Provider は独立したバイナリプロセスとして動作し、Core と gRPC で通信する。
Provider は特定クラウド/サービス（AWS、GCP、Azure 等）の API をラップし、Resource・Data Source の CRUD を実装する。

## Plugin Protocol の進化

| バージョン | 通信方式 | 用途 |
|-----------|---------|------|
| Protocol v5 | gRPC + protobuf | Terraform 0.12 以降 |
| Protocol v6 | gRPC + protobuf | Terraform 1.0 以降（推奨） |

Protocol v6 の主な改善点：

- `GetProviderSchema` のレスポンス形式変更
- Nested attributes のサポート
- `MoveResourceState` RPC の追加（Terraform 1.8+）

## プロセス起動フロー

```
terraform apply
  1. Provider バイナリを exec.Command で起動
  2. Provider が stdout にハンドシェイク情報を出力
  3. Core が gRPC クライアントとして接続
  4. GetProviderSchema で schema 取得
  5. ConfigureProvider でプロバイダー設定（credentials 等）
  6. 各 Resource 操作（Plan/Apply）で RPC 呼び出し
```

## gRPC RPC 一覧（主要）

| RPC | 用途 |
|-----|------|
| `GetProviderSchema` | Resource/DataSource の schema 取得 |
| `ConfigureProvider` | Provider 設定（認証情報等） |
| `ValidateResourceConfig` | Resource 設定の検証 |
| `PlanResourceChange` | 変更計画の計算（Core からの diff 要求） |
| `ApplyResourceChange` | 変更の実際の適用 |
| `ReadResource` | 現在の実際の状態を読み取り（refresh） |
| `ImportResourceState` | 既存リソースの import |
| `ReadDataSource` | Data Source の読み取り |
| `MoveResourceState` | リソースアドレス変更時の state 移行（v6） |

## Proto 定義ファイル

```
internal/tfplugin6/tfplugin6.proto   # Protocol v6 定義
internal/tfplugin5/tfplugin5.proto   # Protocol v5 定義
```

## Provider Schema の構造

```go
// internal/providers/provider.go
type Schema struct {
    Block *configschema.Block  // 属性定義
}

// configschema.Block
type Block struct {
    Attributes map[string]*Attribute
    BlockTypes map[string]*NestedBlock
}

type Attribute struct {
    Type        cty.Type
    Required    bool
    Optional    bool
    Computed    bool
    Sensitive   bool
    Description string
}
```

## cty 型システム

Provider との値やり取りは `cty`（`github.com/zclconf/go-cty`）型を使用。

```
cty.String, cty.Number, cty.Bool     # プリミティブ
cty.List(elemType)                    # リスト
cty.Map(elemType)                     # マップ
cty.Set(elemType)                     # セット
cty.Object(attrTypes)                 # オブジェクト
cty.DynamicPseudoType                 # any（動的型）
```

## Provider の検索・インストール

```
terraform init 時:
  1. 設定の required_providers を解析
  2. ~/.terraform.d/plugins をチェック
  3. なければ registry.terraform.io から取得
  4. .terraform/providers/ にバイナリを保存
  5. .terraform.lock.hcl にハッシュ・バージョン記録
```

## 関連パッケージ

| パッケージ | 内容 |
|-----------|------|
| `internal/providers` | Provider インターフェース定義 |
| `internal/plugin` | gRPC クライアント実装（plugin5） |
| `internal/plugin6` | gRPC クライアント実装（plugin6） |
| `internal/providercache` | Provider バイナリキャッシュ管理 |
| `internal/getproviders` | Provider 取得・インストール |
