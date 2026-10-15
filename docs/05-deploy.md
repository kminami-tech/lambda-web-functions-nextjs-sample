# 05. デプロイ

いよいよ AWS にデプロイします。このステップから AWS 上にリソースが作られ、課金対象になります。

やること:

1. デプロイの設定を確認する
2. pnpm の `node_modules` からシンボリックリンクをなくす
3. `aws lambda-web deploy` でデプロイする
4. URL にアクセスして動作を確認する
5. 作られたリソースを見てみる

## 1. デプロイの設定

| 項目 | 値 | 説明 |
|---|---|---|
| 関数名 | `nextjs-sample` | アカウント内の全リージョンで一意である必要がある |
| リージョン | `ap-northeast-1`(東京) | 02 で設定した `AWS_REGION` |
| アップロードするディレクトリ | `.next/standalone` | 03・04 で作った standalone 出力 |
| エンドポイント名 | `dev` | `deploy` コマンドの既定値 |
| エンドポイントの種類 | `HomeRegion` | 1 つのリージョンだけで動かす。一番単純な構成 |
| 認証方式 | `ApplicationManaged` | **URL を知っていれば誰でもアクセスできる(公開)** |
| 自動デプロイ | `LatestRevision` | 新しいリビジョンができたら、すぐにすべての通信をそちらへ流す |
| 実行環境の上限 | 2 | 公開エンドポイントなので、想定外のアクセスで費用が膨らまないように上限を付ける(設定できる最小値が 2) |

> **公開エンドポイントについて**
> AWS 側の認証も WAF(不正アクセスを防ぐファイアウォール)もありません。URL は秘密になりません。
> 今回は Next.js の初期ページなので漏れて困る情報はありませんが、学習が終わったら 07 で必ず削除します。

### `deploy` コマンドが自動で作るもの

初回は、次のリソースが作られます。

| リソース | 名前 | 役割 |
|---|---|---|
| S3 バケット | `awscli-lambdaweb-<アカウントID>-ap-northeast-1-an` | コードの ZIP を置く場所。バージョニングが有効。同じリージョンで `deploy` を使う関数が共有する |
| IAM ロール | `awscli-lambdaweb-nextjs-sample` | 実行ロール。CloudWatch Logs にログを書く権限だけを持つ |
| Web Function | `nextjs-sample` | 関数本体 |
| リビジョン | (自動で ID が付く) | 今回アップロードしたコードと設定のスナップショット |
| エンドポイント | `dev` | HTTPS の入口 |

## 2. pnpm の `node_modules` からシンボリックリンクをなくす

### 何が起きたか

最初は、この手順なしでデプロイしました。デプロイは成功しましたが、アクセスすると `Bad Gateway` になり、ログには次のエラーが出ていました。

```text
Failed to load entry point ./server.js: Error: Cannot find module 'next'
```

```text
{"type":"platform.initReport", ..., "record":{"status":"failure","errorType":"Runtime.ExitError", ...}}
```

`server.js` が `next` を読み込めず、起動に失敗しています。Web Functions は失敗した実行環境を作り直しては止まるのを繰り返し、リクエストに応答できる環境がないため `Bad Gateway` になっていました。

### 原因

pnpm は、パッケージの実体を `node_modules/.pnpm/` に置き、`node_modules/next` などはそこへのシンボリックリンクにします。standalone 出力も同じ形です。

```text
.next/standalone/node_modules/
├── .pnpm/
│   └── next@16.4.0_.../node_modules/
│       ├── next/           … 実体
│       ├── @next/          … next が使うパッケージ(next と並んで置かれる)
│       ├── styled-jsx/
│       └── sharp/
├── next      -> .pnpm/next@16.4.0_.../node_modules/next   (シンボリックリンク)
├── react     -> .pnpm/react@19.3.0/node_modules/react
└── react-dom -> .pnpm/react-dom@.../node_modules/react-dom
```

デプロイ先では、このシンボリックリンクが残っていませんでした。そのため `node_modules/next` が存在せず、`Cannot find module 'next'` になります。

> リンクを実ファイルに置き換える(`cp -rL` でコピーする)方法は使えません。`next` を `node_modules/next` にコピーしても、`next` が使う `@next/env` などは `.pnpm/` の奥に残ったままで、Node.js がそこを探しに行かないためです。

