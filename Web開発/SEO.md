# SEO メモ

検索エンジン(主に Google)に Web サイトを発見・理解・評価させ、検索結果での露出と順位を改善する技術と運用の総称。実装側 SEO(クローラ制御 + 構造化データ + パフォーマンス)とコンテンツ SEO(キーワード設計 + 内部リンク + E-E-A-T)に分かれる。本ノートは実装側に絞る。

---

## 1. SEO の全体像

検索エンジンの動作は **Crawl → Index → Rank → 表示** の 4 段階。実装側 SEO の役割は段階ごとに分かれる。

| 段階       | 検索エンジンの動作               | 実装側で扱う制御手段                                  |
| ---------- | -------------------------------- | ----------------------------------------------------- |
| **Crawl**  | クローラ(Googlebot)がページ取得 | `robots.txt` / `sitemap.xml` / リンク構造             |
| **Index**  | 取得 HTML を解析し DB に登録     | `<meta name="robots">` / `canonical` / 構造化データ   |
| **Rank**   | クエリに対する順位付け           | コンテンツ品質 / リンク / Core Web Vitals             |
| **表示**   | 検索結果(SERP)に表示           | `<title>` / `<meta description>` / OGP / リッチリザルト |

OG / Twitter Card は厳密には SNS シェア用で SEO 順位には直接寄与しないが、慣例的に同じ枠で扱う。

---

## 2. メタタグ(title / description / canonical 等)

### 基本タグ

```html
<head>
  <title>ページタイトル(50-60 文字目安)</title>
  <meta name="description" content="120-160 文字目安。SERP のスニペットに採用されうる" />
  <link rel="canonical" href="https://example.com/path" />
  <meta name="robots" content="index,follow" />
</head>
```

- `title` は**ページごとにユニーク**にする。サイト名は末尾に `|` または `-` 区切りで添える
- `description` は **検索順位に直接寄与しない**が、SERP の CTR に大きく影響する
- `canonical` は重複コンテンツ(`?utm_source=...` 等のパラメータ違い)の正規 URL を指定し、評価分散を防ぐ
- `robots` は `noindex`(インデックス不可)/ `nofollow`(リンク辿らない)/ `noarchive`(キャッシュ無効)等を組み合わせる

### 言語・地域(多言語サイト)

```html
<html lang="ja">
<link rel="alternate" hreflang="ja" href="https://example.com/ja/..." />
<link rel="alternate" hreflang="en" href="https://example.com/en/..." />
<link rel="alternate" hreflang="x-default" href="https://example.com/..." />
```

`x-default` は言語マッチが無い時のフォールバック。

---

## 3. OGP(Open Graph)/ Twitter Card

SNS でシェアされた時のプレビュー制御。

```html
<meta property="og:title" content="..." />
<meta property="og:description" content="..." />
<meta property="og:image" content="https://.../og.png" />  <!-- 絶対 URL 必須 -->
<meta property="og:type" content="website" />              <!-- website / article 等 -->
<meta property="og:url" content="https://example.com/path" />

<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:title" content="..." />
<meta name="twitter:description" content="..." />
<meta name="twitter:image" content="https://.../og.png" />
```

- 画像は **1200×630px** が事実上の標準(Twitter `summary_large_image` も同サイズで両対応)
- 画像 URL は**絶対 URL 必須**。相対 URL は SNS クローラが解決できない
- `og:type` = `article` の場合は `article:published_time` / `article:author` も併設可能

---

## 4. robots.txt

サイトのルート(`/robots.txt`)に置く、クローラへの指示書。**クロール禁止であってインデックス禁止ではない**点に注意。

```
User-agent: *
Allow: /
Disallow: /admin/
Disallow: /api/

Sitemap: https://example.com/sitemap.xml
```

| ディレクティブ      | 意味                                                        |
| ------------------- | ----------------------------------------------------------- |
| `User-agent: *`     | 全クローラ対象(`Googlebot` 等で個別指定も可)              |
| `Allow: /path`      | クロール許可(明示)                                        |
| `Disallow: /path`   | クロール禁止                                                |
| `Crawl-delay: 10`   | 連続リクエスト間隔(秒、Google は無視。Bing 等は尊重)      |
| `Sitemap: <url>`    | sitemap の場所を明示(必須ではないが推奨)                  |

