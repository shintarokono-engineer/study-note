# Prisma メモ

Node.js / TypeScript 向けの ORM。スキーマ定義言語 + マイグレーションツール + 型付きクライアント + GUI を一体で提供する。

---

## 1. Prisma の構成要素

| 要素                              | 役割                                                           | 主なコマンド                              |
| --------------------------------- | -------------------------------------------------------------- | ----------------------------------------- |
| **Prisma Schema**(`schema.prisma`) | DB の単一の真実の源(モデル・enum・リレーション・インデックス) | —                                         |
| **Prisma Migrate**                | schema の差分から SQL マイグレーションを生成・適用             | `prisma migrate dev` / `migrate deploy`   |
| **Prisma Client**                 | schema から自動生成される型付き DB クライアント               | `prisma generate`                         |
| **Prisma Studio**                 | テーブル/データを操作する GUI(localhost:5555)                | `prisma studio`                           |

---

## 2. schema.prisma の基本構造

```prisma
generator client {
  provider = "prisma-client-js"
}
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Role { OWNER ADMIN MEMBER }   // /// で各値にドキュメントコメント可

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  role      Role     @default(MEMBER)
  posts     Post[]                       // 1:N リレーション
  createdAt DateTime @default(now())

  @@index([email])                       // 複合インデックスは @@index([a, b])
}
```

- フィールド属性: `@id` / `@unique` / `@default(...)` / `@updatedAt` / `@db.Text` / `@db.Decimal(10,4)` 等
- ブロック属性: `@@id([a, b])`(複合 PK)/ `@@index([...])` / `@@unique([...])`
- `onDelete: Cascade` でリレーションの親削除連鎖
- `prisma format` で整形、`prisma validate` で構文チェック

---

