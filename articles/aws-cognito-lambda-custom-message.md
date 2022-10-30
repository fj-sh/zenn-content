---
title: "Cognitoのユーザー作成時にLambdaを使ってカスタマイズしたメールを送信する"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws"]
published: false
---

この記事では、Cognito でユーザーを作成したときに送信するメールを AWS Lambda を使ってカスタマイズする方法を紹介します。

## 記事内でやること

- Cognito でユーザーを作成する
- TypeScript で Lambda を作成する
- Cognito から呼び出す Lambda を設定する
- 実際に Cognito でユーザー登録してみて、メールがカスタマイズされていることを確認する

この記事で一番やりたいことは、メール作成のタイミングで Lambda トリガーを使用して Lambda 関数を呼び出し、メールをカスタマイズすることです。

## Cognito でユーザーを作成する

[Amazon Cognito のコンソール](https://ap-northeast-1.console.aws.amazon.com/cognito/home?region=ap-northeast-1#) から[ユーザープール](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/cognito-user-identity-pools.html)を作成します。

ユーザープールは AWS 内での認証を扱うサービスです。

ユーザープールの管理 > ユーザープールを作成する

をクリックします。

プール名は `SandboxUserPool` とします。

属性は `E メールアドレスおよび電話番号` とします。

![](https://storage.googleapis.com/zenn-user-upload/39a5d62032fd-20221030.jpg)

ポリシーではパスワードの強度を設定します。

![](https://storage.googleapis.com/zenn-user-upload/32ee3c8db675-20221030.jpg)

MFA は `なし` にしました。

ユーザーがアカウントを回復する手段は `使用可能な場合は E メール、それ以外の場合は電話。ただし、MFA にも使用している場合、電話でパスワードをリセットすることは許可しません` としています。

Cognito のデフォルトの機能でも以下のようにメッセージのカスタマイズができます。

![](https://storage.googleapis.com/zenn-user-upload/c6cad3915941-20221030.jpg)

「トリガー」という設定項目があります。
ここで Lambda の関数を設定しますが、現時点では Lambda 関数を作っていないので、設定しません。

![](https://storage.googleapis.com/zenn-user-upload/053cc92f66d4-20221030.jpg)

作成したユーザープールで試しにユーザーを作成してみます。

ユーザーとグループ > ユーザーの作成 をクリックしてユーザーを作成します。

![](https://storage.googleapis.com/zenn-user-upload/75eb853182e1-20221030.jpg)

ユーザーを作成すると、以下のようなメールが届きます。

> ユーザー名は example123@example.com、仮パスワードは HogeHoge123 です。

アカウントのステータスは `FORCE_CHANGE_PASSWORD` で、ユーザーは `Enabled` となっています。

## ローカル環境で Lambda 関数を作成する

[TypeScript で作成した関数を AWS Lambda に zip ファイルでデプロイする](https://zenn.dev/fjsh/articles/aws-typescript-deploy-to-lambda)

[How to Customize Emails in AWS Cognito](https://bobbyhadz.com/blog/aws-cognito-customize-emails)
