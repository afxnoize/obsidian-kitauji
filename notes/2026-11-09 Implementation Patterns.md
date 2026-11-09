# パターン実装例集

ケースごとに「どう実装するとよいか」を短いコードで示す。言語は TypeScript を基本とし、そのパターンの本家である言語が明確なものはその言語で書く。

## 環境

### Muse

- 作成: Muse（パーソナルエージェント、Riri）
- モデル: Muse Spark 1.3
- 作成日: 2026年11月9日

## 生成まわり

### Builder / smart constructor

ケース: 検証を通った値だけを存在させたい。

```ts
class EmailAddress {
  private constructor(readonly value: string) {}
  static parse(raw: string): Checked<EmailAddress> {
    return raw.includes("@")
      ? valid(new EmailAddress(raw))
      : invalid("メールアドレスに @ がありません");
  }
}
// コンストラクタが private なので、生成は parse 経由のみ。
// EmailAddress 型の値は「検証済み」であることが型で保証される。
```

#### 採用例

- OkHttp（Request.Builder 等の Builder 群）
- Rust 標準ライブラリの NonZeroU32::new（smart constructor。不正な値なら None を返し、生成経路を絞る）

実コード（OkHttp の Request.Builder）: ファイルは okhttp/src/commonJvmAndroid/kotlin/okhttp3/Request.kt で、194〜396行が Builder の実装範囲。https://deepwiki.com/square/okhttp/2-core-http-client。抜粋は同ファイルのソース再掲（5.0.0-alpha.10）で確認した。

```kotlin
open class Builder {
    internal var url: HttpUrlRepresentation?
    internal var method: String
    internal var headers: Headers.Builder
    internal var body: RequestBody?
    internal constructor(request: Request)
    open fun url(url: HttpUrl): Builder
    open fun url(url: String): Builder
    open fun header(name: String, value: String): Builder
    open fun method(method: String, body: RequestBody?): Builder
    open fun build(): Request
}
```

実コード（Rust 標準ライブラリの NonZero::new）: https://doc.rust-lang.org/nightly/src/core/num/nonzero.rs.html（new は496行目）。

```rust
/// Creates a non-zero if the given value is not zero.
#[stable(feature = "nonzero", since = "1.28.0")]
#[rustc_const_stable(feature = "const_nonzero_int_methods", since = "1.47.0")]
#[must_use]
#[inline]
pub const fn new(n: T) -> Option<Self> {
    // SAFETY: Memory layout optimization guarantees that `Option<NonZero<T>>` has
    //         the same layout and size as `T`, with `0` representing `None`.
    unsafe { intrinsics::transmute_unchecked(n) }
}
```

### Factory / DI

ケース: 生成方法を差し替えたい、テストで差し替えたい。

```ts
type Logger = { log(msg: string): void };

const createLogger = (env: "dev" | "prod"): Logger =>
  env === "dev"
    ? { log: (m) => console.log(m) }
    : { log: (m) => sendToServer(m) };

// 使う側は生成済みのものを受け取るだけ（DI）
const run = (logger: Logger) => logger.log("start");
```

#### 採用例

- Spring Framework の DI コンテナ
- Angular（DI を前提にした framework 設計）

実コード（Spring Framework の BeanFactory）: https://docs.spring.io/spring-framework/docs/6.2.0-RC2/javadoc-api/org/springframework/beans/factory/BeanFactory.html。ソースファイルは spring-beans/src/main/java/org/springframework/beans/factory/BeanFactory.java。

```java
/**
 * The root interface for accessing a Spring bean container.
 * This is the basic client view of a bean container;
 */
public interface BeanFactory {

    String FACTORY_BEAN_PREFIX = "&";

    Object getBean(String name) throws BeansException;
    // ...
}
```

### Singleton の代替

ケース: 全体で1つだけ共有したい。

```ts
// クラスで管理せず、モジュールが1回だけ評価される性質を使う
export const config = loadConfig(); // import した全員が同じ1つを共有する
```

#### 採用例

