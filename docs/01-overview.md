# 01. 全体像と進め方

このシリーズでは、AWS Lambda Web Functions に Next.js アプリをデプロイするまでを、手を動かしながら学びます。
ドキュメントは 1 ステップずつ追加します。コマンドは自分で実行し、結果を確認してから次に進みます。

## ゴール

- このリポジトリの Next.js アプリ(`create-next-app` の初期状態)を Web Functions にデプロイする
- ブラウザで HTTPS の URL を開き、トップページが表示されることを確認する
- 最後にリソースを削除し、課金が残らない状態に戻す

## Lambda Web Functions とは

Lambda の新しい種類の関数です。従来の Lambda(現在は **Event Functions** と呼ばれます)とは、コードの動かし方が根本的に違います。

| | Event Functions(従来の Lambda) | Web Functions |
|---|---|---|
| コードの形 | `handler(event)` をエクスポートする | **ポートで待ち受ける普通の HTTP サーバー** |
| 1 つの実行環境が同時に処理するリクエスト | 1 件 | 複数件(既定 64、最大 128) |
| HTTP の入口 | API Gateway や Function URL を別に用意する | **エンドポイント**(HTTPS の URL)が標準で付く |
| 課金(時間に関する部分) | 実行時間(壁時計の時間)× メモリ | **CPU を実際に使った時間**、消費したメモリ、リクエスト数、データ転送量 |
| ランタイム | 多数の言語に対応 | Node.js 24 のみ(arm64) |

ポイントは「Express や Next.js のサーバーを、ほぼそのまま Lambda で動かせる」ことです。
`serverless-express` のようなアダプター(イベントを HTTP に変換する部品)は不要です。

課金の特徴も押さえておきます。データベースや外部 API の応答を待っている間は CPU をほとんど使わないため、その待ち時間には CPU 料金がほぼかかりません。

### 3 つのリソース

Web Functions は次の 3 つのリソースで構成されます。

| リソース | 役割 |
|---|---|
| **Web Function** | 一番上の入れ物。名前と設定を持つ |
| **リビジョン(Revision)** | コードと設定のスナップショット。デプロイするたびに新しく作られ、後から変更できない。前のリビジョンに戻せばロールバックになる |
| **エンドポイント(Endpoint)** | HTTPS のドメインを持つ入口。認証方式と、どのリビジョンに通信を流すかを決める |

### 実行環境の約束ごと

アプリ側が守るべきルールは次の 3 つです。

1. **パッケージのルートにある `index.js` が起動される**(変更したい場合は環境変数 `AWS_LAMBDA_NODEJS_ENTRYPOINT` で指定する)
2. **環境変数 `AWS_LAMBDA_HTTP_ENDPOINT` に書かれたホストとポートで待ち受ける**(値は `0.0.0.0:3000` のような `host:port` 形式。`PORT` という環境変数は渡されない)
3. **コードは ZIP にして S3 に置く**(`aws lambda-web deploy` コマンドを使えば、ZIP 作成とアップロードを自動でやってくれる)

実行環境は 2 vCPU / 2 GB メモリ、CPU は arm64(Graviton)です。

## Next.js をどう載せるか

Next.js には `output: 'standalone'` というビルド設定があります。これを有効にすると、本番実行に必要なファイルだけが `.next/standalone/` に集められ、そこに `server.js` という小さなサーバーが出力されます。

この `server.js` を Web Functions で動かすには、2 つの食い違いを確認する必要があります。

**1. 起動するファイル名**

Web Functions は `index.js` を起動しますが、Next.js が出力するのは `server.js` です。
これは `aws lambda-web deploy` コマンドが吸収してくれます。このコマンドは `server.js` を自動で見つけ、それを読み込む小さな `index.js` をパッケージに追加します。

**2. 待ち受けるホストとポート**

`server.js` は環境変数 `PORT` と `HOSTNAME` を見て待ち受けます。Web Functions が渡すのは `AWS_LAMBDA_HTTP_ENDPOINT` です。
ただし既定値は一致しています。

| | 値 |
|---|---|
| `server.js` の既定値 | `PORT` 未設定なら `3000`、`HOSTNAME` 未設定なら `0.0.0.0` |
| `AWS_LAMBDA_HTTP_ENDPOINT` の値 | `0.0.0.0:3000` |

そのため、追加のコードなしで動きます。
心配だったのは `HOSTNAME` です。Linux では、この環境変数にマシン名が自動で入っていることがあり、その場合 Next.js はマシン名で待ち受けてしまいます。05 で指定なしでデプロイしたところ問題なく動いたので、Web Functions の実行環境は `HOSTNAME` を渡していないと分かりました。

