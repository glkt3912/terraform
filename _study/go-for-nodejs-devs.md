# Go 入門（Node.js/NestJS 開発者向け）

## この文書の目的

Terraform のソースコードは Go で書かれている。
Node.js / NestJS を主戦場とする開発者が Terraform の内部実装を読む際に
「この Go のコードは TypeScript で言うと何に相当するのか？」を素早く対応付けるためのガイド。

---

## 1. 型システム：struct と interface

### TypeScript との対比

```typescript
// TypeScript
interface Animal {
  name: string;
  sound(): string;
}

class Dog implements Animal {
  name: string;
  constructor(name: string) { this.name = name; }
  sound(): string { return "woof"; }
}
```

```go
// Go（同等のコード）
type Animal interface {
  Sound() string
}

type Dog struct {
  Name string
}

// "implements" キーワードは不要
// Animal インターフェースのメソッドを持っていれば自動的に実装したことになる（暗黙的実装）
func (d Dog) Sound() string {
  return "woof"
}
```

**最重要ポイント: 暗黙的インターフェース**

Go には `implements` キーワードがない。
メソッドシグネチャが一致していれば、自動的にインターフェースを実装したことになる。

これが Terraform の `providers.Interface` を理解する鍵になる。
AWS Provider も GCP Provider も、同じメソッド群（`PlanResourceChange` 等）を持っているだけで
`providers.Interface` を実装していると見なされる。

```go
// internal/providers/provider.go
type Interface interface {
  GetProviderSchema() GetProviderSchemaResponse
  PlanResourceChange(PlanResourceChangeRequest) PlanResourceChangeResponse
  ApplyResourceChange(ApplyResourceChangeRequest) ApplyResourceChangeResponse
  // ...
}

// AWS Provider も GCP Provider も、これらのメソッドを持つだけで Interface を満たす
// → TypeScript の implements 宣言は不要
```

---

## 2. エラーハンドリング：戻り値としての error

### TypeScript との対比

```typescript
// TypeScript: 例外でエラーを伝える
async function readFile(path: string): Promise<string> {
  try {
    return await fs.readFile(path, "utf-8");
  } catch (e) {
    throw new Error(`Failed to read: ${e}`);
  }
}

// 呼び出し側
try {
  const content = await readFile("./file.txt");
} catch (e) {
  console.error(e);
}
```

```go
// Go: エラーは戻り値として返す（例外なし）
func readFile(path string) (string, error) {
  content, err := os.ReadFile(path)
  if err != nil {
    return "", fmt.Errorf("Failed to read: %w", err)
  }
  return string(content), nil
}

// 呼び出し側
content, err := readFile("./file.txt")
if err != nil {
  log.Fatal(err)
}
```

**Go でエラーハンドリングが `if err != nil` だらけになる理由**

Go に例外（`throw/catch`）はない。
エラーは関数の戻り値として返すのが慣習で、呼び出すたびに `if err != nil` を書く。

Terraform のソースコードでも至る所に出てくるが、
「例外が来るかもしれない」ではなく「エラーが戻り値で返ってきた場合」と読む。

**Terraform 固有: `tfdiags.Diagnostics`**

Terraform は単純な `error` の代わりに `tfdiags.Diagnostics` を使うことが多い。
これは複数のエラー・警告をまとめて返せるリスト型で、
`terraform plan` で複数のエラーを一度に表示するために使われている。

```go
// 複数エラーを一度に収集して返す
var diags tfdiags.Diagnostics
diags = diags.Append(err1)
diags = diags.Append(err2)
return diags  // TypeScript の Error[] に相当するイメージ
```

---

## 3. 非同期・並列：goroutine と channel

### TypeScript との対比

```typescript
// TypeScript: Promise.all で並列実行
async function applyAll(resources: Resource[]): Promise<void> {
  await Promise.all(resources.map(r => r.apply()));
}
```

```go
// Go: goroutine で並列実行
func applyAll(resources []Resource) {
  var wg sync.WaitGroup
  for _, r := range resources {
    wg.Add(1)
    go func(res Resource) {  // "go" キーワードで goroutine（軽量スレッド）として起動
      defer wg.Done()
      res.Apply()
    }(r)
  }
  wg.Wait()  // 全 goroutine の完了を待つ（Promise.all の await に相当）
}
```

