---
title: "NestJS+TypeORM 0.3 でCRUD操作を行う"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NestJS"]
published: true
---

## NestJS アプリケーションを作成する

```
nest new .
npm run start:dev
```

`http://localhost:3000` にアクセスして、「Hello World!」と表示されたら成功です。

## Docker で PostgreSQL の環境を構築する

docker-compose.yml ファイルを作成します。

```shell
touch docker-compose.yml
```

```yml:docker-compose.yml
version: "3"

services:
  postgres:
    image: postgres:14
    volumes:
      - ./postgres/data:/var/lib/postgresql/data
      - ./postgres/initdb:/docker-entrypoint-initdb.d
    ports:
      - "${POSTGRES_PORT}:5432"
    environment:
      - TZ=Asia/Tokyo
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}

volumes:
  postgres:
    driver: local
```

`.env`を作成します。

```shell
touch .env
```

`.env` に PostgreSQL の設定を記載します。

```txt:.env
POSTGRES_HOST=127.0.0.1
POSTGRES_PORT=5432
POSTGRES_DB=todo
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
```

Docker を起動します。

```shell
mkdir -p postgres/data postgres/initdb
docker compose up
```

もし余計なコンテナが残っていて Docker が起動しない場合は、以下の記事を参考に不要なコンテナやイメージを掃除します。

[docker system prune で Docker のお掃除をする話](https://ensekitt.hatenablog.com/entry/2018/07/03/200000)

## TypeORM のインストール

TypeORM をインストールします。

```shell
npm install @nestjs/typeorm typeorm reflect-metadata pg
```

2022 年 9 月 23 日時点では、`0.3.10` がインストールされます。

PostgreSQL の設定情報を環境変数から読み込むために、 `@nestjs/config` をインストールします。

<!--
バリデーションを行うための `Joi` をインストールします。

```shell
npm install @nestjs/config joi
``` -->

```shell
npm install @nestjs/config
```

## データベース接続用のモジュールを作成する

database モジュールを作成します。

```shell
nest g mo database
```

[CLI command reference](https://docs.nestjs.com/cli/usages#nest-generate)

```ts:src/database/database.module.ts
import { Module } from "@nestjs/common";
import { TypeOrmModule } from "@nestjs/typeorm";
import { ConfigModule, ConfigService } from "@nestjs/config";

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      imports: [
        ConfigModule.forRoot({
          envFilePath: [".env"],
        }),
      ],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        type: "postgres",
        host: configService.get("POSTGRES_HOST"),
        port: configService.get("POSTGRES_PORT"),
        database: configService.get("POSTGRES_DB"),
        username: configService.get("POSTGRES_USER"),
        password: configService.get("POSTGRES_PASSWORD"),
        entities: [],
        synchronize: true,
      }),
    }),
  ],
})
export class DatabaseModule {}
```

:::note

`synchronize` を `true` に設定すると、エンティティの変更を自動的に Database に反映してくれます。
本番環境では意図せずデータが削除されてしまうこともあるので、明示的にマイグレーションするのが望ましいです。

:::

## リソースの作成

```shell
nest g resource tasks
? What transport layer do you use? REST API
? Would you like to generate CRUD entry points? Yes
```

## エンティティの作成

```ts:src/tasks/entities/task.entity.ts
import {
  Column,
  CreateDateColumn,
  PrimaryGeneratedColumn,
  UpdateDateColumn,
  Timestamp,
  Entity,
} from "typeorm";

@Entity("tasks")
export class Task {
  @PrimaryGeneratedColumn({
    name: "id",
    unsigned: true,
    type: "smallint",
    comment: "ID",
  })
  readonly id: number;

  @Column("varchar", { comment: "タスク名" })
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

エンティティ作成後、`database.module.ts` の `entities` に `Task` を追加します。

```ts:src/database/database.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { Task } from '../tasks/entities/task.entity';

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
        type: 'postgres',
        host: configService.get('POSTGRES_HOST'),
        port: configService.get('POSTGRES_PORT'),
        database: configService.get('POSTGRES_DB'),
        username: configService.get('POSTGRES_USER'),
        password: configService.get('POSTGRES_PASSWORD'),
        entities: [Task],
        synchronize: true,
      }),
    }),
  ],
})
export class DatabaseModule {}

```

NestJS を再起動して、DataGrip などで確認すると、テーブルが作られていることがわかります。

DB: todo
USER: postgres
PASSWORD: postgres

です。

## Module の作成

```ts:src/tasks/tasks.module.ts
import { Module } from "@nestjs/common";
import { TasksService } from "./tasks.service";
import { TasksController } from "./tasks.controller";
import { TypeOrmModule } from "@nestjs/typeorm";
import { Task } from "./entities/task.entity";

@Module({
  imports: [TypeOrmModule.forFeature([Task])],
  exports: [TypeOrmModule],
  controllers: [TasksController],
  providers: [TasksService],
})
export class TasksModule {}
```

## バリデーションを行う

```shell
npm i --save class-validator class-transformer
```

バリデーション用の DTO を作成します。

```ts:src/tasks/dto/create-task.dto.ts
import { IsNotEmpty, MaxLength } from "class-validator";

