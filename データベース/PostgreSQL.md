# PostgreSQL メモ

PostgreSQL の特徴・MySQL との比較・pgvector / RAG までを 1 ファイルに集約します。

---

## 1. PostgreSQL vs MySQL

### 思想の違い

| 観点         | PostgreSQL                              | MySQL                            |
| ------------ | --------------------------------------- | -------------------------------- |
| 思想         | 標準 SQL 準拠、機能豊富、研究志向       | 速度重視、シンプル、Web で広く使われる |
| 開発元       | PostgreSQL Global Development Group     | Oracle(MySQL AB → Sun → Oracle) |
| ライセンス   | PostgreSQL License(MIT 互換)          | GPL(エンタープライズ版あり)    |

### 機能比較

| 機能             | PostgreSQL                         | MySQL                              |
| ---------------- | ---------------------------------- | ---------------------------------- |
| **JSON**         | JSONB(バイナリ + インデックス可)   | JSON 型あるが PG ほど柔軟ではない |
| **ベクトル検索** | pgvector(公式拡張)                | なし(別 DB が必要)               |
| **全文検索**     | 組み込み、多言語対応               | 組み込みあり、FULLTEXT             |
| **配列型**       | ネイティブサポート                 | なし(JSON で代用)                |
| **enum**         | ネイティブサポート(強力)         | あるが弱い(変更困難)             |
| **カスタム型**   | 自由に定義可能                     | 制限あり                           |
| **トランザクション分離** | 標準的(SERIALIZABLE まで利用可) | InnoDB の MVCC、READ COMMITTED が癖あり |
| **同時接続**     | プロセスベース(コネクションプール推奨) | スレッドベース(軽量)            |
| **同時更新の挙動** | MVCC、ロック粒度が細かい          | InnoDB の MVCC、ギャップロックが特徴 |

### 採用例

| DB         | 採用例                                                  |
| ---------- | ------------------------------------------------------- |
| PostgreSQL | GitHub、Instagram、Reddit、Stripe、Notion、Discord     |
| MySQL      | Facebook、Twitter、YouTube、WordPress、Shopify(初期) |

### 案件市場の傾向(2026 年時点)

- B2B SaaS / スタートアップ: **PostgreSQL がデファクト**
- WordPress / 老舗 Web サービス: MySQL が中心
- レガシー業務システム: MySQL or Oracle

### 学習コスト

MySQL のみ経験者が PostgreSQL に移る場合の摩擦は限定的:

- SQL の基本構文は同じ(SELECT / INSERT / UPDATE / DELETE / JOIN)
- ORM(Prisma など)を使えば CRUD で DB 種別をほぼ意識しない
- 主な差分: `AUTO_INCREMENT` vs `SERIAL/IDENTITY`、`TINYINT(1)` vs `BOOLEAN`、識別子の大文字小文字扱い等

数日〜1 週間で吸収可能。

---

## 2. PostgreSQL の代表的な強み

### JSONB

JSON データを **バイナリ形式 + インデックス可能** な型として保存できます。

```sql
CREATE TABLE webhook_events (
  id SERIAL PRIMARY KEY,
  payload JSONB
);

-- JSON 内のフィールドで検索
SELECT * FROM webhook_events WHERE payload->>'type' = 'checkout.session.completed';

-- JSON 内のフィールドにインデックス
CREATE INDEX idx_payload_type ON webhook_events ((payload->>'type'));
```

Webhook ペイロードを生で保存しつつ高速検索したい等、半構造化データの扱いに強い。

### 配列型

```sql
CREATE TABLE projects (
  id SERIAL PRIMARY KEY,
  tags TEXT[]
);

INSERT INTO projects (tags) VALUES (ARRAY['ai', 'saas', 'mvp']);

-- 配列内を検索
SELECT * FROM projects WHERE 'ai' = ANY(tags);
```

### enum

```sql
CREATE TYPE project_status AS ENUM ('IDEA', 'IN_DEV', 'BETA', 'LAUNCHED');

CREATE TABLE projects (
  status project_status DEFAULT 'IDEA'
);
```

MySQL の enum と異なり、新しい値の追加が後から可能。

### 拡張機能(extensions)

PostgreSQL は拡張で機能追加できる設計:

- `pgvector` — ベクトル検索
- `PostGIS` — 地理空間データ
- `pg_trgm` — 類似文字列検索
- `pg_cron` — DB 内で cron ジョブ
- `uuid-ossp` — UUID 生成

---

## 3. pgvector とベクトル検索

### pgvector とは

**PostgreSQL でベクトル検索を可能にする拡張機能**(オープンソース、AWS RDS / Aurora で標準サポート)。

