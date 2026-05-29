# Anthropic API メモ

Claude を呼ぶ HTTP API。公式 SDK `@anthropic-ai/sdk` で Messages API を叩く。構造化出力が欲しいときは Tool Use を使う。

---

## 1. SDK とクライアント

- `new Anthropic({ apiKey })` で生成。Claude REST API への HTTP クライアント
- API キーは `sk-ant-...`(サーバー側のみ。漏洩厳禁)。**従量課金**(サブスクではない。Claude Code / Claude.ai のサブスクとは別物)
- `apiVersion` 相当の概念はあるが、SDK が適切なバージョンを付ける。明示が必要な場面は少ない

---

## 2. Messages API(`client.messages.create`)

| パラメータ | 意味 |
| --- | --- |
| `model` | モデル ID(`claude-sonnet-4-x` / `claude-haiku-4-x` 等) |
| `max_tokens` | **必須**。生成する最大トークン数 |
| `system` | システムプロンプト(役割・制約を指示する文字列) |
| `messages` | `{ role: 'user' \| 'assistant', content: string \| ContentBlock[] }[]`。会話履歴 |
| `tools` | ツール定義の配列(下記) |
| `tool_choice` | ツール呼び出しの強制度(下記) |
| `temperature` | ランダム度(0〜1。低いほど決定的) |
| `stop_sequences` | 出現したら生成を止める文字列の配列 |

- `messages` は `user` / `assistant` を交互に並べる。ツール結果を返すときは `role: 'user'` で `{ type: 'tool_result', tool_use_id, content }` ブロックを入れる
- モデルの使い分けは「品質重視 = 上位モデル / 構造化中心・大量処理 = 軽量モデル」が定石

---

## 3. Tool Use(構造化出力 / 外部アクション)

- `tools: [{ name, description, input_schema: <JSON Schema> }]` でツールを定義(`input_schema` は `{ type: 'object', properties: {...}, required: [...] }` 形式の JSON Schema)
- モデルがツールを使うと、レスポンスに `type: 'tool_use'` のコンテンツブロック(`id` / `name` / `input` = モデルが作った JSON)が含まれ、`stop_reason: 'tool_use'` になる
- `tool_choice`:
  | 値 | 挙動 |
  | --- | --- |
  | `{ type: 'auto' }`(既定) | モデルが使うかどうか判断 |
  | `{ type: 'any' }` | 何かしらツールを必ず使う |
  | `{ type: 'tool', name: 'X' }` | ツール X を必ず使う |
- **`{ type: 'tool', name: 'X' }` で呼び出しを強制すると、モデルは自由文を返せず必ずツール入力として JSON を返す** → 「Claude から確実に構造化 JSON を取り出す」定石(テキストブロックに JSON を書かせて `JSON.parse` を祈る、より堅い)
- `block.input` は SDK 上 **`unknown` 型**(モデルが任意の JSON を作るので)→ 受け取り側でランタイム検証する(`typeof x.field === 'string'` 等)
- 実際にツールを「実行」したい(関数呼び出し)場合は、`tool_use` を受けて自前で処理 → 結果を `tool_result` ブロックで返す → モデルが続きを生成、という往復になる

---

## 4. レスポンスの形

| フィールド | 内容 |
| --- | --- |
| `content` | `ContentBlock[]`。`text` ブロックや `tool_use` ブロックの配列(`type` で判別可能ユニオン) |
| `stop_reason` | `'end_turn'` / `'tool_use'` / `'max_tokens'` / `'stop_sequence'` |
| `usage` | `{ input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens }` — **コスト/利用上限の記録に使う** |
| `model` / `id` | 使われたモデル ID / メッセージ ID |
| `role` | 常に `'assistant'` |

- `tool_use` ブロックを取り出すには `res.content.find((b) => b.type === 'tool_use')` → `type` の絞り込みで narrow
- トークン課金は `usage` ベース。Prompt Caching を使うと `cache_read_input_tokens` 分が割安になる

---

## 5. その他

- `client.messages.stream(...)` — ストリーミング(イベントを逐次受信)
- `client.messages.countTokens(...)` — 送信前にトークン数を計算
- エラー型: `APIError` / `AuthenticationError`(401)/ `RateLimitError`(429)/ `BadRequestError`(400)/ `InternalServerError`(5xx)等。SDK は一部リトライを内蔵
- 埋め込み(embedding)API は **無い** — RAG 等で必要なら OpenAI / Voyage AI 等を別途使う

---

## 6. Tool Use を使う / 使わないの判断

Tool Use は「構造化出力(複数フィールド / 配列)が必要なときだけ」使う。出力そのものが人間が読む文章なら自由文応答にする(公式も、構造化が不要なら Tool Use を強制しないことを推奨)。

| | Tool Use なし(自由文) | Tool Use あり(`tool_choice` で強制) |
| --- | --- | --- |
| レスポンス | `text` ブロック(文字列) | `tool_use` ブロック(JSON Schema 準拠) |
| 向く用途 | 会話・説明・要約など「読む文章」が成果物 | title+body の分割、配列、型付きデータ |
| メリット | 自然な長文、シンプル、レイテンシ軽い | スキーマ準拠保証、パース不要、required 欠落を防ぐ |
| デメリット | 形式が揺れて構造化データを確実に取れない | 制約のぶん長文説明の質がやや落ちる、レイテンシ増 |

**具体例**: README 生成は title と content を別フィールドで欲しいので Tool Use あり。AI チャット(壁打ち)の回答は Markdown 文章そのものが成果物なので Tool Use なし。後者で無理に Tool Use を使うと「回答をツールの content フィールドに詰める」余計な制約が入り、自然な長文の質が落ちやすい。
