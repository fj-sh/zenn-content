---
title: "NestJS+TypeORM 0.3系でMySQLに接続してエラーなく起動する"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs", "typeorm"]
published: true
---

Docker で動かしている NestJS から、同じく Docker 上で動かしている MySQL に接続してみます。

TypeORM のバージョンは "0.3.10" です。 TypeORM は `0.2` 系と `0.3` 系で微妙に挙動が違うので注意してください。

## TypeORM と MySQL ドライバのインストール

TypeORM を使うために必要なライブラリをインストールします。

```
npm install @nestjs/typeorm typeorm reflect-metadata mysql @nestjs/config
```

データベースに MySQL でなく PostgreSQL を使っている方もいるかもしれません。

使えるドライバーの例は [typeorm](https://www.npmjs.com/package/typeorm) に詳しく載っています。

## 前提となる環境を確認する

MySQL も NestJS も Docker で立ち上げています。

MySQL のサービス名は `db` です。

```yml:docker-compose.yml
version: '3'
services:
  nest:
    build:
      context: .
      dockerfile: Dockerfile # 本番環境は Dockerfile-prod を指定する
    container_name: nest
    depends_on:
      - db
    volumes:
      - ./src:/app/src
      - .env:/app/.env

  nginx:
    build:
      context: .
      dockerfile: Dockerfile-nginx
    container_name: nest-nginx
    env_file:
      - .env
    depends_on:
      - nest
    environment:
      - NGINX_SERVER_NAME=${NGINX_SERVER_NAME}
      - NEST_HOST=${NEST_HOST}
      - NEST_PORT=${NEST_PORT}
      - NGINX_MAX_BODY=${NGINX_MAX_BODY}
      - NGINX_PORT=${NGINX_PORT}
    ports:
      - ${NGINX_PORT}:${NGINX_PORT}

  db:
    image: mysql:5.7
    container_name: nest-db
    env_file:
      - .env
    environment:
      MYSQL_DATABASE: ${DATABASE_NAME}
      MYSQL_USER: ${DATABASE_USER}
      MYSQL_PASSWORD: ${DATABASE_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DATABASE_ROOT_PASSWORD}
    ports:
      - "${DATABASE_PORT}:3306"
    volumes:
      - ./docker/db/data:/var/lib/mysql
      - ./docker/db/my.cnf:/etc/mysql/conf.d/my.cnf

```

環境変数には以下の値を入れています。

`DATABASE_HOST` は `localhost` ではなく、 `db` としています。

```txt:.env
DATABASE_HOST=db
DATABASE_NAME=nestdb
DATABASE_USER=nestuser
DATABASE_PASSWORD=nestpass
DATABASE_ROOT_PASSWORD=nestrootpass
DATABASE_PORT=3306
NGINX_SERVER_NAME=localhost
NEST_HOST=nest
NEST_PORT=3000
NGINX_MAX_BODY=100M
NGINX_PORT=80
```

## データベース接続用のモジュールを作成する

NestJS CLI で database モジュールを作成します。

```shell
nest g mo database
```

`g` は generate、 `mo` は module です。

コマンドを実行すると、 `app.module.ts` には自動で `DatabaseModule` がインポートされています。

```ts:src/app.module.ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { DatabaseModule } from './database/database.module';

@Module({
  imports: [DatabaseModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

## database.module.ts に接続設定を定義する

`.env` から環境設定を読み込み、 MySQL ドライバを通じてデータベースに接続しています。

```ts:src/database/database.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      imports: [
        ConfigModule.forRoot({
          envFilePath: ['.env'],
        }),
      ],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        type: 'mysql',
        host: configService.get('DATABASE_HOST'),
        port: configService.get('DATABASE_PORT'),
        database: configService.get('DATABASE_NAME'),
        username: configService.get('DATABASE_USER'),
        password: configService.get('DATABASE_PASSWORD'),
        entities: [],
        synchronize: false,
      }),
    }),
  ],
})
export class DatabaseModule {}
```

## Docker を起動してもエラーが出ないことを確認する

Docker を立ち上げて、エラーなく起動できることを確認します。

```console
docker compose up
```

エラーが出ませんね。
これで Docker 上で NestJS + TypeORM から MySQL に接続できました。

次は Entity を定義して、データベースにマイグレーションしてみます。

参考：[NestJS - Database](https://docs.nestjs.com/techniques/database)

## NestJS シリーズ

1. [NestJS+Nginx+MySQL の開発環境を Docker 化する](https://zenn.dev/fjsh/articles/nestjs-with-docker)
2. [NestJS+TypeORM 0.3 系で MySQL に接続してエラーなく起動する](https://zenn.dev/fjsh/articles/nestjs-typeorm-connect-mysql)
3. [NestJS+TypeORM 0.3 系でマイグレーションを実行する](https://zenn.dev/fjsh/articles/nestjs-typeorm-migration)
4. [NestJS で JWT 認証を実装する](https://zenn.dev/fjsh/articles/nestjs-auth-user-password)
5. [NestJS でロールベースの認証を実装する](https://zenn.dev/fjsh/articles/nestjs-role-based-authentication)
