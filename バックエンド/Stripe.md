# Stripe メモ

決済処理(特にサブスクリプション課金)の SaaS。クレカ情報を自前で扱わず Stripe にホストさせる。バックエンドからは公式 SDK で REST API を叩き、課金状態の変化は Webhook で通知を受けて自前 DB にミラーする。

---

## 1. 全体像(B2B SaaS のサブスク課金)

```
自前アプリ ──(SDK / REST API)──> Stripe        Customer / Checkout Session / Subscription を作る・読む
Stripe ──(Webhook = HTTP POST)──> 自前アプリ    課金状態の変化(支払い完了・更新・解約・失敗)を通知
```

- 「真実の源」は Stripe 側。自前 DB はそのミラー。Webhook ハンドラで Stripe オブジェクトの内容を**上書き**する(差分を積む発想ではない)
- Webhook は**リアルタイムではない**(配信遅延・再送・順序入れ替わりがある前提)→ 冪等に作る

---

## 2. Stripe SDK(stripe-node)

- `new Stripe(secretKey)` で生成。**REST API の HTTP クライアント + Webhook 署名検証ヘルパー**を兼ねる
- 主な名前空間: `customers.*` / `checkout.sessions.*` / `subscriptions.*` / `prices.*` / `products.*` / `invoices.*` / `webhooks.constructEvent`(これだけ API 通信せずローカル計算)
- シークレットキー(`sk_test_` / `sk_live_`)はサーバー側のみ。漏洩厳禁
- `apiVersion` を省略するとアカウントの既定 API バージョンに追従(固定したいなら明示指定)

---

## 3. Checkout(ホスト型決済ページ)

- 自前でクレカ入力フォームを作る(PCI DSS 準拠が重い)代わりに、Stripe がホストする決済ページにリダイレクトする
- 流れ: API で **Checkout Session** を作成 → 返り値 `session.url` にユーザーをリダイレクト → Stripe 上で決済 → `success_url` / `cancel_url` に戻る
- Checkout Session の主なフィールド:

| フィールド | 意味 |
| --- | --- |
| `mode` | `payment`(一回払い)/ `subscription`(継続課金)/ `setup`(カード登録だけ) |
| `line_items` | 買うもの = `price`(Price ID)+ `quantity` |
| `customer` | 誰が(Stripe Customer ID) |
| `metadata` / `subscription_data.metadata` | 任意の自前データ。後で Webhook 側で読める |
| `success_url` / `cancel_url` | 完了 / キャンセル時のリダイレクト先 URL |

- Session は短命(既定 24h で失効)。完了すると `checkout.session.completed` イベントが発火する

---

## 4. Webhook(Stripe → 自前アプリ)

- Stripe 側に**エンドポイント URL を登録**しておくと、選んだイベントを Stripe がその URL に `POST` する(JSON ボディ + `Stripe-Signature` ヘッダー)
  - 本番: Dashboard → Developers → Webhooks → Add endpoint で URL 登録 → 署名シークレット `whsec_...` を取得
  - ローカル: `stripe listen --forward-to localhost:PORT/path` が Stripe と WebSocket を張ってイベントを受け取り、ローカルへ `POST` で中継。`stripe trigger <event>` でイベントを発火できる
- URL パスは自由(`/webhooks/stripe` でも何でも)。Stripe 仕様で決まっているのは「POST / JSON ボディ / `Stripe-Signature` ヘッダー付き」だけ

### 署名検証(`webhooks.constructEvent`)

`Stripe-Signature` ヘッダー: `t=<Unix秒>,v1=<HMAC-SHA256>`(`t` = 送信時刻、`v1` = 署名)

`constructEvent(rawBody, signatureHeader, secret)` がやること:

1. ヘッダーから `t` と `v1` を取り出す
2. 自分でも `HMAC-SHA256(secret, "{t}.{rawBody}")` を計算
3. `v1` と定数時間比較 → 不一致なら throw(改ざん or 鍵違い)
4. `t` が今から ±5 分(既定 tolerance)以内か → 古すぎたら throw(**リプレイ攻撃対策**)
5. OK なら `JSON.parse(rawBody)` して型付き `Event` を返す

