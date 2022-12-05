---
title: "NestJSで1つの画像をアップロードして、ローカルストレージに保存する"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs"]
published: true
---

この記事では NestJS の REST API 経由でファイルをアップロードして、ローカルストレージに保存する手順を紹介します。
クライアントは Postman を使います。

## ライブラリのインストール

```shell
npm i -D @types/multer
```

[multer](https://www.npmjs.com/package/multer)は Node.js で `multipart/form-data` を扱うためのミドルウェアです。

## リソースの作成

```shell
$ nest g res posts


? What transport layer do you use? REST API
? Would you like to generate CRUD entry points? Yes
```

`res` オプションを使うと REST API に必要なファイルがたくさんできますが、今回使うのは Controller と dto のみです。

## DTO でインプットとアウトプットを定義する

コントローラーのインプットとアウトプットを DTO として定義します。

```ts:src/posts/dto/create-post-input.dto.ts
export class CreatePostInputDto {
  title: string;
  description: string;
}
```

DTO 自体にはファイル（`file`）は定義していません。
送ってくる画像の受け口は別で定義します。

アウトプットにはファイルの格納先のパスを入れてあげます。

```ts:src/posts/dto/create-post-output.dto.ts
export class CreatePostOutputDto {
  title: string;
  description: string;
  imageUrl: string;
}
```

## Interceptor で使う関数を作る

`Multer` の `FileInterceptor` では、ファイル名を編集する機能や、ファイルに制限をかける機能があります。
`FileInterceptor` で使うための関数を作っています。

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
  req: any,
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
  if (!file.originalname.match(/\.(jpg|jpeg|png)$/)) {
    return callback(
      new BadRequestException('おいおい、画像だけを送ってくれよな？'),
      false,
    );
  }
  callback(null, true);
};
```

## コントローラーを作成する

ここが本命です。
`/posts/file` という API を作り、`file` という KEY からファイルを受け取っています。

```ts:src/posts/posts.controller.ts
import {
  Controller,
  Post,
  Body,
  UploadedFile,
  UseInterceptors,
} from '@nestjs/common';
import { PostsService } from './posts.service';
import { CreatePostInputDto } from './dto/create-post-input.dto';

import { FileInterceptor } from '@nestjs/platform-express';
import { diskStorage } from 'multer';
import { editFileName, imageFileFilter } from './interceptors';
import { CreatePostOutputDto } from './dto/create-post-output.dto';

@Controller('posts')
export class PostsController {
  constructor(private readonly postsService: PostsService) {}

  @Post('file')
  @UseInterceptors(
    FileInterceptor('file', {
      storage: diskStorage({
        destination: './files',
        filename: editFileName,
      }),
      fileFilter: imageFileFilter,
      limits: { fileSize: 1024 * 1024 * 4 },
    }),
  )
  uploadFileAndPassValidation(
    @Body() body: CreatePostInputDto,
    @UploadedFile()
    file: Express.Multer.File,
  ): CreatePostOutputDto {
    console.log(`title: ${body.title}, description: ${body.description}`);
    console.log('file', file);
    return {
      title: body.title,
      description: body.description,
      imageUrl: `${file.destination}/${file.filename}`,
    };
  }
}
```

Postman では `form-data` に `file`、`title`、`description` をキーにして、それぞれ値を入れます。

![](https://storage.googleapis.com/zenn-user-upload/0d160cd0a74a-20221205.jpg)

`file`には画像ファイルを指定します。

これでリクエストを送ると、REST API からのレスポンスは以下のように返ってきます。

```json
{
  "title": "公園です。",
  "description": "とても良いです。",
  "imageUrl": "./files/takeru-f75f.jpg"
}
```

コンソールには以下のように表示されます。

```shell
title: 公園です。, description: とても良いです。
file {
  fieldname: 'file',
  originalname: 'takeru.jpg',
  encoding: '7bit',
  mimetype: 'image/jpeg',
  destination: './files',
  filename: 'takeru-619f.jpg',
  path: 'files/takeru-619f.jpg',
  size: 50400
}
```

`プロジェクトのルートディレクトリ/files`に `takeru-f75f.jpg` みたいなファイルが保存されます。
`files`ディレクトリは事前に作っておいてください。

さて、ここから下はちょっとした調査です。

上のサンプルでは `Multer` の Interceptor を使ってバリデーションをかけています。

ですが、NestJS の公式ガイドのサンプルでは `@UploadedFile` 中でバリデーションをかけている例が紹介されています。

それぞれで何か違いがあるのか、確認してみました。

## `FileInterceptor` でバリデーションをかけた場合

まずは上のサンプルと同じく、`FileInterceptor`でバリデーションをかけた場合です。

```ts
  @Post('file')
  @UseInterceptors(
    FileInterceptor('file', {
      storage: diskStorage({
        destination: './files',
        filename: editFileName,
      }),
      fileFilter: imageFileFilter,
      limits: { fileSize: 1024 * 1024 * 4 },
    }),
  )
  uploadFileAndPassValidation(
    @Body() body: CreatePostInputDto,
    @UploadedFile()
    file: Express.Multer.File,
  ): CreatePostOutputDto {
    console.log(`title: ${body.title}, description: ${body.description}`);
    console.log('file', file);
    return {
      title: body.title,
      description: body.description,
      imageUrl: `${file.destination}/${file.filename}`,
    };
  }