- Prisma Client を globalThis に載せて再利用する構成（Prisma 公式ドキュメントが開発時の多重生成防止として推奨）

実コード（Prisma 公式ドキュメントのスニペット、そのまま）: https://www.prisma.io/docs/orm/v7/more/troubleshooting/nextjs

```ts
import { PrismaClient } from "../prisma/generated/client";
const globalForPrisma = global as unknown as { prisma: PrismaClient };
export const prisma = globalForPrisma.prisma || new PrismaClient();
if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

## 状態と振る舞い

### State

ケース: 状態で振る舞いが変わり、分岐が増殖してきた。

```ts
type State =
  | { name: "idle" }
  | { name: "running"; startedAt: number }
  | { name: "done"; result: string };

const next = (s: State, ev: "start" | "finish"): State => {
  switch (s.name) {
    case "idle":
      return ev === "start" ? { name: "running", startedAt: Date.now() } : s;
    case "running":
      return ev === "finish" ? { name: "done", result: "ok" } : s;
    case "done":
      return s;
  }
};
// 状態ごとに持てるデータが違うことを、型で表現できる
```

#### 採用例

- XState（状態機械ライブラリ）
- Erlang/OTP の gen_statem behaviour

実コード（Erlang/OTP の gen_statem の使用例）: 公式 OTP Design Principles の code_lock 例。状態名と同名のコールバックがイベントを処理し、{next_state, 次状態, 新Data} を返して遷移を宣言する。

```erlang
locked(cast, {button,Digit},
       #{code := Code, remaining := Remaining} = Data) ->
    case Remaining of
        [Digit] ->
            do_unlock(),
            {next_state,open,Data#{remaining := Code},10000};
        [Digit|Rest] -> % Incomplete
            {next_state,locked,Data#{remaining := Rest}};
        _Wrong ->
            {next_state,locked,Data#{remaining := Code}}
    end.
```

本体は lib/stdlib/src/gen_statem.erl。https://github.com/erlang/otp/blob/maint/lib/stdlib/src/gen_statem.erl#L217

### Strategy（高階関数）

ケース: アルゴリズムを差し替えたい。

```ts
type Pricing = (price: number) => number;

const regular: Pricing = (p) => p;
const member: Pricing = (p) => p * 0.9;

const checkout = (price: number, pricing: Pricing) => pricing(price);
checkout(1000, member); // 900
```

#### 採用例

- Passport.js（認証方式そのものを Strategy として差し替える設計で、名称も Strategy）
- Java の Comparator

実コード（passport-strategy の README にあるサブクラス化の定型）: https://github.com/jaredhanson/passport-strategy

```js
var util = require('util')
  , Strategy = require('passport-strategy');

function CustomStrategy(...) {
  Strategy.call(this);
}
util.inherits(CustomStrategy, Strategy);

CustomStrategy.prototype.authenticate = function(req, options) {
  // TODO: authenticate request
}
```

### Command

ケース: 操作を保存・キュー・再実行・取り消ししたい。

```ts
type Command = { do(): void; undo(): void };

class AddText implements Command {
  constructor(private doc: string[], private text: string) {}
  do() { this.doc.push(this.text); }
  undo() { this.doc.pop(); }
}

const history: Command[] = [];
const execute = (c: Command) => { c.do(); history.push(c); };
const undo = () => history.pop()?.undo();
```

#### 採用例

- VS Code の Commands（ID で登録し、キーバインドや他の拡張から実行できる）
- Redux の action（シリアライズ可能な Command とみなせる）

定義ファイルは src/vs/platform/commands/common/commands.ts（CommandsRegistry）。

### Observer / signals

ケース: 変更を購読側へ流したい。

```ts
// 最小の signal 実装（Observer の細粒度版が signals）
const signal = <T>(init: T) => {
  let value = init;
  const subs = new Set<(v: T) => void>();
  return {
    get: () => value,
    set: (v: T) => { value = v; subs.forEach((f) => f(v)); },
    subscribe: (f: (v: T) => void) => subs.add(f),
  };
};

const count = signal(0);
count.subscribe((v) => console.log("count =", v));
count.set(1); // count = 1
```

#### 採用例

- RxJS（Observable）
- SolidJS の signals（Observer の細粒度化）

実コード（SolidJS の createSignal）: 本体は packages/solid/src/reactive/signal.ts の227〜261行、Signal 型は198行（https://deepwiki.com/solidjs/solid/3.1-signals-and-derived-state による）。抜粋はシグネチャのみ。

```ts
export function createSignal<T>(
  value?: T,
  options?: SignalOptions<T | undefined>
): Signal<T | undefined>
```

## 構造

### Adapter

ケース: インターフェースが合わない既存実装を、期待する形に合わせたい。

```ts
type OldPrinter = { printOld(s: string): void };
type Printer = { print(s: string): void };

const adapt = (old: OldPrinter): Printer => ({
  print: (s) => old.printOld(s),
});
```

#### 採用例

- Java の Arrays.asList（配列を List のインターフェースで見せる）

実コード（OpenJDK の java.util.Arrays.asList）: https://cr.openjdk.org/~plevart/misc/valhala-hacks/javany-webrev.01/raw_files/new/src/java.base/share/classes/java/util/Arrays.java。GitHub 上のパスは src/java.base/share/classes/java/util/Arrays.java。

```java
public static <T> List<T> asList(T... a) {
    return new ArrayList<>(a);
}

private static class ArrayList<E> extends AbstractList<E>
        implements RandomAccess, java.io.Serializable {
```

### Decorator / middleware

ケース: 機能を層で重ねたい。

```ts
type Handler = (req: string) => string;

const withLog = (h: Handler): Handler => (req) => {
  console.log("req:", req);
  return h(req);
};
const withAuth = (h: Handler): Handler => (req) =>
  req === "secret" ? h(req) : "401";

const app = withLog(withAuth((req) => "ok"));
```

ネストが読みにくい場合は、pipe 形で並べる。reduceRight を使うと、書いた順がそのまま実行順（外側→内側）になる。

```ts
type Wrapper = (h: Handler) => Handler;

const pipe = (base: Handler, ...wrappers: Wrapper[]): Handler =>
  wrappers.reduceRight((h, w) => w(h), base);

const app2 = pipe((req) => "ok", withLog, withAuth);
// withLog(withAuth(base)) と等価
```

#### 採用例

- Koa の middleware（onion model と呼ばれる合成）
- Java I/O の BufferedInputStream 等のラッパー

実コード（koa-compose の dispatch）: 現行版の compose は index.js の30〜73行（https://deepwiki.com/koajs/compose/2.1-compose() による）。以下の抜粋は古典版のもので、現行版は async 化と検証追加により構造が異なる。

```js
return function (context, next) {
  let index = -1
  return dispatch(0)
  function dispatch (i) {
    if (i <= index) return Promise.reject(new Error('next() called multiple times'))
    index = i
    let fn = middleware[i]
    if (i === middleware.length) fn = next
    if (!fn) return Promise.resolve()
    try {
      return Promise.resolve(fn(context, function next () {
        return dispatch(i + 1)
      }))
```

### Composite

ケース: 木構造の葉と枝を同じ操作で扱いたい。

```ts
type Node = { name: string; size(): number };

const file = (name: string, bytes: number): Node => ({
  name,
  size: () => bytes,
});
const dir = (name: string, children: Node[]): Node => ({
  name,
  size: () => children.reduce((sum, c) => sum + c.size(), 0),
});

const root = dir("root", [file("a.txt", 100), dir("sub", [file("b.txt", 50)])]);
root.size(); // 150。枝も葉も同じ size() で再帰的に合計される
```

#### 採用例

- DOM の Element ツリー
- コンパイラの AST（TypeScript Compiler 等）

実コード（TypeScript Compiler の SyntaxKind）: https://github.com/microsoft/TypeScript/blob/1294ee8c1f37c8bf214db67e180edb3220a52236/src/compiler/types.ts#L21。SyntaxKind 列挙体は同ファイルの40〜492行（DeepWiki の索引による）。

```
// Contextual keywords
AbstractKeyword,
AsKeyword,
AssertsKeyword,
AnyKeyword,
AwaitKeyword,
BooleanKeyword,
ConstructorKeyword,
DeclareKeyword,
InferKeyword,
KeyOfKeyword,
NamespaceKeyword,
NeverKeyword,
```

全 AST ノードは共通基底 Node（kind: SyntaxKind を持つ）に従う。

## データ・並行・分散

### Event Sourcing + CQRS

ケース: 変更履歴そのものを正として、状態は後から再構成したい。

```ts
type Event =
  | { type: "deposited"; amount: number }
  | { type: "withdrawn"; amount: number };

// 現在の残高は「保存された値」ではなくイベント列からの再構成
const balance = (events: Event[]) =>
  events.reduce((b, e) => (e.type === "deposited" ? b + e.amount : b - e.amount), 0);

// 書き込みは「イベントを追記するだけ」、読み取りは「再構成した値」で分離するのが CQRS
```

#### 採用例

- EventStoreDB（イベントストア専用 DB）
- Axon Framework（Java の CQRS / Event Sourcing 基盤）

実コード（Axon Framework の利用例、DZone の実装記事より）: 核は AggregateLifecycle.apply で、集約のコマンドハンドラの中からイベントを発行する。

```java
public AccountAggregate(CreateAccountCommand createAccountCommand) {
    AggregateLifecycle.apply(new AccountCreatedEvent(
        createAccountCommand.id, createAccountCommand.accountBalance, createAccountCommand.currency));
}
```

### Durable Execution

ケース: 長い処理を途中で止めたい、障害で落ちても続きから再開したい。

```ts
// Temporal の workflow の概念図（疑似コード）
// 各 activity の完了は記録され、プロセスが落ちても完了済みは再実行されず、続きから再開する
async function orderWorkflow(orderId: string) {
  await activities.reserveStock(orderId);
  await activities.charge(orderId); // ここで落ちても、次回はここから
  await activities.ship(orderId);
}
```

#### 採用例

- Temporal
- AWS Step Functions

実コード（Temporal 公式サンプル、samples-typescript の child-workflows）: https://github.com/temporalio/samples-typescript/blob/main/child-workflows/src/workflows.ts（GitHub のファイルページを開いて確認済み）。

```ts
import { executeChild } from '@temporalio/workflow';

export async function parentWorkflow(...names: string[]): Promise<string> {
  const responseArray = await Promise.all(
    names.map((name) =>
      executeChild(childWorkflow, {
        args: [name],
      }),
    ),
  );
  return responseArray.join('\n');
}
```

### Actor

ケース: 状態の競合を避けたい。状態はプロセスの中に閉じ、外からはメッセージでしか触れない。

```elixir
defmodule Counter do
  use GenServer

  def handle_call(:get, _from, count), do: {:reply, count, count}
  def handle_cast(:inc, count), do: {:noreply, count + 1}
end
# プロセスが1つずつメッセージを処理するため、ロックなしで競合が起きない
```

#### 採用例

- WhatsApp のバックエンド（Erlang/OTP）
- Microsoft Orleans（Halo シリーズのバックエンドで使われた仮想 Actor）

実コード（Erlang/OTP の gen_server 本体）: https://github.com/erlang/otp/blob/master/lib/stdlib/src/gen_server.erl（GitHub のファイルページを開いて確認済み）。抜粋は、コールバック関数の契約を型仕様として固定している箇所。

```erlang
handle_call :: fun((Request :: term(), From :: from(), State :: term()) ->
                       {reply, Reply :: term(), NewState :: term()} |
                       {reply, Reply :: term(), NewState :: term(), Action :: action()} |
                       {noreply, NewState :: term()} |
                       {noreply, NewState :: term(), action()} |
                       {stop, Reason :: term(), Reply :: term(), NewState :: term()} |
                       {stop, Reason :: term(), NewState :: term()}),
```

### CRDT（G-Counter）

ケース: 複数端末で同時編集した結果を、後から矛盾なくマージしたい。

```ts
// 各ノードが自分の分だけ増やす。マージは要素ごとの max、合計は全部の和
type GCounter = Record<string, number>;

const inc = (c: GCounter, node: string) => {
  c[node] = (c[node] ?? 0) + 1;
};
const merge = (a: GCounter, b: GCounter): GCounter => {
  const out = { ...a };
  for (const k in b) out[k] = Math.max(out[k] ?? 0, b[k]);
  return out;
};
const value = (c: GCounter) => Object.values(c).reduce((x, y) => x + y, 0);
// merge は交換・結合・冪等なので、どの順序で同期しても同じ結果になる
```

#### 採用例

- Yjs（CRDT ライブラリ）
- Automerge（CRDT ライブラリ）
- Figma のマルチプレイヤー（CRDT に着想を得た独自実装を技術ブログで公表している）

注記: Yjs の実装ファイルは src/types/YMap.js（YMap クラス）と src/types/YArray.js で、特定は https://deepwiki.com/yjs/yjs/3.6-events-and-observation による。

### 楽観的ロック

ケース: 同時更新の衝突を、ロックせずに検出したい。

```sql
-- version が読み取り時と一致する場合だけ更新する
UPDATE docs SET body = ?, version = version + 1
WHERE id = ? AND version = ?;
```

```ts
const save = async (doc: { id: string; version: number; body: string }) => {
  const res = await db.query(
    "UPDATE docs SET body = $1, version = version + 1 WHERE id = $2 AND version = $3",
    [doc.body, doc.id, doc.version],
  );
  if (res.rowCount === 0) throw new Error("他の人が先に更新しています");
};
```

#### 採用例

- Hibernate / JPA の @Version
- HTTP の ETag と If-Match による条件付きリクエスト
- Rails ActiveRecord の locking 実装

実コード（ActiveRecord の楽観的ロック実装）: https://github.com/rails/rails/blob/d0dcb8fa6073a0c4d42600c15e82e3bb386b27d3/activerecord/lib/active_record/locking/optimistic.rb（GitHub のファイルページを開いて確認済み）。更新時に lock_version を +1 し、旧バージョンを条件に含めて更新して、影響行数が1でなければ StaleObjectError を投げる。

```ruby
def _update_row(attribute_names, attempted_action = "update")
  return super unless locking_enabled?

  locking_column = self.class.locking_column
  lock_attribute_was = @attributes[locking_column]
  update_constraints = _query_constraints_hash
  attribute_names << locking_column
  self[locking_column] += 1
  affected_rows = self.class._update_record(attributes_with_values(attribute_names), update_constraints)

  if affected_rows != 1
    raise ActiveRecord::StaleObjectError.new(self, attempted_action)
```

## 言語に組み込まれたもの

### Iterator / generator

ケース: 列挙の手順を隠し、必要な分だけ遅延して生みたい。

```ts
function* pages(total: number, per: number) {
  for (let p = 1; p <= Math.ceil(total / per); p++) yield p;
}
for (const p of pages(95, 10)) {
  /* 1..10 を1つずつ */
}
```

#### 採用例

- Python の iterator protocol
- Rust 標準ライブラリの Iterator trait

実コード（Rust 標準ライブラリの Iterator trait）: https://doc.rust-lang.org/1.83.0/src/core/iter/traits/iterator.rs.html。

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

### 後片付けの保証

ケース: 確保したリソースを必ず解放したい。

```ts
const openFile = (path: string) => ({
  read: () => "...",
  [Symbol.dispose]: () => console.log("close", path),
});

{
  using f = openFile("a.txt"); // ブロックを抜けると dispose が必ず呼ばれる
  f.read();
}
// 同じ役割: C++/Rust の RAII、Java の try-with-resources、C# の using、Go の defer
```

#### 採用例

- Rust の Drop trait
- C++ 標準ライブラリの std::unique_ptr / std::lock_guard（RAII）

実コード（Rust 標準ライブラリの Drop trait）: https://doc.rust-lang.org/beta/src/core/ops/drop.rs.html。

```rust
#[lang = "drop"]
pub trait Drop {
    fn drop(&mut self);
}
```

### Option / null 安全

ケース: 「値がない」ことを null でなく型で表したい。

```ts
type Option<T> = { some: true; value: T } | { some: false };

// TypeScript 標準では T | undefined + narrowing で近いことができる
const findUser = (id: string): User | undefined => users.get(id);
const u = findUser("1");
if (u !== undefined) u.name; // この中では User として使える
```

#### 採用例

- Haskell の Maybe（起源）
- Rust の Option
- Swift の Optional

実コード（Rust 標準ライブラリの Option）: https://doc.rust-lang.org/1.86.0/src/core/option.rs.html。

```rust
pub enum Option<T> {
    None,
    Some(T),
}
```

### パターンマッチング

ケース: 種類ごとに分岐し、網羅漏れを検出したい。TypeScript には match がないので、discriminated union + switch と never チェックで代用する。

```ts
type Shape =
  | { kind: "circle"; r: number }
  | { kind: "rect"; w: number; h: number };

const area = (s: Shape): number => {
  switch (s.kind) {
    case "circle": return Math.PI * s.r ** 2;
    case "rect": return s.w * s.h;
    default: {
      const _: never = s; // Shape に種類を足すと、ここがコンパイルエラーになる
      throw new Error("unreachable");
    }
  }
};
// Rust、Swift、Python 3.10、Java の switch では match として言語組み込み
```

#### 採用例

- Rust の match
- Elixir の関数頭部でのパターンマッチ

実コード（Elixir 公式コードのモジュールドキュメント内にある例）: https://github.com/elixir-lang/elixir/blob/92e1830bf7548684462be5ce14083dea551b9811/lib/elixir/lib/kernel.ex（GitHub のファイルページを開いて確認済み）。構造体パターンで age を取り出し、when ガードで条件を付ける典型形。

```elixir
def drive(%User{age: age}) when age >= 16 do
  ...
end
```

注記: パターンマッチングは言語機能のため、代表箇所の一箇所特定は困難。これは公式コード自身が示す典型形である。

### async / await

ケース: 非同期の連鎖を手続き的に書きたい。Promise の連鎖が構文になったもの。

```ts
const load = async (id: string) => {
  const user = await fetchUser(id);
  const posts = await fetchPosts(user.id); // 前の結果に依存する処理を上から順に書ける
  return { user, posts };
};
```

#### 採用例

- C# / .NET（async/await を最初に言語へ導入した系統）
- Rust の Tokio ランタイム

実コード（Tokio の runtime ドキュメント例）: https://openrr.github.io/openrr/src/tokio/runtime/mod.rs.html で確認した、#[tokio::main] と spawn の形。

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // ...
    tokio::spawn(async move {
        let mut buf = [0; 1024];
        // ...
    });
}
```

### trait / protocol 的な合成

ケース: 継承でなく「能力の組み合わせ」で型を作りたい。

```ts
type Drawable = { draw(): void };
type Movable = { move(x: number, y: number): void };

// 能力を持つ部品を合成して渡す。Rust の trait、Swift の protocol、Haskell の typeclass と同系統
const sprite = (d: Drawable, m: Movable): Drawable & Movable => ({ ...d, ...m });
```

#### 採用例

- Rust 標準ライブラリの trait 群（Iterator、From / Into 等）
- Swift の protocol

実コード（Rust 標準ライブラリの From trait）: https://doc.rust-lang.org/nightly/src/core/convert/mod.rs.html（588〜594行）。From を実装すると、同ファイルのブランケット実装により Into が自動で得られる（trait 合成の核）。

```rust
pub const trait From<T>: Sized {
    /// Converts to this type from the input type.
    fn from(value: T) -> Self;
}
```
