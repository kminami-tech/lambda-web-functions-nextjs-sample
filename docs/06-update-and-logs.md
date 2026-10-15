# 06. 更新・ロールバック・ログ

デプロイしたアプリを更新し、リビジョンのしくみとログの見方を学びます。

やること:

1. コードを変えて再デプロイし、新しいリビジョンができることを確認する
2. 前のリビジョンに戻す(ロールバック)
3. 自動デプロイに戻す
4. ログを見る

## 1. コードを変えて再デプロイする

### コードを変える

`app/page.tsx` の見出しの文言を変えます。

```tsx
            To get started, edit the{" "}
```

を、次のように変えます。

```tsx
            Hello from Lambda Web Functions! Edit the{" "}
```

### ビルドしてデプロイする

```bash
pnpm build
aws lambda-web deploy \
  --name nextjs-sample \
  --code .next/standalone \
  --revision-description "Change heading"
```

2 回目以降は、`--auth-type` や `--max-environments` を付けなくても、エンドポイントの設定はそのまま引き継がれます。
`--revision-description` は、リビジョンに付けるメモです。あとで一覧を見たときに、どの変更か分かりやすくなります。

### 確認する

```bash
URL=$(aws lambda-web get-web-function-endpoint \
  --function-name nextjs-sample --endpoint-name dev \
  --query domainName --output text)
curl -s "https://$URL/" | grep -o "Hello from Lambda Web Functions"
```

文言が表示されれば、新しいリビジョンに切り替わっています。ブラウザでも確認してください。

### リビジョンを見る

```bash
aws lambda-web list-web-function-revisions --function-name nextjs-sample
```

リビジョンが増えているはずです。05 でのデプロイ(失敗した最初のものを含む)と、今回のデプロイの分があります。
**それぞれの `revisionId`(`rev-xxxxx` の形)と `description` をメモしてください。** 次のロールバックで使います。

エンドポイントが今どのリビジョンに通信を流しているかも確認します。

```bash
aws lambda-web get-web-function-endpoint --function-name nextjs-sample --endpoint-name dev
```

### しくみ

```text
Web Function: nextjs-sample
├── リビジョン rev-aaaaa  … 05 の最初のデプロイ(Cannot find module 'next' で起動失敗)
├── リビジョン rev-bbbbb  … 05 で hoisted にして成功したもの
└── リビジョン rev-ccccc  … 今回(見出しを変更)
        ▲
        │ 100%
エンドポイント dev(autoDeploymentMode: LatestRevision)
```

- リビジョンは、作ったあとに変更できない(コードも設定も)
- `LatestRevision` モードのエンドポイントは、新しいリビジョンが `Active` になると、自動ですべての通信をそちらへ流す
- **起動に成功したかどうかは見ていない。** 05 の最初のデプロイで、起動に失敗したリビジョンにも通信が流れたのはこのため

## 2. ロールバックする

前のリビジョン(見出しを変える前)に戻します。

`LatestRevision` モードのままでは、常に最新のリビジョンに通信が流れます。前のリビジョンに戻すには、エンドポイントを `Disabled` モードにして、どのリビジョンに何 % 流すかを自分で指定します。

`<前のリビジョンID>` には、手順 1 でメモした「05 で hoisted にして成功したもの」の `revisionId` を入れます。

```bash
aws lambda-web update-web-function-endpoint \
  --function-name nextjs-sample \
  --endpoint-name dev \
  --auto-deployment-mode Disabled \
  --revision-weights revisionId=<前のリビジョンID>,weight=100
```

| オプション | 意味 |
|---|---|
| `--auto-deployment-mode Disabled` | 新しいリビジョンができても、自動では切り替えない |
| `--revision-weights revisionId=...,weight=100` | 指定したリビジョンに 100% の通信を流す |

エンドポイントの設定変更は、新しいリビジョンを作らずにその場で反映されます(リビジョンに属するのはコードと実行時の設定、エンドポイントに属するのは通信の振り分けや認証の設定、という分担です)。

反映を待ってから確認します。

```bash
aws lambda-web wait web-function-endpoint-updated \
  --function-name nextjs-sample --endpoint-name dev
curl -s "https://$URL/" | grep -o "To get started"
```

元の文言(`To get started`)に戻っていればロールバック成功です。

> `--revision-weights` には 2 つのリビジョンを指定することもできます(合計 100%)。例えば新しいリビジョンに 10%、古いリビジョンに 90% だけ流して様子を見る「カナリアリリース」ができます。

## 3. 自動デプロイに戻す

