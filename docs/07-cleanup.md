# 07. 後片付け

作ったリソースを削除して、課金が残らない状態に戻します。

`aws lambda-web deploy` が作ったものと、自動で作られたログを、作成と逆の順で消していきます。

| # | リソース | 名前 | 消さないとどうなるか |
|---|---|---|---|
| 1 | Web Function(リビジョン・エンドポイントを含む) | `nextjs-sample` | 公開 URL が残り続ける |
| 2 | CloudWatch Logs のロググループ | `/aws/lambda/web/nextjs-sample` など | ログの保存料金がかかる |
| 3 | IAM ロール | `awscli-lambdaweb-nextjs-sample` | 料金はかからないが、不要な権限が残る |
| 4 | S3 バケット | `awscli-lambdaweb-<アカウントID>-ap-northeast-1-an` | ZIP の保存料金がかかる |

> 作業を始める前に、02 と同じく学習用アカウントを向いているか確認してください。
>
> ```bash
> aws sts get-caller-identity
> echo "$AWS_PROFILE $AWS_REGION"
> ```

以降のコマンドでは、アカウント ID を変数に入れておきます。

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

## 1. Web Function を削除する

```bash
aws lambda-web delete-web-function --function-name nextjs-sample
aws lambda-web wait web-function-deleted --function-name nextjs-sample
```

`delete-web-function` は、関数に属するリビジョンとエンドポイントもまとめて削除します(ヘルプより)。個別に消す必要はありません。

確認します。

```bash
aws lambda-web list-web-functions
```

`nextjs-sample` が一覧から消えていれば OK です。05 で使った URL にアクセスしても、もう応答しないはずです。

## 2. ロググループを削除する

Web Functions のログは、関数を削除しても残ります。まず、何があるかを確認します。

```bash
aws logs describe-log-groups \
  --log-group-name-prefix /aws/lambda/web/nextjs-sample \
  --query 'logGroups[].logGroupName'
```

`/aws/lambda/web/nextjs-sample` と `/aws/lambda/web/nextjs-sample.local` の 2 つが表示されるはずです(違いは 06 の Tips を参照)。両方を削除します。

```bash
aws logs delete-log-group --log-group-name /aws/lambda/web/nextjs-sample
aws logs delete-log-group --log-group-name /aws/lambda/web/nextjs-sample.local
```

もう一度 `describe-log-groups` を実行して、空(`[]`)になれば OK です。

## 3. IAM ロールを削除する

IAM ロールは、付いているポリシーを先に外さないと削除できません。まず、何が付いているかを確認します。

```bash
ROLE=awscli-lambdaweb-nextjs-sample

# ロールに直接書き込まれたポリシー(インラインポリシー)
aws iam list-role-policies --role-name $ROLE

# ロールに付けられた管理ポリシー
aws iam list-attached-role-policies --role-name $ROLE
```

今回は、インラインポリシー `awscli-lambdaweb-logs`(`deploy` が作った、CloudWatch Logs にログを書く権限)が 1 つだけで、管理ポリシーはありませんでした。

表示されたものを外してから、ロールを削除します。

```bash
# インラインポリシーがあれば、表示された名前ごとに実行
aws iam delete-role-policy --role-name $ROLE --policy-name <ポリシー名>

# 管理ポリシーがあれば、表示された ARN ごとに実行
aws iam detach-role-policy --role-name $ROLE --policy-arn <ポリシーARN>

# ロールを削除
aws iam delete-role --role-name $ROLE
```

`aws iam get-role --role-name $ROLE` が `NoSuchEntity` エラーになれば OK です。

## 4. S3 バケットを削除する

### 他の関数が使っていないか確認する

このバケットは、同じリージョンで `deploy` を使うすべての Web Function が共有します。ほかに Web Function がないことを確認します。

```bash
aws lambda-web list-web-functions
```

一覧が空なら、消して問題ありません。ほかの関数が残っている場合は、バケットは消さずにこのステップを終えてください。

### バケットを空にする

バケットは、中身が空でないと削除できません。しかもバージョニングが有効なので、通常の削除(`aws s3 rm` や `aws s3 rb --force`)では古いバージョンが残ってしまいます。**すべてのバージョンを指定して削除する**必要があります。

```bash
BUCKET=awscli-lambdaweb-$ACCOUNT_ID-$AWS_REGION-an

# 残っているものを確認する
aws s3api list-object-versions --bucket $BUCKET \
  --query '{Versions: Versions[].[Key,VersionId], DeleteMarkers: DeleteMarkers[].[Key,VersionId]}'
```

| 種類 | 意味 |
|---|---|
| `Versions` | アップロードした ZIP の各バージョン |
| `DeleteMarkers` | 「削除した」という印。バージョニングが有効なバケットでは、普通に削除するとファイルは消えずにこの印が付く |

両方を削除します。

```bash
# ZIP の全バージョンを削除する
aws s3api delete-objects --bucket $BUCKET --delete "$(aws s3api list-object-versions --bucket $BUCKET \
  --query '{Objects: Versions[].{Key: Key, VersionId: VersionId}}' --output json)"

# 削除マーカーを削除する(なければエラーになるが問題ない)
aws s3api delete-objects --bucket $BUCKET --delete "$(aws s3api list-object-versions --bucket $BUCKET \
  --query '{Objects: DeleteMarkers[].{Key: Key, VersionId: VersionId}}' --output json)"
```

> 1 回の `delete-objects` で消せるのは 1,000 件までです。今回は数件なので問題ありません。
> コマンドが難しければ、AWS マネジメントコンソールの S3 画面でバケットを選び、「空にする」ボタンを使っても同じことができます。

### バケットを削除する

```bash
aws s3api delete-bucket --bucket $BUCKET
```

`aws s3api head-bucket --bucket $BUCKET` がエラー(`Not Found` など)になれば OK です。

## 5. 最終確認

```bash
aws lambda-web list-web-functions
aws logs describe-log-groups --log-group-name-prefix /aws/lambda/web/nextjs-sample --query 'logGroups[].logGroupName'
aws iam get-role --role-name awscli-lambdaweb-nextjs-sample
aws s3api head-bucket --bucket $BUCKET
```

| コマンド | 期待する結果 |
|---|---|
| `list-web-functions` | `nextjs-sample` がない |
| `describe-log-groups` | `[]` |
| `get-role` | `NoSuchEntity` エラー |
| `head-bucket` | `Not Found` などのエラー |

## 手元に残るもの

AWS 上のリソースはすべて消えましたが、手元のリポジトリの変更は残っています。必要に応じてコミットしておくと、あとで同じ手順をもう一度試せます。

| ファイル | 変更内容 | ステップ |
|---|---|---|
| `next.config.ts` | `output: "standalone"` | 03 |
| `package.json` | `build` スクリプトに静的ファイルのコピー | 03 |
| `pnpm-workspace.yaml` | `supportedArchitectures`、`nodeLinker: hoisted` | 04、05 |
| `app/page.tsx` | 見出しの文言 | 06 |
| `docs/` | このシリーズのドキュメント | - |

## このステップのチェックリスト

- [ ] Web Function を削除した
- [ ] ロググループを削除した
- [ ] IAM ロールを削除した
- [ ] S3 バケットを空にして削除した(ほかの Web Function が使っていない場合)
- [ ] 最終確認で、すべて消えていることを確認した

## 参考資料

- `aws lambda-web delete-web-function help`(リビジョンとエンドポイントもまとめて削除される)
- [Deleting a bucket(Amazon S3 ドキュメント)](https://docs.aws.amazon.com/AmazonS3/latest/userguide/delete-bucket.html)
