---
author:
  - Muse（パーソナルエージェント）
  - Claude
model:
  - Muse Spark 1.3
  - Claude Sonnet 5.5
createdAt: 2026-11-09
---

# パターン実装例集

ケースごとに「どう実装するとよいか」を短いコードで示す。言語は TypeScript を基本とし、そのパターンの本家である言語が明確なものはその言語で書く。

各パターンは、ケース、実装例、採用例の順に並ぶ。採用例は1件ごとに見出しを立て、リンクとコード本体を分けて載せる。

> [!note] 採用例のコードについて
> - コードは一次ソース（リポジトリまたは公式ドキュメント）から転記した。取得日は 2026-11-09。
> - `#L` 付きのリンクは、タグまたはコミットで固定している。ブランチ名だけのリンクは、あとで内容が変わりうるので行番号を付けていない。
> - 省略した箇所は `// ...` で示した。インデントは詰めてある。
> - コードを載せていない採用例は、リンクのみとした。
> - 例外は Axon の利用例で、DZone の記事からの引用であり、原典の URL を確認できていない。
> - 各採用例の「要点」は、元の文章のまま残している。事実の確認はしていない。

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

#### 採用例: OkHttp（Request.Builder など Builder 群）

- リンク: [okhttp3/Request.kt](https://github.com/square/okhttp/blob/master/okhttp/src/commonJvmAndroid/kotlin/okhttp3/Request.kt)
- コード: master ブランチから転記

```kotlin
open class Builder {
  internal var url: HttpUrl? = null
  internal var method: String
  internal var headers: Headers.Builder
  internal var body: RequestBody? = null
  internal var cacheUrlOverride: HttpUrl? = null
  internal var tags: Tags = EmptyTags

  constructor() {
    this.method = "GET"
    this.headers = Headers.Builder()
  }

  // ...

  open fun url(url: HttpUrl): Builder =
    apply {
      this.url = url
    }

  // ...

  open fun build(): Request = Request(this)
}
```

#### 採用例: Rust 標準ライブラリの `NonZero<T>::new`

- 要点: smart constructor。0 なら None を返し、生成経路を絞る。`NonZeroU32::new` は `T = u32` の場合。
- リンク: [core/src/num/nonzero.rs（1.86.0）](https://github.com/rust-lang/rust/blob/1.86.0/library/core/src/num/nonzero.rs#L367-L376)
- コード: タグ 1.86.0 から転記

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

#### 採用例: Spring Framework の DI コンテナ（BeanFactory）

- リンク: [BeanFactory.java](https://github.com/spring-projects/spring-framework/blob/main/spring-beans/src/main/java/org/springframework/beans/factory/BeanFactory.java)
- コード: main ブランチから転記

```java
/**
 * The root interface for accessing a Spring bean container.
 *
 * <p>This is the basic client view of a bean container;
 * ...
 */
public interface BeanFactory {

    // ...
    String FACTORY_BEAN_PREFIX = "&";

    // ...
    Object getBean(String name) throws BeansException;

    // ...
}
```

#### 採用例: Angular（DI を前提にした framework 設計）

- リンク（ドキュメント）: [Angular の DI](https://angular.dev/guide/di)
- リンク（ソース）: [packages/core/src/di/injector.ts](https://github.com/angular/angular/blob/main/packages/core/src/di/injector.ts)
- コード: 掲載なし（リンクのみ）

### Singleton の代替

ケース: 全体で1つだけ共有したい。

```ts
// クラスで管理せず、モジュールが1回だけ評価される性質を使う
export const config = loadConfig(); // import した全員が同じ1つを共有する
```

#### 採用例: Prisma Client を globalThis に載せて再利用する構成

- 要点: Prisma 公式ドキュメントが、開発時の多重生成を防ぐ方法として推奨している。Next.js のホットリロードで、モジュールが何度も読み直されることへの対策。
- リンク: [Prisma Docs: Next.js のトラブルシューティング](https://www.prisma.io/docs/orm/v7/more/troubleshooting/nextjs)
- コード: 公式ドキュメントのスニペットを原文のまま転記

```ts
// lib/prisma.ts
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

#### 採用例: XState（状態機械ライブラリ）

- リンク: [packages/core/src/createMachine.ts](https://github.com/statelyai/xstate/blob/main/packages/core/src/createMachine.ts)
- コード: 掲載なし（リンクのみ）

#### 採用例: Erlang/OTP の gen_statem behaviour

- 要点: 状態名と同名のコールバック関数がイベントを処理し、`{next_state, 次状態, 新Data}` を返して遷移を宣言する。
- リンク（例）: [OTP Design Principles: gen_statem Behaviour](https://www.erlang.org/doc/system/statem.html)
- リンク（本体）: [lib/stdlib/src/gen_statem.erl](https://github.com/erlang/otp/blob/maint/lib/stdlib/src/gen_statem.erl)
- コード: 上記ドキュメントの code_lock 例から転記。`locked/3` と `open/3` の部分

```erlang
locked(
  cast, {button,Button},
  #{code := Code, length := Length, buttons := Buttons} = Data) ->
    NewButtons =
        if
            length(Buttons) < Length ->
                Buttons;
            true ->
                tl(Buttons)
        end ++ [Button],
    if
        NewButtons =:= Code -> % Correct
            do_unlock(),
            {next_state, open, Data#{buttons := []},
             [{state_timeout,10_000,lock}]}; % Time in milliseconds
        true -> % Incomplete | Incorrect
            {next_state, locked, Data#{buttons := NewButtons}}
    end.

open(state_timeout, lock,  Data) ->
    do_lock(),
    {next_state, locked, Data};
open(cast, {button,_}, Data) ->
    {next_state, open, Data}.
```

### Strategy（高階関数）

ケース: アルゴリズムを差し替えたい。

```ts
type Pricing = (price: number) => number;

const regular: Pricing = (p) => p;
const member: Pricing = (p) => p * 0.9;

const checkout = (price: number, pricing: Pricing) => pricing(price);
checkout(1000, member); // 900
```

#### 採用例: Passport.js

- 要点: 認証方式そのものを Strategy として差し替える設計で、名称も Strategy。
- リンク: [jaredhanson/passport-strategy](https://github.com/jaredhanson/passport-strategy)
- コード: README のサブクラス化の定型（master ブランチ）を、原文の2ブロックのまま転記

```js
var util = require('util')
  , Strategy = require('passport-strategy');

function CustomStrategy(...) {
  Strategy.call(this);
}

util.inherits(CustomStrategy, Strategy);
```

```js
CustomStrategy.prototype.authenticate = function(req, options) {
  // TODO: authenticate request
}
```

#### 採用例: Java の Comparator

- リンク: [java/util/Comparator.java](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/Comparator.java)
- コード: 掲載なし（リンクのみ）

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

#### 採用例: VS Code の Commands

- 要点: ID で登録し、キーバインドや他の拡張から実行できる。定義は `CommandsRegistry`。
- リンク: [src/vs/platform/commands/common/commands.ts](https://github.com/microsoft/vscode/blob/main/src/vs/platform/commands/common/commands.ts)
- コード: 掲載なし（リンクのみ）

#### 採用例: Redux の action

- 要点: シリアライズ可能な Command とみなせる。
- リンク（ドキュメント）: [Redux Fundamentals: Concepts and Data Flow](https://redux.js.org/tutorials/fundamentals/part-2-concepts-data-flow)
- リンク（リポジトリ）: [reduxjs/redux](https://github.com/reduxjs/redux)
- コード: 掲載なし（リンクのみ）

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

#### 採用例: RxJS（Observable）

- リンク: [src/internal/Observable.ts（7.x）](https://github.com/ReactiveX/rxjs/blob/7.x/src/internal/Observable.ts)
- コード: 掲載なし（リンクのみ）

#### 採用例: SolidJS の signals

- 要点: Observer の細粒度化。
- リンク: [packages/solid/src/reactive/signal.ts](https://github.com/solidjs/solid/blob/main/packages/solid/src/reactive/signal.ts)
- コード: main ブランチから、`createSignal` の宣言部分を転記

```ts
export function createSignal<T>(): Signal<T | undefined>;
export function createSignal<T>(value: T, options?: SignalOptions<T>): Signal<T>;
export function createSignal<T>(
  value?: T,
  options?: SignalOptions<T | undefined>
): Signal<T | undefined> {
  // ...
}
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

#### 採用例: Java の Arrays.asList

- 要点: 配列を List のインターフェースで見せる。
- リンク: [java/util/Arrays.java](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/Arrays.java)
- コード: master ブランチから転記

```java
@SafeVarargs
@SuppressWarnings("varargs")
public static <T> List<T> asList(T... a) {
    return new ArrayList<>(a);
}

/**
 * @serial include
 */
private static class ArrayList<E> extends AbstractList<E>
    implements RandomAccess, java.io.Serializable
{
    // ...
    private final E[] a;

    ArrayList(E[] array) {
        a = Objects.requireNonNull(array);
    }

    // ...
}
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

#### 採用例: Koa の middleware

- 要点: onion model と呼ばれる合成。
- リンク: [koajs/compose の index.js](https://github.com/koajs/compose/blob/master/index.js)
- コード: master ブランチの `composeSlim` を転記。同ファイルの `compose` は、`NODE_ENV` が production のときだけ `composeSlim` を返し、それ以外では `next()` の二重呼び出しなどの検査を足した実装を返す

```js
const composeSlim = (middleware) => async (ctx, next) => {
  const dispatch = (i) => async () => {
    const fn = i === middleware.length
      ? next
      : middleware[i]
    if (!fn) return
    return await fn(ctx, dispatch(i + 1))
  }
  return dispatch(0)()
}
```

#### 採用例: Java I/O の BufferedInputStream 等のラッパー

- リンク: [java/io/BufferedInputStream.java](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/io/BufferedInputStream.java)
- コード: 掲載なし（リンクのみ）

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

#### 採用例: DOM の Element ツリー

- リンク: [DOM Standard: Element](https://dom.spec.whatwg.org/#interface-element)
- コード: 掲載なし（リンクのみ）

#### 採用例: コンパイラの AST（TypeScript Compiler）

- 要点: 全 AST ノードは共通基底 `Node`（`kind: SyntaxKind` を持つ）に従う。
- リンク（SyntaxKind）: [src/compiler/types.ts#L158-L172](https://github.com/microsoft/TypeScript/blob/1294ee8c1f37c8bf214db67e180edb3220a52236/src/compiler/types.ts#L158-L172)
- リンク（Node）: [src/compiler/types.ts#L841-L843](https://github.com/microsoft/TypeScript/blob/1294ee8c1f37c8bf214db67e180edb3220a52236/src/compiler/types.ts#L841-L843)
- コード: コミット 1294ee8 から転記

```ts
// SyntaxKind の途中
// Contextual keywords
AbstractKeyword,
AsKeyword,
AssertsKeyword,
AnyKeyword,
AsyncKeyword,
AwaitKeyword,
BooleanKeyword,
ConstructorKeyword,
DeclareKeyword,
GetKeyword,
InferKeyword,
IntrinsicKeyword,
IsKeyword,
KeyOfKeyword,
```

```ts
export interface Node extends ReadonlyTextRange {
    readonly kind: SyntaxKind;
    readonly flags: NodeFlags;
    // ...
}
```

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

#### 採用例: EventStoreDB

- 要点: イベントストア専用 DB。
- リンク: [EventStore/EventStore](https://github.com/EventStore/EventStore)
- コード: 掲載なし（リンクのみ）

#### 採用例: Axon Framework

- 要点: Java の CQRS / Event Sourcing 基盤。集約のコマンドハンドラの中から `AggregateLifecycle.apply` でイベントを発行する。
- リンク（リポジトリ）: [AxonFramework/AxonFramework](https://github.com/AxonFramework/AxonFramework)
- リンク（API の定義）: [AggregateLifecycle.java（axon-4.10.x）](https://github.com/AxonFramework/AxonFramework/blob/axon-4.10.x/modelling/src/main/java/org/axonframework/modelling/command/AggregateLifecycle.java)
- コード（定義）: axon-4.10.x ブランチから転記

```java
public static ApplyMore apply(Object payload) {
    return AggregateLifecycle.getInstance().doApply(payload, MetaData.emptyInstance());
}
```

- コード（利用例）: DZone の実装記事からの引用。記事の URL は確認できていない

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

#### 採用例: Temporal

- リンク: [samples-typescript/child-workflows/src/workflows.ts](https://github.com/temporalio/samples-typescript/blob/main/child-workflows/src/workflows.ts)
- コード: 公式サンプル（main ブランチ）の `parentWorkflow` を転記

```ts
import { executeChild } from '@temporalio/workflow';

export async function parentWorkflow(...names: string[]): Promise<string> {
  const responseArray = await Promise.all(
    names.map((name) =>
      executeChild(childWorkflow, {
        args: [name],
        // workflowId, // add business-meaningful workflow id here
        // // regular workflow options apply here, with two additions (defaults shown):
        // cancellationType: ChildWorkflowCancellationType.WAIT_CANCELLATION_COMPLETED,
        // parentClosePolicy: ParentClosePolicy.PARENT_CLOSE_POLICY_TERMINATE
      }),
    ),
  );
  return responseArray.join('\n');
}
```

#### 採用例: AWS Step Functions

- リンク: [AWS Step Functions とは](https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html)
- コード: 掲載なし（マネージドサービスで、実装は公開されていない）

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

#### 採用例: WhatsApp のバックエンド（Erlang/OTP）

- リンク: [WhatsApp - Wikipedia](https://en.wikipedia.org/wiki/WhatsApp)（Erlang で書かれていることへの言及を確認した二次資料）
- コード: 掲載なし（実装は公開されていない）

#### 採用例: Microsoft Orleans

- 要点: Halo シリーズのバックエンドで使われた仮想 Actor。
- リンク: [dotnet/orleans](https://github.com/dotnet/orleans)
- コード: 掲載なし（リンクのみ）

#### 採用例: Erlang/OTP の gen_server 本体

- 要点: コールバック関数の契約を、型仕様として固定している箇所。
- リンク: [lib/stdlib/src/gen_server.erl](https://github.com/erlang/otp/blob/master/lib/stdlib/src/gen_server.erl)
- コード: master ブランチから転記。`-record(server_data, ...)` の内側にある `handle_call` のフィールド

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

#### 採用例: Yjs（CRDT ライブラリ）

- 要点: 実装ファイルは `src/types/YMap.js`（YMap クラス）と `src/types/YArray.js`。
- リンク（YMap）: [src/types/YMap.js（v13.6.33）](https://github.com/yjs/yjs/blob/v13.6.33/src/types/YMap.js)
- リンク（YArray）: [src/types/YArray.js（v13.6.33）](https://github.com/yjs/yjs/blob/v13.6.33/src/types/YArray.js)
- コード: 掲載なし（リンクのみ）

> [!note] Yjs のリンクについて
> main ブランチでは、上記のパスが 404 だった。そのためタグ v13.6.33 に固定した。

#### 採用例: Automerge（CRDT ライブラリ）

- リンク: [automerge/automerge](https://github.com/automerge/automerge)
- コード: 掲載なし（リンクのみ）

#### 採用例: Figma のマルチプレイヤー

- 要点: CRDT に着想を得た独自実装を、技術ブログで公表している。
- リンク: [How Figma's multiplayer technology works](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/)
- コード: 掲載なし（実装は公開されていない）

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

#### 採用例: Hibernate / JPA の @Version

- リンク: [jakarta.persistence.Version](https://github.com/jakartaee/persistence/blob/master/api/src/main/java/jakarta/persistence/Version.java)（JPA 側の注釈の定義）
- コード: 掲載なし（リンクのみ）

#### 採用例: HTTP の ETag と If-Match による条件付きリクエスト

- リンク: [RFC 9110: If-Match](https://www.rfc-editor.org/rfc/rfc9110#name-if-match)
- コード: 掲載なし（仕様書のため）

#### 採用例: Rails ActiveRecord の locking 実装

- 要点: 更新時に `lock_version` を +1 し、旧バージョンを条件に含めて更新する。影響行数が1でなければ `StaleObjectError` を投げる。
- リンク: [activerecord/lib/active_record/locking/optimistic.rb#L92-L118](https://github.com/rails/rails/blob/d0dcb8fa6073a0c4d42600c15e82e3bb386b27d3/activerecord/lib/active_record/locking/optimistic.rb#L92-L118)
- コード: コミット d0dcb8f から転記

```ruby
def _update_row(attribute_names, attempted_action = "update")
  return super unless locking_enabled?

  begin
    locking_column = self.class.locking_column
    lock_attribute_was = @attributes[locking_column]

    update_constraints = _query_constraints_hash

    attribute_names = attribute_names.dup if attribute_names.frozen?
    attribute_names << locking_column

    self[locking_column] += 1

    affected_rows = self.class._update_record(
      attributes_with_values(attribute_names),
      update_constraints
    )

    if affected_rows != 1
      raise ActiveRecord::StaleObjectError.new(self, attempted_action)
    end

    affected_rows

  # If something went wrong, revert the locking_column value.
  rescue Exception
    # ...
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

#### 採用例: Python の iterator protocol

- リンク: [Python docs: Iterator Types](https://docs.python.org/3/library/stdtypes.html#iterator-types)
- コード: 掲載なし（リンクのみ）

#### 採用例: Rust 標準ライブラリの Iterator trait

- リンク: [core/src/iter/traits/iterator.rs#L39-L75（1.86.0）](https://github.com/rust-lang/rust/blob/1.86.0/library/core/src/iter/traits/iterator.rs#L39-L75)
- コード: タグ 1.86.0 から転記

```rust
pub trait Iterator {
    /// The type of the elements being iterated over.
    #[rustc_diagnostic_item = "IteratorItem"]
    #[stable(feature = "rust1", since = "1.0.0")]
    type Item;

    // ...

    #[lang = "next"]
    #[stable(feature = "rust1", since = "1.0.0")]
    fn next(&mut self) -> Option<Self::Item>;

    // ...
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

#### 採用例: Rust の Drop trait

- リンク: [core/src/ops/drop.rs#L204-L242（1.86.0）](https://github.com/rust-lang/rust/blob/1.86.0/library/core/src/ops/drop.rs#L204-L242)
- コード: タグ 1.86.0 から転記。doc コメントは省略

```rust
#[lang = "drop"]
#[stable(feature = "rust1", since = "1.0.0")]
#[const_trait]
#[rustc_const_unstable(feature = "const_destruct", issue = "133214")]
pub trait Drop {
    // ...
    #[stable(feature = "rust1", since = "1.0.0")]
    fn drop(&mut self);
}
```

#### 採用例: C++ 標準ライブラリの std::unique_ptr / std::lock_guard（RAII）

- リンク（unique_ptr）: [cppreference: std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)
- リンク（lock_guard）: [cppreference: std::lock_guard](https://en.cppreference.com/w/cpp/thread/lock_guard)
- コード: 掲載なし（リンクのみ）

### Option / null 安全

ケース: 「値がない」ことを null でなく型で表したい。

```ts
type Option<T> = { some: true; value: T } | { some: false };

// TypeScript 標準では T | undefined + narrowing で近いことができる
const findUser = (id: string): User | undefined => users.get(id);
const u = findUser("1");
if (u !== undefined) u.name; // この中では User として使える
```

#### 採用例: Haskell の Maybe

- 要点: 起源。
- リンク: [GHC.Maybe（base）](https://hackage.haskell.org/package/base/docs/GHC-Maybe.html)
- コード: 掲載なし（リンクのみ）

#### 採用例: Rust の Option

- リンク: [core/src/option.rs#L572-L581（1.86.0）](https://github.com/rust-lang/rust/blob/1.86.0/library/core/src/option.rs#L572-L581)
- コード: タグ 1.86.0 から転記

```rust
pub enum Option<T> {
    /// No value.
    #[lang = "None"]
    #[stable(feature = "rust1", since = "1.0.0")]
    None,
    /// Some value of type `T`.
    #[lang = "Some"]
    #[stable(feature = "rust1", since = "1.0.0")]
    Some(#[stable(feature = "rust1", since = "1.0.0")] T),
}
```

#### 採用例: Swift の Optional

- リンク: [stdlib/public/core/Optional.swift](https://github.com/swiftlang/swift/blob/main/stdlib/public/core/Optional.swift)
- コード: 掲載なし（リンクのみ）

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

#### 採用例: Rust の match

- リンク: [The Rust Programming Language: match 制御フロー](https://doc.rust-lang.org/book/ch06-02-match.html)
- コード: 掲載なし（リンクのみ）

#### 採用例: Elixir の関数頭部でのパターンマッチ

- 要点: 構造体パターンで `age` を取り出し、`when` ガードで条件を付ける典型形。
- リンク: [lib/elixir/lib/kernel.ex#L154-L156](https://github.com/elixir-lang/elixir/blob/92e1830bf7548684462be5ce14083dea551b9811/lib/elixir/lib/kernel.ex#L154-L156)
- コード: コミット 92e1830 から転記。`Kernel` モジュールのドキュメント（Guards の節）にある例

```elixir
def drive(%User{age: age}) when age >= 16 do
  ...
end
```

> [!note] 注記
> パターンマッチングは言語機能なので、代表箇所を1か所に特定するのは難しい。上のコードは、公式のドキュメント自身が示す典型形である。

### async / await

ケース: 非同期の連鎖を手続き的に書きたい。Promise の連鎖が構文になったもの。

```ts
const load = async (id: string) => {
  const user = await fetchUser(id);
  const posts = await fetchPosts(user.id); // 前の結果に依存する処理を上から順に書ける
  return { user, posts };
};
```

#### 採用例: C# / .NET

- 要点: async/await を最初に言語へ導入した系統。
- リンク: [Microsoft Learn: C# の非同期プログラミング](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/)
- コード: 掲載なし（リンクのみ）

#### 採用例: Rust の Tokio ランタイム

- リンク: [tokio/src/runtime/mod.rs](https://github.com/tokio-rs/tokio/blob/master/tokio/src/runtime/mod.rs)
- コード: master ブランチのモジュールドキュメント（`//!` コメント内の例）から転記。`//!` は外した

```rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        let (mut socket, _) = listener.accept().await?;

        tokio::spawn(async move {
            let mut buf = [0; 1024];

            // ...
```

### trait / protocol 的な合成

ケース: 継承でなく「能力の組み合わせ」で型を作りたい。

```ts
type Drawable = { draw(): void };
type Movable = { move(x: number, y: number): void };

// 能力を持つ部品を合成して渡す。Rust の trait、Swift の protocol、Haskell の typeclass と同系統
const sprite = (d: Drawable, m: Movable): Drawable & Movable => ({ ...d, ...m });
```

#### 採用例: Rust 標準ライブラリの trait 群（Iterator、From / Into など）

- 要点: `From` を実装すると、同ファイルのブランケット実装（`impl<T, U> Into<U> for T where U: From<T>`）により `Into` が自動で得られる。trait 合成の核。
- リンク: [core/src/convert/mod.rs#L582-L588（1.86.0）](https://github.com/rust-lang/rust/blob/1.86.0/library/core/src/convert/mod.rs#L582-L588)
- コード: タグ 1.86.0 から `From` の定義を転記

```rust
pub trait From<T>: Sized {
    /// Converts to this type from the input type.
    #[rustc_diagnostic_item = "from_fn"]
    #[must_use]
    #[stable(feature = "rust1", since = "1.0.0")]
    fn from(value: T) -> Self;
}
```

#### 採用例: Swift の protocol

- リンク: [The Swift Programming Language: Protocols](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/)
- コード: 掲載なし（リンクのみ）
