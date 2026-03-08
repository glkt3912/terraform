# Terraform Core アーキテクチャ全体像

## 概要

Terraform は HashiCorp が開発した Infrastructure as Code (IaC) ツール。HCL（HashiCorp Configuration Language）で記述したインフラ定義を元に、クラウドリソースを宣言的に管理する。

## 主要コンポーネント

```
CLI (terraform コマンド)
  └── Core
        ├── Config Loader（HCL パース）
        ├── State Manager（tfstate 読み書き）
        ├── Graph Builder（依存グラフ構築）
        ├── Plan Engine（差分計算）
        └── Apply Engine（変更適用）
              └── Provider Plugin（gRPC）
                    └── Cloud API
```

## CLI から Apply までのフロー

```
terraform apply
  1. HCL 読み込み → AST → Config
  2. State 読み込み（local or remote backend）
  3. Provider 初期化（plugin 起動 / gRPC 接続）
  4. Graph 構築（リソース依存関係の DAG）
  5. Plan 実行（desired vs current の diff）
  6. ユーザー確認
  7. Apply（Graph をトポロジカル順に並列実行）
  8. State 更新・保存
```

## 4つの主要概念

| 概念 | 役割 |
|------|------|
| **Provider** | AWS/GCP/Azure 等の API ラッパー（gRPC プラグイン） |
| **Resource** | 管理対象のインフラ単位（EC2, S3 等） |
| **State** | 現在のインフラ状態を記録する JSON（tfstate） |
| **Module** | 再利用可能な設定のまとまり |

## 内部パッケージ構成（Go）

| パッケージ | 内容 |
|-----------|------|
| `internal/command` | CLI サブコマンド実装 |
| `internal/configs` | HCL 設定パーサー |
| `internal/states` | State 型定義・操作 |
| `internal/plans` | Plan 型定義・シリアライズ |
| `internal/providers` | Provider インターフェース |
| `internal/backend` | Backend インターフェース |
| `internal/addrs` | リソースアドレス型（`aws_instance.web` 等） |
| `internal/lang` | 式評価・組み込み関数 |

## 詳細ドキュメント

- `provider.md` — Provider プラグインの gRPC プロトコル
- `state.md` — tfstate 構造・remote backend・locking
- `plan-apply.md` — Plan/Apply の内部フロー
- `modules.md` — モジュールシステム
- `expressions.md` — HCL 式評価・for_each・depends_on
- `backends.md` — S3/GCS/Consul バックエンド