**goroutine とは？**

Node.js のイベントループとは異なり、Go の goroutine は OS スレッドに近い軽量プロセス。
`go func()` と書くだけで新しい goroutine が起動する。
`async/await` のような構文糖衣は不要。

**Terraform での使われ方**

`internal/dag/walk.go` の Graph Walker が、
依存関係のないノードを goroutine で並列実行している。
これが `terraform apply -parallelism=10` の実装の正体。

**channel（データの受け渡し）**

```go
// channel = goroutine 間のデータパイプ
// TypeScript の EventEmitter や Subject（RxJS）に近いイメージ

ch := make(chan string)

go func() {
  ch <- "hello"  // channel に送信
}()

msg := <-ch  // channel から受信（送信されるまでブロック）
```

---

## 4. ジェネリクス的な表現：interface{}（any）

Go 1.18 以前はジェネリクスがなく、型を問わない値を `interface{}`（Go 1.18+ では `any`）で表現していた。
Terraform のコードベースはこの旧スタイルが多い。

```typescript
// TypeScript
function process(value: unknown): void { ... }
```

```go
// Go（旧スタイル）
func process(value interface{}) { ... }

// 型アサーション（TypeScript の型キャストに相当）
str, ok := value.(string)
if !ok {
  // string でなかった
}

// 型スイッチ（TypeScript の typeof / instanceof に相当）
switch v := value.(type) {
case string:
  fmt.Println("string:", v)
case int:
  fmt.Println("int:", v)
}
```

**Terraform での使われ方**

`dag.Graph` のノードは `interface{}` として扱われ、
Walk 時に型アサーションで具体的なノード型（`NodeApplyableResource` 等）にキャストされる。

---

## 5. DI・依存注入：Factory 関数パターン

### NestJS との対比

```typescript
// NestJS: デコレーターベースの DI
@Injectable()
class AwsProvider {
  constructor(private readonly config: AwsConfig) {}
}

@Module({
  providers: [AwsProvider],
})
class AppModule {}
```

```go
// Go: Factory 関数で DI を表現（フレームワークなし）
type ProviderFactory func() (providers.Interface, error)

// 具体的な Provider を生成するファクトリ
func NewAwsProvider(config AwsConfig) ProviderFactory {
  return func() (providers.Interface, error) {
    return &AwsProvider{config: config}, nil
  }
}

// 利用側はインターフェース型で受け取る
var factory ProviderFactory = NewAwsProvider(config)
provider, err := factory()
```

**なぜデコレーターがないのか？**

Go にはリフレクションベースのデコレーターパターンがなく、
明示的な関数呼び出しで依存を組み立てる（Composition Root パターン）。
シンプルだが、NestJS のような自動スキャン・自動注入はない。

---

## 6. ポインタ：参照渡しと値渡し

TypeScript では object は常に参照渡しだが、
Go は明示的にポインタ（`*`）を使わないと値のコピーが渡される。

```typescript
// TypeScript: object は常に参照
function mutate(obj: { count: number }) {
  obj.count++;  // 呼び出し元に影響する
}
```

```go
// Go: ポインタを使わないと値のコピー
func mutateBad(s State) {
  s.Count++  // コピーへの操作。呼び出し元は変わらない
}

func mutateGood(s *State) {  // * でポインタ型
  s.Count++  // 元の値を変更できる
}

// 呼び出し
state := State{Count: 0}
mutateGood(&state)  // & でアドレスを渡す
```

**Terraform での読み方**

`*configs.Config`、`*states.State` のように `*` が付いていたら
「このオブジェクトへの参照を渡している（変更が呼び出し元に反映される）」と読む。

`nil` チェック（`if s == nil`）が多いのも、
ポインタが未初期化（null 相当）の可能性があるため。

---

## 7. メソッドレシーバー：クラスメソッドの書き方

```typescript
// TypeScript
class Graph {
  private nodes: Node[] = [];

  addNode(node: Node): void {
    this.nodes.push(node);
  }
}
```