### 落とし穴

- **インデックス禁止には `<meta name="robots" content="noindex">` を使う**。`robots.txt` の `Disallow` だけだと、外部リンクから URL を拾われた場合「タイトル不明」状態でインデックスされうる
- `noindex` を効かせたい場合は逆に `robots.txt` でクロールを許可しないと、クローラが `noindex` を読めずインデックスが残る
- 認証必須ページは `Disallow` + サーバ側 401/302 の二重防御が安全

---

## 5. sitemap.xml

クローラに**インデックスして欲しい URL を一覧で渡す**ファイル。新規・動的ページの発見スピードを上げる。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://example.com/</loc>
    <lastmod>2026-05-29</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>
  <url>
    <loc>https://example.com/posts/abc</loc>
    <lastmod>2026-05-28</lastmod>
  </url>
</urlset>
```

| 要素           | 内容                                                                                |
| -------------- | ----------------------------------------------------------------------------------- |
| `<loc>`        | URL(必須、絶対 URL)                                                              |
| `<lastmod>`    | 最終更新日時(ISO 8601)。Google は鮮度判定に使う                                  |
| `<changefreq>` | 更新頻度ヒント(`always` `hourly` `daily` `weekly` `monthly` `yearly` `never`)     |
| `<priority>`   | サイト内相対優先度(0.0〜1.0)。**Google 公式は無視と表明**しているが慣例で残す    |

### 制限と運用

- 1 sitemap 当たり **50,000 URL / 50MB(非圧縮)** 上限
- 超える場合は sitemap index を作成して sub-sitemap を束ねる
- 動的 sitemap は SSR で生成。`Cache-Control: max-age=3600` 程度でキャッシュ
- Search Console に手動 submit すると検出が早い
- `noindex` ページは sitemap に**入れない**(Search Console 警告の原因)

---

## 6. 構造化データ(Schema.org / JSON-LD)

ページの意味を機械可読にし、**リッチリザルト**(評価 ★ / 価格 / FAQ アコーディオン等)の対象になる。

### 推奨形式は JSON-LD

埋め込み方式は Microdata / RDFa / JSON-LD の 3 種あるが、**Google は JSON-LD を推奨**。`<head>` でも `<body>` 内でも動作する。

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "SoftwareApplication",
  "name": "Example App",
  "description": "...",
  "applicationCategory": "BusinessApplication",
  "operatingSystem": "Web",
  "offers": {
    "@type": "Offer",
    "price": "0",
    "priceCurrency": "JPY"
  },
  "aggregateRating": {
    "@type": "AggregateRating",
    "ratingValue": "4.5",
    "ratingCount": "120"
  }
}
</script>
```

### よく使う `@type`

| `@type`                  | 用途                                       |
| ------------------------ | ------------------------------------------ |
| `Organization`           | 会社・組織情報(運営元の同定)             |
| `WebSite`                | サイト全体(`SearchAction` でサイト内検索) |
| `Article` / `BlogPosting` | 記事(著者・公開日が表示されうる)         |
| `Product`                | 物販商品(価格・在庫・評価)               |
| `SoftwareApplication`    | SaaS / アプリ(価格・OS・評価)            |
| `BreadcrumbList`         | パンくず(SERP のパス表示)                |
| `FAQPage`                | FAQ(SERP でアコーディオン表示)           |
| `HowTo`                  | 手順記事(SERP でステップ表示)            |
| `VideoObject`            | 動画(サムネ + 再生時間)                  |
| `Person`                 | 著者・人物                                 |

### XSS リスクへの対処

ユーザー生成コンテンツを JSON-LD に注入する場合、データ中に `</script>` が混ざると script 終了で XSS。`JSON.stringify` 後に `<` を `<` に置換するのが定番。

```ts
const safe = JSON.stringify(jsonLd).replace(/</g, '\\u003c')
```

### 検証

