---
title: Terraformと競合しないTypeScript Lambdaデプロイ戦略
tags:
  - Node.js
  - AWS
  - TypeScript
  - lambda
private: false
updated_at: '2025-07-12T04:54:20+09:00'
id: f998abba8aa665d94d10
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

業務で Lambda の実装をすることになり、意外とハマったので備忘録も兼ねて書きました。

# 要件

今回実現したいことはざっくり以下の通りでした。

- Lambda のガワ（= リソース配置や IAM ロール紐付けなど、純粋なインフラ部分）は terraform 管理したい。
- Lambda のソースコードは TypeScript で開発したい。
- axios など npm パッケージも使用したい。
- インフラとソースコードは別リポジトリで管理したい（開発ドメイン分離）
- プルリクマージしてソースコード更新したときは、GithubActions で Lambda に反映したい。

↓ イメージ
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/577028/2122f225-ec27-4e66-86a0-b9519e2a297d.png)

# 検討

## 公式で紹介されている方法

当たり前ですが、TypeScript のまま Lambda にデプロイすることはできず、Node.js にトランスパイルする必要があります。

また今回は、**Node.js 標準ライブラリだけでなく npm パッケージも使いたい**（記事中だと axios）という要件があり、依存も含めいい感じにデプロイする方法ないかなーと記事を探していたところ、以下公式ドキュメントに辿り着きました。

`.zip ファイルアーカイブを使用して、トランスパイルされた TypeScript コードを Lambda にデプロイする`

https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/typescript-package.html

３つの方法が紹介されていました。

1. AWS SAM（Serverless Application Model）を使う
2. AWS CDK を使う
3. AWS CLI と esbuild を使う

:::note info
`3.` は明示的に esbuild を使う必要があります。
とはいえ `1.`, `2.` でも、SAM・CDK が内部的には esbuild を使っているようです。
:::

## どれを採用したか

`3.` の AWS CLI を使う方法を採用しました。

理由は、**AWS SAM や CDK を使うと terraform と競合してしまう**からです。

とはいえ AWS CLI で Lambda の中身・環境変数を更新すると、その後の terraform apply で巻き戻ってしまう問題があります。
そのため、いい感じに terraform 側で差分を無視する必要があります。（詳細は後述）

## 補足: esbuild とは？

### どんなツール？

- TS/JS 向けの超高速ビルド・バンドルツールです。Go 言語で実装されており、従来のツール（webpack や tsc）に比べて数十倍のビルド速度を誇ります。

### 今回は何をしてくれる？

主に以下をやってくれます。（すごい）

- TS を JS にトランスパイル
- npm パッケージを含めたバンドル。つまり axios なども含め 1 つの JS ファイルにまとめる
- Lambda の実行環境と整合性のある形式に出力設定できる（Node.js バージョンなど）

そのため、**別に Lambda 専用ツールというわけではないが、今回のように TS で開発してから Node.js 環境の Lambda で実行したいときに相性が非常に良いです。**

Cloudflare Workers とかでもいい感じに使えると思います。

# 実装

## ソースコード

npm パッケージとして、axios を使って適当なサイトに GET する関数を作成します。
axios.get する URL を、Lambda に設定する環境変数`URL_TO_GET`で指定します。

```index.ts
import axios from "axios";

export const handler = async (event: any): Promise<any> => {
  try {
    console.log("Lambda function started");

    // 環境変数からURLを取得
    const urlToGet = process.env["URL_TO_GET"];
    console.log("Target URL:", urlToGet);

    const response = await axios.get(urlToGet, {
      timeout: 5000,
      headers: {
        "User-Agent": "AWS-Lambda-TypeScript",
      },
    });

    return {
      statusCode: 200,
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        message: "Successfully fetched URL",
        status: response.status,
        contentLength: response.data.length,
        timestamp: new Date().toISOString(),
      }),
    };
  } catch (error) {
    console.error("Error occurred:", error);

    return {
      statusCode: 500,
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        message: "Error occurred while fetching data",
        error: error instanceof Error ? error.message : "Unknown error",
        timestamp: new Date().toISOString(),
      }),
    };
  }
};
```

