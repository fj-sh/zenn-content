---
title: "TypeScriptで作成した関数をAWS Lambdaにzipファイルでデプロイする"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['typescript','aws']
published: true
---

## この記事でやること

- AWS CLI で IAMロールを作成する
- AWS CLI で Lambda 関数を作成する
  - 作成された Lambda 関数を AWS コンソールで確認する
- TypeScript で Lambda 関数を書く
- TypeScript を JavaScript にトランスパイルする
- トランスパイルされたコードを AWS CLI で Lambda にアップロードする
- Lambda 関数を実行してログを確認する
- TypeScript を修正して、Lambda の関数を更新する
- npm スクリプトでZIPパッケージを作成する

## AWS CLI で IAMロールを作成する

`lambda-execute` という IAMロールを作成します。

```console
aws iam create-role --role-name lambda-execute --assume-role-policy-document '{"Version": "2012-10-17","Statement": [{ "Effect": "Allow", "Principal": {"Service": "lambda.amazonaws.com"}, "Action": "sts:AssumeRole"}]}'
```

[create-role](https://docs.aws.amazon.com/cli/latest/reference/iam/create-role.html) はAWSに[新しいロール](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/id_roles.html)を作成するコマンドです。

コマンド実行後、以下のようなレスポンスが返ってきます。

```json
{
    "Role": {
        "Path": "/",
        "RoleName": "lambda-execute",
        "RoleId": "AROA2QGSZAUCDUOCDSUEWZUEH",
        "Arn": "arn:aws:iam::1829967232694291:role/lambda-execute",
        "CreateDate": "2022-10-29T14:32:49+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "Service": "lambda.amazonaws.com"
                    },
                    "Action": "sts:AssumeRole"
                }
            ]
        }
    }
}
```

コンソールでIAMのロールを見ると、`lambda-execute`というロールが作成されています。

ちなみにこの手順は[AWS CLI での Lambda の使用](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/gettingstarted-awscli.html)に載っています。

## ロールにアクセス許可を追加する

`AWSLambdaBasicExecutionRole`マネージドポリシーを追加します。

[AWS マネージドポリシー](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies_managed-vs-inline.html)は、AWS が作成および管理するスタンドアロンポリシーです。
スタンドアロンポリシーとは、ポリシー名を含む独自の Amazon リソースネーム (ARN) の付いたポリシーです。

```console
aws iam attach-role-policy --role-name lambda-execute --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```


## AWS CLI で Lambda 関数を作成する

[TypeScript + Node.js + ESLint + Prettier の環境をとにかく早く構築する](/typescript-start-with-nodejs)の記事のような、TyepScriptの基本的な開発環境はできているものとします。

`@types/aws-lambda` をインストールします。

```
npm install -D @types/aws-lambda 
```

`index.ts` に以下のコードをコピペします。

```ts:src/index.ts
import {Context, APIGatewayProxyResult, APIGatewayEvent} from 'aws-lambda';

export const handler = async (event: APIGatewayEvent, context: Context): Promise<APIGatewayProxyResult> => {
  console.log(`Event: ${JSON.stringify(event, null, 2)}`);
  console.log(`Context: ${JSON.stringify(context, null, 2)}`);
  return {
    statusCode: 200,
    body: JSON.stringify({
      message: 'hello world',
    }),
  };
};

```

build コマンドを実行すると、dist 以下に index.js が作成されます。


```console
npm run build
```

[Deploy transpiled TypeScript code in Lambda with .zip file archives](https://docs.aws.amazon.com/lambda/latest/dg/typescript-package.html)

## zip で JavaScript ファイルを圧縮する

```console
cd dist && zip function.zip index.js
```

## Lambda 関数を作成する

```console
aws lambda create-function --function-name sample-function \
--zip-file fileb://function.zip --handler index.handler --runtime nodejs16.x \
--role arn:aws:iam::xxxxxx:role/lambda-execute
```

ここでの `arn:aws:iam::xxxxx:role/lambda-execute`の値は IAM > ロール > lambda-execute で取得できます。

以下のコマンドで作成した関数一覧をリストにして確認できます。

```console
aws lambda list-functions --max-items 10
```


## Lambda 関数を実行する

```console
aws lambda invoke --function-name sample-function out --log-type Tail
```

AWS コンソールからも実行できます。

ログは CloudWatch Logs に出力されます。

## Lambda関数を更新する

関数を修正して再度Lambdaにアップロードしたい場合は、以下のコマンドを実行します。

```console
aws lambda update-function-code --function-name sample-function \
--zip-file fileb://function.zip 
```


## Lambda関数を削除する

```console
aws lambda delete-function --function-name sample-function
```

