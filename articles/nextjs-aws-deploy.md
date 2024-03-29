---
title: "Next.jsの静的サイトをAWS S3+Cloudfront+Route 53+Lambda@Edge環境にデプロイする"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs"]
published: true
---

## やりたいこと

- Next.js で作った静的サイトを AWS 上にデプロイする
- GitHub に push したタイミングで自動的にデプロイできるようにする

### なぜ Vercel ではないのか？

Next.js のサイトをデプロイするなら Vercel は圧倒的に楽である。
GitHub のリポジトリを連携しただけで勝手にデプロイしてくれる手軽さは捨てがたいものがある。

しかしながら、 Hobby プランには制限がある。

[General Limits](https://vercel.com/docs/concepts/limits/overview)

中でも、`Bandwidth Up to 100 GB`の制限がネックであった。

というのも、万が一、私がデプロイするサイトが 1000 万ページビューを超える大ヒットサイトに育ってしまったら、けっこうお金がかかってしまいそうだからだ。

将来の禍根は事前に詰まねばならない。したがって、Vercel は見送ることにする。

### なぜ Cloudflare ではないのか？

Cloudflare は素晴らしい。

デプロイは無料で、アクセス解析も手軽にできて、しかも通信量による課金がない。

しかしながら、私が作るサイトは巨大サイトである。

SSG で出力されたページが 20,000 を超えると、Cloudflare 上で以下のようなエラーが出て、ビルドができなくなってしまったのだ。

```
Error: Exceeded 20k asset limit. Total assets:
```

残念でならないが、消去法的に AWS を選ばざるを得なくなった。

AWS にデプロイするよりも、Vercel や Cloudflare にデプロイしたほうが圧倒的に楽だ、という点は強調しておきたい。

## ローカルの Next.js の設定

`package.json` を編集する。

```diff json:package.json
- "build": "next build ",
+ "build": "next build && next export",
```

## AWS S3 の設定

Amazon S3 を開き、以下の順で作成していきます。

### バケットを作成

- バケット名
- 任意のもの
- AWS リージョン
  - アジアパシフィック（東京）ap-northeast-1
- オブジェクト所有者
  - ACL 無効
- このバケットのブロックパブリックアクセス設定
  - パブリックアクセスをすべて ブロックのチェックを外す
  - 現在の設定により、このバケットとバケット内のオブジェクトが公開される可能性があることを承認します。にチェックを入れる
- バケットのバージョニング
  - 無効にする
- デフォルトの暗号化
  - 無効にする
- オブジェクトロック
  - 無効にする

### 静的ウェブサイトホスティング

S3 バケットをクリックして、プロパティ タブを開きます。

![](https://storage.googleapis.com/zenn-user-upload/2f70e9999564-20220518.png)

「静的ウェブサイトホスティング」の編集をクリックして、

![](https://storage.googleapis.com/zenn-user-upload/595982261295-20220518.png)

以下のように設定します。

- インデックスドキュメント
  - `index.html`
- エラードキュメント - オプション
  - `404/index.html`

![](https://storage.googleapis.com/zenn-user-upload/41d3cd4be8c6-20220518.png)

「静的ウェブサイトホスティング」に「バケットウェブサイトエンドポイント」ができています。
アクセスすると、`403 Forbidden` が返ってきます。

### アクセス許可の設定

Amazon S3 > バケット の「アクセス許可」のタブを開きます。

バケットポリシーを以下のように設定します。 `[your-bucket-name]` の部分はバケット名に置き換えます。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::[your-bucket-name]/*"
    }
  ]
}
```

## Route 53 でドメインを取得

Route 53 を開き、取得したいドメイン名を入力します。

![](https://storage.googleapis.com/zenn-user-upload/393d440b2bf0-20220518.png)

「チェック」をクリックして、ドメインを購入すると、ダッシュボードに購入したドメインが表示されます。

メールアドレス宛に Amazon から確認リンクが届くので、送られてきたリンクをクリックします。

## CertificateManager で証明書をリクエスト

[Certificate Manager](https://us-east-1.console.aws.amazon.com/acm/home?region=us-east-1#/certificates/list) の「リクエスト」をクリックします。

![](https://storage.googleapis.com/zenn-user-upload/726361599059-20220519.jpg)

「完全修飾ドメイン名」には取得したドメイン名を設定します。

Amazon Certificate Manager > 証明書 > 証明署名

を見ると、証明書のステータスが「保留中の検証」となっていっていたのですが、画面真ん中あたりにある「Route 53 でレコードを作成」をクリックして、タイプ `CNAME` を作成すると、ステータスが「成功」に変化しました（「Route 53 でレコードを作成」をクリックすると、デフォルトで値が入っています）

![](https://storage.googleapis.com/zenn-user-upload/664ded2201ee-20220521.png)

[CloudFront + S3 + Route53 で独自ドメインを SSL 通信(https)設定をする](https://soypocket.com/it/cloudfront-s3-route53-https/)

## CloudFront の設定

[CloudFront](https://us-east-1.console.aws.amazon.com/cloudfront/v3/home?region=us-east-1#/distributions)を開き、「ディストリビューションを作成」をクリックします。

「オリジンドメイン」にはこの記事の上の方で作成した S3 の Bucket を指定します。`xxxx.s3.ap-northeast-1.amazonaws.com`のようなものです。
オリジンを選択すると、自動的に名前も入力されます。

「カスタム SSL 証明書 - オプション」には、CertificateManager でリクエストした証明書を関連付けます。

「代替ドメイン名 (CNAME) - オプション」には Route 53 で設定したドメイン名を設定します。
この設定によって、指定したドメインへの通信を https にします。

「デフォルトルートオブジェクト - オプション」は `index.html` とします。

その他は基本的にはデフォルトのままです。

### オリジン

CloudFront のディストリビューションを作成後、「オリジン」タブを開きます。

作成したオリジン名をクリックして、「編集」を押します。

「S3 バケットアクセス」は「はい、OAI を使用します (バケットは CloudFront のみへのアクセスとなるように制限できます)」を指定します。

「新しい OAI を作成」します（名前はデフォルトで指定されるもので OK です）

「バケットポリシー」は「はい、バケットポリシーを自動で更新します」を選択します。

追加料金が発生するらしいので、「オリジンシールドを有効にする」は「いいえ」にします。

「変更を保存」すると、S3 側の「アクセス許可」にバケットポリシーが追加されます。

### ビヘイビア

「ビヘイビア」タブをクリックして、「ビューワープロトコルポリシー」を「Redirect HTTP to HTTPS」にします。

### 一般 > 設定

一般 > 設定 をクリックして、「編集」ボタンを押します。

![](https://storage.googleapis.com/zenn-user-upload/a731d867bcf5-20220521.png)

代替ドメイン名 (CNAME) - オプション の「項目を追加」をクリックして、Route 53 で取得したドメイン名を設定します。

ここで「代替ドメイン名」設定しておくと、次の手順で「CloudFront ディストリビューションへのエイリアス」が設定できるようになります。

## Route 53 のホストゾーンでレコードを追加

Route 53 > ホストゾーン をクリックします。

作成したドメインを選択します。

「レコードを作成」をクリックします。

「レコード名」のところは テキストボックス + ドメイン名 みたいになっているかと思いますが、サブドメインを設定しない場合は、ここのテキストボックスは空のままで OK です。

「値」の横にある「エイリアス」をオンにします。

「エンドポイントを選択」で CloudFront ディストリビューションへのエイリアス を選択します。

セレクトボックスが出てくるので、作成した CloudFront のディストリビューションドメイン名を選択します。

レコードタイプは「A」で、「値」に CloudFront の「ディストリビューションドメイン名」を設定します。 `[id].cloudfront.net` のようなものです。

![](https://storage.googleapis.com/zenn-user-upload/cd5cb41486f8-20220521.png)

### GitHub Actions の設定

```console
mkdir -p .github/workflows
```

```console
touch .github/workflows/push_deploy.yml
```

```yaml:.github/workflows/push_deploy.yml
name: On push deploy free-av
on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Setup node
        uses: actions/setup-node@v3
        with:
          node-version: '16'

      - name: Install Dependencies
        run: npm install

      - name: Build
        run: npm run build

      - name: Deploy
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.SECRET_ACCESS_KEY }}
        run: |
          echo "AWS s3 sync"
          aws s3 sync --region ap-northeast-1 ./out s3://${{ secrets.AWS_S3_BUCKET}} --delete
          echo "AWS CF reset"
          aws cloudfront create-invalidation --region ap-northeast-1 --distribution-id ${{ secrets.AWS_CF_ID }} --paths "/*"


```

GitHub Actions を作った上で、GitHub 上のリポジトリの Settings > Secrets > Actions で、環境変数を設定します。

- `ACCESS_KEY_ID`：IAM から取得した key-id
- `SECRET_ACCESS_KEY`：IAM から取得した access-key
- `AWS_S3_BUCKET`：S3 のバケット名
- `AWS_CF_ID`: CloudFront の ID

GitHub の main ブランチに push すればデプロイが走ります。

この時点で、`index.html` つまり、トップページだけは見ることができます。

その他の個別ページ（ `/about` など）は `AccessDenied` が出て見ることができません。

それぞれのページを見るためには、Lambda@Edge を設定します。

## Lambda@edge を作成

```console
mkdir lambda-edge
cd lambda-edge
cdk init app --language typescript
npm install @types/aws-lambda --save-dev
touch .env
```

`.env` に以下を設定します。

```.env
CDK_DEFAULT_ACCOUNT="自分のアカウントID（右上クリックで出てくるやつ）"
CDK_DEFAULT_REGION="us-east-1"
```

アカウント ID はコンソールの右上の名前のところをクリックすると出てきます。

### 関数を作る

```console
mkdir lambda
touch lambda/index.ts
```

自動で作られる `lib/lambda-edge-stack.ts` を修正します。

`lambda-edge-stack`の部分はプロジェクト名によって違います。

```js:lib/lambda-edge-stack.ts
import { Stack, StackProps, aws_lambda_nodejs as lambda } from 'aws-cdk-lib'
import { Construct } from 'constructs'
export class LambdaEdgeStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props)
    new lambda.NodejsFunction(this, 'RedirectS3Lambda', {
      entry: 'lambda/index.ts',
    })
  }
}
```

[Next.js で SSG したサイトを AWS CloudFront + S3 にデプロイする](https://qiita.com/shinnoki/items/9cf68e68cbe594732b4b)を参考にして、「拡張子がついていないアクセスの場合 .html を付与して S3 へマッピングする対応」を行います。

```js:lambda/index.ts
import { Handler } from 'aws-lambda'

export const handler: Handler = async (event) => {
  const { request } = event.Records[0].cf

  // "/" へのリクエストはそのまま処理する
  if (request.uri === '/') {
    return request
  }

  // ファイル名 ("/" で区切られたパスの最後) を取得
  const filename = request.uri.split('/').pop()

  if (!filename) {
    // ファイル名が空 (つまり "/" で終わる) の場合、末尾の "/" を除去してリダイレクト
    return {
      status: '302',
      statusDescription: 'Found',
      headers: {
        location: [
          {
            key: 'Location',
            value: request.uri.replace(/\/+$/, '') || '/',
          },
        ],
      },
    }
  } else if (!filename.includes('.')) {
    // ファイル名に拡張子がついていない場合、 ".html" をつける
    request.uri = request.uri.concat('.html')
  }

  return request
}

```

### デフォルトのコードの修正

`bin/lambda-edge.ts` を修正します。

`lambda-edge` の部分はプロジェクト名です。

```js:bin/lambda-edge.ts
#!/usr/bin/env node
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { LambdaEdgeStack } from '../lib/lambda-edge-stack';

const app = new cdk.App();
new LambdaEdgeStack(app, 'LambdaEdgeStack', {
  env: { account: process.env.CDK_DEFAULT_ACCOUNT, region: process.env.CDK_DEFAULT_REGION },
});

```

次にターミナル上で、以下のアクセスキーなどを指定します。

下のキーは公式サイトのサンプルで、私のアクセスキーではありません。

```console
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7C832CWXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/KCD)JDG/bPxRfiCYEXAMPLEKEY
export AWS_DEFAULT_REGION=us-east-1
```

[Environment variables to configure the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html)

次に以下のコマンドでデプロイします。

```console
npm run cdk -- deploy
```

上記のコマンドを実行すると、[CloudFormation](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks?filteringStatus=active&filteringText=&viewNested=true&hideStacks=false)にスタックというものが作られます。

[Lambda 関数](https://us-east-1.console.aws.amazon.com/lambda/home?region=us-east-1#/functions) を開くと、`LambdaEdgeStack...` のような関数が作られています。

## Lambda を @edge へデプロイする

Lambda > 関数 > 作成した関数名 を開きます。

真ん中あたりのタブに「設定」があるので、クリックします。

「設定」の「環境変数」に `AWS_NODEJS_CONNECTION_REUSE_ENABLED` があるので、これは削除します。
Lambda@Edge にデプロイする際に、環境変数があったらエラーが出るためです。

## 関数の実行ロールを設定する

普通に Lambda@Edge を利用しようとすると、以下のようなエラーが出るかもしれません。

```
関数の実行ロールは、edgelambda.amazonaws.com サービスプリンシパルによって引き受け可能である必要があります。
```

[IAM](https://us-east-1.console.aws.amazon.com/iamv2/home#/home) を開いて、使用しているユーザーをクリックします。

左メニューに「ロール」があるので、クリックします。

`lambda` でフィルタをかけると、今回作成した Lambda 関数の名前がひっかかるかと思います（今回作成した関数名に lambda がない場合は、`lambda` 以外でフィルタしてください）

「信頼関係」タブがあるので、`"edgelambda.amazonaws.com"` を追加します。

```diff json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "lambda.amazonaws.com",
+                    "edgelambda.amazonaws.com"
                ]
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

## Lambda@Edge にデプロイ

[Lambda](https://us-east-1.console.aws.amazon.com/lambda/home?region=us-east-1#/functions)に戻ります。

作成した関数名をクリックします。

右上の「アクション」をクリックすると「Lambda@Edge へデプロイ」という機能があります。

![](https://storage.googleapis.com/zenn-user-upload/98abb77666b3-20220521.jpg)

- ディストリビューション → 作成した CloudFront の ID を指定する
- キャッシュ動作 →`*`
- CloudFront イベント → オリジンリクエスト
- Lambda@Edge へのデプロイを確認 → チェックを入れる

## trailingSlash を false にして再デプロイ

ローカルの Next.js の `next.config.js` を編集して、 trailingSlash を false にします。

```diff json:next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
+  trailingSlash: false,
}

module.exports = nextConfig

```

この状態で再度デプロイを走らせると、個別ページを見ることができるようになります。