## 3. PostgreSQL 拡張(pgvector 等)の扱い

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]   // ← これが必要
}
datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [pgvector(map: "vector", schema: "public")]
}
model Doc {
  embedding Unsupported("vector(1536)")?       // Prisma がネイティブ型を持たないカラム
}
```

- `extensions = [...]` を書くと `prisma migrate` が `CREATE EXTENSION` を自動で migration に含める
- Prisma がネイティブ型を持たないカラム(`vector` 等)は `Unsupported("...")` で宣言。query では生 SQL かサードパーティ拡張が必要
- **`Unsupported` カラムへのインデックス(HNSW 等)は Prisma が生成できない** → 手書き SQL マイグレーションで別途追加

---

## 4. マイグレーション

| 操作                                                       | コマンド / 方法                                                                          | 用途                       |
| ---------------------------------------------------------- | ---------------------------------------------------------------------------------------- | -------------------------- |
| 開発用(差分検出 → SQL 生成 → 適用 → Client 再生成)       | `prisma migrate dev --name <名前>`                                                       | ローカル開発               |
| 本番適用(未適用 migration を順に適用、対話なし)          | `prisma migrate deploy`                                                                  | CI / 本番デプロイ          |
| DB を初期化して全 migration 再実行                         | `prisma migrate reset`                                                                   | スキーマがおかしくなった時 |
| 手書き SQL マイグレーション                                | `prisma/migrations/<timestamp>_<名前>/migration.sql` を手で作成 → `migrate dev`/`deploy` が拾う | 拡張機能・特殊インデックス・データ移行 |

- `migration_lock.toml` に provider を記録(誤って別 DB に切替える事故を防ぐ)
- **ハマりどころ: advisory lock の残留**
  - `prisma migrate` を実行中に途中で kill すると `pg_advisory_lock` が解放されず残る
  - 次回 `migrate dev` が `P1002: Timed out trying to acquire a postgres advisory lock` で失敗
  - 対処: DB サーバー(コンテナ)を再起動して全セッションをリセット

---

## 5. prisma generate

- `schema.prisma` を読んで `node_modules/@prisma/client` に **型付きクライアントコード** を生成
- **schema を編集したら必ず再実行**(モデル追加・変更 → generate しないと型が古いまま)
- `migrate dev` 実行時は自動で generate も走る
- `--watch` で schema 変更を監視して自動 regenerate
- pnpm の場合、生成物の実体は `node_modules/.pnpm/@prisma+client@x.x.x/...` で symlink 参照

---

## 6. Prisma Client Extension(`$extends`)

query が実行される直前に割り込んで `args` を書き換えたり結果を加工したりできる仕組み。マルチテナント分離(全 query に `WHERE tenantId = ?` を自動付与)などに使う。

```ts
const ext = Prisma.defineExtension({
  name: 'tenant-isolation',
  query: {
    $allModels: {                          // 全モデルを対象
      $allOperations({ model, operation, args, query }) {
        // args を書き換えてから query(modifiedArgs) を呼ぶ
        return query({ ...args, where: { ...args?.where, tenantId } });
      },
    },
  },
});
const prisma = new PrismaClient().$extends(ext);
```

- **`query` フックは実行時の振る舞いだけ変えて TypeScript の型は変えない**(create の data に必須フィールドを自動補完しても型エラーは消えない)
- `$extends` の戻り値は `PrismaClient` のサブクラスではなく **別の型**(model メソッドがラップされた新型)。`$connect` / `$disconnect` / `$transaction` は残る
- 拡張の種類: `query`(クエリ書き換え)/ `result`(計算フィールド追加)/ `model`(カスタムメソッド追加)/ `client`(トップレベルメソッド追加)
- **`extendedWhereUnique`(Prisma 5.0+ で GA)**: `findUnique` / `update` / `delete` の `where` に非ユニークフィールドを足せる。`{ where: { id: 'x', tenantId: 'y' } }` が有効になる(Extension での tenantId 注入に必須)

---

## 7. モノレポでの Prisma パッケージ配布

`packages/db` のような共有パッケージで Prisma をラップする場合の注意点:

| 問題                                                                                       | 対処                                                                                          |
| ------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| `main: "./src/index.ts"` だと CommonJS の consumer(NestJS 等)がランタイムで `require` できない(Node は `.ts` を require 不可) | `tsc` で `dist/` にコンパイルし、`main: "./dist/index.js"` / `types: "./dist/index.d.ts"` を指す |
| Next.js は `.ts` のままでも `transpilePackages` で扱えるが、API(NestJS)は無理            | コンパイル済み JS にしておけば両方の consumer で動く                                          |
| consumer が古いビルドを参照する                                                            | turbo の `build` タスクに `dependsOn: ["^build"]` を付けて依存パッケージを先にビルド          |

`@prisma/client` 自体は `prisma generate` でコンパイル済み JS が出るので、`import { PrismaClient } from '@prisma/client'` は常に動く。問題になるのは **自前のラッパーパッケージの index** だけ。

---

## 8. NestJS との統合パターン

```ts
@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit()    { await this.$connect(); }
  async onModuleDestroy() { await this.$disconnect(); }
}
```

- `@Global()` Module で provide すれば全 feature module から(import 不要で)注入できる
- **Client Extension 適用時**: `$extends` の戻り値はクラスの `extends` 句に書けない型なので、「コンストラクタが拡張クライアントを返すクラス式」を作る:
  ```ts
  const ExtendedCtor = class {
    constructor() { return new PrismaClient().$extends(ext); }
  } as new () => ReturnType<typeof createExtended>;
  @Injectable() export class PrismaService extends ExtendedCtor implements OnModuleInit { ... }
  ```

---

## 9. PrismaClient シングルトン(dev HMR 対策)

```ts
const g = globalThis as unknown as { prisma?: PrismaClient };
export const prisma = g.prisma ?? new PrismaClient();
if (process.env.NODE_ENV !== 'production') g.prisma = prisma;
```

- PrismaClient は内部に接続プールを持つので **アプリ全体で 1 インスタンス**
- dev の Hot Reload でモジュールが再評価されるたびに `new PrismaClient()` すると接続が枯渇する → `globalThis` にキャッシュして使い回す

---

## 10. ER 図の自動生成

| ジェネレータ            | 出力                          | 必要なもの                                                                            |
| ----------------------- | ----------------------------- | ------------------------------------------------------------------------------------- |
| `prisma-erd-generator`  | SVG / PNG / Mermaid Markdown  | SVG/PNG は `@mermaid-js/mermaid-cli`(mmdc)= Puppeteer + Chromium。Markdown 出力なら不要 |
| `prisma-dbml-generator` | `.dbml`(dbdiagram.io 互換)   | 軽量。生成した `.dbml` を https://dbdiagram.io/d に貼ってビジュアル表示                |

`schema.prisma` に `generator erd { provider = "prisma-erd-generator"; output = "..." }` を足すと `prisma generate` のたびに自動生成される。`///` ドキュメントコメントは DBML の note として保持される。

---

## 11. Prisma Client のクエリメソッド早見