学習を続けやすいように、`LatestRevision` モードに戻しておきます。

```bash
aws lambda-web update-web-function-endpoint \
  --function-name nextjs-sample \
  --endpoint-name dev \
  --auto-deployment-mode LatestRevision
```

```bash
aws lambda-web wait web-function-endpoint-updated \
  --function-name nextjs-sample --endpoint-name dev
curl -s "https://$URL/" | grep -o "Hello from Lambda Web Functions"
```

最新のリビジョン(見出しを変えたもの)に戻っていれば OK です。
エラーになった場合は、その出力を共有してください。

## 4. ログを見る

### リアルタイムで見る

```bash
aws logs tail /aws/lambda/web/nextjs-sample --follow --format short
```

別のターミナルやブラウザからアクセスしながら眺めてみてください。止めるときは `Ctrl+C` です。

### ログの種類

05 のトラブル対応で見たように、ログには Web Functions 自体が出すもの(`type` が `platform.` で始まる)と、アプリが出すものがあります。

| `type` | いつ出るか | 見どころ |
|---|---|---|
| `platform.initStart` | 実行環境の起動開始 | どのリビジョン(`revisionId`)の環境か |
| `platform.initReport` | 実行環境の起動完了 | `status` が `success` か `failure` か。05 で最初に見るべきだったもの |
| `platform.invokeSummary` | 実行環境ごとに 1 分おき | その 1 分間のリクエスト数、同時に処理した最大数、使った CPU 時間 |
| `platform.spinDownStart` / `platform.spinDownReport` | 実行環境の停止 | 環境が生きていた時間 |
| (`type` なし) | アプリの `console.log` などの出力 | Next.js の起動メッセージなど |

### 集計してみる

`platform.invokeSummary` を CloudWatch Logs Insights で集計すると、過去 30 分に何個の実行環境が使われ、何件のリクエストを処理し、CPU を何秒使ったかが分かります。

```bash
QID=$(aws logs start-query \
  --log-group-name /aws/lambda/web/nextjs-sample \
  --start-time $(($(date +%s)-1800)) --end-time $(date +%s) \
  --query-string 'filter type="platform.invokeSummary" | stats count_distinct(record.instanceId) as environments, sum(record.metrics.requests) as totalRequests, max(record.metrics.concurrentRequests) as peakConcurrency, sum(record.metrics.cpuTimeMs)/1000 as totalCpuSeconds' \
  --query queryId --output text) && sleep 6 && aws logs get-query-results --query-id "$QID"
```

| 項目 | 意味 |
|---|---|
| `environments` | 使われた実行環境の数 |
| `totalRequests` | 処理したリクエストの合計 |
| `peakConcurrency` | 1 つの実行環境で同時に処理したリクエストの最大数 |
| `totalCpuSeconds` | 使った CPU 時間の合計(秒)。Web Functions の CPU の課金はこの時間に対してかかる |

`totalCpuSeconds` は、ページを何度か開いた程度なら 1 秒にも満たないはずです。実行環境が起きている時間ではなく、CPU を実際に使った時間で課金されることを数字で確認できます。

## Tips

### `--revision-weights` の使い道

エンドポイントに来た通信を、どのリビジョンにどれだけ流すかを決める設定です。指定できるリビジョンは最大 2 つで、割合の合計は 100% にします。`Disabled` モードのときだけ使えます。

| 使い方 | 設定の例 | 目的 |
|---|---|---|
| 固定・ロールバック | 前のリビジョンに 100% | このステップの手順 2 でやったこと |
| カナリアリリース | 新 10% / 旧 90% → 新 50% / 旧 50% → 新 100% | 一部の利用者だけに新しい版を見せ、問題がなければ少しずつ増やす |
| Blue/Green デプロイ | 旧 100% → 新 100% に一気に切り替える | 新しい版を用意して確認してから切り替え、問題があれば旧に戻す |

**Next.js で割合を分けるときの注意(バージョンスキュー)**
2 つのリビジョンに通信を分けると、同じ利用者のリクエストでも 1 回ごとに別のリビジョンに届くことがあります。
Next.js はビルドのたびに `/_next/static/` の JavaScript のファイル名が変わります。そのため、旧リビジョンから受け取った HTML が旧ビルドの JavaScript を読みに行き、そのリクエストが新リビジョンに届くと、ファイルがなく 404 になることがあります。
100% ずつ切り替える使い方ならほぼ影響はありません。割合を分ける場合の対策は、Next.js のドキュメント(`node_modules/next/dist/docs/01-app/02-guides/self-hosting.md` のバージョンスキューの節)を参照してください。

