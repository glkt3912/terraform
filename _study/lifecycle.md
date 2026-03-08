# リソースライフサイクル（create_before_destroy・ignore_changes・precondition/postcondition）

## 概要

Terraform の `lifecycle` ブロックはリソースの作成・更新・削除の挙動を細かく制御する。
`precondition` / `postcondition` はカスタム検証ルールを定義する。

## lifecycle ブロックの全オプション

```hcl
resource "aws_instance" "app" {
  # ...

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [tags, ami]
    replace_triggered_by  = [aws_launch_template.app.id]

    precondition {
      condition     = var.instance_type != "t2.micro"
      error_message = "t2.micro は本番環境では使用禁止"
    }

    postcondition {
      condition     = self.public_ip != ""
      error_message = "インスタンスに public IP が割り当てられていない"
    }
  }
}
```

## create_before_destroy

### 通常の Replace フロー

```
1. 旧リソース削除（DestroyResourceChange）
2. 新リソース作成（ApplyResourceChange）
```

### create_before_destroy = true の Replace フロー

```
1. 新リソース作成（ApplyResourceChange）
   → State の Current に追加
2. 旧リソースを Deposed キューに移動
   → State: { Current: new, Deposed: { key: old } }
3. 旧リソース削除（ApplyResourceChange with empty planned state）
   → Deposed から削除

クラッシュ時:
   → State に Deposed が残る
   → 次回 apply で "deposed object" として検出・削除
```

Go での Deposed 管理：

```go
// internal/states/resource.go
type ResourceInstance struct {
    Current *ResourceInstanceObjectSrc
    Deposed map[DeposedKey]*ResourceInstanceObjectSrc
}

type DeposedKey [8]byte  // ランダム生成のキー

// Plan 時に DeposedKey を生成
// internal/plans/changes.go
type ResourceInstanceChange struct {
    DeposedKey states.DeposedKey
    // ...
}
```

### create_before_destroy の伝播

`create_before_destroy = true` の依存グラフへの影響：

```
resource A (create_before_destroy = true)
  ↑ 依存
resource B

→ B も暗黙的に create_before_destroy = true として扱われる
  （A を再作成する前に B も再作成が必要なため）
```

## prevent_destroy

```hcl
lifecycle {
  prevent_destroy = true
}
```

- Plan 時に削除アクションが検出されるとエラーで中断
- `terraform destroy` も対象
- `terraform state rm` は影響を受けない（State から除くだけ）

内部実装：

```go
// internal/terraform/node_resource_abstract_instance.go
if rs.Current.CreateBeforeDestroy {
    // ...
}
// prevent_destroy チェック
if n.Config.Managed.PreventDestroy && action == plans.Delete {
    diags = diags.Append(&hcl.Diagnostic{
        Severity: hcl.DiagError,
        Summary:  "Instance cannot be destroyed",
    })
}
```

## ignore_changes

```hcl
lifecycle {
  ignore_changes = [
    tags,           # 特定属性
    tags["Name"],   # ネストした属性
    all,            # すべての属性（外部で管理されるリソース用）
  ]
}
```

内部動作：

```
PlanResourceChange 呼び出し前に:
  1. State の現在値を読み込み
  2. ignore_changes に指定された属性を
     「コード側の値」ではなく「State の現在値」で上書き
  3. 実質的にその属性の変更を無視

→ Provider には「変更なし」として渡されるため、
  Provider も更新を試みない
```

## replace_triggered_by（Terraform 1.2+）

```hcl
lifecycle {
  replace_triggered_by = [
    aws_launch_template.app,        # リソース全体
    aws_launch_template.app.id,     # 特定属性
  ]
}
```

- 参照先が変更されると、このリソースを強制的に Replace
- 直接の属性参照がなくても再作成をトリガーできる

## precondition / postcondition（Terraform 1.2+）

### precondition

Plan フェーズで評価される事前条件：

```hcl
resource "aws_instance" "app" {
  ami = var.ami_id

  lifecycle {
    precondition {
      condition     = data.aws_ami.selected.architecture == "x86_64"
      error_message = "選択した AMI は x86_64 アーキテクチャである必要があります"
    }
  }
}
```

評価タイミング：

```
terraform plan:
  1. 参照する値（data source 等）を評価
  2. condition 式を評価
  3. false → Plan エラーで中断
  4. true → Plan 継続
```

### postcondition

Apply フェーズ後に評価される事後条件：

```hcl
resource "aws_instance" "app" {
  lifecycle {
    postcondition {
      condition     = self.public_ip != null
      error_message = "インスタンスに public IP が必要です"
    }
  }
}
```

評価タイミング：

```
terraform apply:
  1. リソースを Apply（ApplyResourceChange）
  2. 返却された NewState で condition を評価
  3. false → Apply エラー（リソースは作成済みだが State は tainted）
  4. true → Apply 継続・State 更新
```

`self` は適用後のリソースの属性を参照する特殊キーワード。

### output の precondition

```hcl
output "instance_ip" {
  value = aws_instance.app.public_ip

  precondition {
    condition     = aws_instance.app.public_ip != null
    error_message = "インスタンスに public IP がありません"
  }
}
```

### variable の validation との違い

| 機能 | 評価タイミング | 参照できる値 |
|------|--------------|-------------|
| `variable validation` | 変数評価時（Plan 前） | `var.xxx` のみ |
| `precondition` | Plan 時（リソース評価時） | data source・他リソースも可 |
| `postcondition` | Apply 後 | `self`（適用後の値） |

## 関連パッケージ

| パッケージ | 内容 |
|-----------|------|
| `internal/configs` | lifecycle ブロックのパース |
| `internal/terraform` | precondition/postcondition の評価ロジック |
| `internal/states` | Deposed・Tainted の管理 |
| `internal/plans` | replace_triggered_by の Plan への反映 |
