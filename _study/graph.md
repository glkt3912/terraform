# 依存グラフ（DAG walk アルゴリズム・並列実行・サイクル検出）

## 概要

Terraform はリソース間の依存関係を DAG（有向非巡回グラフ）で表現し、トポロジカル順に並列実行する。
グラフの構築・走査は `internal/dag` と `internal/terraform` パッケージが担当する。

## dag パッケージの基本構造

```go
// internal/dag/graph.go
type Graph struct {
    vertices Set         // ノードの集合
    edges    *EdgeSet    // エッジの集合（依存関係）
    downEdges map[interface{}]Set  // ノード → 依存先
    upEdges   map[interface{}]Set  // ノード → 依存元
}

// AcyclicGraph は循環検出付き
type AcyclicGraph struct {
    Graph
}

// エッジ（依存関係）
type Edge interface {
    Source() Vertex  // 依存元
    Target() Vertex  // 依存先（先に実行される）
}
```

## グラフ構築フロー

```
planGraphBuilder.Build():
  1. ConfigTransformer    — config の resource を頂点に追加
  2. OrphanResourceTransformer — state にあるが config にないリソースを追加
  3. StateTransformer     — state の resource を参照頂点として追加
  4. ProviderTransformer  — provider 頂点を追加・resource と接続
  5. ReferenceTransformer — 属性参照を走査してエッジを追加
  6. DependsOnTransformer — depends_on のエッジを追加
  7. DestroyEdgeTransformer — destroy 順序のエッジを追加
  8. TransitiveReductionTransformer — 冗長なエッジを削除
```

各 Transformer は `GraphTransformer` インターフェースを実装：

```go
type GraphTransformer interface {
    Transform(*Graph) error
}
```

## Walk アルゴリズム（並列トポロジカル走査）

```go
// internal/dag/walk.go
type Walker struct {
    Callback   WalkFunc           // 各ノードで実行する関数
    Reverse    bool               // 逆順（Destroy 時）
    changedDeps map[Vertex]bool   // 依存が変化したノードの追跡
    // ...
}
```

Walk の実行フロー：

```
Walker.Update(g) / Walker.Wait():
  1. グラフを解析し、in-degree = 0（依存なし）のノードを起動可能リストに追加
  2. 各ノードを goroutine で並列実行
  3. ノード完了時に、そのノードに依存していた全ノードの
     in-degree をデクリメント
  4. in-degree が 0 になったノードを新たに goroutine で起動
  5. 全ノード完了で Walk 終了
```

並列実行の制御：

```go
// 並列度は semaphore で制限（デフォルト 10）
// internal/terraform/context.go
type ContextOpts struct {
    Parallelism int  // デフォルト 10
}
```

## サイクル検出

```go
// internal/dag/graph.go
func (g *AcyclicGraph) Validate() error {
    // 1. stronglyConnected() で強連結成分を検出（Tarjan's algorithm）
    // 2. 2頂点以上の強連結成分 = サイクル
    // 3. サイクルを含むパスを列挙してエラーメッセージ生成
}

// terraform plan 時に実行
// "Error: Cycle: resource_a -> resource_b -> resource_a"
```

## ノードの種類（Plan Graph）

| ノード型 | 役割 |
|---------|------|
| `NodePlannableResource` | Plan 対象の managed resource |
| `NodePlannableResourceInstance` | 各インスタンス（count/for_each） |
| `NodeAbstractProvider` | Provider の初期化・設定 |
| `NodeApplyableProvider` | Apply 時の Provider ノード |
| `NodeDestroyableResource` | Destroy 対象リソース |
| `NodeOrphanResourceInstance` | State にあるが config にない（削除予定） |
| `NodeAbstractResourceInstance` | 共通基底型 |
| `nodeModuleExpand` | モジュール展開（for_each モジュール） |

## Apply Graph での実行順序例

```
# 設定例
resource "aws_vpc" "main" { ... }
resource "aws_subnet" "main" { vpc_id = aws_vpc.main.id }
resource "aws_instance" "app" { subnet_id = aws_subnet.main.id }

# グラフエッジ（→ は「先に実行」）
aws_vpc.main → aws_subnet.main → aws_instance.app

# 実行順序
goroutine 1: aws_vpc.main（依存なし、即開始）
  ↓ 完了
goroutine 2: aws_subnet.main（vpc 完了後に開始）
  ↓ 完了
goroutine 3: aws_instance.app（subnet 完了後に開始）
```

## Destroy Graph の逆順

```
# Destroy 時はエッジを逆転
# 通常: A → B（A を先に作る）
# Destroy: B → A（B を先に消す）

// internal/dag/walk.go
walker.Reverse = true
```

## デバッグ: グラフの可視化

```bash
# DOT 形式でグラフを出力
terraform graph | dot -Tsvg > graph.svg

# Plan グラフ
terraform graph -type=plan

# Apply グラフ
terraform graph -type=apply

# Destroy グラフ
terraform graph -type=plan-destroy
```

## 関連パッケージ

| パッケージ | 内容 |
|-----------|------|
| `internal/dag` | 汎用 DAG・Walk・サイクル検出 |
| `internal/terraform` | Terraform 固有の Graph ノード・Transformer |
| `internal/graphs` | Graph Builder インターフェース |
| `internal/command/graph.go` | `terraform graph` コマンド実装 |