### 対処: pnpm を npm と同じ配置にする

pnpm の `nodeLinker` 設定を `hoisted` にすると、npm と同じように、すべてのパッケージが `node_modules/` 直下に実体として置かれます。シンボリックリンクがなくなるので、standalone 出力もそのまま ZIP にできます。

`pnpm-workspace.yaml` に次の 1 行を追加します。

```yaml
nodeLinker: hoisted
```

| | 既定(`isolated`) | `hoisted` |
|---|---|---|
| 配置 | `.pnpm/` に実体、ほかはシンボリックリンク | npm と同じ平らな配置 |
| `package.json` に書いていないパッケージの `import` | エラーになる(厳しい) | できてしまう |
| standalone 出力のシンボリックリンク | あり | なし |

`hoisted` にすると、ローカルでも pnpm の厳しさは失われます。学習用なので、手軽さを優先してこの方法にします。

> ローカルは `isolated` のままにしたい場合は、プロジェクトを別のディレクトリにコピーし、そこで `pnpm install --node-linker hoisted` を実行してビルドする方法があります(`--node-linker` オプションは設定ファイルより優先されます)。手順が増えるので、このシリーズでは採用していません。

入れ直して、ビルドします。04 と同じく、設定を変えただけでは `pnpm install` が何もしないので、`node_modules` を消してから実行します。

```bash
rm -rf node_modules && pnpm install
pnpm build
```

確認します。

```bash
# シンボリックリンクが 0 個になったか
find .next/standalone -type l | wc -l

# next、react などが実体として置かれているか(先頭が d で始まるディレクトリになっている)
ls -la .next/standalone/node_modules/ | head -20

# arm64 用の sharp も入っているか(04 と同じく 2 つ表示される)
find .next/standalone -name "*.node"
```

最後に、ローカルで起動して表示を確認します(03 と同じ手順)。

```bash
HOSTNAME=0.0.0.0 PORT=3000 node .next/standalone/server.js
```

## 3. デプロイする

### 事前確認

```bash
# 学習用アカウント・東京リージョンを向いているか
aws sts get-caller-identity
echo "$AWS_PROFILE $AWS_REGION"

# 最新の状態でビルドしておく
pnpm build
```

### デプロイ

```bash
aws lambda-web deploy \
  --name nextjs-sample \
  --code .next/standalone \
  --auth-type ApplicationManaged \
  --max-environments 2
```

| オプション | 意味 |
|---|---|
| `--code .next/standalone` | このディレクトリを ZIP にしてアップロードする |
| `--auth-type ApplicationManaged` | 公開エンドポイントにする |
| `--max-environments 2` | 実行環境の数の上限。指定できる範囲は 2〜10000 で、1 は指定できない(`--max-environments must be between 2 and 10000` というエラーになる)。1 つの環境で同時に 64 リクエストまで処理できるので、学習用には十分 |

初回は、作成するリソースの確認を求められます。内容を確認して Enter を押してください(確認を省略する `--create` オプションもありますが、何が作られるかを見るためにあえて付けていません)。
すでに一度デプロイしている場合は、確認なしで新しいリビジョンが作られます。`--auth-type` などのエンドポイントの設定は、同じ値なのでそのまま維持されます。

完了すると、エンドポイントの URL(`xxxx.xxx.lambda-web.ap-northeast-1.on.aws` のような形)が表示されます。初回はエンドポイントの準備に数分かかることがあります。

#### 補足: 隠しファイルの扱い

`deploy` は、名前が `.` で始まるファイルやディレクトリを既定で ZIP から除外します(`.env` などを誤ってアップロードしないため)。`.next/` は例外として自動で含まれます。
手順 2 で `hoisted` にしたので、standalone 出力には `.next/` 以外の隠しディレクトリがなく、隠しファイルを含める `--include-hidden-files` オプションは不要です(既定の `isolated` のままだと、`node_modules/.pnpm/` が除外されてしまいます)。

#### 補足: `server.js` だけで動くしくみ

`index.js` を自分で書かなくても動くのは、次の 2 つがかみ合っているからです。

**1. 入口: `deploy` が `index.js` を自動で追加する**

