# zenn-content

[Zenn](https://zenn.dev) の記事をGitHub連携で管理するリポジトリです。

## 構成

- `articles/` — 記事（1ファイル1記事）

## 公開のしかた

1. [Zenn](https://zenn.dev) に Google ログインでアカウント作成（ユーザー名: `ai_lifehackdays`）
2. [デプロイ設定](https://zenn.dev/dashboard/deploys) でこのリポジトリを連携
3. 公開したい記事の frontmatter を `published: true` に変更して push

## ローカルでプレビュー（任意）

```bash
npx zenn-cli preview
```

## 記事一覧

- `articles/ytmeta-youtube-cli.md` — ytmeta（YouTubeメタデータ一括管理CLI）の紹介。現在 `published: false`（下書き）
