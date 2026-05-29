# React メモ

React 18/19 の重要機能(React Server Components アーキテクチャ、新フック、Server Action 連携)をまとめます。Next.js などの実装に依存しない、React 自体の機能を扱います。

---

## 1. Server Component と Client Component の使い分け

### 違い

| | Server Component | Client Component |
| --- | --- | --- |
| マーカー | 既定(何も書かない) | ファイル先頭 `'use client'` |
| 実行 | サーバーのみ | サーバー(SSR 初回)+ ブラウザ |
| JS バンドル | 0 KB | コード分含まれる |
| 使える機能 | DB / env / 秘密情報 / `async/await` | フック / イベント / ブラウザ API |

### 判断指針

状態 / イベント / フック / ブラウザ API のどれかが必要なら Client、それ以外は Server。**「困ったら Server、必要になったら葉だけ Client」** が原則。親をいきなり Client にすると子孫が全部 Client 化してバンドルが膨らむ。

### 例

```tsx
// データ取得 + 表示 → Server
export default async function Page() {
  const data = await fetchData();
  return <div>{data.name}</div>;
}

// Modal / Dropdown(Radix が内部でフック使用)→ Client
'use client';
export function MyDialog() {
  const [open, setOpen] = useState(false);
  return <Dialog open={open} ...>...</Dialog>;
}
```

---

## 2. 'use client' 境界の自動伝播

### 概要

`'use client'` を書いたファイルが import するモジュールは、**自身に `'use client'` を書いていなくても Client Component として扱われる**。

### 要点

- 境界の入口だけマーカーを付ければ十分(子に書く必要なし)
- 同じユーティリティでも、Server から import すれば Server、Client から import すれば Client として扱われる
- 「絶対に Server だけで動かしたい」モジュールは `import 'server-only'` で防御(誤 import するとビルドエラー)
- 逆に「絶対 Client」を強制する `client-only` もあるが用途は稀

### 例

```ts
// utils.tsx('use client' なし)
export function Helper() { ... }

// pageA.tsx(Server)→ Helper は Server として動く、JS バンドル 0
// pageB.tsx('use client')→ Helper は Client 化される、JS バンドルに含まれる
```

---

## 3. 'use server' は書く位置で意味が変わる

### 概要

`'use server'` はファイル先頭にも関数先頭にも書けるが、意味が違う。**Server Component と Server Action は別概念**で、ファイル先頭の `'use server'` を Server Component ファイルに書くと壊れる。

### 使い分け

| 位置 | 意味 | 制約 |
| --- | --- | --- |
| **ファイル先頭** | ファイル全体が Server Action モジュール | export は async function のみ(オブジェクト・const は不可) |
| **関数先頭(インライン)** | その関数だけ Server Action 化 | Server Component 内で定義可能、クロージャを使える |

### 例

```tsx
// ❌ Server Component ファイルの先頭に書く
'use server';
export default async function Page() { ... }  // RPC として登録される → 壊れる

// ⭕ 関数先頭のインライン(Server Component 内で Co-located Server Action)
export default async function Page() {
  async function deleteItem(formData: FormData) {
    'use server';
    await prisma.item.delete({ where: { id: String(formData.get('id')) } });
  }
  return <form action={deleteItem}>...</form>;
}

// ⭕ 別ファイルでファイル先頭(複雑な Action / 複数箇所で利用)
// _actions/create-project.ts
'use server';
export async function createProjectAction(...) { ... }
```

---

## 4. Server util(`'use server'` を付けない server side 関数)

### 概要

サーバー側からしか呼ばない普通の async 関数(Server Component / Server Action から import される共通処理)には `'use server'` を **付けない**。`'use server'` は「Client から RPC で呼べるエンドポイント」を意味し用途が違う。

### 要点

| 状況                                                       | 書き方                                                                 |
| ---------------------------------------------------------- | ---------------------------------------------------------------------- |
| Server Component / Server Action から import される共通 util | `'use server'` 無しの普通の async function。誤 import 防御は `import 'server-only'` |
| Client Component から直接呼ばれる(form の action 等)     | `'use server'` 付きファイル(または関数先頭インライン)              |

- `'use server'` を付けると **RPC エンドポイントとして公開** + export 制約(async function 以外不可)+ ランタイムオーバーヘッド
- `server-only` は「Client にバンドルされたらビルドエラー」を強制する別の防御マーカー
- 上流に `server-only` を持つモジュール(例: 認証付き fetch ラッパー)を import している util は transitive に保護される

### 例

```ts
// lib/api/workspaces.ts ('use server' なし)
import { apiFetch } from './client';   // client.ts が 'server-only' を持つ

export async function fetchWorkspace(slug: string) {
  return apiFetch(`/workspaces/${slug}`);
}
// → Server Component / Server Action から import して使う
// → Client から import すると transitive に server-only に当たりビルドエラー
```

---

## 5. Client から Server Component を呼ぶ方法

### 結論

**`children` または props で ReactNode として渡す**のが唯一の合法手段。Client Component が Server Component を直接 import してはいけない(import すると Client 化される、2 節参照)。

### なぜ

