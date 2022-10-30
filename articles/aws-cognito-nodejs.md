---
title: "Node.jsからCognitoを操作して、アカウントのステータスがどう変化していくかまとめる"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "nodejs"]
published: true
---

Amazon Cognito を Node.js から操作してみます。

- ユーザーの作成
- サインイン
- サインアウト
- パスワードの変更
- メールアドレスを「検証済」に変更

などの操作を試してみます。

## Cognito でユーザープールを作成する

[Amazon Cognito](https://ap-northeast-1.console.aws.amazon.com/cognito/home?region=ap-northeast-1)を開きます。

ユーザープールの管理 > ユーザープールを作成する

をクリックして、ユーザープールを作成します。

- プール名：`CognitoNodeSample`
  - デフォルトを確認する
- アプリクライアントの追加
  - `CognitoAppClientSample` で作成
  - `ユーザー名パスワードベースの認証を有効にする (ALLOW_USER_PASSWORD_AUTH)` にチェックを追加
  - `クライアントシークレットを生成` のチェックは外す

Cognito でユーザープールを作成すると、プール ID という ID が生成されるので、メモしておきます。
アプリクライアントをクリックすると、「アプリクライアント ID」が表示されるので、これもメモします。

## プール ID とアプリクライアント ID を環境変数に設定する

プール ID とアプリクライアント ID を環境変数に設定します。

MY_USERNAME にはメールアドレスを、
MY_PASSWORD には `gefAuC@0982`のような、大文字小文字記号混じりの適当な文字列を入れてください。

```shell:.env
USER_POOL_ID=
COGNITO_CLIENT_ID=
REGION=
MY_USERNAME=
MY_PASSWORD=
```

## aws-sdk をインストール

```console
npm i aws-sdk
npm i -D @aws-sdk/types
```

## adminCreateUser でユーザーを作成する

[AWS Cognito UserPool をサーバーサイドで使うサンプル (Node.js)](https://note.kiriukun.com/entry/20191023-server-side-aws-cognito-user-pool-authentication-in-nodejs)を参考に Node.js からユーザーを作成します。

```ts
import AWS from "aws-sdk";
import "dotenv/config";

const { USER_POOL_ID, COGNITO_CLIENT_ID, REGION, MY_USERNAME, MY_PASSWORD } =
  process.env;

function getEnvVariables() {
  if (!USER_POOL_ID) {
    console.log("exception");
    throw new Error(`USER_POOL_IDが falsy ッス。${USER_POOL_ID}`);
  }
  if (!COGNITO_CLIENT_ID) {
    throw new Error(`COGNITO_CLIENT_ID が falsy ッス。${COGNITO_CLIENT_ID}`);
  }
  if (!REGION) {
    throw new Error(`REGION が falsy ッス。${REGION}`);
  }
  if (!MY_USERNAME) {
    throw new Error(`MY_USERNAME null が falsy ッス。${MY_USERNAME}`);
  }
  if (!MY_PASSWORD) {
    throw new Error(`MY_PASSWORD が falsy ッス。${MY_PASSWORD}`);
  }
  return {
    userPoolId: USER_POOL_ID,
    cognitoClient: COGNITO_CLIENT_ID,
    region: REGION,
    userName: MY_USERNAME,
    password: MY_PASSWORD,
  };
}

async function adminCreateUser() {
  const { userPoolId, userName, region } = getEnvVariables();
  AWS.config.update({
    region,
  });
  const cognito = new AWS.CognitoIdentityServiceProvider({
    apiVersion: "2016-04-18",
  });
  try {
    const user = cognito
      .adminCreateUser({
        UserPoolId: userPoolId,
        Username: userName,
      })
      .promise();
    console.log(
      "ユーザーの登録が完了しました。",
      JSON.stringify(user, null, 4)
    );
    return user;
  } catch (error) {
    console.error(error);
  }
}

async function main() {
  await adminCreateUser();
}

main();
```

実行結果：

```
ユーザーの登録が完了しました。 {}
```

Cognito 上には以下のようにユーザーが作成されます。

![](https://storage.googleapis.com/zenn-user-upload/901f44ec2222-20221030.jpg)

アカウントのステータスは `FORCE_CHANGE_PASSWORD` となっています。
E メールは確認済になっていません。

## adminSetUserPassword でユーザーのパスワードを変更する

```ts
async function adminSetUserPassword() {
  const { userPoolId, userName, region, password } = getEnvVariables();
  AWS.config.update({
    region,
  });
  const cognito = new AWS.CognitoIdentityServiceProvider({
    apiVersion: "2016-04-18",
  });
  try {
    const result = await cognito
      .adminSetUserPassword({
        UserPoolId: userPoolId,
        Username: userName,
        Password: password,
      })
      .promise();
    console.log("パスワードの変更が完了しました。", result);
  } catch (error) {
    console.error(error);
  }
}
```

結果：

```
パスワードの変更が完了しました。 {}
```

Cognito のステータス `FORCE_CHANGE_PASSWORD` のまま変化なしです。

![](https://storage.googleapis.com/zenn-user-upload/cd6a012916ab-20221030.png)

## Cognito の E メールのステータスを「確認済」にする

```ts
async function setEmailVerified() {
  const { userPoolId, userName, region } = getEnvVariables();
  AWS.config.update({
    region,
  });
  const cognito = new AWS.CognitoIdentityServiceProvider({
    apiVersion: "2016-04-18",
  });
  try {
    const param = {
      UserPoolId: userPoolId,
      Username: userName,
      UserAttributes: [
        {
          Name: "email_verified",
          Value: "true",
        },
      ],
    };
    const result = await cognito.adminUpdateUserAttributes(param).promise();
    console.log("Eメールを確認済のステータスに変更しました。", result);
  } catch (error) {
    console.error(error);
  }
}
```

E メールが「確認済」のステータスになりました。

![](https://storage.googleapis.com/zenn-user-upload/4cd0511ab422-20221030.png)

## adminInitiateAuth を使ってサインイン

```ts
async function signIn() {
  const { userPoolId, cognitoClientId, userName, password, region } =
    getEnvVariables();
  AWS.config.update({
    region,
  });
  const cognito = new AWS.CognitoIdentityServiceProvider({
    apiVersion: "2016-04-18",
  });
  try {
    const user = await cognito
      .adminInitiateAuth({
        UserPoolId: userPoolId,
        ClientId: cognitoClientId,
        AuthFlow: "ADMIN_USER_PASSWORD_AUTH",
        AuthParameters: {
          USERNAME: userName,
          PASSWORD: password,
        },
      })
      .promise();
    console.log("user.AuthenticationResult", user.AuthenticationResult);
    console.log("user.ChallengeName", user.ChallengeName);
    console.log("user.ChallengeParameters", user.ChallengeParameters);
  } catch (error) {
    console.error(error);
  }
}
```

結果：

```console
user.AuthenticationResult undefined
user.ChallengeName NEW_PASSWORD_REQUIRED
user.ChallengeParameters {
  USER_ID_FOR_SRP: 'xxxx@gmail.com',
  requiredAttributes: '["userAttributes.email"]',
  userAttributes: '{"email_verified":"true","email":""}'
}
```

## adminUserGlobalSignOut でサインアウト

```ts
async function signOut() {
  const { userPoolId, userName, region } = getEnvVariables();
  AWS.config.update({
    region,
  });
  const cognito = new AWS.CognitoIdentityServiceProvider({
    apiVersion: "2016-04-18",
  });

  const response = await cognito
    .adminUserGlobalSignOut({
      UserPoolId: userPoolId,
      Username: userName,
    })
    .promise();

  console.log(
    "サインアウトしました。",
    response.$response.httpResponse.statusCode
  );
}
```

## ユーザーのステータスを CONFIRMED にする

```ts
async function adminSetUserPasswordAndStatusConfirmed() {
  const { userPoolId, userName, region, password } = getEnvVariables();
  AWS.config.update({
    region,
  });
  const cognito = new AWS.CognitoIdentityServiceProvider({
    apiVersion: "2016-04-18",
  });
  try {
    const result = await cognito
      .adminSetUserPassword({
        UserPoolId: userPoolId,
        Username: userName,
        Password: password,
        // permanent = true にすれば CONFIRMED、指定しなければ FORCE_CHANGE_PASSWORD
        Permanent: true,
      })
      .promise();
    console.log("パスワードの変更が完了しました。", result);
  } catch (error) {
    console.error(error);
  }
}
```

![](https://storage.googleapis.com/zenn-user-upload/da4aadcb4419-20221030.png)

## AccessToken, idToken, RefreshToken を取得する

ユーザーのアカウントのステータスを `CONFIRMED` に変更した後で、singIn の関数を実行すると、レスポンスで idToken などが取得できます。

```ts
async function signIn() {
  const { userPoolId, cognitoClientId, userName, password, region } =
    getEnvVariables();
  AWS.config.update({
    region,
  });
  const cognito = new AWS.CognitoIdentityServiceProvider({
    apiVersion: "2016-04-18",
  });
  try {
    const user = await cognito
      .adminInitiateAuth({
        UserPoolId: userPoolId,
        ClientId: cognitoClientId,
        AuthFlow: "ADMIN_USER_PASSWORD_AUTH",
        AuthParameters: {
          USERNAME: userName,
          PASSWORD: password,
        },
      })
      .promise();
    console.log(user);
    console.log("user.AuthenticationResult", user.AuthenticationResult);
    console.log("user.ChallengeName", user.ChallengeName);
    console.log("user.ChallengeParameters", user.ChallengeParameters);
  } catch (error) {
    console.error(error);
  }
}
```

`user.AuthenticationResult`に入ってくるレスポンス：

```json
{
  AccessToken: 'xxx',
  ExpiresIn: 3600,
  TokenType: 'Bearer',
  RefreshToken: 'xxx,
  IdToken: 'xxxxx'
}
```