| メソッド | 用途 / 挙動 |
| --- | --- |
| `findUnique({ where: {<一意キー>}, select })` | 一意キーで 0/1 件取得。`where` は `@id` / `@unique` / 複合一意キーのみ |
| `findFirst({ where, orderBy })` | 条件に合う最初の 1 件。`where` に非一意フィールド・`OR` / `AND` / `NOT` も使える |
| `findMany({ where, select, include, orderBy, take, skip })` | 0+ 件取得 |
| `create({ data })` | 1 件 INSERT |
| `upsert({ where, create, update })` | 「`where` に合う行があれば `update`、無ければ `create`」。**冪等な書き込み**(再実行しても結果が同じ)を作りやすい。`where` は **`@id` / `@unique` / `@@unique` のいずれか必須**。id が auto-generate で未確定 / FK が鶏卵 / unique 値が編集で変動するケースでは使えず、`findFirst → 分岐` に逃げる |
| `update({ where: {<一意>}, data })` | 1 件 UPDATE。**該当行が無いと例外**(P2025) |
| `updateMany({ where, data })` | 0+ 件 UPDATE。**該当行 0 でも例外にならない**。非一意条件で更新したい / 行の有無が不確実なときに使う |
| `delete` / `deleteMany` | `update` / `updateMany` と同じ「1 件 vs 0+ 件」の使い分け |
| `count` / `aggregate` / `groupBy` | 集計系 |

### よく使う引数

- **`select`** — 返すカラムを絞る(指定しないと全スカラーカラム)。`include` とは排他
  ```ts
  prisma.user.findUnique({ where: { id }, select: { id: true, email: true } });
  ```
- **複合主キー / 複合一意キーの `where`** — `@@id([a, b])` や `@@unique([a, b])` には Prisma が `a_b` という合成キー名を生成。オブジェクト 1 個で渡す
  ```ts
  prisma.tenantMember.findUnique({ where: { tenantId_userId: { tenantId, userId } } });
  ```
- **リレーションのネスト `select` / `include`** — 関連レコードも一緒に取る
  ```ts
  prisma.tenant.findUnique({ where: { slug }, select: { id: true, owner: { select: { email: true } } } });
  ```
- **`_count`** — 関連の件数を `select` に混ぜる。`{ _count: { members: 3 } }` の形で返る
  ```ts
  prisma.tenant.findUnique({ where: { slug }, select: { _count: { select: { members: true } } } });
  ```
- **リレーションの書き込み(ネスト `data`)** — 子(`@relation(fields, references)` で FK 列を持つ側)の FK を `connect` / `create` 等で宣言的に操作
  ```ts
  // connect: 既存の親を unique で探し、子の FK 列にその id をセット(親は新規作成しない)
  prisma.blogPost.update({
    where: { id },
    data: { delivery: { connect: { announcementId_channel: { announcementId, channel: 'BLOG' } } } },
  });
  // SQL: 1) SELECT id FROM Delivery WHERE ... 2) UPDATE BlogPost SET deliveryId = ...
  ```
  | キーワード | 意味 |
  | --- | --- |
  | `create` | 親を新規作成 + 紐付け |
  | `connect` | 既存の親を unique で探す + 紐付け(不在は `RecordNotFoundError`) |
  | `connectOrCreate` | 探して無ければ create |
  | `disconnect: true` | FK 列を null に |
  | `update` / `delete` | 親側のフィールド更新 / 削除 |
  | `set` | 1:N で「この id 集合に置き換える」 |
  - 親側から書いても子側から書いても **同じ SQL**(子の FK 列を書き換える)になる syntactic sugar
  - 1:1 で BlogPost ↔ Delivery が相互参照する鶏卵パターンは「先に子を FK 抜きで `create` → 親を `upsert` → `connect` で後付け」 の 3 段に分けるのが定石

### enum の扱い

- `schema.prisma` の `enum Plan { FREE PRO TEAM }` は `@prisma/client` に **値 + 型**として生成される(`Plan.PRO === 'PRO'`、型 `Plan = 'FREE' | 'PRO' | 'TEAM'`)
- 入出力型が文字列ユニオンなので `data: { plan: 'PRO' }` でも型安全。だが**マジック文字列を散らさない**ために `data: { plan: Plan.PRO }` と書く方がリネーム耐性がある
- モノレポで Prisma を別パッケージにしている場合、その index で `export { Plan, Role, ... } from '@prisma/client'` して再エクスポートすると、利用側パッケージから enum を import できる

---

## 12. where 句の演算子と複合一意キー

