---
title: "YouTubeのメタデータ一括編集を「事故らない」CLIにした — ytmeta"
emoji: "🎬"
type: "tech"
topics: ["youtube", "cli", "nodejs", "oss", "googleapis"]
published: true
---

## 作ったもの

YouTubeの動画メタデータ（タイトル・概要欄・コメント・透かし・アップロード）を、
コマンドラインから一括管理できるツール **ytmeta** を作って公開しました。

- npm: https://www.npmjs.com/package/ytmeta
- GitHub: https://github.com/crazyathlete220-stack/ytmeta

```bash
npm install -g ytmeta
```

## なぜ作ったか

チャンネルの運用をしていると、こういう作業が地味に重い:

- 全動画の概要欄の末尾に、同じ導線（リンク・ハッシュタグ）を追加したい
- タイトルの命名ルールを変えて、まとめて付け直したい
- 高再生の動画にだけ、固定用のコメントを投稿したい

YouTube Studio は基本的に**1本ずつ**。本数が増えるほど手作業が溶けていきます。

かといって自分でスクリプトを書くと、今度は **怖さ** が出てきます。
`videos.update` を一括で回すコードは、1行間違えれば数十本の概要欄を一瞬で上書きします。**取り消しはできません。**

「一括でやりたい。でも事故りたくない」——これを両立させたかった、というのが動機です。

## 設計：dry-run ファースト

ytmeta は、**すべての書き込みコマンドがデフォルトで dry-run（プレビューのみ）** です。
実際に変更が走るのは `--execute` を付けたときだけ。

```bash
# まず何が変わるかを確認（API は叩くが、変更はしない）
ytmeta titles titles.json

# 内容に納得してから、実際に適用
ytmeta titles titles.json --execute
```

加えて、

- **アップロードはデフォルトで private**（誤って公開されない）
- **削除は確認フレーズの入力必須**（`--execute` を付けても、もう一段ガードがある）
- **認証情報はローカルのみ**。`client_secret.json` / `token.json` はリポジトリに含めず `.gitignore` 済み

## 使い方

### 認証（最初の1回）

Google Cloud Console で YouTube Data API v3 を有効化し、OAuthクライアント（Desktop app）の
JSONを `client_secret.json` として置いてから:

```bash
ytmeta login
```

ブラウザが開いて認可すると、`token.json` がローカルに保存されます（loopback フローなのでコード手貼り不要）。

### 概要欄に共通フッターを一括追加

```json
[
  { "id": "VIDEO_ID_1", "append": "\n—\nチャンネル登録: https://youtube.com/@you" },
  { "id": "VIDEO_ID_2", "append": "\n—\nチャンネル登録: https://youtube.com/@you" }
]
```

```bash
ytmeta descriptions data.json            # プレビュー
ytmeta descriptions data.json --execute  # 適用
```

`description`（全置換）/ `prepend` / `append` を動画ごとに選べます。

### その他のコマンド

| コマンド | 内容 |
|---|---|
| `ytmeta titles <file>` | タイトル一括更新 |
| `ytmeta comments <file>` | コメント投稿 |
| `ytmeta watermark <image> --channel <id>` | チャンネル透かし設定 |
| `ytmeta upload <file> --title ...` | 動画アップロード（private既定） |
| `ytmeta delete <id...>` | 動画削除（確認フレーズ必須） |

## 技術的なメモ

- Node.js 18+ / `googleapis` / `commander`
- テストは Node 標準の `node:test`（追加依存なし）。CI は GitHub Actions で Node 18/20/22
- コメントの「固定」は API では不可なので、投稿後に Studio で手動固定する想定（URLを出力）

## おわりに

「自動化したいけど事故が怖い」という、運用ツールの定番のジレンマに対する一つの答えとして作りました。
MIT ライセンスです。使ってみて要望やバグがあれば、Issue / PR を歓迎します。

⭐ https://github.com/crazyathlete220-stack/ytmeta
