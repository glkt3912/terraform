# モジュールシステム（モジュール解決・変数・出力の伝播）

## 概要

Terraform のモジュールは再利用可能な設定の単位。すべての Terraform 設定は「ルートモジュール」であり、他のモジュールを呼び出してネストできる。
モジュールは入力変数（`variable`）で設定を受け取り、出力値（`output`）で結果を返す。

## モジュール呼び出しの構文

```hcl
# ルートモジュールから子モジュールを呼び出す
module "network" {
  source  = "./modules/network"   # ローカルパス
  version = "~> 1.0"              # バージョン制約（レジストリ時のみ）

  # 変数の渡し方
  vpc_cidr   = "10.0.0.0/16"
  env_name   = var.environment
}

# 子モジュールの output を参照
resource "aws_instance" "app" {
  subnet_id = module.network.public_subnet_id
}
```

## モジュールのソース種別

| ソース | 例 |
|--------|-----|
| ローカルパス | `./modules/vpc` |
| Terraform Registry | `hashicorp/consul/aws` |
| GitHub | `github.com/org/repo//modules/vpc` |
| S3 | `s3::https://s3.amazonaws.com/bucket/module.zip` |
| GCS | `gcs::https://www.googleapis.com/storage/v1/...` |
| HTTP | `https://example.com/module.zip` |

## 内部の Module アドレス体系

```go
// internal/addrs/module.go

// Module はモジュール階層のパス
type Module []string
// 例: [] (root), ["network"], ["network", "subnet"]

// ModuleInstance はインスタンス化されたモジュール
// count や for_each 使用時に複数インスタンスが存在する
type ModuleInstance []ModuleInstanceStep

type ModuleInstanceStep struct {
    Name        string
    InstanceKey InstanceKey  // count の場合は IntKey、for_each は StringKey
}
// 例: module.network[0].module.subnet["public"]
```

## Config ロード時のモジュール解決

```
terraform init:
  1. ルートの設定を読み込み
  2. 各 module ブロックの source を解析
  3. ローカル: そのまま参照
  4. レジストリ/Git 等: .terraform/modules/ にダウンロード
  5. .terraform/modules/modules.json にマニフェスト記録

terraform plan:
  1. configs.LoadConfig() でルートから再帰的に読み込み
  2. 各 module ブロックの source を解決
  3. Module ツリーを構築（configs.Config）
```

## 変数の伝播

```
ルートモジュール
  変数: var.env = "prod"
  ↓ module "network" { env_name = var.env }
子モジュール (module.network)
  変数: var.env_name = "prod"（親から受け取る）
  ↓ module "subnet" { env = var.env_name }
孫モジュール (module.network.module.subnet)
  変数: var.env = "prod"（祖父から間接的に受け取る）
```

変数のデフォルト値・検証：

```hcl
variable "environment" {
  type        = string
  description = "deployment environment"
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod"
  }
}
```

## Output の伝播

```hcl
# 子モジュール内
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "The ID of the VPC"
  sensitive   = false
}

# 親モジュール
resource "aws_subnet" "main" {
  vpc_id = module.network.vpc_id  # 子モジュールの output を参照
}
```

## for_each / count によるモジュールインスタンス化

```hcl
# count
module "servers" {
  count  = 3
  source = "./modules/server"
  index  = count.index
}
# → module.servers[0], module.servers[1], module.servers[2]

# for_each
module "regions" {
  for_each = toset(["us-east-1", "ap-northeast-1"])
  source   = "./modules/regional"
  region   = each.key
}
# → module.regions["us-east-1"], module.regions["ap-northeast-1"]
```

## Go の Config 型構造

```go
// internal/configs/config.go
type Config struct {
    Module   *Module              // このモジュールの設定
    Path     addrs.Module         // モジュールパス
    Children map[string]*Config   // 子モジュール（キーはラベル名）
    Root     *Config              // ルートモジュールへの参照
    Parent   *Config              // 親モジュールへの参照
}

type Module struct {
    SourceDir   string
    Variables   map[string]*Variable
    Outputs     map[string]*Output
    ManagedResources map[string]*Resource
    DataResources    map[string]*Resource
    ModuleCalls      map[string]*ModuleCall
}
```

## 関連パッケージ

| パッケージ | 内容 |
|-----------|------|
| `internal/configs` | モジュール Config のロード・パース |
| `internal/addrs` | Module/ModuleInstance アドレス型 |
| `internal/modsdir` | モジュールキャッシュ（.terraform/modules） |
| `internal/getmodules` | モジュールのダウンロード・取得 |
| `internal/registry` | Terraform Registry API クライアント |
