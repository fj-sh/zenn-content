---
title: "SpectaQLを使ってNestJS+GraphQLのドキュメントを自動作成する"
emoji: "🐷"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs"]
published: true
---

普段、NestJS + GraphQL で開発していて、GraphQL のドキュメントを自動でいい感じに生成したい方のための記事です。

[Spectaql](https://github.com/anvilco/spectaql) を使って、schema.gql からドキュメントを作ります。

手順通りにやると、以下のようなユーザーインターフェースの HTML ドキュメントが自動生成できます。

![](https://storage.googleapis.com/zenn-user-upload/3c343d38bc5c-20230104.jpg)

一つの HTML ファイルとして表示されているため、ブラウザで簡単に検索できます。

QUERY + VARIABLES のサンプルも自動で生成されるので、お試しで GraphQL を叩いてみたいときなどに重宝します。

## 環境の準備

```shell
mkdir -p docs/graphql/html
```

```shell
touch docs/graphql/config.yml
```

## config ファイルの内容

Spectaql 用の config ファイルです。

```yaml:docs/graphql/config.yml
introspection:
  schemaFile:
    - ./src/schema.gql

servers:
  - url: https://dev-api.example.com/graphql
    description: GraphQL の API サーバーのエンドポイント

info:
  title: GraphQL の API サーバーのドキュメント
  description: GraphQL の API サーバーのドキュメントです。
```

公式のサンプルは以下です。

[https://github.com/anvilco/spectaql/blob/main/examples/config.yml](https://github.com/anvilco/spectaql/blob/main/examples/config.yml)

## ドキュメントの生成

```shell
npx spectaql --target-dir ./docs/graphql/html ./docs/graphql/config.yml
```

コマンドを実行すると、`./docs/graphql/html` 以下に `javascripts`, `index.html`, `stylesheets` が生成されます。

## ドキュメントを開く

以下のコマンドでドキュメントを開けます。

```shell
npx spectaql --target-dir ./docs/graphql/html -d ./docs/graphql/config.yml
```

## npm scripts に登録する

コマンドをいちいち打ち込むのは面倒なので、npm scripts に登録します。

```json:package.json
"scripts": {
  "doc:gen": "npx spectaql --target-dir ./docs/graphql/html ./docs/graphql/config.yml",
  "doc:open": "npx spectaql --target-dir ./docs/graphql/html -d ./docs/graphql/config.yml"
},
```

## schema.gql はどこで生成しているか？

spectaql は schema.gql を元にしてドキュメントを作ります。

NestJS + GraphQL のプロジェクトでは、`GraphQLModule` を import するときにオプションで `autoSchemaFile` を設定することで、 `schema.gql` を自動で生成することができます。

```ts:src/app.module.ts
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";
import { DatabaseModule } from "./database/database.module";
import { BookmarksModule } from "./bookmarks/bookmarks.module";
import { ApolloDriver, ApolloDriverConfig } from "@nestjs/apollo";
import { GraphQLModule } from "@nestjs/graphql";
import { join } from "path";
import { CategoriesModule } from "./categories/categories.module";
import { TagsModule } from "./tags/tags.module";
import { UsersModule } from "./users/users.module";

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      debug: true,
      playground: true,
      autoSchemaFile: join(process.cwd(), "src/schema.gql"),
      sortSchema: true,
    }),
    DatabaseModule,
    BookmarksModule,
    CategoriesModule,
    TagsModule,
    UsersModule,
  ],

  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

## GraphQLModule.forRoot 以外から schema.gql を自動で生成する方法

プロジェクトの設定で `GraphQLModule.forRoot` のオプションに `autoSchemaFile` を設定していない人もいるかもしれません。

その場合は NestJS の機能を使って、 Resolver から schema.gql を手動で作ることができます。

ドキュメントは [Generating SDL](https://docs.nestjs.com/graphql/generating-sdl) です。

```shell
mkdir scripts
```

```shell
touch scripts/generate-schema.ts
```

```ts:scripts/generate-schema.ts
import { NestFactory } from '@nestjs/core';
import { printSchema } from 'graphql/utilities';
import {
  GraphQLSchemaBuilderModule,
  GraphQLSchemaFactory,
} from '@nestjs/graphql';
import { BookmarksResolver } from '../src/bookmarks/bookmarks.resolver';
import { CategoriesResolver } from '../src/categories/categories.resolver';
import { TagsResolver } from '../src/tags/tags.resolver';
import { UsersResolver } from '../src/users/users.resolver';
import { writeFileSync } from 'fs';

async function generateSchema() {
  const app = await NestFactory.create(GraphQLSchemaBuilderModule);
  await app.init();

  const gqlSchemaFactory = app.get(GraphQLSchemaFactory);

  // ★　ここで Resolver を設定している
  const schema = await gqlSchemaFactory.create([
    BookmarksResolver,
    CategoriesResolver,
    TagsResolver,
    UsersResolver,
  ]);
  console.log(printSchema(schema));
  writeFileSync('src/schema.gql', printSchema(schema));
}

generateSchema();
```

以下のコマンドで schema を生成するスクリプトを実行できます。

```shell
npx ts-node scripts/generate-schema.ts
```

コマンドを生成するのは面倒なので、npm scirpts に登録します。

```json:package.json
"scripts": {
  "doc:gen:schema": "npx ts-node scripts/generate-schema.ts",
  "doc:gen": "npx spectaql --target-dir ./docs/graphql/html ./docs/graphql/config.yml",
  "doc:open": "npx spectaql --target-dir ./docs/graphql/html -d ./docs/graphql/config.yml"
},
```

## 参考

- [Harnessing the power of TypeScript & GraphQL](https://docs.nestjs.com/graphql/quick-start)
