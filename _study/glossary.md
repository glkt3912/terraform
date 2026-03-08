# 用語集（Terraform 内部コードの型・概念一覧）

## アドレス体系（internal/addrs）

Terraform 内部ではあらゆるオブジェクトを「アドレス」で識別する。

| 型 | 例 | 説明 |
|----|-----|------|
| `addrs.Module` | `["network", "subnet"]` | モジュール階層パス（文字列スライス） |
| `addrs.ModuleInstance` | `module.network[0].module.subnet` | インスタンス化されたモジュールパス |
| `addrs.Resource` | `aws_instance.web` | モジュール内のリソース参照 |
| `addrs.AbsResource` | `module.network.aws_instance.web` | モジュールパス込みの絶対リソース参照 |
| `addrs.ResourceInstance` | `aws_instance.web[0]` | count/for_each の特定インスタンス |
| `addrs.AbsResourceInstance` | `module.network.aws_instance.web["prod"]` | 絶対パスのリソースインスタンス |
| `addrs.Provider` | `registry.terraform.io/hashicorp/aws` | Provider の完全修飾名 |
| `addrs.AbsProviderConfig` | `provider["registry.terraform.io/hashicorp/aws"].alias` | Provider 設定のアドレス |
| `addrs.InputVariable` | `var.env_name` | 入力変数 |
| `addrs.LocalValue` | `local.base_tags` | ローカル値 |
| `addrs.OutputValue` | `output.vpc_id` | モジュール出力値 |
| `addrs.InstanceKey` | `IntKey(0)`, `StringKey("prod")` | count/for_each のインスタンスキー |

## Plan 関連型（internal/plans）

| 型 | 説明 |
|----|------|
| `plans.Plan` | Plan 全体（変更セット・変数・バックエンド設定を含む） |
| `plans.Changes` | Plan 内の全変更のセット |
| `plans.ResourceInstanceChange` | 単一リソースインスタンスの変更（Before/After 含む） |
| `plans.Change` | 変更の基本型（Action + Before/After の cty.Value） |
| `plans.Action` | Create/Read/Update/Delete/NoOp/Replace/Forget |
| `plans.DynamicValue` | msgpack でエンコードされた cty.Value（Provider との値受け渡し） |

## State 関連型（internal/states）

| 型 | 説明 |
|----|------|
| `states.State` | tfstate 全体（Modules マップ） |
| `states.Module` | 特定モジュールの State（Resources + Outputs） |
| `states.Resource` | リソースの State（EachMode + Instances マップ） |
| `states.ResourceInstance` | 単一インスタンスの State（Current + Deposed） |
| `states.ResourceInstanceObject` | インスタンスの実際の値（cty.Value + Status） |
| `states.ResourceInstanceObjectSrc` | シリアライズ済みのインスタンスオブジェクト（msgpack） |
| `states.EachMode` | NoEach / EachList（count）/ EachMap（for_each） |
| `states.ObjectStatus` | ObjectReady / ObjectTainted / ObjectPlanned |
| `DeposedKey` | create_before_destroy での旧インスタンスの識別キー |

## Config 関連型（internal/configs）

| 型 | 説明 |
|----|------|
| `configs.Config` | モジュールツリーのノード（Module + Children） |
| `configs.Module` | 単一モジュールのパース済み設定 |
| `configs.Resource` | resource ブロックの設定 |
| `configs.Variable` | variable ブロックの定義 |
| `configs.Output` | output ブロックの定義 |
| `configs.ModuleCall` | module ブロックの呼び出し定義 |
| `configs.Provider` | provider ブロックの設定 |
| `configs.Backend` | backend ブロックの設定 |

## Graph 関連型（internal/terraform, internal/dag）

| 型 | 説明 |
|----|------|
| `dag.Graph` | 汎用 DAG（有向非巡回グラフ）データ構造 |
| `dag.AcyclicGraph` | 循環検出付き DAG |
| `dag.Walker` | Graph のトポロジカル走査（並列実行サポート） |
| `terraform.Graph` | Terraform 固有の DAG（dag.AcyclicGraph のラッパー） |
| `terraform.GraphNode` | Graph ノードのインターフェース |
| `terraform.GraphNodeExecutable` | Execute メソッドを持つ実行可能ノード |
| `terraform.EvalContext` | Graph Walk 中のコンテキスト（Provider/State へのアクセス） |

## Provider 関連型（internal/providers）

| 型 | 説明 |
|----|------|
| `providers.Interface` | Provider の Go インターフェース（全 RPC に対応） |
| `providers.Factory` | Provider インスタンスを生成するファクトリ関数 |
| `providers.GetProviderSchemaResponse` | GetProviderSchema の応答（全 Schema を含む） |
| `providers.PlanResourceChangeRequest` | PlanResourceChange の引数 |
| `providers.PlanResourceChangeResponse` | PlanResourceChange の応答（PlannedState + RequiresReplace） |
| `providers.ApplyResourceChangeRequest` | ApplyResourceChange の引数 |
| `providers.ApplyResourceChangeResponse` | ApplyResourceChange の応答（NewState） |

## HCL / cty 関連

| 型/概念 | 説明 |
|---------|------|
| `hcl.Expression` | HCL の式（未評価） |
| `hcl.EvalContext` | HCL 式の評価コンテキスト（変数マップ） |
| `hcl.Diagnostics` | エラー・警告のリスト |
| `cty.Value` | 型付きの値（Terraform 内部で値を表現する基本型） |
| `cty.Type` | cty の型（String/Number/Bool/List/Map/Object 等） |
| `cty.NilVal` | nil に相当する無効な cty.Value |
| `cty.NullVal(t)` | null 値（型はあるが値がない） |
| `cty.UnknownVal(t)` | unknown 値（Plan 時に Computed 属性が取る値） |
| `cty.DynamicVal` | 型が不明な unknown 値 |

## 重要な概念

| 概念 | 説明 |
|------|------|
| **Tainted** | 部分的に作成されたリソース（次回 apply で強制再作成） |
| **Deposed** | `create_before_destroy` で旧インスタンスが廃棄待ちの状態 |
| **Lineage** | State ファイルの UUID（異なる State の混在を防ぐ） |
| **Serial** | State の更新カウンター（競合検出に使用） |
| **Workspace** | 同一設定で独立した State を持つ環境（default が標準） |
| **Provisioner** | リソース作成後にローカル/リモートでコマンド実行（非推奨） |
| **Data Source** | 既存リソースの情報を読み取る（`data` ブロック） |
| **Sensitive** | マスク表示される値（State には平文で保存される点に注意） |
| **Lock** | 同時実行防止のロック（DynamoDB/GCS 等で実装） |
| **Backend** | State 保存先とオペレーション実行場所の定義 |
| **Registry** | Provider/Module の配布サーバー（registry.terraform.io） |
| **Plugin Protocol** | Core と Provider の通信プロトコル（gRPC + protobuf） |
| **Graph Walk** | DAG をトポロジカル順に並列走査する処理 |
| **Refresh** | 実際のインフラ状態を読み取り State を更新する処理 |
| **Import** | Terraform 管理外のリソースを State に取り込む操作 |
| **Moved** | `moved` ブロックによるリソースアドレス変更の宣言 |
