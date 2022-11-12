---
title: "NestJSでロールベースの認証を実装する"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs"]
published: true
---

この記事では NestJS で ROLE ベースの認証を実装していきます。

ユーザーに Admin や User などのロールをもたせて、

「Admin ロールを持ってるならこの API を叩ける」

「User ロールを持ってるならこの API を叩ける」

というような、ロールごとに叩ける API を設定していきます。

## プロジェクトの作成と準備

プロジェクトを作成します。

```console
npx @nestjs/cli new role-based-athorization
```

起動してみます。

```console
cd role-based-athorization
npm run start:dev
```

必要なライブラリをインストールします。

```console
npm install --save @nestjs/passport @nestjs/jwt passport passport-local passport-jwt
npm install --save-dev @types/passport-local @types/passport-jwt
```

NestJS CLI を使って必要なモジュールを生成します。

```console
npx @nestjs/cli g module auth --no-spec
npx @nestjs/cli g module users --no-spec
npx @nestjs/cli g service auth --no-spec
npx @nestjs/cli g service users --no-spec
```

## User のエンティティを作成する。

`users` ディレクトリ配下に User のエンティティを作成します。
この記事のサンプルでは OR マッパーは使わないので、特にデコレータを貼ることもなく、普通に interface を定義しているだけです。

```ts:src/users/user.entity.ts
export enum Role {
  User = 'user',
  Admin = 'admin',
}

export interface User {
  userId: number;
  username: string;
  password: string;
  roles: Role[];
}
```

## `users/users.service.ts` を修正する

ユーザー情報は Repository 経由でデータベースに保存するのが一般的かと思いますが、今回はサンプルなのでユーザー情報はメモリ上に配列で定義します。

```ts:src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { Role, User } from './user.entity';

@Injectable()
export class UsersService {
  private readonly users: User[] = [
    {
      userId: 1,
      username: 'naruto',
      password: '12345',
      roles: [Role.User],
    },
    {
      userId: 2,
      username: 'sasuke',
      password: '12345',
      roles: [Role.Admin],
    },
        {
      userId: 3,
      username: 'sakura',
      password: '12345',
      roles: [Role.Admin, Role.User],
    },
  ];

  async findOne(username: string): Promise<User | undefined> {
    return this.users.find((user) => user.username === username);
  }
}
```

上のコードを作成したら、 `UsersModule` に `exports` を追加します。
`UsersService` はあとで `AuthService` で使うので、 `exports` に定義してモジュールの外で使えるようにするのです。

```ts:src/users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';

@Module({
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

## `auth/auth.service.ts` を修正する。

認証に関する機能は `AuthService` で作っていきます。

```ts:src/auth/auth.service.ts
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';
import { Payload } from './payload.interface';
import { User } from '../users/user.entity';

@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
  ) {}

  async validateUser(username: string, pass: string): Promise<any> {
    const user = await this.usersService.findOne(username);
    if (user && user.password === pass) {
      const { password, ...result } = user;
      return result;
    }
    return null;
  }

  async login(user: User) {
    const payload: Payload = {
      username: user.username,
      sub: user.userId,
      roles: user.roles,
    };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}
```

`AuthService` の `validateUser` はユーザー名とパスワードを認証する `LocalAuthGuard` から呼ばれます。
ユーザー名とパスワードが一致した場合は `req.user` にユーザーの情報がセットされます。

`LocalStrategy` の `validate` を通したリクエストに対して、ユーザーデータを `req.user` として生やすのは Passport の仕様のようです。

[how to change return req.user object name of passport-local?](https://stackoverflow.com/questions/72677417/how-to-change-return-req-user-object-name-of-passport-local)

`AuthService` の `login` には LocalStrategy を通して、作られた `req.user` が渡されます。

渡された User から `username` と `sub` と `roles` を持つ PAYLOAD を含む JWT トークンを作って返します。

JWT トークンとは `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im5hcnV0byIsInN1YiI6MSwicm9sZXMiOlsidXNlciJdLCJpYXQiOjE2NjgyMzg5MDYsImV4cCI6MTY2ODI0MjUwNn0.tsfKbae4gU-DVNlU3JF2sxsIDyyKx0BrtsLS11zLEMQ`のような文字列です。

この文字列はデコードできます（ [jwt.io](https://jwt.io/) などに貼り付けるだけで OK）
デコードすると、ペイロードと呼ばれる部分に以下のように `username`,`sub`, `roles` と一緒にトークンの有効期限が含まれていることがわかります。

```json
{
  "username": "naruto",
  "sub": 1,
  "roles": ["user"],
  "iat": 1668238906,
  "exp": 1668242506
}
```

`AuthService` で使っている `Payload` は以下のように定義します。

```ts:src/auth/payload.interface.ts
import { Role } from '../users/user.entity';

