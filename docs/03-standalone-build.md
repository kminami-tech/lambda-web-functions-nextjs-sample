# 03. Next.js を standalone 出力にする

Next.js のビルド結果を、Web Functions に載せられる形(standalone 出力)にします。
このステップはすべて手元で完結し、AWS は使いません。

やること:

1. `next.config.ts` に `output: 'standalone'` を追加する
2. ビルドスクリプトに静的ファイルのコピーを追加する
3. ビルドして、出力の中身を確認する
4. 本番と同じ起動方法で、ローカルで動かしてみる

## 1. `next.config.ts` を変更する

`nextConfig` に `output: 'standalone'` を 1 行追加します。

```ts
const nextConfig: NextConfig = {
  output: "standalone",
  /* config options here */
  experimental: {
    agentFeedback: true,
  },
  // ...以下は変更なし
};
```

これで `next build` のときに、本番実行に必要なファイルだけが `.next/standalone/` に集められます。

### standalone 出力のしくみ

Next.js はビルド時に、各ページが `import` や `require` で読み込むファイルを静的に解析します(この処理を「トレース」と呼びます)。
その結果、実際に使われるファイルだけが `node_modules` から `.next/standalone/node_modules` にコピーされます。開発用ツール(TypeScript、ESLint、Tailwind CSS など)は含まれないので、パッケージが小さくなります。

## 2. ビルドスクリプトを変更する

01 で触れたとおり、standalone 出力には `public/` と `.next/static/` が含まれません。
コピーを忘れないように、`package.json` の `build` スクリプトに書いておきます。

```json
"scripts": {
  "dev": "next dev",
  "build": "next build && cp -r public .next/standalone/ && cp -r .next/static .next/standalone/.next/",
  "start": "next start",
  "lint": "eslint"
},
```

| コピーするもの | 中身 | このアプリでの例 |
|---|---|---|
| `public/` | そのまま配信する静的ファイル | `next.svg`、`vercel.svg` など |
| `.next/static/` | ビルドで生成された JavaScript と CSS | `/_next/static/...` で配信されるファイル |

`next build` は毎回 `.next/` を作り直すので、古いファイルが残る心配はありません。

## 3. ビルドして中身を確認する

```bash
pnpm build
```

### 構成を確認する

```bash
ls -la .next/standalone/
ls -la .next/standalone/.next/
```

次のようになっていれば成功です。

```text
.next/standalone/
├── server.js        … 起動用のサーバー。05 でこれを Web Functions の入口にする
├── package.json     … プロジェクトの package.json のコピー
├── node_modules/    … トレースで選ばれた依存パッケージだけ
├── public/          … 手順 2 でコピーしたもの
└── .next/
    ├── static/      … 手順 2 でコピーしたもの
    └── server/ など … サーバー側のビルド結果
```

`server.js` が `.next/standalone/` の直下にない場合(さらに深いディレクトリにある場合)は、Next.js がプロジェクトのルートを誤って判定しています。その場合は知らせてください。

### サイズを確認する

```bash
du -sh .next/standalone/
```

Web Functions のパッケージ上限は 512 MB です。このアプリなら十分に収まるはずです。

### シンボリックリンクを確認する

```bash
find .next/standalone -type l | head
find .next/standalone -type l | wc -l
```

このプロジェクトは pnpm を使っています。pnpm は `node_modules` の中をシンボリックリンクで組み立てるので、standalone 出力にもシンボリックリンクがそのまま残ります。
ZIP にしたときにリンクが正しく扱われるかは、05 で確認します。ここでは「リンクが含まれている」ことだけ把握しておけば十分です。

## 4. ローカルで起動する

Web Functions と同じ条件(`server.js` を直接起動し、`0.0.0.0:3000` で待ち受ける)で動かします。

```bash
HOSTNAME=0.0.0.0 PORT=3000 node .next/standalone/server.js
```

起動したら、別のターミナルで確認します。

```bash
# トップページ(HTML)
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/

# public/ のファイル
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/next.svg
```

どちらも `200` なら成功です。ブラウザで http://localhost:3000 を開き、ロゴ画像やスタイルが崩れずに表示されることも確認してください。確認が終わったら `Ctrl+C` で止めます。

> **このアプリは `pnpm dev` ではなく `server.js` で動いている点が大事です。** 05 で Web Functions に載せるのは、まさにこの `.next/standalone/` ディレクトリです。

### 補足: なぜ `HOSTNAME` を明示するのか

`server.js` は環境変数 `HOSTNAME` の値で待ち受けます。次のコマンドで、手元のシェルの状態を見てみてください。

```bash
echo "$HOSTNAME"        # bash が持っている値(マシン名)
env | grep HOSTNAME     # 子プロセスに渡る環境変数
```

bash は `HOSTNAME` にマシン名を入れますが、通常は環境変数として子プロセスに渡しません。そのため、手元では `HOSTNAME` を付けなくても動きます。
一方、Docker などでは `HOSTNAME` が環境変数として渡されます。その場合 Next.js はマシン名で待ち受けてしまい、外から届きません。
Web Functions の実行環境がどちらなのかは、05 で `HOSTNAME` を指定せずにデプロイして確かめます(結果: 指定なしで動いた。つまり Web Functions の実行環境は `HOSTNAME` を渡していない)。

## うまくいかないとき

| 症状 | 考えられる原因 |
|---|---|
| ページは出るが CSS が当たらない | `.next/static/` のコピーが漏れている |
| ロゴ画像が 404 になる | `public/` のコピーが漏れている |
| `Cannot find module` で起動しない | トレースで必要なファイルが漏れている。エラー全文を共有してください |

## このステップのチェックリスト

- [ ] `next.config.ts` に `output: "standalone"` を追加した
- [ ] `package.json` の `build` スクリプトにコピー処理を追加した
- [ ] `.next/standalone/server.js` がある
- [ ] `.next/standalone/public/` と `.next/standalone/.next/static/` がある
- [ ] `node .next/standalone/server.js` で起動し、トップページと画像が表示された

**次のステップのために、次の 3 つを教えてください。**

1. `du -sh .next/standalone/` の結果
2. `find .next/standalone -type l | wc -l` の結果
3. ローカルで表示できたかどうか

## 参考資料

- Next.js の `output` 設定: `node_modules/next/dist/docs/01-app/03-api-reference/05-config/01-next-config-js/output.md`
- [Web Function deployment package(AWS ドキュメント)](https://docs.aws.amazon.com/lambda/latest/dg/web-functions-configuration-deployment-package.html)
