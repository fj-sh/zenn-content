---
title: "NestJSで Interceptor を使ってみる"
emoji: "😀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NestJS"]
published: true
---

[NestJS+TypeORM 0.3 で CRUD 操作を行う](https://zenn.dev/fjsh/articles/nestjs-with-typeorm) の続きです。
上記の記事では NestJS で簡単な CRUD 機能を作成しました。

ここで実験的に interceptor（インターセプター）の機能を追加してみます。

## インターセプターとは

関数の前後に割り込み処理を入れるための機能です。
たとえば、HTTP 通信に interceptor を入れることで、リクエストを投げる前やレスポンスを受け取った後に共通の処理を行うことができます。

公式で紹介されているのは以下のような用途です。

- メソッドの前後に追加のロジックをバインド
- 関数の結果を変換
- 関数から throw された例外を変換
- 基本関数の動作を拡張

[overview-interceptors](https://zenn.dev/kisihara_c/books/nest-officialdoc-jp/viewer/overview-interceptors)

インターセプタをどう設定するかは次の単位から選べます。

- クラス単位に設定
- メソッド単位に設定
- グローバルに設定

たとえば、NestJS を GraphQL サーバーとして動かしているとして、受け取ったクエリやミューテーションのバリデーションを行ったり、ログを出力する処理を挟み込むことができます。

ヘッダーを検証し、 Valid なら API 呼び出しを許可する、といった使い方もできます。

## 実際に使ってみる

以下のようなインターセプタを作ってみます。

関数を実行する「前」に受け取ったリクエストの内容をコンソールに出力し、関数を実行した「後」に処理にかかった時間を表示します。

```ts:src/interceptors/logging.interceptor.ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { Observable } from "rxjs";
import { tap } from "rxjs/operators";

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(
    context: ExecutionContext,
    next: CallHandler<any>
  ): Observable<any> | Promise<Observable<any>> {
    console.log("関数を実行するする前...");

    const http = context.switchToHttp();
    console.log("リクエストのタイプ", context.getType());
    console.log("HTTPバージョン", http.getRequest().httpVersion);
    console.log("URL", http.getRequest().url);
    console.log("メソッド", http.getRequest().method);

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() =>
          console.log(`関数を実行した後... かかった時間(${Date.now() - now}ms)`)
        )
      );
  }
}
```

TasksController クラスにインターセプタを設定してみます。

```ts:src/tasks/tasks.controller.ts
import {
  Controller,
  Get,
  Post,
  Body,
  Patch,
  Param,
  Delete,
  UseInterceptors,
  Logger,
} from '@nestjs/common';
import { TasksService } from './tasks.service';
import { CreateTaskDto } from './dto/create-task.dto';
import { UpdateTaskDto } from './dto/update-task.dto';
import { LoggingInterceptor } from '../interceptors/logging.interceptor';

@UseInterceptors(LoggingInterceptor)
@Controller('tasks')
export class TasksController {
  constructor(private readonly tasksService: TasksService) {}

  @Post()
  create(@Body() createTaskDto: CreateTaskDto) {
    return this.tasksService.create(createTaskDto);
  }

  @Get()
  findAll() {
    Logger.log('/tasks is called.');
    return this.tasksService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.tasksService.findOne(+id);
  }

  @Patch(':id')
  update(@Param('id') id: string, @Body() updateTaskDto: UpdateTaskDto) {
    return this.tasksService.update(+id, updateTaskDto);
  }

  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.tasksService.remove(+id);
  }
}

```

実際にリクエストを投げてみます。

```console
$ curl --location --request PATCH 'localhost:3000/tasks/2' \
    --header 'Content-Type: application/json' \
    --data-raw '{ "name": "サッカーする"}'
{"name":"サッカーする","id":2,"created_at":"2022-10-12T14:01:35.027Z","updated_at":"2022-10-13T14:38:14.711Z"}%
```

すると、ログには以下のように処理の前後でログが吐かれています。

```console
関数を実行するする前...
リクエストのタイプ http
HTTPバージョン 1.1
URL /tasks/2
メソッド PATCH
関数を実行した後... かかった時間(67ms)
```

## 特定の関数の返り値を変換する

公式にもあるサンプルです。
関数で処理した結果を `{ data: 結果オブジェクト }` の形に変換して返しています。

```ts:src/interceptors/transform.interceptor.ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from '@nestjs/common';
import { map, Observable } from 'rxjs';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, Response<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler<T>,
  ): Observable<Response<T>> | Promise<Observable<Response<T>>> {
    return next.handle().pipe(map((data) => ({ data })));
  }
}

```

### 変換インターセプタを使わない場合

上の変換インターセプタを使わない場合は、以下のようなレスポンスですが...

```console
$ curl --location --request GET 'localhost:3000/tasks/2' \
    --header 'Content-Type: application/json'
{"name":"サッカーする","id":2,"created_at":"2022-10-12T14:01:35.027Z","updated_at":"2022-10-13T14:38:14.711Z"}%
```

以下のようにコントローラのメソッドにインターセプタを設定すると...

```ts:src/tasks/tasks.controller.ts
  @UseInterceptors(TransformInterceptor)
  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.tasksService.findOne(+id);
  }
```

レスポンスは以下のようになります。

```console
$ curl --location --request GET 'localhost:3000/tasks/2' \
    --header 'Content-Type: application/json'
{"data":{"name":"サッカーする","id":2,"created_at":"2022-10-12T14:01:35.027Z","updated_at":"2022-10-13T14:38:14.711Z"}}%
```

## ExecutionContext に何が入ってくるか？

`ExecutionContext` は `ArgumentHost` を拡張したものです。

`ArgumentHost` はハンドラに渡される引数を取得するためのメソッドを提供する...と公式ガイドでは紹介されているのですが、実際に使ってみないとわかりづらいので、動かしてみます。

[Execution context](https://docs.nestjs.com/fundamentals/execution-context)

たとえば以下のような Interceptor を作ってみます。

```ts:src/app.interceptor.ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AppInterceptor implements NestInterceptor {
  intercept(
    context: ExecutionContext,
    next: CallHandler<any>,
  ): Observable<any> | Promise<Observable<any>> {
    console.log('context.getType(): ', context.getType());
    console.log('context.getClass(): ', context.getClass());
    console.log('context.getHandler(): ', context.getHandler());

    const [req] = context.getArgs();
    console.log('req?.body:', req?.body);

    return next.handle();
  }
}
```

作成した Interceptor を `AppController` に適用してみます。

```ts:src/app.controller.ts
import { Controller, Get, UseInterceptors } from '@nestjs/common';
import { AppService } from './app.service';
import { AppInterceptor } from './app.interceptor';

@UseInterceptors(AppInterceptor)
@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}
```

適当に Body に値を詰め込んでリクエストを投げてみると...

![](https://storage.googleapis.com/zenn-user-upload/4565e73099ff-20221123.jpg)

以下のような値がコンソールに表示されます。

```console
context.getType():  http
context.getClass():  [class AppController]
context.getHandler():  [Function: getHello]
req?.body: { message: 'こんにちは！' }
```

## Interceptor でヘッダの値を取得する

リクエストで投げられたヘッダの値を見て、何らかの確認を行いたい場合もあるかもしれません。
ここでは Interceptor でリクエストに含まれるヘッダの値を取得してみましょう。

以下のような Interceptor を作ります。

```ts:src/app.interceptor.ts
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AppInterceptor implements NestInterceptor {
  verifySignature(signature: string) {
    return signature === 'himitudayo';
  }

  intercept(
    context: ExecutionContext,
    next: CallHandler<any>,
  ): Observable<any> | Promise<Observable<any>> {
    if (context.getType() === 'http') {
      const request = context.switchToHttp().getRequest();
      const signature = request.headers['signature'];
      if (signature !== undefined) {
        console.log('signature:', signature);
        if (!this.verifySignature(signature)) {
          throw new Error('署名が不正です！貴様は誰だ？');
        }
      }
      console.log('署名は正しいな！通ってヨシ！');
    }
    return next.handle();
  }
}
```

Headers に `signature`: `himitudayo` を設定してリクエストを投げてみます。

![](https://storage.googleapis.com/zenn-user-upload/49f6c5443355-20221123.jpg)

コンソールには以下のように表示されます。

```console
signature: himitudayo
署名は正しいな！通ってヨシ！
```

では「正しくない署名」が投げられた場合はどうでしょう？

`signature: himitudayo` の部分を `signature: akunin` に変更してリクエストを投げてみます。

するとサーバーからは `Internal server error` が帰ってきます。

![](https://storage.googleapis.com/zenn-user-upload/ed8593adfc5a-20221123.jpg)

コンソールには以下のように表示されます。

```console
signature: akunin
[Nest] 57284  - 11/23/2022, 4:36:31 PM   ERROR [ExceptionsHandler] 署名が不正です！貴様は誰だ？
Error: 署名が不正です！貴様は誰だ？
```

実際には、リクエストボディを共通鍵でハッシュ化した値をクライアントから `signature` として送り、
サーバー側でも共通鍵で送られてきたリクエストボディをハッシュ化して、ハッシュ値がクライントから送信された `signature` と同じであることを確認して、改ざん検知などを行ったりします。
