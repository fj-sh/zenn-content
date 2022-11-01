---
title: "NestJS+TypeORM 0.3系でマイグレーションを実行する"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs", "typeorm"]
published: true
---

マイグレーションとは、モデルの変更をデータベースに変更させるための SQL です。

NestJS と TypeORM を組み合わせて使う場合、マイグレーションは TypeORM の機能を使います。

## エンティティの作成

NestJS CLI で CRUD リソースを作成します。

`res` は `resource` の略です。
`--no-spec` でテストの作成を省略します。

```shell
nest g --no-spec res product
```

[CLI command reference](https://docs.nestjs.com/cli/usages)

自動生成される DTO で `@nestjs/mapped-types` が必要なのでインストールします。

```shell
npm i @nestjs/mapped-types
```

- [@nestjs/mapped-types](https://www.npmjs.com/package/@nestjs/mapped-types)
- [Mapped types](https://docs.nestjs.com/openapi/mapped-types)

npm のモジュールを install したときは Docker を再度ビルドします。

```shell
docker compose build --no-cache
```

## Entity を作成する

`nest g --no-spec res product` で `src/product/entities` 以下に `product.entity.ts` が作られます。

以下のように Entity を作ってみてください。

```ts:src/product/entities/product.entity.ts
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
export class Product {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @Column()
  description: string;

  @Column()
  image: string;

  @Column()
  price: number;
}
```

## Entity を データソースに設定する

マイグレーション時に読み込むためのファイルを作成します。

```shell
touch src/database/database.source.ts
```

`src/database/database.module.ts` に NestJS で使うためのデータベースの接続定義があるので、2 重管理をできるだけ防ぐために、まず NestJS で使う`src/database/database.module.ts` を以下のように編集します。

```ts:src/database/database.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Product } from '../product/entities/product.entity';

export const databaseEntities = [Product];
export const migrationFilesDir = 'src/migration/*.ts';

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
        entities: databaseEntities,
        synchronize: false,
        migrations: [migrationFilesDir],
      }),
    }),
  ],
})
export class DatabaseModule {}
```

上の方にある `databaseEntities` に作成した Entity を入れています。

これを`src/database/database.source.ts` で使います。

`src/database/database.source.ts` は次のようにします。

```ts:src/database/database.source.ts
import { databaseEntities, migrationFilesDir } from './database.module';
import 'dotenv/config';
import { DataSource } from 'typeorm';

export const AppDataSource = new DataSource({
  type: 'mysql',
  host: process.env.DATABASE_HOST,
  port: Number(process.env.DATABASE_PORT),
  database: process.env.DATABASE_NAME,
  username: process.env.DATABASE_USER,
  password: process.env.DATABASE_PASSWORD,
  entities: databaseEntities,
  synchronize: false,
  migrations: [migrationFilesDir],
});
```

## マイグレーションを実行する

マイグレーション用のファイルを格納するためのディレクトリを作成します。

```shell
mkdir src/migration
```

### entity からマイグレーションファイルを作成する

Docker のシェルを起動します。

```shell
docker exec -it nest /bin/sh
```

Docker のコンテナの中で以下のコマンドを実行すると、`src/migration`　以下にマイグレーションファイルが作成されます。

`-d` でデータソースファイルを指定します。

```shell
/app $ npx typeorm-ts-node-esm migration:generate src/migration/CreateProduct -d src/database/database.source.ts

# Migration /app/src/migration/1667314200578-CreateProduct.ts has been generated successfully.
```

作った Entity からマイグレーションファイルを作ってくれます。

ここでは `1667314200578-CreateProduct.ts` という名前でマイグレーションファイルができました。

### マイグレーションを実行する

次のコマンドでマイグレーションを実行します。

```shell
/app $ npx typeorm-ts-node-esm migration:run -d src/database/database.source.ts
```

すると、実際にデータベースに `product` テーブルを作ってくれます。

その他のコマンドも見ておきましょう。

### migration:show

すべてのマイグレーションと実行の有無を以下のコマンドで確認できます。

```shell
/app $ npx typeorm-ts-node-esm migration:show -d src/database/database.source.ts
[X] CreateProduct1667314200578
```

### migration:revert

`migration:revert` を実行すると、前回実行したマイグレーションを元に戻すことができます。

```shell
/app $ npx typeorm-ts-node-esm migration:revert -d src/database/database.source.ts

query: SELECT VERSION() AS `version`
query: SELECT * FROM `INFORMATION_SCHEMA`.`COLUMNS` WHERE `TABLE_SCHEMA` = 'nestdb' AND `TABLE_NAME` = 'migrations'
query: SELECT * FROM `nestdb`.`migrations` `migrations` ORDER BY `id` DESC
1 migrations are already loaded in the database.
CreateProduct1667314200578 is the last executed migration. It was executed on Tue Nov 01 2022 14:50:00 GMT+0000 (Coordinated Universal Time).
Now reverting it...
query: START TRANSACTION
query: DROP TABLE `product`
query: DELETE FROM `nestdb`.`migrations` WHERE `timestamp` = ? AND `name` = ? -- PARAMETERS: [1667314200578,"CreateProduct1667314200578"]
Migration CreateProduct1667314200578 has been  reverted successfully.
query: COMMIT
```

データベースを見てみると、product テーブルが消えていました。
`migration:show` でも様子を確認できます。

```shell
/app $ npx typeorm-ts-node-esm migration:show -d src/database/database.source.ts
[ ] CreateProduct1667314200578
```

ここで `npx typeorm-ts-node-esm migration:run -d src/database/database.source.ts` を実行すると、また product テーブルが作成されます。

## 参考

- [TypeORM 0.3 系のマイグレーション](https://qiita.com/Aurum64/items/f5962bd2a643447dbef9)
- [Using CLI](https://typeorm.io/using-cli)

## NestJS シリーズ

1. [NestJS+Nginx+MySQL の開発環境を Docker 化する](https://zenn.dev/fjsh/articles/nestjs-with-docker)
2. [NestJS+TypeORM 0.3 系で MySQL に接続してエラーなく起動する](https://zenn.dev/fjsh/articles/nestjs-typeorm-connect-mysql)
3. [NestJS+TypeORM 0.3 系でマイグレーションを実行する](https://zenn.dev/fjsh/articles/nestjs-typeorm-migration)
