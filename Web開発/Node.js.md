# Node.js メモ

Node.js ランタイム固有の標準ライブラリと挙動。

---

## Buffer

生のバイト列を扱う型(`Uint8Array` のサブクラス、Node.js 専用)。暗号化 / ファイル I/O / ネットワーク I/O で使う。

```ts
Buffer.from('hello', 'utf8')       // 文字列 → Buffer
Buffer.from('aGVsbG8=', 'base64')  // base64 → Buffer
Buffer.alloc(32)                   // 0 で埋めた 32 バイト
crypto.randomBytes(12)             // 暗号学的に安全な乱数

buf.toString('utf8' | 'base64url' | 'hex')  // バイト列 → 文字列
buf.subarray(start, end)                    // 部分参照(コピーなし)
buf.length                                  // バイト数(文字数ではない)
```

- `toString()` 引数なしは utf8。暗号文には必ず `'base64url'` / `'hex'` を明示。
- `subarray` は参照なので書き換えると元に伝播。独立コピーは `Buffer.from(buf.subarray(...))`。

---

## Buffer.concat

複数の Buffer を連結して新しい Buffer を返す。`+` 演算子や `Array.concat` は使えない。

```ts
Buffer.concat([a, b, c])  // 順に連結した新 Buffer
```
