# 式と関数（HCL 評価・depends_on・for_each）

## 概要

Terraform の式（Expression）は HCL の評価エンジンによって処理される。変数参照・関数呼び出し・条件式・for 式など豊富な構文を持ち、`internal/lang` パッケージが評価を担当する。

## 式の評価エンジン

```go
// internal/lang/eval.go
// HCL の式を評価して cty.Value を返す
type Scope struct {
    Data         lang.Data       // 変数・リソース参照を解決
    SelfAddr     addrs.Referenceable
    PureOnly     bool
    ConsoleMode  bool
}

func (s *Scope) EvalExpr(expr hcl.Expression, wantType cty.Type) (cty.Value, tfdiags.Diagnostics)
```

## 参照できるオブジェクト一覧

| 参照 | 説明 |
|------|------|
| `var.name` | Input 変数 |
| `local.name` | ローカル値 |
| `module.name.output` | 子モジュールの Output |
| `resource_type.name.attr` | 管理リソースの属性 |
| `data.type.name.attr` | Data Source の属性 |
| `path.module` | 現在のモジュールディレクトリ |
| `path.root` | ルートモジュールディレクトリ |
| `terraform.workspace` | 現在のワークスペース名 |
| `count.index` | count 使用時のインデックス |
| `each.key` / `each.value` | for_each 使用時のキー・値 |
| `self` | provisioner 内での自リソース参照 |

## 条件式（Conditional Expression）

```hcl
# condition ? true_val : false_val
instance_type = var.env == "prod" ? "t3.large" : "t3.micro"

# null 結合（null チェックに便利）
tags = var.extra_tags != null ? var.extra_tags : {}
```

## for 式

```hcl
# リストの変換
upper_names = [for name in var.names : upper(name)]

# フィルタリング
prod_instances = [for i in var.instances : i if i.env == "prod"]

# マップ変換
name_to_id = {for r in aws_instance.servers : r.tags.Name => r.id}

# セット → マップ
region_map = {for r in var.regions : r => "enabled"}
```

## splat 式

```hcl
# リソースの特定属性をリストで取得
instance_ids = aws_instance.servers[*].id
# ↑ count/for_each で複数リソースがある場合に使用

# 旧スタイル（legacy splat）
all_amis = aws_instance.servers.*.ami
```

## for_each の内部動作

```hcl
resource "aws_instance" "web" {
  for_each = tomap({
    "server-1" = "us-east-1"
    "server-2" = "ap-northeast-1"
  })
  # ...
}
```

内部処理：

1. `for_each` の式を評価して `map` または `set` を取得
2. 各キーに対して `addrs.StringKey` の InstanceKey を生成
3. グラフに `aws_instance.web["server-1"]`、`aws_instance.web["server-2"]` ノードを追加
4. 既存 State とのキー差分でCreate/Update/Delete を決定

## depends_on の内部動作

```hcl
resource "aws_instance" "app" {
  depends_on = [aws_iam_role_policy.app]
}
```

- 通常の属性参照では Terraform が暗黙的依存を検出できるが、副作用（IAM が有効になるまでの遅延等）には `depends_on` を使う
- `depends_on` はグラフのエッジとして追加され、トポロジカルソートで順序を強制
- `depends_on` を使うと、参照先リソースの「すべての属性」が unknown になるまで評価を遅延する

## 組み込み関数カテゴリ

| カテゴリ | 関数例 |
|---------|--------|
| 文字列 | `format`, `join`, `split`, `replace`, `trimspace`, `regex` |
| 数値 | `abs`, `ceil`, `floor`, `max`, `min`, `parseint` |
| コレクション | `length`, `keys`, `values`, `merge`, `flatten`, `toset`, `zipmap` |
| エンコード | `base64encode`, `jsonencode`, `jsondecode`, `yamlencode`, `urlencode` |
| ファイル | `file`, `filebase64`, `templatefile` |
| 日時 | `timestamp`, `formatdate`, `timeadd` |
| ハッシュ | `md5`, `sha256`, `sha512`, `bcrypt` |
| IP | `cidrhost`, `cidrnetmask`, `cidrsubnet`, `cidrsubnets` |
| 型変換 | `tostring`, `tonumber`, `tobool`, `tolist`, `tomap` |

## locals の評価

```hcl
locals {
  # 他の local を参照可能（依存順に自動ソートされる）
  base_tags = { env = var.environment, project = "myapp" }
  all_tags  = merge(local.base_tags, var.extra_tags)

  # 複雑な変換
  instance_map = {
    for idx, inst in var.instances :
    inst.name => {
      id   = idx
      size = inst.size
      tags = local.all_tags
    }
  }
}
```

## dynamic ブロック

```hcl
resource "aws_security_group" "app" {
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

## 関連パッケージ

| パッケージ | 内容 |
|-----------|------|
| `internal/lang` | 式評価スコープ・組み込み関数登録 |
| `internal/lang/funcs` | 組み込み関数の実装 |
| `internal/configs` | HCL パース・式の AST |
| `github.com/zclconf/go-cty` | 型システム（cty） |
| `github.com/hashicorp/hcl/v2` | HCL パーサー・評価エンジン |
