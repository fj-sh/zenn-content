---
title: "NestJS+TypeORM0.3でseedデータを投入する"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NestJS"]
published: true
---

## はじめに

NestJS で作成したアプリケーションに初期データを投入する処理を作ってみます。

seed はアプリケーション動作に必須な所謂マスタテーブルのデータを最初に登録する際などに利用します。

テストに使うデータを投入する場合は fixtures を利用します（この記事は seed データを投入する方法について書いたものです）

[NestJS+TypeORM 0.3 で CRUD 操作を行うために TODO アプリを作ってみる](https://zenn.dev/fjsh/articles/nestjs-with-typeorm)が前提となっていますが、この記事単体でも概要は掴めるはずです。

TypeORM で データベースに接続する方法がよくわからん、という方は過去記事を参考にしてみてください。

## 対象読者

- NestJS の仕組みに乗っかって seed データを投入してみたい方
- NestJS のプロジェクトで OR マッパーに TypeORM を採用している方

## 環境

登場するライブラリのバージョンは以下のとおりです。

```json:package.json
{
    "@nestjs/common": "^9.0.0",
    "@nestjs/config": "^2.2.0",
    "@nestjs/core": "^9.0.0",
    "@nestjs/typeorm": "^9.0.1",
    "typeorm": "^0.3.10",
}
```

## ディレクトリ構造

今回の記事で登場するファイルがあるディレクトリ構成は以下のようになっています。

```
$ tree src
src
├── seed.ts
├── seeders
│   ├── seeder.module.ts
│   ├── seeder.ts
│   └── tasks
│       ├── data.ts
│       └── tasks.seeder.service.ts
```

## data.ts に投入するデータを定義する

```ts:src/seeders/tasks/data.ts
import { ITask } from '../../tasks/interfaces/task.interface';

export const tasks: ITask[] = [
  { name: 'コード' },
  { name: 'C言語' },
  { name: 'NestJS' },
  { name: 'C#' },
];

/**
 * タスクの型定義 src/tasks/interfaces/task.interface.ts に置いてある
 */
// export interface ITask {
//   name: string;
// }


```

## TasksSeederService にデータを投入する処理を書く

```ts:src/seeders/tasks/tasks.seeder.service.ts
import { Injectable } from '@nestjs/common';
import { TaskRepository } from '../../tasks/task.repository';
import { Task } from '../../tasks/entities/task.entity';
import { tasks } from './data';

@Injectable()
export class TasksSeederService {
  constructor(private readonly taskRepository: TaskRepository) {}

  create(): Array<Promise<Task>> {
    return tasks.map(async (task) => {
      const result = await this.taskRepository.findOneBy({ name: task.name });
      if (!result) {
        return Promise.resolve(
          await this.taskRepository
            .save(task)
            .catch((error) => Promise.reject(error)),
        );
      } else {
        return Promise.resolve(null);
      }
    });
  }
}
```

## SeederModule で必要なモジュールを import する

```ts:src/seeders/seeder.module.ts
import { TasksModule } from '../tasks/tasks.module';
import { Logger, Module } from '@nestjs/common';
import { DatabaseModule } from '../database/database.module';
import { Seeder } from './seeder';
import { TasksSeederService } from './tasks/tasks.seeder.service';

@Module({
  imports: [TasksModule, DatabaseModule],
  providers: [Logger, Seeder, TasksSeederService],
})
export class SeederModule {}

```

## Seeder で seed 処理をまとめる

上で作った `tasksSeederService` を使って seeding を行っています。
seed 対象のエンティティが増えたら、`seed()`から呼び出すメソッドを追加する形になります。

```ts:src/seeders/seeder.ts
import { Injectable, Logger } from '@nestjs/common';
import { TasksSeederService } from './tasks/tasks.seeder.service';

@Injectable()
export class Seeder {
  constructor(
    private readonly logger: Logger,
    private readonly tasksSeederService: TasksSeederService,
  ) {}

  async seed() {
    await this.tasks()
      .then((completed) => {
        this.logger.debug('Successfully completed seeding tasks...');

        Promise.resolve(completed);
      })
      .catch((error) => {
        this.logger.error('Failed seeding tasks...');

        Promise.reject(error);
      });
  }

  async tasks() {
    return await Promise.all(this.tasksSeederService.create())
      .then((createdTasks) => {
        this.logger.debug(
          '作成されたタスク数: ' +
            createdTasks.filter(
              (nullValueOrCreatedTask) => nullValueOrCreatedTask,
            ).length,
        );

        return Promise.resolve(true);
      })
      .catch((error) => Promise.reject(error));
  }
}
```

## package.json に起動スクリプトを追加

import 時にパスが参照できないエラーを解消するために、 `tsconfig-paths` をインストールします。

```console
npm i -D tsconfig-paths
```

`"scripts"` に以下を追加します。

```json
{
  "seed": "ts-node -r tsconfig-paths/register src/seed.ts"
}
```

## 実行した結果

```console
$ npm run seed

> nestjs-basic@0.0.1 seed
> ts-node -r tsconfig-paths/register src/seed.ts

[Nest] 14286  - 10/17/2022, 11:33:21 PM     LOG [NestFactory] Starting Nest application...
[Nest] 14286  - 10/17/2022, 11:33:21 PM     LOG [InstanceLoader] DatabaseModule dependencies initialized +28ms
[Nest] 14286  - 10/17/2022, 11:33:21 PM     LOG [InstanceLoader] TypeOrmModule dependencies initialized +0ms
[Nest] 14286  - 10/17/2022, 11:33:21 PM     LOG [InstanceLoader] ConfigHostModule dependencies initialized +0ms
[Nest] 14286  - 10/17/2022, 11:33:21 PM     LOG [InstanceLoader] ConfigModule dependencies initialized +1ms
[Nest] 14286  - 10/17/2022, 11:33:21 PM     LOG [InstanceLoader] TypeOrmCoreModule dependencies initialized +187ms
[Nest] 14286  - 10/17/2022, 11:33:21 PM     LOG [InstanceLoader] TypeOrmModule dependencies initialized +1ms
[Nest] 14286  - 10/17/2022, 11:33:21 PM     LOG [InstanceLoader] SeederModule dependencies initialized +0ms
[Nest] 14286  - 10/17/2022, 11:33:21 PM     LOG [InstanceLoader] TasksModule dependencies initialized +1ms
[Nest] 14286  - 10/17/2022, 11:33:21 PM   DEBUG 作成されたタスク数: 4
[Nest] 14286  - 10/17/2022, 11:33:21 PM   DEBUG Successfully completed seeding tasks...
[Nest] 14286  - 10/17/2022, 11:33:21 PM   DEBUG Seeding complete
```

`data.ts` に書いたデータがデータベースに投入されていました 👏

## 参考

- [An E2E test scenario in NestJS for a GraphQL mutation](https://selleo.com/til/posts/epi7h0iajl-an-e2e-test-scenario-in-nestjs-for-a-graphql-mutation)
