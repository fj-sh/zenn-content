---
title: "NestJSで複数の画像をアップロードして、ローカルストレージに保存する"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs"]
published: false
---

[NestJS で 1 つの画像をアップロードして、ローカルストレージに保存する](https://zenn.dev/fjsh/articles/nestjs-save-single-image) の続編ですが、一通りの作成手順を記録します。

今回は複数の画像をアップロードしてみます。

最初に `@types/multer` をインストールします。

```shell
npm i -D @types/multer
```

## interceptor を作成する

Controller で使うための interceptor を作成します。

```ts:src/posts/interceptors/index.ts
import { Request } from 'express';
import { parse } from 'path';
import { BadRequestException } from '@nestjs/common';

export const editFileName = (
  req: Request,
  file: Express.Multer.File,
  callback: (error: Error | null, filename: string) => void,
) => {
  const fileBaseName = parse(file.originalname).name;
  const fileExtName = parse(file.originalname).ext;
  const randomName = Array(4)
    .fill(null)
    .map(() => Math.round(Math.random() * 16).toString(16))
    .join('');
  callback(null, `${fileBaseName}-${randomName}${fileExtName}`);
};

export const imageFileFilter = (
  req: Request,
  file: {
    fieldname: string;
    originalname: string;
    encoding: string;
    mimetype: string;
    size: number;
    destination: string;
    filename: string;
    path: string;
    buffer: Buffer;
  },
  callback: (error: Error | null, acceptFile: boolean) => void,
) => {
  if (!file.originalname.match(/\.(jpg|jpeg|png|webp|gif|avif)$/)) {
    return callback(
      new BadRequestException('おいおい、画像だけを送ってくれよな？'),
      false,
    );
  }
  callback(null, true);
};

```

## Controller を作成する

```ts:src/posts/posts.controller.ts
import {
  Controller,
  Post,
  UseInterceptors,
  UploadedFiles,
} from "@nestjs/common";

import { FileFieldsInterceptor } from "@nestjs/platform-express";
import { diskStorage } from "multer";
import { editFileName, imageFileFilter } from "./interceptors";

@Controller("posts")
export class PostsController {
  @Post("files")
  @UseInterceptors(
    FileFieldsInterceptor([{ name: "files", maxCount: 4 }], {
      storage: diskStorage({
        destination: "./files",
        filename: editFileName,
      }),
      fileFilter: imageFileFilter,
      limits: { fileSize: 1024 * 1024 * 4 },
    })
  )
  uploadFiles(
    @UploadedFiles()
    files: Array<Express.Multer.File>
  ): void {
    console.log(files);
  }
}
```

## 作ったコントローラをモジュールに登録する

```ts:src/posts/posts.module.ts
import { Module } from '@nestjs/common';
import { PostsService } from './posts.service';
import { PostsController } from './posts.controller';

@Module({
  controllers: [PostsController],
  providers: [PostsService],
})
export class PostsModule {}
```

```ts:src/app.module.ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { PostsModule } from './posts/posts.module';

@Module({
  imports: [PostsModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

## Postman でファイルを送信してみる

`http://localhost:3000/posts/files` に対してリクエストを投げます。

`Body` > `form-data` の KEY に `files` を設定し、VALUE にファイルを選択します。
（KEY のセルの右端に`Text`か`File` を選ぶドロップダウンがあるので、`File`を選択します）

![](https://storage.googleapis.com/zenn-user-upload/fb7b6ad063ed-20221212.jpg)

ファイルを 4 つ選んでリクエストを投げると、プロジェクトの `files` ディレクトリ以下に 4 つのファイルが保存されます。

コンソールには以下のように表示されます。

```shell
[Object: null prototype] {
  files: [
    {
      fieldname: 'files',
      originalname: 'sample1.jpeg',
      encoding: '7bit',
      mimetype: 'image/jpeg',
      destination: './files',
      filename: 'sample1-10710.jpeg',
      path: 'files/sample1-10710.jpeg',
      size: 213304
    },
    {
      fieldname: 'files',
      originalname: 'sample2.jpeg',
      encoding: '7bit',
      mimetype: 'image/jpeg',
      destination: './files',
      filename: 'sample2-9102a.jpeg',
      path: 'files/sample2-9102a.jpeg',
      size: 378541
    },
    {
      fieldname: 'files',
      originalname: 'sample3.jpeg',
      encoding: '7bit',
      mimetype: 'image/jpeg',
      destination: './files',
      filename: 'sample3-f09c.jpeg',
      path: 'files/sample3-f09c.jpeg',
      size: 300479
    },
    {
      fieldname: 'files',
      originalname: 'sample4.jpg',
      encoding: '7bit',
      mimetype: 'image/jpeg',
      destination: './files',
      filename: 'sample4-b7f8.jpg',
      path: 'files/sample4-b7f8.jpg',
      size: 149810
    }
  ]
}
```

### 5 つ以上のファイルを送信した場合

`maxCount: 4` でファイル数を 4 つに制限しているため、5 つファイルを送ろうとすると以下のようにエラーが返されます。

```json
{
  "statusCode": 400,
  "message": "Unexpected field",
  "error": "Bad Request"
}
```

## 参考

[File upload](https://docs.nestjs.com/techniques/file-upload)
