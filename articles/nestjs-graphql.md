---
title: "NestJS+GraphQL+TypeORM"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs"]
published: false
---

- [NestJS で GraphQL サーバを実装する](https://zenn.dev/hakushun/articles/7daac74ae9af25) の記事で、2022 年 10 月時点でエラーが出る部分を修正する
  - オンメモリで動く ToDo アプリの「データ取得」を実装する
- ToDo アプリに登録・更新・削除機能を追加する
- 上の記事で作成した Todo アプリとデータベースを連携させる

## NestJS のプロジェクトを作成する

```console
mkdir nestjs-graphql
cd nestjs-graphql
nest new .

? Which package manager would you ❤️  to use? (Use arrow keys)
❯ npm
```

## 必要なパッケージをインストールする

```console
npm i @nestjs/graphql @nestjs/apollo graphql apollo-server-express
```

参考: [Harnessing the power of TypeScript & GraphQL](https://docs.nestjs.com/graphql/quick-start)

## `GraphQLModule` をインポートする

[元記事](<(https://zenn.dev/hakushun/articles/7daac74ae9af25)>)と異なる部分です。

[@nestjs/graphql バージョン 9 からバージョン 10 への移行のためのガイドライン](https://docs.nestjs.com/graphql/migration-guide)では、アプリケーションが使用するドライバを明示的に指定する必要があるそうです。

Apollo や Mercurius など、どの GraphQL ライブラリをプロジェクトで使用するか選択するためです。
以下では `ApolloDriver` を利用しています。

```ts title=src/app.module.ts
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";
import { GraphQLModule } from "@nestjs/graphql";
import { ApolloDriver, ApolloDriverConfig } from "@nestjs/apollo";
import { join } from "path";
import { TodoModule } from "./todo/todo.module";

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: join(process.cwd(), "src/schema.gql"),
      sortSchema: true,
    }),
    TodoModule,
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```
