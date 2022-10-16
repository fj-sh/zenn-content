---
title: "NestJSで作ったアプリケーションをGraphQLサーバとして動かしてみる"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NestJS"]
published: true
---

## はじめに

[NestJS+TypeORM 0.3 で CRUD 操作を行う](https://zenn.dev/fjsh/articles/nestjs-with-typeorm) の続きです。
上記の記事では NestJS で簡単な TODO アプリを実装しています。

これまで、 CRUD 機能を持つ REST API を作成しました。

この記事では REST API で作ってきた NestJS のプロジェクトを GraphQL サーバーとして動かす手順を紹介します。

## 対象読者

- NestJS で GraphQL サーバーを動かしてみたい人

## 環境

登場するライブラリのバージョンは以下のとおりです。

| ライブラリ              | バージョン |
| ----------------------- | ---------- |
| `@nestjs/core`          | `^9.0.0`   |
| `@nestjs/graphql`       | `^10.1.3`  |
| `graphql`               | `^16.6.0`  |
| `apollo-server-express` | `^3.10.3`  |

また、GraphQL の開発には

- スキーマファースト
- コードファースト

の 2 種類がありますが、ここではコードファーストのアプローチを取ります。

コードファーストとは、デコレータと TypeScript のクラスのみを使用して対応するものです。
スキーマファーストとは、 GraphQL SDL（スキーマ定義言語）をもとにして、TypeScript 定義を自動的に生成します。

## この記事を最後まで通してやってみた結果、どうなるか？

`http://localhost:3000/graphql` で Playground が開けるようになります。

GraphQL のクエリ（ミューテーション）を投げて、結果を取得できるようになります。

![](https://storage.googleapis.com/zenn-user-upload/1498b296f360-20221016.jpg)

## 必要なパッケージをインストールする

```console
npm i @nestjs/graphql @nestjs/apollo graphql apollo-server-express
```

参考: [Harnessing the power of TypeScript & GraphQL](https://docs.nestjs.com/graphql/quick-start)

## app.module.ts で `GraphQLModule` をインポートする

```ts title=src/app.module.ts
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";
import { DatabaseModule } from "./database/database.module";
import { TasksModule } from "./tasks/tasks.module";
import { GraphQLModule } from "@nestjs/graphql";
import { ApolloDriver, ApolloDriverConfig } from "@nestjs/apollo";
import { join } from "path";

@Module({
  imports: [
    DatabaseModule,
    TasksModule,
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: join(process.cwd(), "src/schema.gql"),
      sortSchema: true,
    }),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

[@nestjs/graphql バージョン 9 からバージョン 10 への移行のためのガイドライン](https://docs.nestjs.com/graphql/migration-guide)によると、`@nestjs/graphql`のバージョン 10 以降は、アプリケーションが使用するドライバを明示的に指定する必要があるそうです。

Apollo や Mercurius など、どの GraphQL ライブラリをプロジェクトで使用するか選択するためです。
この記事の例では `ApolloDriver` を利用しています。

## リゾルバを作成する

リゾルバとは、実際のデータ操作を行うものです。
特定のフィールドを返す関数になります。

公式の定義は以下です。

> リゾルバ（Resolvers）とは、GraphQL の operation（query や mutation や subscription）が、実際にどのような処理を行って返すのかという指示書です。
> [Graph の resolver を作成する](https://apollographql-jp.com/tutorial/resolvers/)

```ts:src/tasks/tasks.resolver.ts
import { Args, Mutation, Query, Resolver } from '@nestjs/graphql';
import { Task } from './entities/task.entity';
import { TasksService } from './tasks.service';
import { CreateTaskDto } from './dto/create-task.dto';
import { UpdateTaskDto } from './dto/update-task.dto';
import { DeleteTaskDto } from './dto/delete-task.dto';
import { DeleteResponseDto } from '../shared/dto/delete-response.dto';
import { FindTaskDto } from './dto/find-task.dto';

@Resolver()
export class TasksResolver {
  constructor(private readonly tasksService: TasksService) {}

  @Query((returns) => Task)
  async getTask(@Args('findTask') findTaskDto: FindTaskDto): Promise<Task> {
    return this.tasksService.findOne(findTaskDto.id);
  }

  @Query((returns) => [Task])
  async getAllTasks(): Promise<Task[]> {
    return this.tasksService.findAll();
  }

  @Mutation((returns) => Task)
  async createTask(@Args('newTask') newTaskDto: CreateTaskDto): Promise<Task> {
    return this.tasksService.create(newTaskDto);
  }

  @Mutation((returns) => Task)
  async updateTask(
    @Args('updateTask') updateTaskDto: UpdateTaskDto,
  ): Promise<Task> {
    return this.tasksService.update(updateTaskDto);
  }

  @Mutation((returns) => DeleteResponseDto)
  async deleteTask(@Args('deleteTask') deleteTaskDto: DeleteTaskDto) {
    return this.tasksService.remove(deleteTaskDto.id);
  }
}

```

## `@ObjectType()` アノテーションで GraphQL の 型（Type） を設定する

Entity のクラスに `@ObjectType()` アノテーションを付けることで、そのクラスを GraphQL の型として定義できます。
フィールドには`@Field()` アノテーションを付けます。

```ts:src/tasks/entities/task.entity.ts
import {
  Column,
  CreateDateColumn,
  PrimaryGeneratedColumn,
  UpdateDateColumn,
  Timestamp,
  Entity,
} from 'typeorm';
import { Field, ObjectType } from '@nestjs/graphql';

@ObjectType()
@Entity('tasks')
export class Task {
  @PrimaryGeneratedColumn({
    name: 'id',
    unsigned: true,
    type: 'smallint',
    comment: 'ID',
  })
  @Field()
  readonly id: number;

  @Column('varchar', { comment: 'タスク名' })
  @Field()
  name: string;

  @CreateDateColumn({ comment: '登録日時' })
  readonly created_at?: Timestamp;

  @UpdateDateColumn({ comment: '最終更新日時' })
  readonly updated_at?: Timestamp;

  constructor(name: string) {
    this.name = name;
  }
}

```

上のクラスは `schema.gql` に以下のように出力されます。

```gql
type Task {
  id: Float!
  name: String!
}
```

### `@InputType()` で入力値の型を定義する

GraphQL でミューテーションを投げるとき、インプットとなる引数を指定するかと思います。
具体的なスキーマ定義は以下のようなイメージです。

```gql
type Mutation {
  createTask(newTask: CreateTaskDto!): Task!
  deleteTask(deleteTask: DeleteTaskDto!): DeleteResponseDto!
  updateTask(updateTask: UpdateTaskDto!): Task!
}
```

`(newTask: CreateTaskDto!)` のような入力値を `@InputType()` アノテーションをつけて定義します。

```ts:src/tasks/dto/find-task.dto.ts
import { IsNotEmpty } from 'class-validator';
import { Field, InputType } from '@nestjs/graphql';

@InputType()
export class FindTaskDto {
  @IsNotEmpty({ message: 'IDは必須項目です' })
  @Field()
  id: number;
}
```

```ts:src/tasks/dto/update-task.dto.ts
import { IsNotEmpty, MaxLength } from 'class-validator';
import { Field, InputType } from '@nestjs/graphql';

@InputType()
export class UpdateTaskDto {
  @IsNotEmpty({ message: 'IDは必須項目です' })
  @Field()
  id: number;

  @IsNotEmpty({ message: '名前は必須項目です' })
  @MaxLength(255, { message: '名前は255文字以内で入力してください' })
  @Field()
  name: string;
}
```

```ts:src/tasks/dto/create-task.dto.ts
import { IsNotEmpty, MaxLength } from 'class-validator';
import { Field, InputType } from '@nestjs/graphql';

@InputType()
export class CreateTaskDto {
  @IsNotEmpty({ message: '名前は必須項目です。' })
  @MaxLength(255, { message: '名前は255文字以内で入力してください' })
  @Field()
  name: string;
}
```

```ts:src/tasks/dto/delete-task.dto.ts
import { IsNotEmpty } from 'class-validator';
import { Field, InputType } from '@nestjs/graphql';

@InputType()
export class DeleteTaskDto {
  @IsNotEmpty({ message: 'IDは必須項目です' })
  @Field()
  id: number;
}
```

```ts:src/shared/dto/delete-response.dto.ts
import { Field, ObjectType } from '@nestjs/graphql';

@ObjectType()
export class DeleteResponseDto implements DeleteResponse {
  @Field() message: string;
  @Field() delete: boolean;
}
```

```ts:src/shared/interfaces/delete-response.interface.ts
interface DeleteResponse {
  message: string;
  delete: boolean;
}
```

## GraphQL のクエリ実行と結果

`http://localhost:3000/graphql` を開いて GraphQL のクエリを投げてみます。

### getAllTasks

```gql
query {
  getAllTasks {
    id
    name
  }
}
```

結果：

```json
{
  "data": {
    "getAllTasks": [
      {
        "id": 4,
        "name": "洗濯"
      },
      {
        "id": 5,
        "name": "サッカー"
      },
      {
        "id": 3,
        "name": "プログラミング"
      }
    ]
  }
}
```

### findTask

```gql
query {
  getTask(findTask: { id: 4 }) {
    id
    name
  }
}
```

結果：

```json
{
  "data": {
    "getTask": {
      "id": 4,
      "name": "洗濯"
    }
  }
}
```

### createTask

```gql
mutation {
  createTask(newTask: { name: "Breakingdown" }) {
    id
    name
  }
}
```

結果：

```json
{
  "data": {
    "createTask": {
      "id": 6,
      "name": "Breakingdown"
    }
  }
}
```

### updateTask

```gql
mutation {
  updateTask(updateTask: { id: 6, name: "THE MATCH" }) {
    id
    name
  }
}
```

結果：

```json
{
  "data": {
    "updateTask": {
      "id": 6,
      "name": "THE MATCH"
    }
  }
}
```

### deleteTask

```gql
mutation {
  deleteTask(deleteTask: { id: 2 }) {
    message
    delete
  }
}
```

結果：

```json
{
  "data": {
    "deleteTask": {
      "message": "3を削除しました。",
      "delete": true
    }
  }
}
```

## エラー対応の TIPS

### `GraphQLError: Query root type must be provided.` というエラーが出たときは？

> GraphQL サーバーは少なくとも （schema.gql に）1 つの `@Query()` を持っている必要があります。
> `@Query()`がないと、 `applo-server` は例外を投げてサーバの起動に失敗します。
> [GraphQLError: Query root type must be provided](https://stackoverflow.com/questions/64105940/graphqlerror-query-root-type-must-be-provided)

`schema.gql` に以下のような Query が存在していないとエラーになる、ということです。

```gql
type Query {
  getAllTasks: [Task!]!
  getTask(findTask: FindTaskDto!): Task!
}
```

### `Error: Cannot determine a GraphQL output type for the "". Make sure your class is decorated with an appropriate decorator.`というエラーが出たときは？

`ObjectType()`と `@Field()` を Entity に指定する必要があります。

```ts
import {
  Column,
  CreateDateColumn,
  PrimaryGeneratedColumn,
  UpdateDateColumn,
  Timestamp,
  Entity,
} from "typeorm";
import { Field, ObjectType } from "@nestjs/graphql";

@ObjectType()
@Entity("tasks")
export class Task {
  @PrimaryGeneratedColumn({
    name: "id",
    unsigned: true,
    type: "smallint",
    comment: "ID",
  })
  @Field()
  readonly id: number;

  @Column("varchar", { comment: "タスク名" })
  @Field()
  name: string;

  @CreateDateColumn({ comment: "登録日時" })
  readonly created_at?: Timestamp;

  @UpdateDateColumn({ comment: "最終更新日時" })
  readonly updated_at?: Timestamp;

  constructor(name: string) {
    this.name = name;
  }
}
```
