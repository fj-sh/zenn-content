---
title: "NestJS でデコレータからRequestContextに入っているユーザー情報を取得する"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs"]
published: true
---

NestJS + GraphQL でアプリケーションを作っていて、

「カスタマイズした `Guard` で検証したユーザーの情報をリクエストコンテキストに入れて、`Resolver` や `Controller` の関数で、デコレータ経由でユーザー情報を取得したい。どうすればいい？」

とやり方を探している人のための記事です。

`@Request() request` を引数に渡して、`request.user` ユーザー情報を取得しようとしても `undefined` になってしまって困っている、という人には何か役に立てるかもしれません。

Resolver のパラメータに `@CurrentUser user: UserModel` を渡して、引数からユーザー情報を取得する方法を紹介します。

## 結論

以下のようなデコレータを作って、 `ExecutionContext` からユーザー情報を引っこ抜きます（「`ExecutionContext` に入っているユーザー情報」は `Guard` で付与しています）

```ts:user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';
import { UserModel } from '../../users/user.model';

export const CurrentUser = createParamDecorator(
  (data: unknown, context: ExecutionContext) => {
    const ctx = GqlExecutionContext.create(context);
    const requestUser = ctx.getContext().req['user'];
    const currentUser: UserModel = {
      name: requestUser.name,
      uid: requestUser.uid,
    };
    return currentUser;
  },
);
```

デコレータを利用する側は以下のように、パラメータとしてユーザー情報を取得できます。
以下は GraphQL の Resolver で先ほど作った `@CurrentUser` デコレータを使っている例です。

```ts:
  @Mutation(() => CreateBookmarkOutput)
  @UseGuards(AuthGuard)
  createBookmark(
    @CurrentUser() user: UserModel,
    @Args('createBookmarkInput') createBookmarkInput: CreateBookmarkInput,
  ): Promise<Result> {
    const userUid = user.uid;
    return this.bookmarksService.create(createBookmarkInput, userUid);
  }
```

`@CurrentUser() user: UserModel` でユーザー情報を取得できます。

## Guard で JWT トークンからユーザー情報を取得して、リクエストコンテキストに付与する

リクエストコンテキストからユーザー情報を取得して、デコレータ経由でそのユーザーの情報を受け取る方法を見てきました。

1. `Guard` でトークンを検証
2. トークンからユーザー情報を取得（Firebase の認証機能を使っている）
3. 取得したユーザー情報をリクエストコンテキストに付与する
4. リクエストコンテキストに付与されたユーザー情報を `@CurrentUser` デコレータ経由で取得する（★ 上の例で説明したもの）
5. `@CurrentUser` 経由で、 Resolver や Controller でユーザーの情報を受け取る（★ 上の例で説明したもの）

という流れでした。

以下は、`Guard` でトークンを検証して、ユーザー情報をリクエストコンテキスに付与している部分です。

```ts:auth.guard.ts
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from "@nestjs/common";
import { AuthService } from "../auth.service";
import { GqlExecutionContext } from "@nestjs/graphql";

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly authService: AuthService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const ctx = GqlExecutionContext.create(context);
    const requestHeaders = ctx.getContext().req.headers;
    if (!requestHeaders) {
      throw new Error("ヘッダが正しく設定されていません。");
    }
    const idToken: string = requestHeaders.authorization.replace("Bearer ", "");
    try {
      const user = await this.authService.validateUser(idToken);
      ctx.getContext().req["user"] = user;
      return user !== undefined;
    } catch (error) {
      throw new UnauthorizedException("認証情報が正しくありません。");
    }
  }
}
```

上のコードでは以下のようなことをしています。

- ヘッダー(`ctx.getContext().req.header`)から JWT トークンを取り出します
- JWT トークンを検証し、ユーザー情報を取り出します（`await this.authService.validateUser(idToken);` のところ）
- 取り出したユーザーの情報をリクエストに付与する（`ctx.getContext().req["user"] = user;`のところ）

認証部分の内容の詳細は、以下の記事を参照してください。Firebase で JWT トークンを検証しています。

[Next.js + Firebase で「Google でログイン」して、NestJS でトークンを検証・認可を行う](https://zenn.dev/fjsh/articles/nextjs-login-with-google)

さて、ここまで `Guard` でリクエストコンテキストにユーザーを設定して、ユーザーをデコレータで取得できるようにして...という手順を見てきました。

```ts
  @UseGuards(JwtAuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user;
  }
```

[Implementing Passport JWT](https://docs.nestjs.com/security/authentication#implementing-passport-jwt)