```

`FileInterceptor`では、バリデーションエラーとなったときは、エラーレスポンスが返ってきて、ファイルは保存されませんでした。

GIF ファイルを送ったときのエラーレスポンス ↓

```json
{
  "statusCode": 400,
  "message": "おいおい、画像だけを送ってくれよな？",
  "error": "Bad Request"
}
```

サイズオーバーのファイルを送ったときのエラーレスポンス ↓

```json
{
  "statusCode": 413,
  "message": "File too large",
  "error": "Payload Too Large"
}
```

## `@UploadedFile` でバリデーションをかけた場合

次に `@UploadedFile` を使う例です。

```ts
  @Post('file')
  @UseInterceptors(
    FileInterceptor('file', {
      storage: diskStorage({
        destination: './files',
        filename: editFileName,
      }),
      fileFilter: imageFileFilter,
      limits: { fileSize: 1024 * 1024 * 4 },
    }),
  )
  uploadFileAndPassValidation(
    @Body() body: CreatePostInputDto,
    @UploadedFile(
      new ParseFilePipeBuilder()
        .addFileTypeValidator({
          fileType: '.(png|jpeg|jpg)',
        })
        .addMaxSizeValidator({
          maxSize: 1024 * 1024 * 4,
        })
        .build({
          errorHttpStatusCode: HttpStatus.UNPROCESSABLE_ENTITY,
        }),
    )
    file: Express.Multer.File,
  ): CreatePostOutputDto {
    console.log(`title: ${body.title}, description: ${body.description}`);
    console.log('file', file);
    return {
      title: body.title,
      description: body.description,
      imageUrl: `${file.destination}/${file.filename}`,
    };
  }
```

`@UploadedFile` を使ったときは、バリデーションエラーのレスポンスは返ってきますが、ファイルは保存されていました（！）

GIF ファイルを送ったときのエラーレスポンス ↓

```json
{
  "statusCode": 422,
  "message": "Validation failed (expected type is .(png|jpeg|jpg))",
  "error": "Unprocessable Entity"
}
```

サイズオーバーのときのエラーレスポンス（わざとリミットを 10b にした）

```json
{
  "statusCode": 422,
  "message": "Validation failed (expected size is less than 10)",
  "error": "Unprocessable Entity"
}
```

そして `files` 以下にファイルが保存されている感じです。
バリデーションエラーになったファイルが保存されるのはちょっと挙動がおかしい気がするので、以下のような Validator の使い方は何かが足りない可能性があります。

```ts
@UploadedFile(
    new ParseFilePipeBuilder()
    .addFileTypeValidator({
        fileType: '.(png|jpeg|jpg)',
    })
    .addMaxSizeValidator({
        maxSize: 1024 * 1024 * 4,
    })
    .build({
        errorHttpStatusCode: HttpStatus.UNPROCESSABLE_ENTITY,
    }),
)
```

## 参考

- [File upload](https://docs.nestjs.com/techniques/file-upload)
- [How to Upload Files to Azure using NestJS and typeORM with MySQL](https://www.freecodecamp.org/news/upload-files-to-azure-using-nestjs-and-typeorm-with-mysql/)
- [Nestjs file uploading using Multer](https://gabrieltanner.org/blog/nestjs-file-uploading-using-multer/)