- Rich Results Test(https://search.google.com/test/rich-results)— リッチリザルト対象か判定
- Schema Markup Validator(https://validator.schema.org/)— Schema.org 仕様の純粋検証
- Search Console「拡張」レポート — 本番でのエラー集計

---

## 7. Core Web Vitals(UX 性能指標)

Google が公式にランキングシグナルとして使う UX 指標。

| 指標    | 意味                                                  | 良好    | 要改善  | 不良    |
| ------- | ----------------------------------------------------- | ------- | ------- | ------- |
| **LCP** | Largest Contentful Paint(主要コンテンツ表示時間)    | ≤ 2.5s  | ≤ 4s    | > 4s    |
| **INP** | Interaction to Next Paint(UI 応答性、2024〜 FID から置換) | ≤ 200ms | ≤ 500ms | > 500ms |
| **CLS** | Cumulative Layout Shift(レイアウトずれ累積)         | ≤ 0.1   | ≤ 0.25  | > 0.25  |

### 主な改善手段

| 指標 | 対策                                                                                                              |
| ---- | ----------------------------------------------------------------------------------------------------------------- |
| LCP  | 画像最適化(`next/image` / `srcset` / hero 画像に `fetchpriority="high"`)、CDN、SSR/SSG、フォント `display: swap` |
| INP  | メインスレッドを塞ぐ JS の細分化(`scheduler.yield` / `requestIdleCallback`)、巨大ライブラリ削減、`use client` 縮小 |
| CLS  | `<img>` / `<iframe>` に `width`/`height` 指定、フォント差し替えで `size-adjust`、広告枠のサイズ予約                  |

### 計測ツール

- **本番**: Search Console の Core Web Vitals レポート(CrUX = 実ユーザーデータ、28 日集計)
- **開発**: Lighthouse / PageSpeed Insights(ラボデータ、シミュレーション)/ `web-vitals` npm パッケージで RUM 送信

---

## 8. Next.js 15 App Router での SEO 実装

App Router は規約ファイル + Metadata API で大半をカバーする。

### Metadata API

```ts
// app/layout.tsx or app/page.tsx
import type { Metadata } from 'next'

export const metadata: Metadata = {
  metadataBase: new URL('https://example.com'),         // 相対 URL の解決基点
  title: { default: 'Example', template: '%s | Example' }, // 子で %s が title に
  description: '...',
  openGraph: {
    title: 'Example',
    description: '...',
    type: 'website',
    url: '/',
    images: ['/og.png'],
  },
  twitter: { card: 'summary_large_image' },
}
```

動的ページは `generateMetadata` を export:

```ts
export async function generateMetadata(
  { params }: { params: { id: string } },
): Promise<Metadata> {
  const post = await fetchPost(params.id)
  return {
    title: post.title,
    description: post.summary,
    alternates: { canonical: `/posts/${params.id}` },
  }
}
```

### 規約ファイル

```ts
// app/robots.ts
import type { MetadataRoute } from 'next'

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [{ userAgent: '*', allow: '/', disallow: ['/admin/', '/api/'] }],
    sitemap: 'https://example.com/sitemap.xml',
  }
}
```

```ts
// app/sitemap.ts
import type { MetadataRoute } from 'next'

export const revalidate = 3600    // ISR(1 時間キャッシュ)

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await fetchAllPosts()
  return [
    { url: 'https://example.com/', lastModified: new Date(), priority: 1.0 },
    ...posts.map((p) => ({
      url: `https://example.com/posts/${p.id}`,
      lastModified: p.updatedAt,
      priority: 0.8,
    })),
  ]
}
```

- ファイルを置くだけで `/robots.txt` / `/sitemap.xml` として配信される
- 50,000 URL を超える場合は `generateSitemaps` で sub-sitemap に分割

### OG 画像の動的生成

```tsx
// app/opengraph-image.tsx
import { ImageResponse } from 'next/og'

export const size = { width: 1200, height: 630 }
export const contentType = 'image/png'