```go
// Go: struct にメソッドを定義する（クラスはない）
type Graph struct {
  nodes []Node
}

// (g *Graph) がレシーバー（TypeScript の this に相当）
func (g *Graph) AddNode(node Node) {
  g.nodes = append(g.nodes, node)
}

// 呼び出し
graph := &Graph{}
graph.AddNode(myNode)
```

`(g *Graph)` の `g` が TypeScript の `this` に相当する。
`*Graph`（ポインタレシーバー）を使うと、メソッド内で struct を変更できる。

---

## 8. Terraform コードを読む上での頻出パターン

### パターン① : インターフェース + 複数実装

```go
// インターフェース定義
type Backend interface {
  StateMgr(workspace string) (statemgr.Full, error)
}

// S3 実装
type S3Backend struct { ... }
func (b *S3Backend) StateMgr(workspace string) (statemgr.Full, error) { ... }

// GCS 実装
type GCSBackend struct { ... }
func (b *GCSBackend) StateMgr(workspace string) (statemgr.Full, error) { ... }
```

→ NestJS の `abstract class` + 複数の `extends` に相当

### パターン② : 関数型インターフェース（Transformer）

```go
type GraphTransformer interface {
  Transform(*Graph) error
}

// 匿名 struct で実装することもある
type myTransformer struct{}
func (t *myTransformer) Transform(g *Graph) error { ... }
```

→ NestJS の `Interceptor` / `Middleware` パターンに近い

### パターン③ : オプション struct（設定の渡し方）

```go
// 設定を struct にまとめて渡す
type ContextOpts struct {
  Parallelism int
  Providers   map[addrs.Provider]providers.Factory
}

ctx, err := terraform.NewContext(&ContextOpts{
  Parallelism: 10,
})
```

→ NestJS の `@Module({ imports: [...], providers: [...] })` に近い設定パターン

### パターン④ : defer（後処理の保証）

```go
func (m *StateMgr) apply() error {
  lockID, err := m.Lock()
  if err != nil { return err }
  defer m.Unlock(lockID)  // 関数終了時（エラーでも正常でも）に必ず実行

  // ... 処理 ...
  return nil
}
```

→ TypeScript の `try/finally` に相当。
Terraform の State ロック解放は必ずこのパターンで書かれている。

---

## 9. Go のツールチェーン基礎

| コマンド | NestJS 相当 | 説明 |
|---------|------------|------|
| `go build` | `tsc` / `nest build` | コンパイル |
| `go test ./...` | `jest` / `npm test` | テスト実行 |
| `go run main.go` | `ts-node src/main.ts` | 実行 |
| `go mod tidy` | `npm install` | 依存解決 |
| `go fmt` | `prettier` | フォーマット |
| `go vet` | `eslint` | 静的解析 |

**Terraform のビルド方法:**

```bash
# リポジトリ直下で
go build -o terraform .

# テスト実行
go test ./internal/terraform/... -v
```

---

## 10. よく見る Go の構文クイックリファレンス

```go
// 変数宣言（TypeScript の const/let）
x := 42          // 型推論
var y string     // 明示的宣言（ゼロ値で初期化）

// スライス（TypeScript の配列）
items := []string{"a", "b", "c"}
items = append(items, "d")
for i, v := range items { ... }  // for...of に相当

// マップ（TypeScript の Map / object）
m := map[string]int{"a": 1}
m["b"] = 2
v, ok := m["c"]  // ok が false なら存在しない

// nil チェック（TypeScript の null チェック）
if ptr == nil { ... }

// 構造体の初期化
s := &State{
  Modules: make(map[string]*Module),
}

// 無名関数（クロージャ）
fn := func(x int) int { return x * 2 }
go fn(10)  // goroutine として実行
```

---

## 関連ドキュメント

- `architecture.md` — Terraform 全体像（このドキュメントを読んだ後に読む）
- `provider.md` — Go インターフェースの実践例（`providers.Interface`）
- `graph.md` — goroutine 並列実行の実践例（`dag.Walker`）
- `glossary.md` — Terraform 固有の型・概念一覧