各フィールドにフィルタ演算子を渡せる(`{ field: value }` は `{ field: { equals: value } }` の糖衣):

| 種別 | 例 | SQL 相当 |
| --- | --- | --- |
| 比較(数値・日付) | `createdAt: { gte: d }` / `{ gt, lte, lt, equals, not }` | `createdAt >= d` |
| in / notIn | `id: { in: ['a', 'b'] }` / `{ notIn: [...] }` | `id IN ('a', 'b')` |
| 文字列 | `name: { contains: 'foo', mode: 'insensitive' }` / `{ startsWith, endsWith }` | `name ILIKE '%foo%'` |
| null | `field: null` / `field: { not: null }` | `IS NULL` / `IS NOT NULL` |
| 論理結合(トップレベル) | `OR: [{ a: 1 }, { b: 2 }]` / `AND: [...]` / `NOT: {...}` | `(a = 1 OR b = 2)` |
| リレーション(1:N) | `members: { some: {...}, every: {...}, none: {...} }` | EXISTS 系 |
| リレーション(1:1) | `owner: { is: {...}, isNot: {...} }` | |

### 複合一意キーの where(合成キー名)

`@@id([a, b])` や `@@unique([a, b])` を持つモデルでは、Prisma が「一意キーで 1 行を特定する操作」(`findUnique` / `update` / `delete` / `upsert`)の `where` 用に、**フィールド名を `_` で繋いだ合成キー名**を自動生成する。

```ts
// @@id([tenantId, userId]) のモデル
prisma.tenantMember.findUnique({ where: { tenantId_userId: { tenantId: 'x', userId: 'y' } } });
// SQL 相当: WHERE "tenantId" = 'x' AND "userId" = 'y'
```

- 合成名はフィールドの**宣言順**(`[tenantId, userId]` → `tenantId_userId`)。`@@id([a, b], name: "primaryKey")` で改名可
- 単一フィールドの一意キー(`@id` / `@unique` 1 個)は `where: { id: 'x' }` と直接書く。`_` 繋ぎになるのは**複数フィールドの組**のときだけ
- `findUnique` 等は一意キーしか受け付けない(`where: { tenantId: ..., userId: ... }` のようにバラでは渡せない)
- `extendedWhereUnique`(Prisma 5.0+ GA)を使うと、`findUnique` / `update` / `delete` の `where` に「一意キー + 非一意フィールド」を併記できる(追加の絞り込み・楽観ロック向け): `where: { tenantId_userId: {...}, role: 'OWNER' }`

---

## 13. 集計フィールド `_count` / `_sum` / `_avg` / `_min` / `_max`

Prisma が **動的に提供するメタフィールド**(DB の実カラムではなく、SQL 集計を Prisma が組んで返す)。アンダーバー始まりはユーザー定義カラム(例: `count`)との **名前衝突を避ける規約**。

### `_count` を `select` / `include` で使う(リレーション件数)

子の **件数だけ欲しい** ときに使う。`include` で全行取得するより圧倒的に軽い(行データを転送しない)。

```ts
const project = await prisma.project.findUnique({
  where: { id },
  select: {
    id: true,
    name: true,
    _count: { select: { checklist: true, documents: true } },
  },
});
// → { id, name, _count: { checklist: 5, documents: 3 } }
```

- 1 SELECT に COUNT サブクエリを混ぜ込むので **追加クエリ不要**(N+1 防止)
- `where` で件数の絞り込みも可: `_count: { select: { documents: { where: { deletedAt: null } } } }`

### `aggregate` / `groupBy` で `_sum` / `_avg` / `_min` / `_max` / `_count`

数値カラムの集計を返す。`GROUP BY` を Prisma で表現するときに使う。

```ts
// テナント月次の AI トークン合計
await prisma.aIUsage.aggregate({
  where: { tenantId, createdAt: { gte: monthStart } },
  _sum: { tokensIn: true, tokensOut: true },
  _count: { id: true },
});
// → { _sum: { tokensIn: 12345, tokensOut: 6789 }, _count: { id: 42 } }

// プロジェクト別の Document 件数
await prisma.projectDocument.groupBy({
  by: ['projectId'],
  _count: { id: true },
});
// → [{ projectId: 'p1', _count: { id: 3 } }, ...]
```