export interface Payload {
  username: string;
  sub: number;
  roles: Role[];
}
```

## `local.strategy.ts` の作成

ユーザー名とパスワードでログインできるように、 `LocalStrategy` を作ります。

中では `AuthService` の `validateUser` を呼び出しています。
ユーザー名とパスワードが一致しているときは `true` を返して、存在しない場合は `null` で返ってきます。

```ts:src/auth/local.strategy.ts
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-local';
import { AuthService } from './auth.service';
import { Injectable, UnauthorizedException } from '@nestjs/common';

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super();
  }

  async validate(username: string, password: string): Promise<any> {
    const user = await this.authService.validateUser(username, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

`LocalAuthGuard` も作っておきます。
コントローラにデコレーションを設定するときに `AuthGuard('local')` と書かなくても済むようにするためです。

```ts:src/auth/local-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class LocalAuthGuard extends AuthGuard('local') {}
```

## `jwt.strategy.ts` の作成

`JwtStrategy` を作成します。

```ts:src/auth/jwt.strategy.ts
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { jwtConstants } from './constants';
import { Payload } from './payload.interface';

export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: jwtConstants.secret,
    });
  }

  async validate(payload: Payload) {
    return {
      userId: payload.sub,
      username: payload.username,
      roles: payload.roles,
    };
  }
}
```

秘密鍵は `jwtConstants` というクラスにベタ書きしていますが、プロダクトでは環境変数から読み込んでください。

```ts:src/auth/constants.ts
export const jwtConstants = {
  secret: 'secretKey',
};
```

`AuthModule` に JWT モジュールを登録します。
また、 `providers` に `LocalStrategy` と `JwtStrategy` を設定します。

```ts:src/auth/auth.module.ts
import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { UsersModule } from '../users/users.module';
import { PassportModule } from '@nestjs/passport';
import { LocalStrategy } from './local.strategy';
import { JwtModule } from '@nestjs/jwt';
import { jwtConstants } from './constants';
import { JwtStrategy } from './jwt.strategy';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: '1h' },
    }),
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

## ユーザーのロールを確認するためのデコレータを作成する

コントローラのメソッドに `@HasRoles(Role.Admin)` のようなデコレータを付けて、ロールを持っているユーザー以外は値を取得できないようにします。
まずはユーザーが持つ `Role[]` を受け取るデコレータを作ります。

```ts:src/auth/has-roles.decorator.ts
import { SetMetadata } from "@nestjs/common";
import { Role } from "../users/user.entity";

export const HasRoles = (...roles: Role[]) => SetMetadata("roles", roles);
```

次に Guard を作ります。
上の `has-roles.decorator.ts` で `SetMetadata("roles", roles)` のように書きました。

`HasRoles` というデコレータのメタデータは `roles` となります。
メタデータの `roles` に `Role[]` が格納されます。

```ts:src/auth/roles.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Role } from '../users/user.entity';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>('roles', [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user?.roles?.includes(role));
  }
}
```

以下の部分でメソッドに付与された `hasRoles` デコレータに渡された `Role[]` を取り出しています。
`Role[]`を取り出して、「必要な Role 」としているわけです。

```ts
const requiredRoles = this.reflector.getAllAndOverride<Role[]>("roles", [
  context.getHandler(),
  context.getClass(),
]);
```

## AppController にガードされたエンドポイントを設定してみる

```ts:src/app.controller.ts
import { Controller, Get, Post, UseGuards, Request } from '@nestjs/common';
import { AuthService } from './auth/auth.service';
import { HasRoles } from './auth/has-roles.decorator';
import { RolesGuard } from './auth/roles.guard';
import { LocalAuthGuard } from './auth/local-auth.guard';
import { JwtAuthGuard } from './auth/jwt-auth.guard';
import { Role } from './users/user.entity';

@Controller()
export class AppController {
  constructor(private readonly authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('auth/login')
  async login(@Request() req) {
    return this.authService.login(req.user);
  }

  @UseGuards(JwtAuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }

  @HasRoles(Role.Admin)
  @UseGuards(JwtAuthGuard, RolesGuard)
  @Get('admin')
  onlyAdmin(@Request() req) {
    return req.user;
  }

  @HasRoles(Role.User)
  @UseGuards(JwtAuthGuard, RolesGuard)
  @Get('user')
  onlyUser(@Request() req) {
    return req.user;
  }

  @HasRoles(Role.User)
  @UseGuards(JwtAuthGuard, RolesGuard)
  @Post('post')
  post(@Request() req) {
    return req.user;
  }
}

```

## 動作確認してみる

```console
npm run start:dev
```

ROLE が「USER」のユーザーでログインして、アクセストークンを取得します。

```console
curl -X POST http://localhost:3000/auth/login \
-H "Content-Type: application/json" \
--data-raw '{
"username": "naruto",
"password": "12345"
}'