export default async function OG() {
  return new ImageResponse(
    <div style={{ fontSize: 64, display: 'flex' }}>Example</div>,
    size,
  )
}
```

Edge Runtime で動く。フォントは `fetch` で読み込んで `fonts` オプションに渡す。

### JSON-LD

Server Component で `<script type="application/ld+json">` を直接出力。XSS 対策の `<` 置換を忘れない。

```tsx
const jsonLd = { '@context': 'https://schema.org', '@type': 'Article' /* ... */ }
return (
  <>
    <script
      type="application/ld+json"
      dangerouslySetInnerHTML={{
        __html: JSON.stringify(jsonLd).replace(/</g, '\\u003c'),
      }}
    />
    {/* ページ本体 */}
  </>
)
```

---

## 9. 検証 / モニタリングツール

| ツール                          | 用途                                                                       |
| ------------------------------- | -------------------------------------------------------------------------- |
| **Google Search Console**       | インデックス状況 / 検索流入 / CWV / 構造化データレポート(本番運用の起点) |
| **Bing Webmaster Tools**        | Bing 用(国内シェアは小さいが Edge 既定)                                  |
| **Lighthouse**(Chrome DevTools) | パフォーマンス / SEO / a11y / Best Practices のローカル監査                |
| **PageSpeed Insights**          | CrUX 実データ + Lighthouse 統合                                            |
| **Rich Results Test**           | 構造化データ単体検証(リッチリザルト判定)                                 |
| **Schema Markup Validator**     | Schema.org 仕様準拠検証(リッチリザルト無関係でも仕様違反検出)            |
| `site:example.com` 検索         | Google のインデックス済 URL 確認(手動 / 即時性高い)                       |

### 初期セットアップ手順

1. Search Console にプロパティ追加(DNS TXT or HTML meta タグで所有権確認)
2. `sitemap.xml` を Search Console に submit
3. インデックス済 URL 数 / 表示回数 / クリック数 / CWV を週次でレビュー
4. 検索クエリレポートで CTR の低いページの `title` / `description` を改善

---

## 10. アンチパターンと注意点

| アンチパターン                                       | リスク                                                  | 正しい対処                                      |
| ---------------------------------------------------- | ------------------------------------------------------- | ----------------------------------------------- |
| `robots.txt` の `Disallow` でインデックスを止める     | URL だけ拾われ「タイトル不明」状態でインデックス        | `<meta name="robots" content="noindex">` を使う |
| 全ページ同じ `title` / `description`                  | 評価分散 / SERP CTR 低下                                | ページ単位でユニーク化(`generateMetadata`)    |
| 重複コンテンツ(末尾 `/` 有無、`?utm_*`)             | 評価分散                                                | `canonical` + 必要なら 301 リダイレクト         |
| JSON-LD にユーザー入力をそのまま埋め込み              | `</script>` 注入で XSS                                  | `JSON.stringify` 結果を `<` → `<` 置換     |
| OG 画像が相対 URL                                     | SNS で表示崩れ                                          | 絶対 URL 必須(`metadataBase` で解決)          |
| クライアントサイドのみでメタが入る SPA                | クローラが空 HTML を見て内容不明                        | SSR / SSG / Next.js Metadata API                |
| `sitemap.xml` に `noindex` ページを混ぜる             | Search Console 警告                                     | 公開対象のみを sitemap に列挙                   |
| 大量の `noindex,nofollow` を全リンクに撒く            | 評価伝搬が止まる                                        | 必要なページに限定                              |
| `noindex` を効かせたいページを `robots.txt` でブロック | クローラが `noindex` を読めずインデックスが残り続ける    | 該当ページは**クロール許可** + `noindex` 併用    |
| 大量の薄いページ(自動生成のタグ一覧等)             | 低品質サイト判定で全体評価が下がる                      | パラメータ別ページは `canonical` で 1 つに集約  |

---

## 11. 参考リンク

- Google Search Central: https://developers.google.com/search/docs
- Schema.org: https://schema.org/
- web.dev (Core Web Vitals): https://web.dev/vitals/
- Sitemaps protocol: https://www.sitemaps.org/protocol.html
- robots.txt 仕様(RFC 9309): https://www.rfc-editor.org/rfc/rfc9309
- Next.js Metadata API: https://nextjs.org/docs/app/api-reference/file-conventions/metadata
