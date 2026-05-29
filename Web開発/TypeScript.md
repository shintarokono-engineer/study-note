# TypeScript メモ

TypeScript 本体の機能・型表現・実用パターン、および TypeScript で扱う頻度の高い JavaScript 標準機能をまとめます。

---

## 1. `readonly` 配列(`readonly T[]`)と `as const`

`Array<T>` の **読み取り専用版**。配列を変更するメソッド(`push` / `pop` / `splice` / `sort` / `reverse` / 添字代入)を **型レベルで剥がす**。コンパイル時のみの制約(実行時には普通の配列で `Object.freeze` ではない)。

### 書き方の比較

| 書き方 | 同じ意味 | 主な使い所 |
| --- | --- | --- |
| `readonly T[]` | 推奨(短い・現代的) | 関数パラメータ、型注釈 |
| `ReadonlyArray<T>` | 同等(型エイリアス) | 古いコード / 明示性重視 |
| `as const` | readonly + literal 型固定 | 定数(enum 代替) |

```ts
// すべて同じ型
function f(items: readonly number[]) {}
function f(items: ReadonlyArray<number>) {}

// as const: 要素まで literal 型に固定される
const STATUSES = ['draft', 'published', 'archived'] as const;
// 型: readonly ['draft', 'published', 'archived']
type Status = (typeof STATUSES)[number]; // 'draft' | 'published' | 'archived'
```

### 何ができなくなる / できる

```ts
const arr: readonly number[] = [1, 2, 3];

arr.push(4);   // ❌ Property 'push' does not exist
arr[0] = 99;   // ❌ Index signature only permits reading
arr.sort();    // ❌ sort も in-place 変更扱い

arr.length;          // ✅
arr.map(x => x * 2); // ✅(新しい配列を返すメソッドは OK)
arr.slice(0, 2);     // ✅
[...arr];            // ✅(展開してコピー)
```

### 用途 1: 関数パラメータで「中身を変更しない」を **型で約束**

```ts
// ❌ 危ない: 受け取った配列を中で sort してしまう(呼び出し元の配列まで並び替わる)
function sortDescending(items: number[]): number[] {
  items.sort((a, b) => b - a);
  return items;
}

// ✅ 安全: readonly で受ける → 中で sort しようとすると TS エラー
function sortDescending(items: readonly number[]): number[] {
  return [...items].sort((a, b) => b - a);
}
```

→ 「**この関数は引数を変更しない**」を型シグネチャで宣言できる。実装を読まずに安全性が分かる。

### 用途 2: 不変データの定数(`as const` で literal 型固定)

```ts
const ROLES = ['OWNER', 'ADMIN', 'DEVELOPER'] as const;
type Role = (typeof ROLES)[number]; // 'OWNER' | 'ADMIN' | 'DEVELOPER'

function hasRole(role: Role) {
  if (ROLES.includes(role)) { ... }
}
```

enum を使わずに Union 型 + 配列の両方を持てるので、ループも型チェックも一気にできる。React/Next.js の constants で頻出。

### 用途 3: API レスポンス型 / Redux state 等

```ts
interface AppState {
  readonly users: readonly User[];   // state は不変
  readonly projects: readonly Project[];
}
```

immutable update(`{ ...state, users: [...state.users, newUser] }`)を強制できる。

### ハマりどころ

| ハマり | 説明 |
| --- | --- |
| **ネストは浅い** | `readonly { items: number[] }[]` の `items` は **mutable**。深くするには `DeepReadonly<T>` 等を自作 |
| **mutable 関数に渡せない** | `readonly T[]` を `T[]` 引数に渡すと TS エラー(逆は OK)。これは仕様 |
| **実行時には普通の配列** | `readonly` は **コンパイル時のみ**。JS 実行時は push できてしまう |
| **`as const` 配列を mutable 関数に渡せない** | 同上の理由。受け側を `readonly` にするか、`[...arr]` でコピー |

### `Readonly<T>` との関係(オブジェクト用)

```ts
interface User { id: string; name: string; }
const u: Readonly<User> = { id: 'x', name: 'y' };
u.name = 'z';   // ❌ Cannot assign to 'name' because it is read-only
```

`Readonly<T>` は **オブジェクトのプロパティを全部 readonly** にするマップ型。配列は `readonly T[]` の方を使う。

### 引数の readonly 化は習慣化が効く

新しい関数を書くとき、**「この引数を中で変更しないなら readonly を付ける」** をデフォルトにすると:

- 副作用のある実装が型エラーで弾かれる
- 呼び出し側の `as const` 配列がそのまま渡せる
- レビュー時に意図が伝わる

---

## 2. `Function.prototype.bind` で引数を部分適用(JS 標準)

### 概要

JavaScript 標準の関数メソッド。元の関数の引数を **部分適用** して新しい関数を返す。シグネチャを変えたい場面(例: React 19 `useActionState` が期待する `(prevState, formData)` の 2 引数に、外部値を先注入したい)で頻出。

### 要点

- `fn.bind(thisArg, arg1, arg2, ...)` で arg を先固定、新しい関数オブジェクトを返す(元の関数は不変)
- `thisArg` は内部 `this` の指定。アロー関数 / `this` を使わない関数では `null` で OK
- React render 毎に再生成されるので、コンポーネント内で使うときは `useMemo` で固定するのが行儀よい
- TypeScript は `bind` の戻り値型を推論してくれる(`lib.es5.d.ts` で定義)

### 例

```ts
// 基本: 引数の部分適用
function add(a: number, b: number, c: number) { return a + b + c; }
const addOne = add.bind(null, 1);  // a を 1 で固定 → (b, c) => number
addOne(2, 3);  // → 6

// React 19 + Server Action: useActionState は (state, formData) の 2 引数を要求
// action 本体は (slug, state, formData) の 3 引数なので bind で slug を先固定
const boundAction = useMemo(
  () => createProjectAction.bind(null, slug),
  [slug],
);
const [state, formAction, pending] = useActionState(boundAction, INITIAL_STATE);
```

### 似た選択肢との比較

| 方法 | 用途 |
| --- | --- |
| `fn.bind(null, x)` | 部分適用。JS 標準、追加 import 不要 |
| `() => fn(x, ...args)` | アロー関数で包む。同等だが render 毎に新規関数 |
| hidden field でフォームに値を入れる | Server Action 専用の代替手段。**DOM に晒される / 改ざんリスク / 文字列キャスト必要** で bind より劣る |