ただし、これは既定値がたまたま一致しているだけです。本番で使うなら、`AWS_LAMBDA_HTTP_ENDPOINT` を `PORT` と `HOSTNAME` に読み替える数行の `index.js` を自分で書いておくほうが確実です(05 の補足を参照)。

```text
ブラウザ
  │ HTTPS
  ▼
Web Functions エンドポイント(https://xxxx.lambda-web.<region>.on.aws)
  │
  ▼
実行環境(Node.js 24 / arm64)
  └─ index.js           … deploy コマンドが自動で追加。server.js を読み込むだけ
       └─ server.js     … Next.js の standalone サーバー(SSR・静的ファイル配信)
```

## 事前に分かっている注意点

このリポジトリと手元の環境を確認したところ、次の点で対応が必要になりそうです。それぞれ該当するステップで扱います。

| 注意点 | 内容 | 扱うステップ |
|---|---|---|
| AWS CLI が古い | 手元の AWS CLI 2.36.7 では `aws lambda-web` コマンドが使えなかった。更新が必要 | 02 |
| CPU アーキテクチャの違い | 手元(WSL)は x86_64、実行環境は arm64。画像最適化に使う `sharp` は CPU ごとに別のバイナリなので、x86_64 版のままだと実行環境で読み込めない。今のトップページの画像は SVG なので `sharp` は使われないが、デプロイ先に合わせて arm64 用も入れておく | 04 |
| 静的ファイルのコピー | standalone 出力には `public/` と `.next/static/` が含まれない(静的ファイルは CDN から配信する前提のため。Next.js 12 から変わらない仕様)。ビルドスクリプトにコピー処理を書いておく | 03 |
| pnpm を使っている | 既定の配置ではパッケージの実体が `node_modules/.pnpm/` にあり、ほかはシンボリックリンクになる。デプロイ先でリンクが失われ `Cannot find module 'next'` になったので、`nodeLinker: hoisted`(npm と同じ平らな配置)に切り替える | 05 |
| エンドポイントは既定で公開 | 認証方式を `ApplicationManaged`(誰でもアクセスできる)と `IamAuth`(IAM 署名が必要)から選ぶ。今回は `ApplicationManaged` にして、学習後すぐ削除する | 05 |

## 進め方(予定)

| # | ドキュメント | やること |
|---|---|---|
| 01 | 全体像と進め方(このページ) | 概要をつかむ |
| 02 | 事前準備 | AWS CLI の更新、認証情報とリージョンの設定、`aws lambda-web` が使えるか確認 |
| 03 | Next.js を standalone 出力にする | `next.config.ts` とビルドスクリプトの変更、ローカルで起動確認 |
| 04 | arm64 向けの準備 | `sharp` など、CPU ごとに異なる依存パッケージを arm64 用もインストールする |
| 05 | デプロイ | `aws lambda-web deploy` で初回デプロイ、URL にアクセスして確認 |
| 06 | 更新とログ | コードを変えて再デプロイ、リビジョンの確認、CloudWatch Logs でのログ確認 |
| 07 | 後片付け | 作成したリソースの削除 |

途中で分かったことに応じて、ステップを増やしたり順番を変えたりします。

## 用語メモ

| 用語 | 意味 |
|---|---|
| standalone 出力 | Next.js の本番用ビルド形式の 1 つ。必要なファイルだけを集め、`node_modules` ごと持ち運べる形にする |
| arm64 / Graviton | AWS が提供する ARM 系 CPU。Web Functions の実行環境はこれで動く |
| SigV4 | AWS の認証情報でリクエストに署名する方式。`IamAuth` のエンドポイントを呼ぶときに必要 |

## 参考資料

- [Creating a Lambda Web Function(AWS ドキュメント)](https://docs.aws.amazon.com/lambda/latest/dg/web-functions-create.html)
- [Web Function deployment package(AWS ドキュメント)](https://docs.aws.amazon.com/lambda/latest/dg/web-functions-configuration-deployment-package.html)
- [Exploring AWS Lambda Web Functions for building web applications(AWS Compute Blog)](https://aws.amazon.com/blogs/compute/exploring-aws-lambda-web-functions-for-building-web-applications/)
- Next.js の `output` 設定: `node_modules/next/dist/docs/01-app/03-api-reference/05-config/01-next-config-js/output.md`