Web Functions のランタイムは必ず `index.js` を起動します。`deploy` は、`--entry-point` を指定しないと次の順で入口を探します(`aws lambda-web deploy help` の `--entry-point` の説明より)。

1. `package.json` の `main` と `exports`
2. `index.js` や `server.js` などのよくあるファイル名
3. 見つかったファイルが `index.js` 以外なら、それを読み込む `index.js` を追加する

standalone の `package.json` には `main` も `exports` もないので、`server.js` が見つかり、`index.js` が追加されました。中身は次のとおりです(ヘルプには中身までは書かれていないので、実際にデプロイされたものを載せています)。

```js
// Auto-generated by 'aws lambda-web deploy'. Loads your entry point.
const entryPoint = "./server.js";
import(entryPoint).catch((err) => {
  console.error(`Failed to load entry point ${entryPoint}:`, err);
  process.exit(1);
});
```

`import()` は、ファイルを実行中に読み込む書き方です。CommonJS(`require` を使う形式)のファイルの中でも使え、読み込む側も読み込まれる側も CommonJS か ES Modules かを問いません。そのため、Next.js が出力した CommonJS の `server.js` もそのまま読み込めます。
手順 2 で見たエラー `Failed to load entry point ./server.js: ...` は、この `catch` の中で出力されたものです。

**2. 待ち受け先: 既定値がたまたま一致している**

| | 値 |
|---|---|
| `server.js` の既定値(`PORT`・`HOSTNAME` が未設定のとき) | `0.0.0.0:3000` |
| Web Functions が指定する待ち受け先(`AWS_LAMBDA_HTTP_ENDPOINT`) | `0.0.0.0:3000` |

実際に `HOSTNAME` を指定せずに動いたので、Web Functions の実行環境は `HOSTNAME` を環境変数として渡していないことも分かりました。

> **本番で使うなら:** AWS は「`AWS_LAMBDA_HTTP_ENDPOINT` の値で待ち受けること」とだけ定めていて、`0.0.0.0:3000` という値が変わらないとは約束していません。確実にするには、`AWS_LAMBDA_HTTP_ENDPOINT` を `PORT` と `HOSTNAME` に読み替えてから `server.js` を読み込む `index.js` を自分で書きます。

## 4. 動作を確認する

> **「デプロイが成功した」と「アプリが動いている」は別物です。**
> Web Functions は、アプリの起動に失敗してもリビジョンを `Active` にします。必ず実際にアクセスして確認します。

### URL を変数に入れる

```bash
URL=$(aws lambda-web get-web-function-endpoint \
  --function-name nextjs-sample --endpoint-name dev \
  --query domainName --output text)
echo "https://$URL"
```

### アクセスする

```bash
# トップページ
curl -s -o /dev/null -w "%{http_code}\n" "https://$URL/"

# public/ のファイル
curl -s -o /dev/null -w "%{http_code}\n" "https://$URL/next.svg"

# .next/static/ のファイル(HTML から CSS のパスを取り出してアクセス)
CSS=$(curl -s "https://$URL/" | grep -o '/_next/static/[^"]*\.css' | head -1)
curl -s -o /dev/null -w "%{http_code}\n" "https://$URL$CSS"
```

3 つとも `200` なら成功です。ブラウザで `https://<URL>` を開き、ローカルと同じ画面が表示されることも確認してください。

## 5. 動かなかったとき

まずログを見ます。Web Functions のログは、CloudWatch Logs の `/aws/lambda/web/nextjs-sample` に出ます。

```bash
aws logs tail /aws/lambda/web/nextjs-sample --since 15m
```

確認するポイント:

- **起動に成功したか:** `platform.initReport` という行の `status` が `success` か `failure` か
- **どこで待ち受けているか:** Next.js の起動メッセージ `- Network: http://xxx:3000` の `xxx` 部分

症状ごとの対処:

| ログの内容 | 原因 | 対処 |
|---|---|---|
| `Network: http://` の後ろが `0.0.0.0` 以外 | 実行環境に `HOSTNAME` が渡されている(今回は起きなかった) | `--env HOSTNAME=0.0.0.0` を付けて再デプロイする |
| `Cannot find module 'next'` など | pnpm のシンボリックリンクがデプロイ先で失われている | 手順 2 の `nodeLinker: hoisted` を設定したか確認する |
| `SyntaxError` など、`index.js` の読み込みで失敗 | 自動で追加された `index.js` と `server.js` の形式が合っていない | 結果を共有してください |
| その他 | | ログの該当部分を共有してください |

