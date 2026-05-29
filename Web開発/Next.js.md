# Next.js App Router メモ

Next.js App Router 固有の規約(ファイル/ディレクトリ命名、middleware、Route Handler 等)をまとめます。React Server Components アーキテクチャ・`'use client'` / `'use server'` 全般・新フック等の **React 自体の機能** は [React.md](React.md) を参照。

---

## 1. Private Folder(`_` プレフィックス)による Route 除外

### 概要

App Router では、ディレクトリ名を `_` で始めるとそのフォルダ全体が **ルーティング対象から除外** される(Private Folder)。ページ専用の補助コード(コンポーネント / Server Action / フック / ユーティリティ)を `app/` 配下に置きつつ URL 化を防ぐために使う。

### 要点

| ディレクトリ | URL になるか |
| --- | --- |
| `app/foo/page.tsx` | ⭕ `/foo` |
| `app/foo/bar/page.tsx` | ⭕ `/foo/bar` |
| `app/foo/_components/card.tsx` | ❌ |
| `app/foo/_actions/save.ts` | ❌ |

- 慣用名: `_components/` / `_actions/` / `_hooks/` / `_utils/` 等
- ページ固有の部品を「ページと同じディレクトリツリー」に置けて、責務スコープを物理配置で表現できる
- 全体共通の部品は `app/` の外(例: `src/components/`)に置く
- 似た仕組み: route group `(name)` は URL からセグメント名を消すがルーティング自体は有効、Private Folder は **ルーティングごと除外**

### 例

```
app/w/[slug]/
├── page.tsx               # /w/{slug} に対応
├── layout.tsx             # /w/{slug} 配下の共通レイアウト
├── _actions/
│   └── create-project.ts  # ルート化されない(Server Action 群)
└── _components/
    ├── project-card.tsx
    └── new-project-dialog.tsx
```

---

## 2. Server Component のデータ取得タイミング

### 概要

Server Component(`page.tsx` 等の async function)内の `fetch` / `await` は「ページの HTTP リクエストごと」 に走る。クライアントの再レンダー(`useState` 変化等)では走らない。

### 要点

| タイミング | Server Component の fetch |
| --- | --- |
| URL アクセス / ページ遷移 | 走る(Server Component が実行される) |
| `revalidatePath` / `revalidateTag` 後の再生成 | 走る |
| Client Component の再レンダー(`useState` / 入力中など) | 走らない |

- Server Component は「サーバーで 1 回 render された結果」 がブラウザに渡る。クライアント側の再描画は Client 部分だけで、Server の fetch は再実行されない
- `fetch(url, { cache: 'no-store' })` = データキャッシュに乗せない = 毎リクエスト fetch(動的レンダリング)。認証付き・ユーザー別データはこれが基本
- デフォルト fetch や `next: { revalidate: N }` は時間ベースキャッシュ。静的化できるデータ向け
- 同一リクエスト内で同じデータを複数回取りに行く重複は `React.cache` で 1 回に集約できる(→ [React.md](React.md) §8)

### 例

```tsx
// page.tsx — ページにアクセスするたびに実行される(no-store なら毎回 API fetch)
export default async function Page() {
  const sessions = await listSessions();  // ページ遷移ごとに走る / 入力中の再レンダーでは走らない
  return <SessionList items={sessions} />;
}
```