| フィールド | 用途 | 出る場所 |
| --- | --- | --- |
| `_count` | 件数 | `select` / `include` / `aggregate` / `groupBy` |
| `_sum` | 合計 | `aggregate` / `groupBy` |
| `_avg` | 平均 | `aggregate` / `groupBy` |
| `_min` | 最小 | `aggregate` / `groupBy` |
| `_max` | 最大 | `aggregate` / `groupBy` |

### なぜアンダーバー prefix か

| 理由 | 内容 |
| --- | --- |
| 名前衝突回避 | ユーザーが `count` という DB カラムを定義する自由を残す |
| メタ情報の視覚的区別 | 実カラム(`id` / `name`)と派生カラム(`_count`)を見分けられる |

### `include` との対比

| 書き方 | 意味 | 戻り値 |
| --- | --- | --- |
| `include: { checklist: true }` | 子レコードを全行 SELECT | `{ ..., checklist: ChecklistItem[] }` |
| `select: { _count: { select: { checklist: true } } }` | 件数だけ取る | `{ ..., _count: { checklist: 5 } }` |

---

## 14. raw SQL の動的構築(`Prisma.sql` / `Prisma.empty`)

`$queryRaw` の tagged template に動的フラグメントを差し込むときに使うヘルパー。文字列連結は SQL インジェクションになるため、必ずこれを通す。

| API | 役割 |
| --- | --- |
| `` Prisma.sql`...` `` | フラグメント生成、`${value}` は自動でパラメータ化 |
| `Prisma.empty` | 空フラグメント(条件分岐の「なし」側) |
| `Prisma.join(items)` | 配列を `$1, $2, ...` に展開(IN リスト) |
| `Prisma.raw(str)` | 生文字列。**ユーザー入力 NG**(SQLi 直撃) |

### 例: 条件付きフラグメント

```ts
const exclude = excludeId
  ? Prisma.sql`AND "projectId" != ${excludeId}`
  : Prisma.empty;

await prisma.$queryRaw`
  SELECT id FROM "Document"
  WHERE "tenantId" = ${tenantId}
    ${exclude}
  LIMIT ${limit}
`;
```

---

## 15. 業務分類列の表現選定(`enum` / `String` / `Json` / `String[]`)

「セレクトの選択肢」「カテゴリ」「複数選択タグ」 等を schema でどう持つか。基幹マスター(`Plan` / `Role` のように安定 + 集計頻発)は `enum`、業務分類で値が増減しそうなら `String?` / `Json?` + アプリ層検証がデフォルト。

### 表現の比較

| 表現                  | 値追加コスト     | 多選択             | nested 構造 | DB 妥当性 | 主なクエリ性             |
| --------------------- | ---------------- | ------------------ | ----------- | --------- | ------------------------ |
| `enum`                | migration 必須   | `enum[]`(癖あり)  | ❌ 不可     | ✅ 強制   | enum 比較が速い          |
| `String?` `@db.Text`  | コード 1 箇所    | ❌(1 値のみ)      | ❌ 不可     | ⚠️ アプリ層 | LIKE / 完全一致          |
| `Json?`               | コード 1 箇所    | ✅ 配列で素直      | ✅ object 可 | ⚠️ アプリ層 | JSONB GIN で `@>` 検索   |
| `String[]`            | コード 1 箇所    | ✅ ネイティブ配列  | ❌ 不可     | ⚠️ アプリ層 | `has` / `hasEvery`       |

### 判断軸

- **増減頻度**: 半年で値が変わりそう → `String` / `Json` / 何年も固定 → `enum`
- **nested 属性**: `{ scale, role }` 等の複合は `Json` 一択(`enum` は scalar 制約)
- **多選択 + 検索要件**: JSONB の `@>` 検索(GIN インデックス可)が将来クエリで詰みにくい
- **DB 妥当性の代替**: `class-validator` `@IsIn(...)` + FE membership filter の 2 層で実質的に enum 同等のガードを構築できる

### 具体例: 複数選択タグ(JSONB + アプリ層検証)

```prisma
model Project {
  /// 想定コア機能(`["AI", "API", "WEBHOOK", ...]`)
  coreFeatures Json?
}
```

```ts
// 値の SSoT(TS)
export const CORE_FEATURES = ['AI', 'API', 'WEBHOOK', ...] as const;
export type CoreFeature = (typeof CORE_FEATURES)[number];

// BE 検証(NestJS DTO)
@IsArray()
@ArrayMaxSize(CORE_FEATURES.length)
@IsIn(CORE_FEATURES, { each: true })
coreFeatures?: CoreFeature[];
```

値追加は配列に push するだけ、migration 不要。
