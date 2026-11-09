# パターン実装例集

ケースごとに「どう実装するとよいか」を短いコードで示す。言語は TypeScript を基本とし、そのパターンの本家である言語が明確なものはその言語で書く。
## 前提環境

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

### Singleton の代替

ケース: 全体で1つだけ共有したい。

```ts
// クラスで管理せず、モジュールが1回だけ評価される性質を使う
export const config = loadConfig(); // import した全員が同じ1つを共有する
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

### Strategy（高階関数）

ケース: アルゴリズムを差し替えたい。

```ts
type Pricing = (price: number) => number;

const regular: Pricing = (p) => p;
const member: Pricing = (p) => p * 0.9;

const checkout = (price: number, pricing: Pricing) => pricing(price);
checkout(1000, member); // 900
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

### Option / null 安全

ケース: 「値がない」ことを null でなく型で表したい。

```ts
type Option<T> = { some: true; value: T } | { some: false };

// TypeScript 標準では T | undefined + narrowing で近いことができる
const findUser = (id: string): User | undefined => users.get(id);
const u = findUser("1");
if (u !== undefined) u.name; // この中では User として使える
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

### async / await

ケース: 非同期の連鎖を手続き的に書きたい。Promise の連鎖が構文になったもの。

```ts
const load = async (id: string) => {
  const user = await fetchUser(id);
  const posts = await fetchPosts(user.id); // 前の結果に依存する処理を上から順に書ける
  return { user, posts };
};
```

### trait / protocol 的な合成

ケース: 継承でなく「能力の組み合わせ」で型を作りたい。

```ts
type Drawable = { draw(): void };
type Movable = { move(x: number, y: number): void };

// 能力を持つ部品を合成して渡す。Rust の trait、Swift の protocol、Haskell の typeclass と同系統
const sprite = (d: Drawable, m: Movable): Drawable & Movable => ({ ...d, ...m });
```