## 6. 作られたリソースを見てみる

01 で説明した 3 つのリソース(Web Function、リビジョン、エンドポイント)を実際に確認します。

```bash
# Web Function
aws lambda-web get-web-function --function-name nextjs-sample

# リビジョンの一覧(手順 2 で失敗した最初のデプロイの分も残っている)
aws lambda-web list-web-function-revisions --function-name nextjs-sample

# エンドポイント(どのリビジョンに通信を流しているかが分かる)
aws lambda-web get-web-function-endpoint --function-name nextjs-sample --endpoint-name dev
```

AWS マネジメントコンソールでも見られます。Lambda のコンソールを開き、左のメニューから **Web Functions** を選んでください。

## Tips

### エンドポイント名 `dev` は `deploy` コマンドの既定値

`aws lambda-web deploy` は、最初のエンドポイントに `dev` という名前を付けます(`--endpoint-name` の説明より)。初回に `--endpoint-name prod` のように指定すれば、別の名前にできます。
下位の API(`create-web-function`)を直接使う場合は、エンドポイント名を必ず自分で指定します。

### `deploy` はフレームワークのビルド出力を意識している

`--include-hidden-files` のヘルプには、このオプションなしでも常に含めるディレクトリとして `.next/`、`.nuxt/`、`.svelte-kit/`、`.output/` が挙がっています。それぞれ Next.js、Nuxt、SvelteKit、Nitro のビルド出力です。
一方、入口として `server.js` を探して `index.js` を追加する動きは、Express などの自作サーバーでも使える汎用的なしくみです。Next.js の既定の待ち受け先(`0.0.0.0:3000`)が Web Functions とたまたま一致していたので、そのまま動きました。

### `node_modules/.pnpm/` が ZIP に入っていても動かない理由

pnpm の既定の配置では、`node_modules/next` は `.pnpm/` の奥の実体を指すシンボリックリンクです。Node.js は `require('next')` で `node_modules/next` を探しますが、自分から `.pnpm/` の中を探しには行きません。
そのため、リンクが失われると、`.pnpm/` に実体があっても `Cannot find module 'next'` になります。

リンクが失われるのが、`deploy` が ZIP を作るときなのか、Web Functions が展開するときなのかは確認できていません。`--force-include` のヘルプに「シンボリックリンクは含められない」とあるので、`deploy` が除外している可能性が高いと考えています。

### 実際の開発では、CD でだけ `hoisted` にする

このシリーズでは手軽さを優先して `pnpm-workspace.yaml` に `nodeLinker: hoisted` を書きました。実際の開発では、次のように分けるのが一般的です。

- **ローカル:** `nodeLinker` は書かず、pnpm の既定の配置(`isolated`)の厳しさを活かす
- **CD:** まっさらな環境で `pnpm install --frozen-lockfile --node-linker hoisted` → `pnpm build` → デプロイ

`--node-linker` オプションは設定ファイルより優先されます。CD を arm64 のマシン(GitHub Actions の arm64 ランナーなど)で動かせば、ビルド環境と実行環境の CPU がそろうので、04 の `supportedArchitectures` も不要になります。

注意点として、ローカルと CD で配置が違うので、成果物で起きた問題はローカルで再現しにくくなります。

## このステップのチェックリスト

- [ ] `pnpm-workspace.yaml` に `nodeLinker: hoisted` を追加し、standalone 出力のシンボリックリンクが 0 個になった
- [ ] `aws lambda-web deploy` が完了し、URL が表示された
- [ ] トップページ、`/next.svg`、CSS がすべて `200` を返した
- [ ] ブラウザでローカルと同じ画面が表示された

## 参考資料

- [Creating a Lambda Web Function(AWS ドキュメント)](https://docs.aws.amazon.com/lambda/latest/dg/web-functions-create.html)
- `aws lambda-web deploy help`(隠しファイルの扱いと `--entry-point` の説明)
