# CLI コマンド構造（internal/command の登録・実行・UI フロー）

## 概要

Terraform の CLI は Go の `github.com/mitchellh/cli` パッケージをベースに構築されている。
`main.go` からコマンドが登録され、各サブコマンドが `internal/command` パッケージに実装されている。

## エントリーポイント

```
main.go
  └── commands.go  (init() でコマンドマップを構築)
        └── internal/command/xxx.go  (各サブコマンド)
```

```go
// main.go
func main() {
    os.Exit(realMain())
}

func realMain() int {
    // シグナルハンドラ設定
    // メタデータ初期化
    // CLI オブジェクト構築
    cli := &cli.CLI{
        Args:       args,
        Commands:   Commands,  // コマンドマップ
        HelpWriter: os.Stdout,
    }
    exitCode, err := cli.Run()
    // ...
}
```

## コマンド登録マップ

```go
// commands.go
var Commands map[string]cli.CommandFactory = map[string]cli.CommandFactory{
    "apply": func() (cli.Command, error) {
        return &command.ApplyCommand{Meta: meta}, nil
    },
    "plan": func() (cli.Command, error) {
        return &command.PlanCommand{Meta: meta}, nil
    },
    "init": func() (cli.Command, error) {
        return &command.InitCommand{Meta: meta}, nil
    },
    // ... 全サブコマンド
}
```

## Meta 構造体（共通状態）

全コマンドが埋め込む共通の構造体：

```go
// internal/command/meta.go
type Meta struct {
    // UI
    Ui cli.Ui  // 入出力インターフェース

    // 設定
    color            bool
    noColor          bool
    input            bool  // インタラクティブ入力の有無

    // バックエンド
    backendState     *legacy.BackendState
    ContextOpts      *terraform.ContextOpts

    // 作業ディレクトリ
    WorkingDir *workdir.Dir
}
```

## コマンド実装の基本構造

```go
// internal/command/apply.go
type ApplyCommand struct {
    Meta
}

func (c *ApplyCommand) Run(args []string) int {
    // 1. フラグパース
    cmdFlags := c.Meta.defaultFlagSet("apply")
    // ...
    if err := cmdFlags.Parse(args); err != nil { return 1 }

    // 2. Backend の初期化
    b, backendDiags := c.Backend(&BackendOpts{...})

    // 3. Operation の構築と実行
    opReq := c.RunOperation(b, &backend.Operation{
        Type:      backend.OperationTypeApply,
        PlanFile:  planFile,
        // ...
    })
    return opReq.ExitCode
}

func (c *ApplyCommand) Synopsis() string {
    return "Create or update infrastructure"
}

func (c *ApplyCommand) Help() string {
    return strings.TrimSpace(helpTextApply)
}
```

## UI インターフェース

```go
// github.com/mitchellh/cli
type Ui interface {
    Ask(string) (string, error)      // ユーザー入力（確認プロンプト）
    AskSecret(string) (string, error) // パスワード入力
    Output(string)                    // 標準出力
    Info(string)                      // 情報（stderr または色付き）
    Error(string)                     // エラー（stderr）
    Warn(string)                      // 警告
}

// カラー出力対応
type ColoredUi struct {
    Ui          Ui
    OutputColor UiColor
    InfoColor    UiColor
    ErrorColor   UiColor
    WarnColor    UiColor
}
```

## Views（出力フォーマット）

Terraform 1.0+ では出力を `views` パッケージで管理し、Human 形式と JSON 形式を切り替えられる：

```go
// internal/command/views/
type Apply interface {
    Operation() Operation
    Hooks() []terraform.Hook
    Diagnostics(tfdiags.Diagnostics)
    HelpPrompt()
}

// JSON 出力モード（-json フラグ）
type ApplyJSON struct { ... }

// Human 読み取り可能な出力
type ApplyHuman struct { ... }
```

```bash
terraform apply -json   # 機械可読な JSON ストリーム出力
terraform plan -json    # Plan を JSON で出力
```

## Operation の実行フロー

```
CLI コマンド
  → backend.Operation を構築
  → b.Operation(ctx, op) を呼び出し
       ↓
  ローカルバックエンド (backend/local)
  → opApply() / opPlan()
       ↓
  terraform.NewContext(opts)    // Terraform Core の Context 生成
  → ctx.Plan() または ctx.Apply()
       ↓
  Graph Build → Graph Walk → Provider RPC
```

## ワーキングディレクトリの管理

```go
// internal/command/workdir/dir.go
type Dir struct {
    mainDir        string   // -chdir または カレントディレクトリ
    dataDir        string   // .terraform/ ディレクトリ
    overrideDataDir string
}

// .terraform/ 配下の主要ファイル
// .terraform/terraform.tfstate  → バックエンド設定
// .terraform/providers/         → Provider バイナリキャッシュ
// .terraform/modules/           → Module ダウンロードキャッシュ
// .terraform.lock.hcl           → Provider バージョンロック
```

## 主要コマンド一覧

| コマンド | 実装ファイル | 主な処理 |
|---------|------------|---------|
| `init` | `command/init.go` | Backend・Provider・Module の初期化 |
| `plan` | `command/plan.go` | Plan 実行・planfile 出力 |
| `apply` | `command/apply.go` | Apply 実行（Plan 含む） |
| `destroy` | `command/apply.go` | `-destroy` フラグ付き apply |
| `validate` | `command/validate.go` | 設定の静的検証 |
| `state` | `command/state*.go` | State 操作サブコマンド群 |
| `import` | `command/import.go` | 既存リソースの import |
| `output` | `command/output.go` | Output 値の表示 |
| `show` | `command/show.go` | State/planfile の表示 |
| `graph` | `command/graph.go` | 依存グラフの DOT 形式出力 |
| `fmt` | `command/fmt.go` | HCL フォーマット |
| `test` | `command/test.go` | terraform test 実行（1.6+） |
| `workspace` | `command/workspace*.go` | Workspace 管理 |

## 終了コード

| コード | 意味 |
|--------|------|
| `0` | 成功 |
| `1` | エラー |
| `2` | `plan -detailed-exitcode` で変更あり |

## 関連パッケージ

| パッケージ | 内容 |
|-----------|------|
| `internal/command` | 全 CLI コマンド実装 |
| `internal/command/views` | 出力フォーマット（Human/JSON） |
| `internal/command/arguments` | フラグパース用引数型 |
| `internal/command/workdir` | ワーキングディレクトリ管理 |
| `internal/backend/local` | ローカル Operation 実行 |
| `github.com/mitchellh/cli` | CLI フレームワーク |