Server Component は「サーバーで render された結果(JSON 化された VDOM)」として props 経由でしか Client に渡せない。import は「render する関数の呼び出し」になり、Client 化が伝播する。

### 例

```tsx
// page.tsx (Server)
<ClientWrapper>
  <ServerPanel />   {/* ⭕ children 経由で Server Component を渡せる */}
</ClientWrapper>

// client-wrapper.tsx
'use client';
import { ServerPanel } from './server-panel';   // ❌ こうすると ServerPanel が Client 化
export function ClientWrapper({ children }) {
  const [open, setOpen] = useState(true);
  return open ? <div>{children}</div> : null;
}
```

応用: Client Component の中で Server で取得したデータを使いたいときは、Server 側で取得 → props で Plain Object を渡す(シリアライズ可能な値のみ)。

---

## 6. useActionState で Server Action とフォームを繋ぐ

### 概要

React 19 で導入されたフック。`<form action={...}>` の送信を Server Action と統合し、結果状態 / pending フラグを返す。

### 要点

| 返り値        | 内容                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------ |
| `state`       | action が前回返した値(初回は `initialState`)                                            |
| `formAction`  | `<form action={...}>` に渡す関数。submit で `action(state, formData)` を実行し戻り値が次の `state` |
| `isPending`   | 送信中フラグ(ボタン disable / spinner 用)                                                |

- 旧 `useFormState`(react-dom)が React 19 で `useActionState` に改名 + React core 移動
- 戻り値で state を更新するパターンを言語化、エラー時だけ画面に残して表示するフローと相性が良い
- JS 無効でも `<form action={...}>` は動く(Progressive Enhancement)

### 例

```tsx
const [state, formAction, pending] = useActionState(myAction, { ok: false });

return (
  <form action={formAction}>
    <input name="email" />
    {state.error && <p>{state.error}</p>}
    <button disabled={pending}>{pending ? '送信中...' : '送信'}</button>
  </form>
);
```

---

## 7. useOptimistic で楽観的 UI 更新

### 概要

React 19 のフック。非同期処理(Server Action 等)の確定を待たずに UI を先行更新する。
チャット送信で、サーバー応答を待たず送信メッセージを即表示する用途など。

### 要点

| 要素 | 内容 |
| --- | --- |
| `actualState` | 確定済みの実データ(props / useState の値) |
| `updateFn(current, value)` | 楽観更新の計算関数(現在 state + 楽観値 → 新 state) |
| `optimisticState` | 表示用 state(楽観更新が乗った状態) |
| `addOptimistic(value)` | 楽観更新をトリガー。**transition 内でしか呼べない**(form action / startTransition / Server Action 内) |

- `addOptimistic` 後、`optimisticState` が即更新 → 裏で非同期処理 → 完了で `actualState` 更新 → 楽観分は**自動でリセット**され実データに置き換わる
- 楽観更新は一時的(永続化しない)。失敗時も transition 完了で自動的に消える(実データに戻る)
- `useState` で手動 push でも似たことは可能だが、失敗時ロールバック + 実データとの二重管理が自前。`useOptimistic` はそれを React に任せられる
- 典型パターン: `useOptimistic` + `useActionState` + Server Action + `revalidatePath`(楽観表示しつつ確定データはサーバー再取得で置換)

### 例

```tsx
const [optimisticMessages, addOptimistic] = useOptimistic(
  messages,                                  // 実データ(props)
  (current, text: string) => [...current, { id: 'tmp', text, pending: true }],
);

function handleAction(formData: FormData) {
  addOptimistic(String(formData.get('text')));  // 即表示
  formAction(formData);                          // 裏で Server Action → revalidate
}

<form action={handleAction}>...</form>
```

---

## 8. React.cache でリクエスト内 dedup

### 概要

`cache`(`import { cache } from 'react'`)は関数をラップし、**同一リクエスト(1 レンダリングパス)内で同じ引数の呼び出し結果をメモ化**する。Server Component の階層(layout / page / 子)で同じデータを複数回取得する重複呼び出しを 1 回に集約する。

### 要点

| 項目 | 内容 |
| --- | --- |
| `cache(fn)` | `fn` をラップした新しい関数を返す |
| メモ化キー | 引数(プリミティブ等価 / オブジェクトは参照等価)。引数が違えば別実行 |
| 有効範囲 | **1 リクエスト(1 レンダリングパス)のみ**。リクエストをまたがない |

- 解決する問題: layout と page が同じデータを別々に取りに行く重複呼び出し
- `fetch` の自動 dedup は `fetch` API を使う呼び出しだけ。`React.cache` は任意の関数(ORM 直呼び、複数 fetch をまとめた関数、エラーハンドリング込みラッパー)に使える
- リクエストをまたぐ永続キャッシュは別物(Next.js の Data Cache / `unstable_cache`)

### 例

```tsx
import { cache } from 'react';

export const fetchUser = cache(async (id: string) => {
  const res = await fetch(`/api/users/${id}`, { cache: 'no-store' });
  return res.json();
});
// layout.tsx と page.tsx が両方 fetchUser('123') を呼んでも実行は 1 回
```
