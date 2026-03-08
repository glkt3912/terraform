# Plan/Apply ライフサイクル（diff 計算・変更適用の内部フロー）

## 概要

`terraform plan` は「望む状態」と「現在の状態」の差分を計算し、実行計画を生成する。`terraform apply` はその計画を実行してインフラを実際に変更する。

## Plan フェーズの全体フロー

```
terraform plan
  1. Config ロード（HCL → internal/configs.Config）
  2. State ロード（Backend から tfstate 読み込み）
  3. Provider 初期化（plugin 起動・ConfigureProvider）
  4. Graph 構築（planGraphBuilder）
  5. Graph Walk（各ノードを評価）
     a. Provider の ReadResource で現在の実態を確認（refresh）
     b. PlanResourceChange RPC で変更計画を計算
     c. 依存リソースの順序を考慮して並列実行
  6. Plan オブジェクト生成（internal/plans.Plan）
  7. planfile に書き出し（-out オプション時）
```

## Graph の構築

Terraform はリソース間の依存関係を DAG（有向非巡回グラフ）で表現。

```go
// internal/terraform/graph_builder_plan.go
type PlanGraphBuilder struct {
    Config       *configs.Config
    State        *states.State
    RootVariableValues map[string]plans.DynamicValue
    // ...
}

// 主要な Graph ノード型
type NodeAbstractResource      // 抽象リソースノード
type NodeApplyableResource     // Apply 可能なリソース
type NodeDestroyableResource   // Destroy 対象のリソース
type NodeAbstractProvider      // Provider ノード
```

## 依存関係の解決

```hcl
# 明示的依存 (depends_on)
resource "aws_instance" "app" {
  depends_on = [aws_security_group.app]
}

# 暗黙的依存（属性参照）
resource "aws_instance" "app" {
  subnet_id = aws_subnet.main.id  # aws_subnet.main への依存が自動生成
}
```

依存解決フロー：

1. 属性参照を走査して暗黙的依存を抽出
2. `depends_on` の明示的依存を追加
3. DAG を構築してトポロジカルソート
4. 並列実行可能なノードを goroutine で同時処理

## PlanResourceChange の動作

```
Core → Provider:
  PlanResourceChange(
    TypeName: "aws_instance",
    PriorState: <現在の tfstate の値>,
    ProposedNewState: <コードの設定値>,
    Config: <HCL の設定>,
  )

Provider → Core:
  PlanResourceChange Response:
    PlannedState: <計画後の状態（Computed 値は unknown）>
    RequiresReplace: [<変更時に再作成が必要な属性>]
```

## Plan の変更アクション

| アクション | 説明 |
|-----------|------|
| `Create` | 新規リソース作成 |
| `Update` | in-place 更新（既存リソースの属性変更） |
| `Delete` | リソース削除 |
| `Replace` | 削除して再作成（破壊的変更） |
| `Read` | Data Source の読み取り |
| `NoOp` | 変更なし |

## Apply フェーズの全体フロー

```
terraform apply [planfile]
  1. Plan の読み込み（planfile or 再 Plan）
  2. State ロック取得
  3. Apply Graph 構築（applyGraphBuilder）
  4. Graph Walk（トポロジカル順・並列実行）
     a. Create/Update: ApplyResourceChange RPC
     b. Delete: ApplyResourceChange RPC（空の planned state）
     c. 各ステップ完了後に State を部分更新
  5. State 書き込み・ロック解放
```

## Apply 中の State 更新

```go
// 各リソース Apply 完了ごとに State を更新
// クラッシュ時でも途中までの State が保持される
func (n *NodeApplyableResource) Execute(ctx EvalContext, op walkOperation) error {
    // ... Apply 実行 ...
    state.SetResourceInstanceCurrent(addr, newState, providerAddr)
    // ↑ この時点でメモリの State は更新される
    // PersistState() は全体の Apply 完了後
}
```

## Refresh フロー（terraform refresh / plan -refresh）

```
ReadResource RPC:
  Core → Provider: { TypeName, CurrentState }
  Provider → Core: { NewState }  # 実際のクラウドの現在状態

差分がある場合:
  - State を実際の状態に更新
  - Plan で「State との差分」として検出
```

## Plan ファイルの構造

```
plan.tfplan（バイナリ、protobuf）
  ├── Changes（各リソースの変更アクション）
  ├── Variables（入力変数の値）
  ├── ProviderSHA256s（使用 Provider のハッシュ）
  └── Backend（バックエンド設定）
```

## 関連パッケージ

| パッケージ | 内容 |
|-----------|------|
| `internal/terraform` | Plan/Apply の主要ロジック・Graph Walker |
| `internal/plans` | Plan 型定義・protobuf シリアライズ |
| `internal/graphs` | DAG 実装（走査・並列実行） |
| `internal/dag` | 汎用 DAG データ構造 |
| `internal/command/views` | Plan 結果の表示フォーマット |