export class CreateTaskDto {
  @IsNotEmpty({ message: "名前は必須項目です" })
  @MaxLength(255, { message: "名前は255文字以内で入力してください" })
  name: string;
}
```

```ts:src/tasks/dto/update-task.dto.ts
import { IsNotEmpty, MaxLength } from 'class-validator';

export class UpdateTaskDto {
  @IsNotEmpty({ message: 'IDは必須項目です' })
  id: number;

  @IsNotEmpty({ message: '名前は必須項目です' })
  @MaxLength(255, { message: '名前は255文字以内で入力してください' })
  name: string;
}

```

`main.ts` にバリデーションを適用するための処理を追加します。

```ts:src/main.ts
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { ValidationPipe } from "@nestjs/common";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

## Service の作成

```ts:src/tasks/tasks.service.ts
import {
  Injectable,
  InternalServerErrorException,
  NotFoundException,
} from "@nestjs/common";
import { CreateTaskDto } from "./dto/create-task.dto";
import { UpdateTaskDto } from "./dto/update-task.dto";
import { Task } from "./entities/task.entity";
import { DeleteResult, Repository } from "typeorm";
import { InjectRepository } from "@nestjs/typeorm";

@Injectable()
export class TasksService {
  constructor(
    @InjectRepository(Task) private taskRepository: Repository<Task>
  ) {}

  async create(createTaskDto: CreateTaskDto): Promise<Task> {
    return await this.taskRepository
      .save({ name: createTaskDto.name })
      .catch((e) => {
        throw new InternalServerErrorException(e.message);
      });
  }

  async findAll() {
    return await this.taskRepository.find().catch((e) => {
      throw new InternalServerErrorException(e.message);
    });
  }

  async findOne(id: number) {
    const task = await this.taskRepository.findOneBy({ id });

    if (!task) {
      throw new NotFoundException(
        `${id}に一致するデータが見つかりませんでした。`
      );
    }

    return task;
  }

  async update(id: number, updateTaskDto: UpdateTaskDto) {
    await this.taskRepository
      .update(id, { name: updateTaskDto.name })
      .catch((e) => {
        throw new InternalServerErrorException(e.message);
      });
    const updatedPost = await this.taskRepository.findOneBy({ id });
    if (updatedPost) {
      return updatedPost;
    }

    throw new NotFoundException(
      `${id}に一致するデータが見つかりませんでした。`
    );
  }

  async remove(id: number): Promise<DeleteResult> {
    const response = await this.taskRepository.delete(id).catch((e) => {
      throw new InternalServerErrorException(e.message);
    });

    if (!response.affected) {
      throw new NotFoundException(
        `${id} に一致するデータが見つかりませんでした`
      );
    }

    return response;
  }
}
```

## Controller の作成

```ts:src/tasks/tasks.controller.ts
import {
  Controller,
  Get,
  Post,
  Body,
  Patch,
  Param,
  Delete,
} from "@nestjs/common";
import { TasksService } from "./tasks.service";
import { CreateTaskDto } from "./dto/create-task.dto";
import { UpdateTaskDto } from "./dto/update-task.dto";
import { Task } from "./entities/task.entity";

@Controller("tasks")
export class TasksController {
  constructor(private readonly tasksService: TasksService) {}

  @Post()
  async create(@Body() createTaskDto: CreateTaskDto): Promise<Task> {
    return await this.tasksService.create(createTaskDto);
  }

  @Get()
  async findAll(): Promise<Task[]> {
    return await this.tasksService.findAll();
  }

  @Get(":id")
  findOne(@Param("id") id: string) {
    return this.tasksService.findOne(+id);
  }

  @Patch(":id")
  update(@Param("id") id: string, @Body() updateTaskDto: UpdateTaskDto) {
    return this.tasksService.update(+id, updateTaskDto);
  }

  @Delete(":id")
  remove(@Param("id") id: string) {
    return this.tasksService.remove(+id);
  }
}
```

## 動作確認

動作確認してみます。

### 登録

```shell
curl --location --request POST 'localhost:3000/tasks' \
    --header 'Content-Type: application/json' \
    --data-raw '{"name": "洗濯をする"}'
```

### 全件取得

```shell
curl --location --request GET 'localhost:3000/tasks' \
    --header 'Content-Type: application/json'
```

### 1 件取得する

```shell
curl --location --request GET 'localhost:3000/tasks/1' \
    --header 'Content-Type: application/json'
```

### 更新

```shell
curl --location --request PATCH 'localhost:3000/tasks/1' \
    --header 'Content-Type: application/json' \
    --data-raw '{"name": "勉強する"}'
{"statusCode":400,"message":["IDは必須項目です"],"error":"Bad Request"}

curl --location --request PATCH 'localhost:3000/tasks/1' \
    --header 'Content-Type: application/json' \
    --data-raw '{"id":1, "name": "勉強する"}'
{"name":"勉強する","id":1,"created_at":"2022-10-12T13:39:18.123Z","updated_at":"2022-10-12T13:39:51.396Z"}%
```

### 削除

```shell
curl --location --request DELETE 'localhost:3000/tasks/1' \
    --header 'Content-Type: application/json'
```