### `--revision-description` にコミットハッシュを入れる

リビジョンの一覧を見たときに、どのコードがデプロイされているかを Git の履歴と照らし合わせられます。手順 2 のロールバックで、どれが前の版かを探しやすくなります。

```bash
aws lambda-web deploy \
  --name nextjs-sample \
  --code .next/standalone \
  --revision-description "$(git rev-parse --short HEAD) $(git log -1 --pretty=%s)"
```

- 未コミットの変更があるとハッシュとコードが一致しないので、手元からデプロイするときはコミットしてから実行する
- CD では、CI が用意する環境変数(GitHub Actions なら `GITHUB_SHA`)を使う
- 説明の最大文字数は確認していない。長いメッセージでエラーになったら、ハッシュだけにする

### ロググループは 2 つある(`.local` 付きと、なし)

CloudWatch Logs を見ると、`/aws/lambda/web/nextjs-sample` と `/aws/lambda/web/nextjs-sample.local` の 2 つのロググループがあります。Web Functions は、ログを 2 段階で集めているためです。

```text
リクエストを処理したリージョン            ホームリージョン(今回は東京)
/aws/lambda/web/nextjs-sample.local  ──集約──▶  /aws/lambda/web/nextjs-sample
(そのリージョンのログだけ。すぐ書かれる)        (全リージョンのログをまとめたもの)
```

| | `.local` | 末尾なし(集約先) |
|---|---|---|
| 置かれる場所 | リクエストを処理したすべてのリージョン(ホームリージョンを含む) | ホームリージョンだけ |
| 中身 | そのリージョンのログだけ | 全リージョンのログ |
| 書かれる早さ | すぐ | `.local` から集約されるので、少し遅れる |
| 向いている用途 | 特定のリージョンで起きた問題の調査 | アプリ全体を対象にした検索、アラーム、メトリクス |

運用するときの注意:

- 集約したあとも `.local` のログは消えない。同じログが 2 か所に保存され、それぞれに保存料金がかかる
- どちらのロググループも、**保存期間が無期限**で作られる。運用では両方に保存期間を設定しておく(`aws logs put-retention-policy --log-group-name <名前> --retention-in-days 30` など)
- ロググループは、関数が最初のリクエストを処理したときに作られる。関数を削除しても残るので、07 で別に削除する

### デプロイするたびに S3 の保存料金が少しずつ増える

- `deploy` は実行するたびに、コードの ZIP を S3 バケット(`awscli-lambdaweb-<アカウントID>-<リージョン>-an`)にアップロードする
- バケットはバージョニングが有効なので、上書きしても古い ZIP は「以前のバージョン」として残る
- Lambda がリビジョンを削除するときに S3 の ZIP まで消すとは、ドキュメントに書かれていない。残り続けると考えておく

保存されている ZIP は次のコマンドで確認できます。

```bash
aws s3api list-object-versions \
  --bucket awscli-lambdaweb-<アカウントID>-ap-northeast-1-an \
  --query 'Versions[].[Key,Size,LastModified]' --output table
```

**ただし、古い ZIP を気軽に消してはいけません。** 使用中のリビジョンが参照している ZIP のバージョンを消すと、そのリビジョンは `Failed` 状態になり、通信を処理できなくなります(HTTP 409 が返る)。`Failed` は元に戻せず、ZIP を復元しても `Active` には戻りません。S3 のライフサイクルルールで古いバージョンを自動削除するときも、この点に注意が必要です。

## このステップのチェックリスト

- [ ] 見出しを変えて再デプロイし、新しいリビジョンに切り替わった
- [ ] `Disabled` モードと `--revision-weights` で前のリビジョンに戻せた
- [ ] `LatestRevision` モードに戻した
- [ ] ログで `platform.initReport` や `platform.invokeSummary` を確認した

## 参考資料

- [Web Function logs in Amazon CloudWatch Logs(AWS ドキュメント)](https://docs.aws.amazon.com/lambda/latest/dg/web-functions-logging.html)
- [Deploying Web Function code from Amazon S3 - Object lifecycle(AWS ドキュメント)](https://docs.aws.amazon.com/lambda/latest/dg/web-functions-code-s3.html)
- `aws lambda-web update-web-function-endpoint help`(`--auto-deployment-mode` と `--revision-weights` の説明)
- [Exploring AWS Lambda Web Functions for building web applications](https://aws.amazon.com/blogs/compute/exploring-aws-lambda-web-functions-for-building-web-applications/)(`platform.invokeSummary` の集計クエリ)