package.json を定義します。
scripts の`build`, `bundle:config`は後述の CI/CD で使います。

```package.json
{
  "name": "lambda-ts-deploy",
  "version": "1.0.0",
  "description": "TypeScript Lambda function with axios",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc --noEmit",
    "bundle:config": "node esbuild.config.js"
  },
  "dependencies": {
    "axios": "^1.6.0"
  },
  "devDependencies": {
    "@types/node": "^22.15.0",
    "esbuild": "^0.19.0",
    "typescript": "^5.0.0"
  }
}
```

esbuild のバンドル設定は、今回はこうしました。

```esbuild.config.js
const esbuild = require("esbuild");

esbuild
  .build({
    entryPoints: ["index.ts"],
    bundle: true,
    platform: "node",
    target: "node22",
    outfile: "dist/index.js",
    external: ["aws-sdk"],
    format: "cjs", // CommonJS
    minify: true,
    sourcemap: true,
    logLevel: "info",
  })
  .catch(() => process.exit(1));
```

主なオプションの説明

| オプション                 | 説明                                                                                                             |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `bundle: true`             | 依存パッケージ（node_modules）も含めて 1 つのファイルにまとめる                                                  |
| `platform: "node"`         | Node.js 用としてバンドル（require()などをサポートします）                                                        |
| `target: "node22"`         | 使用する Node.js のバージョンを指定（Lambda のバージョンに合わせる）。執筆時点で安定バージョンの 22 にしました。 |
| `outfile: "dist/index.js"` | 出力ファイル形式                                                                                                 |

:::note info
オプションが少なければ、ファイル作らず scripts に直接オプション定義しても良いです。
esbuild index.ts --bundle --platform=node --target=node22 --outfile=dist/index.js
:::

## CI/CD（GithubActions）

Lambda をデプロイする GithubActions ワークフローを作成します。

### 補足

- AWS CLI によりソースコード（.zip）をデプロイした後、Lambda に環境変数をセットします。
- `npm run bundle:config` で esbuild が トランスパイルとファイルのバンドルをやってくれます。これにより、Lambda は単一の js ファイルを実行するだけで済むようになります。
- ワークフローが AWS アクセスするための OIDC 設定や IAM ロールはあらかじめ作成しました。手順はこの記事では省略します。

### 注意点

#### 注意 1

.zip のデプロイステップ直後に環境変数セットのステップを入れると、Lambda の更新が完了しておらず失敗しました。
そのため `aws lambda wait` により 更新完了を待つステップを間に入れることで解決しました。

```
An error occurred (ResourceConflictException) when calling the UpdateFunctionConfiguration operation: The operation cannot be performed at this time. An update is in progress
```

#### 注意 2: 型チェク

**esbuild はトランスパイル・バンドルをしますが、型チェックはしてくれません。**
そのためデプロイの前段に npm run build（tsc --noEmit） を挟んで誤爆を防いでいます。

### コード

ワークフローはこちら。

