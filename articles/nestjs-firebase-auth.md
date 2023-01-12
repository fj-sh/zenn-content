---
title: "NestJS+Firebaseで「メールアドレス＋パスワード」でユーザー登録する"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs"]
published: true
---

Firebase の認証機構を使って、NestJS からメールアドレス＋パスワードでユーザーを作成する手順を記録しています。

## NestJS アプリを作成する

```shell
nest new .
```

## firebase ライブラリをインストールする

```shell
npm i firebase-admin
```

## Firebase でプロジェクトを作成する

Firebase Console [https://console.firebase.google.com/u/0/](https://console.firebase.google.com/u/0/) を開きます。

URL 末尾の `/u/0/` は Chrome にログインした 1 つ目のアカウントを示しています。
`/u/1/` だと 2 つ目のアカウントになるので、自分が利用しているアカウントに合わせてください。

「プロジェクトを追加」をクリックします。

次に、任意のプロジェクト名を設定します。自分は `nestjs-auth-firebase` としました。

次は Google アナリティクスを使うか聞かれます。
ここは必要になってから後でも設定できるため、設定せずに進みます。

「新しいプロジェクトの準備ができました」と表示されたら、「続行」をクリックします。

![](https://storage.googleapis.com/zenn-user-upload/739da75eb007-20230109.jpg)

「アプリに Firebase を追加して利用を開始しましょう」という画面が出たら、丸アイコンの左から 3 番目の `</>` をクリックします。

アプリのニックネームを聞かれます。

`nestjs-auth-firebase-web`とします。

「このアプリの Firebase Hosting も設定します」というチェックボックスがあります。
これは Web アプリとして Firebase のホスティング機能を使ってインターネットに公開するかどうか、という問いです。

公開するので、チェックを入れます。

- Firebase SDK の追加
- Firebase CLI のインストール
- Firebase Hosting へのデプロイ

はただの説明なので「次へ」で進んでいき、「コンソールに進む」をクリックします。

### デフォルトのロケーションを設定する

コンソールを開き、「プロジェクトの概要」の横にある歯車マークをクリックします。

![](https://storage.googleapis.com/zenn-user-upload/9520b81e440a-20230109.jpg)

「プロジェクトの設定」を選択した後、「デフォルトのリソース ロケーションの設定」でクラウドリソースのロケーションを`asia-northeast1`（東京）に設定します。

### 秘密鍵の作成

歯車マーク > プロジェクトの設定

で、「サービスアカウント」のタブを開きます。

![](https://storage.googleapis.com/zenn-user-upload/dc3ab42108f4-20230109.jpg)

「新しい秘密鍵の生成」をクリックします。
すると、xxxx.json というファイルがダウンロードできます。

`.env` に以下の値を設定します。

```txt
FIREBASE_PROJECT_ID=JSONのproject_idの値
FIREBASE_PRIVATE_KEY="JSON の private_key の値"
FIREBASE_CLIENT_EMAIL=JSONのclient_emailの値
```

## Firebase Authentication を始める

プロジェクトのコンソール画面の左ペインから `構築 > Authentication` を選択して、「始める」をクリックします。

「ログイン方法を追加して Firebase Auth の利用を開始しましょう」と出るので、ここでは「メール/パスワード」を選択します。

- メール / パスワード（有効にする）
- メールリンク（パスワードなしでログイン）は無効のまま

で、「保存」します。

次に、お試しでユーザーを作ってみます。

Authentication ページの 「Users」タブを開いて、「ユーザーを追加」をクリックすると、サンプルユーザーが作成できます。

## main.ts で firebase-admin を初期化する

```ts:src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import * as admin from 'firebase-admin';
import { ServiceAccount } from 'firebase-admin';
import { ConfigService } from '@nestjs/config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const configService: ConfigService = app.get(ConfigService);
  const adminConfig: ServiceAccount = {
    projectId: configService.get<string>('FIREBASE_PROJECT_ID'),
    privateKey: configService
      .get<string>('FIREBASE_PRIVATE_KEY')
      .replace(/\\n/g, '\n'),
    clientEmail: configService.get<string>('FIREBASE_CLIENT_EMAIL'),
  };
  admin.initializeApp({
    credential: admin.credential.cert(adminConfig),
  });

  await app.listen(3000);
}
bootstrap();
```

`main.ts` で `firebase-admin` を初期化する方法は以下の記事を参考にしました。

[【TS】NestJS で Firebase を使ってプッシュ通知を送るモック作成](https://zenn.dev/nekoniki/articles/d4bd396476c107)

## コントローラーを作成する

```shell
nest g controller auth
```

お試しで以下のようなコントローラーを作ってみます。

```ts:src/auth/auth.controller.ts
import { Controller, Get } from '@nestjs/common';
import * as admin from 'firebase-admin';

@Controller('auth')
export class AuthController {
  constructor() {}

  @Get()
  async sample() {
    const users = await admin.auth().listUsers();
    users.users.forEach((user) => {
      console.log(user.toJSON());
    });
    return users;
  }
}
```

`http://localhost:3000/auth` を叩くと、前の手順で Firebase に手動で登録したメールアドレスがコンソールに表示されます。

## firebase-admin でユーザーを作成してみる

`signUp`という関数を追加します。

```ts:src/auth/auth.controller.ts
import { Body, Controller, Get, Post } from '@nestjs/common';
import * as admin from 'firebase-admin';
import { SignUpModel } from './models/signup.model';

@Controller('auth')
export class AuthController {
  constructor() {}

  @Get()
  async sample() {
    const users = await admin.auth().listUsers();
    users.users.forEach((user) => {
      console.log(user.toJSON());
    });
    return users;
  }

  @Post('signup')
  async signUp(@Body() signupModel: SignUpModel) {
    console.log('signupModel', signupModel);
    const user = await admin.auth().createUser({
      email: signupModel.email,
      password: signupModel.password,
    });
    return user;
  }
}
```

`signup` で受け取っている `SignupModel` は以下のとおりです。

```ts:src/auth/models/signup.model.ts
export class SignUpModel {
  email: string;
  password: string;
}
```

CURL で POST リクエストを投げてみます。

```shell
curl -d '{"email":"sample@gmail.com", "password":"samplepassword"}' -H "Content-Type: application/json" -X POST http://localhost:3000/auth/signup
```

Firebase のコンソールを開き、 Authentication > Users を見ると、メールアドレスでユーザーが登録できていることがわかります。

- [Firebase Admin Auth でよくつかうものまとめ(ユーザ一覧など)](https://www.memory-lovers.blog/entry/2020/03/20/120000)
- [Authentication によるユーザー認証の動作確認](https://www.wakuwakubank.com/posts/709-firebase-authentication/)
- [curl.md](https://gist.github.com/subfuzion/08c5d85437d5d4f00e58)
- [Introduction to RESTful APIs with NestJS](https://www.thisdot.co/blog/introduction-to-restful-apis-with-nestjs)

## firebase/app でユーザーを作成してみる

歯車アイコンからプロジェクトの設定 > 全般 を開くと下の方に「マイアプリ」という項目があります。

そのマイアプリの中に「SDK の設定と構成」」という項目があり、その中で `firebaseConfig` の内容が書かれています。
`firebase/app` はその `firebaseConfig` を使って初期化します。

環境変数に以下を追記してください。

```txt:.env
API_KEY=
AUTH_DOMAIN=
PROJECT_ID=
STORAGE_BUCKET=
MESSAGING_SENDER_ID=
APP_ID=
```

依存関係に `firebase` を追加します。

```shell
npm install firebase
```

```ts:src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import * as admin from 'firebase-admin';
import { ServiceAccount } from 'firebase-admin';
import { ConfigService } from '@nestjs/config';
import { initializeApp } from 'firebase/app';
import { FirebaseApp } from '@firebase/app';

export let firebaseApp: FirebaseApp = undefined;

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const configService: ConfigService = app.get(ConfigService);
  const adminConfig: ServiceAccount = {
    projectId: configService.get<string>('FIREBASE_PROJECT_ID'),
    privateKey: configService
      .get<string>('FIREBASE_PRIVATE_KEY')
      .replace(/\\n/g, '\n'),
    clientEmail: configService.get<string>('FIREBASE_CLIENT_EMAIL'),
  };
  const firebaseConfig = {
    apiKey: configService.get<string>('API_KEY'),
    authDomain: configService.get<string>('AUTH_DOMAIN'),
    projectId: configService.get<string>('PROJECT_ID'),
    storageBucket: configService.get<string>('STORAGE_BUCKET'),
    messagingSenderId: configService.get<string>('MESSAGING_SENDER_ID'),
    appId: configService.get<string>('APP_ID'),
  };

  firebaseApp = initializeApp(firebaseConfig);
  admin.initializeApp({
    credential: admin.credential.cert(adminConfig),
  });

  await app.listen(3000);
}
bootstrap();
```

コントローラーに `firebase/app` バージョンのユーザー作成のコードを作ってみます。

```ts:src/auth/auth.controller.ts
@Post('signup2')
async signUp2(@Body() signupModel: SignUpModel) {
  const auth = getAuth(firebaseApp);
  const userCredential = await createUserWithEmailAndPassword(
    auth,
    signupModel.email,
    signupModel.password,
  );
  const user = userCredential.user;
  return user;
}
```

リクエストを投げてみます。

```shell
curl -d '{"email":"sample2@gmail.com", "password":"samplepassword2"}' -H "Content-Type: application/json" -X POST http://localhost:3000/auth/signup2
```

firebase-admin でのユーザー作成と違い、作成した時点でログイン状態になっていました。

このことからも、管理者としてユーザーを操作したいときは firebase-admin を使い、

公開 API として Firebase とやり取りする場合は普通に `firebase/app` でユーザーを作成するのが良いのでは、と思いました。

ただ、ユーザーの更新や削除は `firebase-admin`を使っているサンプルが多いです。

- [Javascript を使用してパスワードベースのアカウントを使用して Firebase で認証する](https://firebase.google.com/docs/auth/web/password-auth)
- [コピペで使う Firebase【Authentication 編】](https://qiita.com/koffee0522/items/38e650067c8920e116b9)

## 作成したユーザーのメールアドレスとパスワードで Firebase アプリにログインする

```ts
@Post('signin')
async signIn(@Body() signInModel: SignInModel) {
  const auth = getAuth(firebaseApp);
  const userCredential = await signInWithEmailAndPassword(
    auth,
    signInModel.email,
    signInModel.password,
  );

  return userCredential.user;
}
```

リクエストを投げるとログインできます。

```shell
curl -d '{"email":"sample@gmail.com", "password":"samplepassword"}' -H "Content-Type: application/json" -X POST http://localhost:3000/auth/signin
```

## 参考

- [NestJs: Firebase Auth secured NestJs app using PassportJs](https://medium.com/nerd-for-tech/nestjs-firebase-auth-secured-nestjs-app-using-passport-60e654681cff)
- [NestJs: Firebase Auth secured NestJs app](https://medium.com/nerd-for-tech/nestjs-firebase-auth-secured-nestjs-resource-server-9649bcebd0de)
- [Using Firebase Authentication in NestJS apps](https://blog.logrocket.com/using-firebase-authentication-in-nestjs-apps/)
- [firebase-auth-nestjs-passport](https://github.com/sharmavikashkr/firebase-auth-nestjs-passport)
- [nestjs-passport-firebase](https://github.com/whitecloakph/nestjs-passport-firebase/blob/main/src/firebase.strategy.ts)
