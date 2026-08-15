# Zenn メモ

GitHub 連携で Zenn に記事を公開する手順をまとめます。

---

## 1. 仕組み

`articles/<slug>.md` を書いて push すると Zenn が自動で同期します。**push が公開操作**で、ダッシュボードでの公開操作は不要です。

- `articles/` `images/` はリポジトリのルート直下
- 連携できるリポジトリは 2 つまで、連携時に対象ブランチを指定する

## 2. セットアップ

```bash
npx zenn-cli@latest init   # articles/ books/ images/ を作る
```

初回のみ https://zenn.dev/dashboard/deploys で GitHub 連携する。

## 3. 記事を書く

```bash
npx zenn-cli@latest new:article   # 雛形を作る
npx zenn-cli@latest preview       # localhost:8000 で確認
npx zenn-cli@latest list:articles # frontmatter の検証
```

```yaml
---
title: 'タイトル'
emoji: '🧭' # 1 文字
type: 'tech' # tech または idea
topics: ['個人開発', 'aws'] # 最大 5 つ
published: false # true で公開
---
```

**ファイル名がそのまま slug**(`a-z0-9-_` の 12〜50 字)。URL は `zenn.dev/<ユーザー名>/articles/<slug>`。

## 4. 公開する

1. `published: true` に変更
2. 連携ブランチに push
3. https://zenn.dev/dashboard/deploys で結果を確認

修正は編集して push、非公開は `published: false` に戻して push、削除はファイルごと削除して push。