- **必ず raw body(未加工バイト列)を渡す**。`JSON.parse → JSON.stringify` するとキー順・空白・数値表記が変わって HMAC がズレる。Web フレームワーク側で「JSON パース前のボディ」を保持する設定が要る(例: NestJS は `NestFactory.create(App, { rawBody: true })` で `req.rawBody` に Buffer が残る)

### ステータスコード = Stripe の再送制御

| 返すコード | Stripe の挙動 | いつ返すか |
| --- | --- | --- |
| 2xx | 受領完了、再送しない | 正常処理(処理スキップ含む) |
| 4xx | 「直しても無駄」とみなし**再送しない** | 署名不正・ヘッダー欠落・不正なペイロード |
| 5xx / タイムアウト | 一時障害かもとみなし**指数バックオフで再送** | DB エラーなど一時的な失敗 |

→ 「処理は失敗したがリトライで直るかも」なら 5xx を返す(= 例外を投げる)、「リトライしても無駄」なら 4xx。

### 冪等性(Idempotency)

- Stripe は同じイベント(`event.id` = `evt_...`)を**何度でも送ってくる**前提
- 自前で「処理済みイベント」テーブルを持ち、**`event.id` に UNIQUE 制約**を張る。受信時に「既に処理済みなら何もしない」→ 処理 → 処理済み記録(upsert)。ロジックではなく**DB 制約で構造的に**二重処理を防ぐ
- Stripe API を**呼ぶ**側の冪等性は別途 `Idempotency-Key` ヘッダー(リクエスト単位)もある

---

## 5. 主要なイベント(サブスク課金で扱うもの)

| イベント | 意味 | `event.data.object` |
| --- | --- | --- |
| `checkout.session.completed` | Checkout が成功完了(subscription モードなら Subscription が作られた) | Checkout Session |
| `customer.subscription.created` | Subscription が新規作成 | Subscription |
| `customer.subscription.updated` | Subscription が変化(プラン・数量・status: trialing→active / past_due→active 等) | Subscription |
| `customer.subscription.deleted` | Subscription が解約 / 終了 | Subscription |
| `invoice.paid` | 請求書(初回 or 月次更新)の支払い成功 | Invoice |
| `invoice.payment_failed` | 請求書の支払い失敗(カード拒否等) | Invoice |

- Stripe には ~200 種のイベントがある。`event.type`(`"invoice.paid"` 等の文字列)で分岐し、知らない型は無視(ログだけ)するのが基本
- `customer.subscription.*` の `data.object`(Subscription)から読むもの: `status`(`active` / `past_due` / `canceled` / `incomplete` / `trialing` …)、`items.data[0].price.id`(どの Price = どのプランか)、`items.data[0].current_period_end`(次回更新日。新しめの API では Subscription 直下ではなく item 側にある)、`canceled_at`、`metadata`、`customer`

---

## 6. データモデルの対応(B2B SaaS の場合)

- **Stripe Customer ⟷ テナント(組織 / ワークスペース)を 1:1** にするのが定石(ユーザー個人単位ではない)。Customer の `metadata.tenantId` に自前テナント ID を入れておくと、Webhook で来た Subscription / Invoice からテナントを逆引きできる
- Free プランでも先に Stripe Customer を作っておくと、後で有料へ切り替えるとき速い
- 人数課金(seat-based)は Subscription の `quantity` で表現。メンバー増減のたびに Quantity を更新、按分(proration)は Stripe デフォルトに任せる
- Customer の `email` / `name` は請求書の表示用。誰の名前を入れるか(オーナー等)はアプリ側の都合で決める

---

## 7. テスト

- テストモード(`sk_test_` / `pk_test_`)では事業者情報・本人確認なしで使える
- テストカード: `4242 4242 4242 4242`(成功)、`4000 0000 0000 0002`(拒否)など。有効期限は未来の任意の月/年、CVC は任意 3 桁
- `stripe trigger <event>` は「そのイベントが現実的に出るための前提オブジェクト(fixture)を順に作って」からイベントを発火する。汎用のテストオブジェクトなので自前 `metadata` は付かない点に注意(自前メタデータ込みの動作を見たいなら実際に Checkout を通す)
- `stripe events resend <evt_id>` で過去イベントを再配信できる → 冪等性の確認に使える