ベクトル検索 = テキストや画像を高次元ベクトル(数値の配列)に変換して、「**意味的に近いもの**」を検索する技術。AI / RAG の中核。

### 基本的な使い方

```sql
-- 拡張を有効化
CREATE EXTENSION vector;

-- ベクトルカラムを持つテーブル
CREATE TABLE documents (
  id SERIAL PRIMARY KEY,
  content TEXT,
  embedding vector(1536)  -- OpenAI text-embedding-3-small は 1536 次元
);

-- HNSW インデックス(近似最近傍検索)
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- 類似検索(クエリベクトル <-> 各 row の距離順)
SELECT id, content
FROM documents
ORDER BY embedding <-> '[0.1, 0.2, ...]'::vector
LIMIT 5;
```

### 距離演算子

| 演算子 | 距離                  | 用途                        |
| ------ | --------------------- | --------------------------- |
| `<->`  | L2(ユークリッド)    | 一般的                      |
| `<#>`  | 内積(負の値)        | 速度優先                    |
| `<=>`  | コサイン(1 - cos類似度) | テキスト埋め込み(推奨)   |

OpenAI `text-embedding-3-small` は **L2 正規化済み** のため、コサイン類似度を使うのが定石。

### インデックス方式

| 方式  | 特徴                                                    |
| ----- | ------------------------------------------------------- |
| **HNSW** | 近似最近傍、高速・高精度・メモリ多め(現代の主流)    |
| IVFFlat  | やや古い、訓練必要、メモリ少なめ                      |
| なし   | 厳密検索、データ少量なら十分(数千件以下)            |

実用では **HNSW** を選ぶのが定石(< 50ms で数百万件を検索可能)。

---

## 4. RAG の基本パターン

RAG = **Retrieval-Augmented Generation**(検索拡張生成)。LLM の回答精度を上げるパターン。

### 基本フロー

```
[1. 事前準備]
ドキュメント(README、社内 wiki 等)
   ↓ embedding 生成(OpenAI / Cohere / 自前モデル)
   ↓
ベクトル DB に保存(pgvector / Pinecone / Qdrant 等)

[2. クエリ時]
ユーザーの質問
   ↓ embedding 生成
   ↓ 類似ベクトル検索(上位 N 件取得)
   ↓
取得したドキュメントを LLM のコンテキストに注入
   ↓
LLM が回答生成
```

### なぜ RAG が必要か

LLM は学習時点のデータしか知らない。社内ドキュメント・最新情報・個人の過去資産を**リアルタイムに**参照させる仕組みが RAG。

### embedding モデルの選び方

| モデル                         | 次元   | コスト                  | 用途                  |
| ------------------------------ | ------ | ----------------------- | --------------------- |
| OpenAI text-embedding-3-small | 1536   | $0.02 / 1M tokens(安) | 汎用、コスパ最強      |
| OpenAI text-embedding-3-large | 3072   | $0.13 / 1M tokens       | 高精度版              |
| Cohere embed-multilingual-v3  | 1024   | $0.10 / 1M tokens       | 多言語性能高          |
| 自前(Sentence-BERT 等)      | 任意   | サーバー代              | プライバシー要件あり |

個人〜中規模 SaaS では **text-embedding-3-small** が定番。

### Prisma での型表現

```prisma
model Document {
  id        String  @id @default(cuid())
  content   String
  embedding Unsupported("vector(1536)")?
}
```

`Unsupported` 型で扱う(Prisma が直接対応していないが、Raw クエリで挿入・検索可能)。

### コールドスタート問題

RAG は **既存データがないと意味がない**。新規ユーザー / 新規プロジェクトでは:

- サンプルデータを seed として投入
- 過去資産のインポート機能を提供
- 「使い込むほど精度が上がる」とメッセージング

これらの工夫がないと、初回利用時に「ただの LLM」と区別がつかず差別化が伝わらない。

---

## 5. Partial Index(部分インデックス)

通常 index は **全行** を index に入れるが、partial index は `WHERE 条件` を満たす行だけを入れる PostgreSQL の機能(MySQL には無い)。

```sql
-- 例: soft delete テーブルで「生存行のみ」を index 化
CREATE INDEX users_active_email_idx
  ON users (email)
  WHERE deleted_at IS NULL;
```

「アクティブ行 << 全行」のテーブル(soft delete / 状態テーブル / フラグ)で **index サイズと書き込みコストを節約** できる。クエリの WHERE が index の WHERE を含意するときにだけ使われる(`deleted_at IS NULL` を付けないと使われない)。Prisma `@@index` には `where` 句が無いため、raw SQL マイグレーションで管理する。
