# 02. 事前準備

デプロイの前に、手元の環境を整えます。このステップでは AWS 上にリソースを作りません。

やること:

1. AWS CLI を更新し、`aws lambda-web` コマンドを使えるようにする
2. AWS にログインし、使うプロファイルとリージョンを決める
3. Web Functions の API を呼べることを確認する

## 1. AWS CLI を更新する

Web Functions の操作は `aws lambda-web` というコマンドで行います。このコマンドは新しいバージョンの AWS CLI にしか入っていません。

### 現状を確認する

```bash
aws --version
aws lambda-web help
```

01 を書いた時点では 2.36.7 で、`aws lambda-web help` は次のエラーになりました。

```text
aws: [ERROR]: An error occurred (ParamValidation): argument command: Found invalid choice 'lambda-web'
```

これは「機能がない」のではなく、「CLI が古い」という意味です。

### 更新する

手元の AWS CLI は公式インストーラーで `/usr/local/aws-cli` に入っています。AWS のドキュメントでは、公式インストーラーで入れた場合は次のコマンドで更新できるとされています。

```bash
aws update
```

`/usr/local` 以下への書き込みで権限エラーになる場合は、公式インストーラーを `--update` 付きで実行し直します。

```bash
cd /tmp
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip -q awscliv2.zip
sudo ./aws/install --bin-dir /usr/local/bin --install-dir /usr/local/aws-cli --update
```

### 確認する

```bash
aws --version
aws lambda-web help
```

`aws lambda-web help` で、`deploy` や `create-web-function` などのサブコマンド一覧が表示されれば完了です。

## 2. ログインとプロファイル・リージョンの設定

### プロファイルを決める

`~/.aws/config` には複数のプロファイルがあります。**本番環境のプロファイルは使わず、学習用(サンドボックス)の AWS アカウントのプロファイルを使ってください。**
以降の手順では、プロファイル名を `<profile>` と書きます。

### ログインする

プロファイルの種類に合わせて、どちらかを実行します。

```bash
# IAM Identity Center(SSO)のプロファイルの場合
aws sso login --profile <profile>

# それ以外(IAM ユーザーやルートユーザーでコンソールにサインインしている場合)
aws login --profile <profile>
```

どちらもブラウザが開き、サインインすると短時間で期限が切れる認証情報が手元に保存されます。アクセスキーをファイルに直接書く方式(`aws configure`)より安全です。

### このターミナルで使うプロファイルとリージョンを固定する

毎回 `--profile` や `--region` を付けなくて済むように、環境変数で指定します。

```bash
export AWS_PROFILE=<profile>
export AWS_REGION=ap-northeast-1
```

リージョンは東京(`ap-northeast-1`)を使います。Web Functions は 2026 年 10 月の提供開始時点で、次の 5 リージョンで使えます。

- 米国東部(バージニア北部)`us-east-1`
- 米国東部(オハイオ)`us-east-2`
- 米国西部(オレゴン)`us-west-2`
- アジアパシフィック(東京)`ap-northeast-1`
- 欧州(アイルランド)`eu-west-1`

> `export` はそのターミナルでだけ有効です。新しいターミナルを開いたら、もう一度実行してください。

## 3. 確認する

### ログインできているか

```bash
aws sts get-caller-identity
```

`Account`(アカウント ID)と `Arn`(誰としてログインしているか)が表示されます。**学習用アカウントの ID になっているか**を必ず確認してください。

### Web Functions の API を呼べるか

```bash
aws lambda-web list-web-functions
```

まだ何も作っていないので、空の一覧が返れば成功です。
`AccessDenied` などのエラーになった場合は、ログインしたユーザーやロールに Web Functions を操作する権限がありません。

### Node.js のバージョン

```bash
node --version
```

Web Functions の実行環境は Node.js 24 です。手元も 24 系(01 作成時点で v24.16.0)なので、ローカルでの動作確認と本番の差が出にくい状態です。

## 補足: 必要な権限

学習用アカウントで管理者権限を持っていれば、この節は読み飛ばして構いません。

05 で使う `aws lambda-web deploy` は、初回実行時に S3 バケットと IAM ロールを自動で作ります。そのため、ログインしたユーザーには次の権限が必要です。

| 用途 | 権限 |
|---|---|
| Web Functions の操作 | `lambda:CreateWebFunction` など、Web Functions 関連の操作 |
| 実行ロールを Lambda に渡す | `iam:PassRole` |
| アカウントの確認 | `sts:GetCallerIdentity` |
| コードのアップロード | `s3:ListBucket`、`s3:PutObject`、`s3:GetObject`、`s3:GetObjectVersion` |
| バケットとロールの自動作成 | `s3:CreateBucket`、`s3:PutBucketPolicy`、`s3:PutBucketVersioning`、`iam:GetRole`、`iam:CreateRole`、`iam:PutRolePolicy` |

## このステップのチェックリスト

- [ ] `aws lambda-web help` でサブコマンド一覧が表示される
- [ ] `aws sts get-caller-identity` で学習用アカウントの ID が表示される
- [ ] `aws lambda-web list-web-functions` が空の一覧を返す
- [ ] `AWS_PROFILE` と `AWS_REGION=ap-northeast-1` を設定した

## 参考資料

- [Creating a Lambda Web Function - Prerequisites(AWS ドキュメント)](https://docs.aws.amazon.com/lambda/latest/dg/web-functions-create.html)
- [AWS Lambda announces Lambda Web Functions(提供リージョンの記載あり)](https://aws.amazon.com/about-aws/whats-new/2026/10/aws-lambda-web-functions/)
- [Installing or updating the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [Sign in through the AWS CLI(`aws login`)](https://docs.aws.amazon.com/signin/latest/userguide/command-line-sign-in.html)