```yaml
name: Deploy Lambda for ts-cli-deploy

on:
  push:
    branches:
      - "main"
    paths:
      - "3_cli/src/**"
  workflow_dispatch:

env:
  NODE_VERSION: "22.15.0"
  AWS_REGION: ap-northeast-1
  LAMBDA_FUNCTION_NAME: ts-cli-deploy-test
  LAMBDA_PATH: 3_cli/src # ご自身の環境に適宜合わせてください

jobs:
  deploy:
    name: Deploy to Lambda
    runs-on: ubuntu-latest
    timeout-minutes: 5
    permissions:
      id-token: write
      contents: read

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_LAMBDA_DEPLOY_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"
          cache-dependency-path: ${{ env.LAMBDA_PATH }}/package-lock.json

      - name: Install dependencies
        working-directory: ${{ env.LAMBDA_PATH }}
        run: npm ci

      # esbuild は型チェックしないので、デプロイ前に型チェックを行う
      - name: Type check TypeScript
        working-directory: ${{ env.LAMBDA_PATH }}
        run: npm run build

      - name: Build and bundle TypeScript with esbuild
        working-directory: ${{ env.LAMBDA_PATH }}
        run: npm run bundle:config

      - name: Prepare deployment package
        working-directory: ${{ env.LAMBDA_PATH }}
        run: |
          cd dist
          # distディレクトリをzip化（依存関係は前ステップでバンドル済み）
          zip -r ../lambda-deployment.zip .

      - name: Deploy to Lambda
        working-directory: ${{ env.LAMBDA_PATH }}
        run: |
          aws lambda update-function-code \
            --function-name $LAMBDA_FUNCTION_NAME \
            --zip-file fileb://lambda-deployment.zip \
            --region $AWS_REGION

      # デプロイが完了するまで待ってから環境変数セットする
      - name: Wait for Lambda update to complete
        run: |
          echo "Waiting for Lambda function update to complete..."
          aws lambda wait function-updated \
            --function-name $LAMBDA_FUNCTION_NAME \
            --region $AWS_REGION
          echo "Lambda function update completed!"

      - name: Update Lambda environment variables
        run: |
          aws lambda update-function-configuration \
            --function-name $LAMBDA_FUNCTION_NAME \
            --environment Variables={URL_TO_GET=${{ vars.SITE_URL_TO_GET }}} \
            --region $AWS_REGION

      - name: Deployment result
        run: |
          echo "Deployment completed!"
          echo "Function: ${{ env.LAMBDA_FUNCTION_NAME }}"
          echo "Region: ${{ env.AWS_REGION }}"
          echo "Commit SHA: ${{ github.sha }}"
    outputs:
      commit-sha: ${{ github.sha }}
```

デプロイ後、Lambda コンソールを覗くと ↓ のようになってました。
バンドル後なので整形されてないですが、ちゃんと動くので安心してください。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/577028/8d7ef3f3-ab17-4f17-921e-1f3abad37f7d.png)

## terraform

説明前後しますが、開発したソースコードを CI/CD で反映する前に Lambda 自体の インフラを作っておく必要があります。

記事では IAM まわり省略しますが、以下のようにしました。
ポイントは以下 2 点

- `archive_file` により適当な .zip を作る。これで Lambda 初回作成時でも terraform apply 通るようにする。
- `lifecycle.ignore_changes` で ソースコードが反映された後に apply しても Lambda の中身と環境変数が巻き戻らないようにする。

```main.tf
resource "aws_lambda_function" "ts_deploy_test" {
  filename      = data.archive_file.lambda_zip.output_path # ローカルに作成された.zipを指定
  function_name = "ts-cli-deploy-test"
  role          = aws_iam_role.lambda_execution.arn
  handler       = "index.handler"
  runtime       = "nodejs22.x"
  timeout       = 30
  memory_size   = 512

  # 巻き戻り防止
  lifecycle {
    ignore_changes = [
      filename,
      environment
    ]
  }
}

# terraform 上で Zip ファイルとして lambda コードを作成
# ソースコード管理用リポジトリにてコード管理しているので、ここでは .zip を作成するだけ
data "archive_file" "lambda_zip" {
  type        = "zip"
  output_path = "${path.module}/ts-cli-deploy-test.zip" # ローカルのカレントディレクトリに.zip が作られる

  source {
    content  = "not_use_here" # 本来ソースコードはここに書くが、terraform管理しないので書かない。初回 apply 時のみ適用される。
    filename = "index.js"
  }
}
```

## Github

今回作成・動作確認したコードは以下に格納しています。ご参考までにどうぞ！

https://github.com/MakotoTaguchi1/lambda-ts-deploy

## 最後に

個人的に、インフラや CI/CD 含め、Lambda をゼロイチで実装したことがなかったので、意外と苦戦しました。

esbuild のことも理解できたし、良い勉強になりました！