# {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im5hcnV0byIsInN1YiI6MSwicm9sZXMiOlsidXNlciJdLCJpYXQiOjE2NjgyMzg5MDYsImV4cCI6MTY2ODI0MjUwNn0.tsfKbae4gU-DVNlU3JF2sxsIDyyKx0BrtsLS11zLEMQ"}
```

[jwt.io](https://jwt.io/)で受け取った access_token をデコードすると、 Payload には以下のような情報が入っていることがわかります。

```json
{
  "username": "naruto",
  "sub": 1,
  "roles": ["user"],
  "iat": 1668238906,
  "exp": 1668242506
}
```

JWT トークンがあれば叩ける `getProfile` という API にリクエストを投げてみます。

```ts
  @UseGuards(JwtAuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
```

```console
curl http://localhost:3000/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im5hcnV0byIsInN1YiI6MSwicm9sZXMiOlsidXNlciJdLCJpYXQiOjE2NjgyMzg5MDYsImV4cCI6MTY2ODI0MjUwNn0.tsfKbae4gU-DVNlU3JF2sxsIDyyKx0BrtsLS11zLEMQ"

# {"userId":1,"username":"naruto","roles":["user"]}%
```

では `Admin` ロールでないと叩けない以下の API にリクエストを投げたらどうなるでしょうか？

```ts
  @HasRoles(Role.Admin)
  @UseGuards(JwtAuthGuard, RolesGuard)
  @Get('admin')
  onlyAdmin(@Request() req) {
    return req.user;
  }
```

`403 Forbidden Error` が返ってきました。

```console
curl http://localhost:3000/admin -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6Im5hcnV0byIsInN1YiI6MSwicm9sZXMiOlsidXNlciJdLCJpYXQiOjE2NjgyMzg5MDYsImV4cCI6MTY2ODI0MjUwNn0.tsfKbae4gU-DVNlU3JF2sxsIDyyKx0BrtsLS11zLEMQ"

# {"statusCode":403,"message":"Forbidden resource","error":"Forbidden"}%
```

Admin ロールを持つユーザーのアクセストークンを取得してみましょう。

```console
curl -X POST http://localhost:3000/auth/login \
-H "Content-Type: application/json" \
--data-raw '{
"username": "sasuke",
"password": "12345"
}'

# {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6InNhc3VrZSIsInN1YiI6Miwicm9sZXMiOlsiYWRtaW4iXSwiaWF0IjoxNjY4MjM5MTg4LCJleHAiOjE2NjgyNDI3ODh9.9h2c5btALoGA7qsHcshOy7fRjhs-foYQm6RyPCVmSlE"}
```

Admin ロールのユーザーならしっかりとリクエストが返ってきました。

```console
curl http://localhost:3000/admin -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6InNhc3VrZSIsInN1YiI6Miwicm9sZXMiOlsiYWRtaW4iXSwiaWF0IjoxNjY4MjM5MTg4LCJleHAiOjE2NjgyNDI3ODh9.9h2c5btALoGA7qsHcshOy7fRjhs-foYQm6RyPCVmSlE"

# {"userId":2,"username":"sasuke","roles":["admin"]}%
```

## 参考

- [Role-Based Authorization with JWT Using NestJS](https://shpota.com/2022/07/16/role-based-authorization-with-jwt-using-nestjs.html)
- [Guards](https://docs.nestjs.com/guards)

## NestJS シリーズ

1. [NestJS+Nginx+MySQL の開発環境を Docker 化する](https://zenn.dev/fjsh/articles/nestjs-with-docker)
2. [NestJS+TypeORM 0.3 系で MySQL に接続してエラーなく起動する](https://zenn.dev/fjsh/articles/nestjs-typeorm-connect-mysql)
3. [NestJS+TypeORM 0.3 系でマイグレーションを実行する](https://zenn.dev/fjsh/articles/nestjs-typeorm-migration)
4. [NestJS で JWT 認証を実装する](https://zenn.dev/fjsh/articles/nestjs-auth-user-password)
5. [NestJS でロールベースの認証を実装する](https://zenn.dev/fjsh/articles/nestjs-role-based-authentication)
