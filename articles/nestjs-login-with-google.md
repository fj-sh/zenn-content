---
title: "[NestJS]Googleでログインして取得したメールアドレスとIDを使ってJWT認証を行う"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs"]
published: true
---

## プロジェクトの作成とライブラリのインストール

以下のコマンドでまずは新規にプロジェクトを作成します。

```shell
nest new google-login
cd google-login
```

次に必要なライブラリをインストールします。

```shell
npm i @nestjs/passport passport passport-google-oauth20
npm i -D @types/passport-google-oauth20
```

## Google アプリケーションを作成する

[https://console.cloud.google.com/](https://console.cloud.google.com/) を開き、左側のメニューから「API とサービス」を選びます。

左側の「認証情報」をクリックしてから、上の方にある「＋認証情報を作成」→「OAuth クライアント ID」をクリックします。

「OAuth 同意画面」を一度も触ってない場合は、「OAuth クライアント ID の作成」という画面が出てくるので、「同意画面を設定」をクリックしていきます。

![](https://storage.googleapis.com/zenn-user-upload/86aa8c80b748-20221219.jpg)

「OAuth 同意画面」で「外部」を選択して「作成」をクリックします。

![](https://storage.googleapis.com/zenn-user-upload/781f644cce93-20221219.jpg)

「OAuth 同意画面」を編集していきます。

- アプリ名: `nextjs-example`
- ユーザーサポートメール: 自分のメールアドレス
- デベロッパーの連絡先情報: 自分のメールアドレス

を設定しました。

スコープには `/auth/userinfo.email` と `/auth/userinfo.profile`、`openid` を設定しました。

![](https://storage.googleapis.com/zenn-user-upload/41b694cc2dc1-20221219.jpg)

もう一度、左メニューの「認証情報」をクリックして、次に上に表示される「＋認証情報を作成」→ OAuth クライアント ID の作成 をクリックします。

アプリケーションの種類は「ウェブアプリケーション」を選びます。

![](https://storage.googleapis.com/zenn-user-upload/ea7dc1dc5ed9-20221219.jpg)

- 承認済みの Javascript 生成元： `http://localhost:3000`
- 承認済みのリダイレクト URI: `http://localhost:3000/auth/redirect`

とします。

ここで「作成」をクリックすると、「クライアント ID」と「クライアントシークレット」が表示されるので、コピーして保存しておきます。

## `.env` にクライアント ID とクライアントシークレットを設定する

環境変数に先ほどダウンロードしたクライアント ID とシークレットを設定します。

```shell
touch .env
```

```txt:.env
GOOGLE_CLIENT_ID=xxxx
GOOGLE_CLIENT_SECRET=xxx
GOOGLE_AUTH_CALLBACK_URL=http://localhost:3000/auth/redirect
```

## 前準備の続き

以下のコマンドを実行して、認証用のモジュールを作成します。

```shell
nest g res auth --no-spec

? What transport layer do you use? REST API
? Would you like to generate CRUD entry points? Yes
```

環境変数を読み込むための ConfigModule も一緒にインストールします。

```shell
npm i --save @nestjs/config
```

`src/app.module.ts` に ConfigModule を設定します。

```ts:src/app.module.ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { AuthModule } from './auth/auth.module';
import { ConfigModule } from '@nestjs/config';
import configuration from './config/configuration';

@Module({
  imports: [
    AuthModule,
    ConfigModule.forRoot({
      isGlobal: true,
      load: [configuration],
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

`src/config.configuration.ts` を作ります。

```ts:src/config.configuration.ts
export default () => ({
  google: {
    clientId: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackUrl: process.env.GOOGLE_AUTH_CALLBACK_URL,
  },
});
```

### interface を定義する

Google から受け取ったユーザー情報を入れるための interface です。

```ts:src/auth/interfaces/user.interface.ts
export interface User {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
  picture: string;
  accessToken: string;
  refreshToken?: string;
}
```

トークン情報をまとめた interface です。

```ts:src/auth/interfaces/token.interface.ts
export interface Token {
  accessToken: string;
  refreshToken: string;
}
```

## Google 認証用のストラテジを作成

```shell
mkdir src/auth/strategies
touch src/auth/strategies/google.strategy.ts
```

ストラテジを作成します。 `Injectable()` をつけるのを忘れないようにしてください。

```ts:src/auth/strategies/google.strategy.ts
import { PassportStrategy } from '@nestjs/passport';
import { Profile, Strategy, VerifyCallback } from 'passport-google-oauth20';
import { User } from '../interfaces/user.interface';
import { ConfigService } from '@nestjs/config';
import { Injectable } from '@nestjs/common';

@Injectable()
export class GoogleStrategy extends PassportStrategy(Strategy, 'google') {
  constructor(configService: ConfigService) {
    super({
      clientID: configService.get<string>('google.clientId'),
      clientSecret: configService.get<string>('google.clientSecret'),
      callbackURL: configService.get<string>('google.callbackUrl'),
      scope: ['email', 'profile', 'openid'],
      accessType: 'offline',
    });
  }

  async validate(
    accessToken: string,
    refreshToken: string,
    profile: Profile,
    _done: VerifyCallback,
  ): Promise<User> {
    const { id, name, emails, photos } = profile;
    const user: User = {
      id,
      email: emails[0].value,
      firstName: name.givenName,
      lastName: name.familyName,
      picture: photos[0].value,
      accessToken,
      refreshToken,
    };
    // _done(null, user);
    return user;
  }
}
```

`auth.module.ts` の `providers` に `GoogleStrategy` を設定します。

```ts:src/auth/auth.module.ts
import { Module } from "@nestjs/common";
import { AuthService } from "./auth.service";
import { AuthController } from "./auth.controller";
import { GoogleStrategy } from "./strategies/google.strategy";

@Module({
  controllers: [AuthController],
  providers: [AuthService, GoogleStrategy],
})
export class AuthModule {}
```

次に Guard を作成します。

```ts:src/auth/guards/google-oauth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class GoogleOauthGuard extends AuthGuard('google') {}
```

[スコープ](https://developers.google.com/identity/protocols/oauth2/scopes#google-sign-in)は以下のものがあるようです。

| スコープ | 説明                                                                         |
| -------- | ---------------------------------------------------------------------------- |
| email    | Google アカウントのメインのメールアドレスを表示する                          |
| openid   | Google で公開されているお客様の個人情報とお客様を関連付ける                  |
| profile  | ユーザーの個人情報の表示（ユーザーが一般公開しているすべての個人情報を含む） |

## Service の作成

認証サービスを作ります。

```ts:src/auth/auth.service.ts
import { Injectable, InternalServerErrorException } from "@nestjs/common";
import { User } from "./interfaces/user.interface";
import { Token } from "./interfaces/token.interface";

@Injectable()
export class AuthService {
  async login(user: User | undefined): Promise<Token> {
    if (user === undefined) {
      throw new InternalServerErrorException(
        `Googleからユーザー情報が渡されていませんね？ ${user}`
      );
    }
    console.log("Googleから渡されたユーザーの情報です。", user);

    return {
      accessToken: user.accessToken,
      refreshToken: user.refreshToken,
    };
  }
}
```

## Controller の作成

Google 認証を行うための API を作ります。

```ts:src/auth/auth.controller.ts
import { Controller, Get, Req, Request, UseGuards } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthGuard } from '@nestjs/passport';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Get()
  @UseGuards(AuthGuard('google'))
  async googleAuth(@Req() _req) {}

  @Get('redirect')
  @UseGuards(AuthGuard('google'))
  redirect(@Request() req) {
    return this.authService.login(req?.user);
  }
}
```

## Google 認証を使ってみる

`http://localhost:3000/auth`をブラウザで開くと、Google のログイン画面に飛ばされます。
Google でログインすると、アクセストークンがブラウザに表示されます。

コンソールには以下のように表示されます。

```json
Googleから渡されたユーザーの情報です。 {
  id: '192719212',
  email: 'xxxx@gmail.com',
  firstName: 'xxx',
  lastName: 'xxxx',
  picture: 'https://lh3.googleusercontent.com/a/21212121=2121',
  accessToken: 'ssss',
  refreshToken: undefined
}
```

## Google ログインと JWT 認証を組み合わせる

Google ログインで取得したユーザーの情報を使って JWT トークンを発行してみます。
ネット上で同じようなサンプルを作っている記事は [Google OAuth2 Authentication with NestJS explained](https://medium.com/@flavtech/google-oauth2-authentication-with-nestjs-explained-ab585c53edec) の 1 つしか見つけられなかったので、あまり一般的ではないのかもしれません。

無難にやるなら AWS Cognito を使うほうがサンプルが豊富です。

ターミナルで以下のコマンドを実行して、秘密鍵を生成します。

```shell
node -e "console.log(require('crypto').randomBytes(256).toString('base64'));"
```

`.env` に `JWT_ACCESS_SECRET` と `JWT_REFRESH_SECRET` を追記します。

```txt:.env
JWT_ACCESS_SECRET=5f9a+mSLNiK5R6sDEleBN/2pmmnrK+XuFv9drMtra9OJqcRMOuFvwTx9s+UI
JWT_REFRESH_SECRET=CpwWyDXmUmnBuOyNITWvA3W9NjFftWT099olyDOJjnpJh
```

`configuration.ts` で `JWT_ACCESS_SECRET` と `JWT_REFRESH_SECRET` を読み込みます。

```ts:config/configuration.ts
export default () => ({
  google: {
    clientId: process.env.GOOGLE_CLIENT_ID,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET,
    callbackUrl: process.env.GOOGLE_AUTH_CALLBACK_URL,
  },
  jwt: {
    accessSecret: process.env.JWT_ACCESS_SECRET,
    refreshSecret: process.env.JWT_REFRESH_SECRET,
  },
});

```

必要なライブラリのインストールをします。

```shell
npm i @nestjs/jwt passport-jwt
npm i -D @types/passport-jwt
```

## JWT 認証用のストラテジを作成する

accessToken を validation するためのストラテジです。

```ts:src/auth/strategies/access-token.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';

type JwtPayload = {
  sub: string;
  username: string;
};

@Injectable()
export class AccessTokenStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor(private configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: configService.get<string>('jwt.accessSecret'),
    });
  }

  validate(payload: JwtPayload) {
    return payload;
  }
}
```

```ts:src/auth/strategies/refresh-token.strategy.ts
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { Request } from 'express';
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class RefreshTokenStrategy extends PassportStrategy(
  Strategy,
  'jwt-refresh',
) {
  constructor(private configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: configService.get<string>('jwt.refreshSecret'),
      passReqToCallback: true,
    });
  }

  validate(req: Request, payload: any) {
    const refreshToken = req.get('Authorization').replace('Bearer', '').trim();
    return { ...payload, refreshToken };
  }
}
```

## JWT 認証用の Guard を作成する

コントローラーで `UseGuard` するためのガードを作成します。
accessToken 用のガードと、refreshToken 用のガードです。

```ts:src/auth/guards/access-token.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class AccessTokenGuard extends AuthGuard('jwt') {}
```

refreshToken のガードはその名の通り、トークンを更新する API を呼び出すときに使います。

```ts:src/auth/guards/refresh-token.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class RefreshTokenGuard extends AuthGuard('jwt-refresh') {}
```

## AuthService に JWT 認証用のメソッドを作成する

`getTokens` で accessToken と refreshToken を取得します。
accessToken は 15 分で期限切れするものにします。
refreshToken は 7 日間です。

accessToken の期限が切れたら refreshToken を使って更新します。

```ts:src/auth/auth.service.ts
import { Injectable, InternalServerErrorException } from '@nestjs/common';
import { User } from './interfaces/user.interface';
import { Token } from './interfaces/token.interface';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class AuthService {
  constructor(
    private jwtService: JwtService,
    private configService: ConfigService,
  ) {}

  async getTokens(userId: string, username: string) {
    const [accessToken, refreshToken] = await Promise.all([
      this.jwtService.signAsync(
        {
          sub: userId,
          username,
        },
        {
          secret: this.configService.get<string>('jwt.accessSecret'),
          expiresIn: '15m',
        },
      ),
      this.jwtService.signAsync(
        {
          sub: userId,
          username,
        },
        {
          secret: this.configService.get<string>('jwt.refreshSecret'),
          expiresIn: '7d',
        },
      ),
    ]);

    return {
      accessToken,
      refreshToken,
    };
  }

  async refreshTokens(userId: string, username: string, refreshToken: string) {
    // ちゃんとやりたい場合は、データベースに保存しておいたハッシュ化された refreshToken と
    // ユーザーから渡された refreshToken をハッシュ化して比較して、一致しない場合は例外を投げる。
    // ここはサンプルなので、 refreshTokens が呼ばれたら新しい accessToken を返す。
    const tokens = await this.getTokens(userId, username);
    return tokens;
  }

  async login(user: User | undefined): Promise<Token> {
    if (user === undefined) {
      throw new InternalServerErrorException(
        `Googleからユーザー情報が渡されていませんね？ ${user}`,
      );
    }
    console.log('Googleから渡されたユーザーの情報です。', user);

    const { accessToken, refreshToken } = await this.getTokens(
      user.id,
      user.email,
    );

    return {
      accessToken,
      refreshToken,
    };
  }
}
```

## AuthController に refreshToken 用の API を作る

`refreshTokens` というメソッドを追加します。
トークンを更新するための API です。

```ts:src/auth/auth.controller.ts
import { Controller, Get, Req, Request, UseGuards } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthGuard } from '@nestjs/passport';
import { RefreshTokenGuard } from './guards/refresh-token.guard';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Get()
  @UseGuards(AuthGuard('google'))
  async googleAuth(@Req() _req) {}

  @Get('redirect')
  @UseGuards(AuthGuard('google'))
  redirect(@Request() req) {
    return this.authService.login(req?.user);
  }

  @Get('refresh')
  @UseGuards(RefreshTokenGuard)
  async refreshTokens(@Request() req) {
    const userId = req?.user.sub;
    const username = req.user.username;
    const refreshToken = req.user.refreshToken;

    return this.authService.refreshTokens(userId, username, refreshToken);
  }
}
```

## AuthModule で JwtModule を import する

```ts:src/auth/auth.module.ts
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { GoogleStrategy } from './strategies/google.strategy';
import { JwtModule } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
import { RefreshTokenStrategy } from './strategies/refresh-token.strategy';
import { AccessTokenStrategy } from './strategies/access-token.strategy';

@Module({
  imports: [
    // registerAsync の中で ConfigService を inject するやり方は以下の記事を参照する。
    // https://stackoverflow.com/questions/53426486/best-practice-to-use-config-service-in-nestjs-module
    // https://stackoverflow.com/questions/64337784/nestjs-use-configservice-in-simple-provider-class
    JwtModule.registerAsync({
      inject: [ConfigService],
      useFactory: async (configService: ConfigService) => ({
        secret: configService.get<string>('jwt.accessSecret'),
        signOptions: { expiresIn: '60s' },
      }),
    }),
  ],
  controllers: [AuthController],
  providers: [
    AuthService,
    GoogleStrategy,
    AccessTokenStrategy,
    RefreshTokenStrategy,
  ],
})
export class AuthModule {}
```

## 実際に JWT トークンを使ってみる

`app.controller.ts` accessToken でガードされた API を作ってみます。

```ts:src/app.controller.ts
import { Controller, Get, UseGuards } from '@nestjs/common';
import { AppService } from './app.service';
import { AccessTokenGuard } from './auth/guards/access-token.guard';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }

  @UseGuards(AccessTokenGuard)
  @Get('/secret')
  getGuardContent(): string {
    return '保護されたコンテンツです。';
  }
}
```

Authorization - BearerToken を設定せずに `localhost:3000/secret` を叩くと、以下のように 401 が返ってきます。

```json
{
  "statusCode": 401,
  "message": "Unauthorized"
}
```

`localhost:3000` は何もしなくても叩けます。

`http://localhost:3000/auth` をブラウザで開くと、Google でログイン画面にリダイレクトされます。

ログインすると、ブラウザ上に accessToken と refreshToken が表示されるはずです。

```json
{
  "accessToken": "xxx.xxx.xxx",
  "refreshToken": "xxx.xxx.xxx"
}
```

ここで取得した `accessToken` を Authorization - Bearer Token に設定して `localhost:3000/secret` にリクエストを投げると、

「保護されたコンテンツです。」

というレスポンスが返ってきます。ガードされていたコンテンツを取得できました。

`accessToken` は 15 分で切れてしまうので、 refreshToken で更新してみましょう。

ブラウザで取得した `refreshToken` を Authorization - Bearer Token に設定して `localhost:3000/auth/refresh` を叩きます。

すると、新たに `accessToken` と `refreshToken` がレスポンスとして返されます。

```json
{
  "accessToken": "xxx.xxx.xxx",
  "refreshToken": "xxx.xxx.xxx"
}
```

更新した `accessToken` を使って、またコンテンツを取得していきます。

## 参考

- [How to Implement Login with Google in Nest JS](https://dev.to/imichaelowolabi/how-to-implement-login-with-google-in-nest-js-2aoa)
- [NestJS で google oauth 認証を実装する](https://zenn.dev/vbbvbbvbbv/articles/15b73c6329471a)
- [Implement Google OAuth in NestJS using Passport](https://dev.to/chukwutosin_/implement-google-oauth-in-nestjs-using-passport-1j3k)
- [Google API の Access Token をお手軽に取得する](https://qiita.com/shin1ogawa/items/49a076f62e5f17f18fe5)
- [OAuth2 in NestJS for Social Login (Google, Facebook, Twitter, etc)](https://javascript.plainenglish.io/oauth2-in-nestjs-for-social-login-google-facebook-twitter-etc-8b405d570fd2)
- [NestJS JWT Authentication with Refresh Tokens Complete Guide](https://www.elvisduru.com/blog/nestjs-jwt-authentication-refresh-token)
- [thisismydesign/nestjs-starter](https://github.com/thisismydesign/nestjs-starter)
