# lambda-web-functions-nextjs-sample

Next.js アプリを AWS Lambda Web Functions にデプロイする学習用のサンプルです。
[`create-next-app`](https://nextjs.org/docs/app/api-reference/cli/create-next-app) で作ったアプリに、デプロイに必要な設定だけを加えています。

## 手順書

[`docs/`](docs/) に、ステップごとの手順と、つまずいた点の解説があります。最初から順に進めてください。

| # | ドキュメント | 内容 |
|---|---|---|
| 01 | [全体像と進め方](docs/01-overview.md) | Web Functions の概要と、Next.js の載せ方 |
| 02 | [事前準備](docs/02-preparation.md) | AWS CLI の更新、ログイン、リージョンの設定 |
| 03 | [standalone 出力](docs/03-standalone-build.md) | standalone 出力の設定と、ローカルでの起動確認 |
| 04 | [arm64 向けの準備](docs/04-arm64.md) | arm64 用の `sharp` を入れる |
| 05 | [デプロイ](docs/05-deploy.md) | `aws lambda-web deploy` でデプロイする |
| 06 | [更新・ロールバック・ログ](docs/06-update-and-logs.md) | 再デプロイ、リビジョンの切り替え、ログの見方 |
| 07 | [後片付け](docs/07-cleanup.md) | 作成したリソースの削除 |

## デプロイのための設定

| ファイル | 設定 | 理由 |
|---|---|---|
| `next.config.ts` | `output: "standalone"` | 本番実行に必要なファイルだけを `.next/standalone/` に集める |
| `package.json` | `build` で `public/` と `.next/static/` を standalone にコピー | standalone 出力には静的ファイルが含まれないため |
| `pnpm-workspace.yaml` | `supportedArchitectures` に `arm64` | Web Functions の実行環境は arm64 のため、`sharp` の arm64 用バイナリも入れる |
| `pnpm-workspace.yaml` | `nodeLinker: hoisted` | pnpm のシンボリックリンクはデプロイ先で失われるため、平らな配置にする |

## コマンド

```bash
# 開発サーバー
pnpm dev

# ビルド(.next/standalone/ に出力)
pnpm build

# デプロイ用の出力をローカルで起動
node .next/standalone/server.js

# デプロイ(初回は docs/05 のオプションも参照)
aws lambda-web deploy --name nextjs-sample --code .next/standalone
```

デプロイには、`aws lambda-web` コマンドが使える AWS CLI と、Node.js 24 が必要です。

> デプロイしたエンドポイントは公開(`ApplicationManaged`)です。学習が終わったら [07. 後片付け](docs/07-cleanup.md) でリソースを削除してください。
