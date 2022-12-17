---
title: "NestJSでAWS S3にファイルをアップロードする"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs"]
published: true
---

この記事では NestJS のアプリケーションで AWS S3 にファイルをアップロードするための手順を紹介します。

まずはサンプルプロジェクトを作成しましょう。

```shell
nest new nest-fileupload
```

## AWS S3 でバケットを作成

- バケット名：`nestjs-fileupload`
- AWS リージョン: `ap-northeast-1`

## 環境変数の設定

IAM から `AWS_ACCESS_KEY_ID` と `AWS_SECRET_ACCESS_KEY` を取得して環境変数に設定します。

```txt:.env
AWS_REGION=ap-northeast-1
AWS_ACCESS_KEY_ID=accessKey
AWS_SECRET_ACCESS_KEY=secretKey
AWS_BUCKET_NAME='nestjs-fileupload'
```

## CRUD エンドポイントを作成

不要なファイルも作成されますが、便利なので NestCLI 経由でファイルを作成します。

```shell
nest g res file-upload
```

## 必要なライブラリのインストール

次に今回のサンプルで使うライブラリをインストールします。

```shell
npm i @nestjs/config aws-sdk @types/aws-sdk uuid
npm i -D @types/multer @types/uuid
```

## ConfigService の作成

環境変数を読み込むためのサービスを作っていきましょう。

`app.module.ts` で ConfigModle を import します。

```ts:app.module.ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { FileUploadModule } from './file-upload/file-upload.module';
import { ConfigModule } from '@nestjs/config';
import configuration from './config/configuration';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [configuration],
      isGlobal: true,
    }),
    FileUploadModule,
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

`configuration` は以下のように作ります。
ここで `.env` の値を読み込んでいます。

```ts:src/config/configuration.ts
export default () => ({
  aws: {
    region: process.env.AWS_REGION,
    accessKey: process.env.AWS_ACCESS_KEY_ID,
    secretKey: process.env.AWS_SECRET_ACCESS_KEY,
    s3BucketName: process.env.AWS_BUCKET_NAME,
  },
});
```

次に `main.ts` で AWS の config を設定します。

```ts:main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { config } from 'aws-sdk';
import { ConfigService } from '@nestjs/config';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const configService = app.get(ConfigService);
  config.update({
    accessKeyId: configService.get('aws.accessKey'),
    secretAccessKey: configService.get('aws.secretKey'),
    region: configService.get('aws.region'),
  });
  await app.listen(3000);
}
bootstrap();
```

`FileUploadService` 内で設定を読み込むために、 `file-upload.module.ts` で `ConfigModule` を import します。

```ts:src/file-upload/file-upload.module.ts
import { Module } from '@nestjs/common';
import { FileUploadService } from './file-upload.service';
import { FileUploadController } from './file-upload.controller';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule],
  controllers: [FileUploadController],
  providers: [FileUploadService],
})
export class FileUploadModule {}
```

ここまでで、環境面の設定は完了です。

## ファイルをアップロードするサービスを作成する

コントローラーで受け取ったファイルをサービスに渡して、サービスから AWS S3 にアップロードします。

```ts:src/file-upload/file-upload.service.ts
import { Injectable } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { S3 } from "aws-sdk";
import { v4 as uuid } from "uuid";

@Injectable()
export class FileUploadService {
  constructor(private readonly configService: ConfigService) {}
  async uploadFile(dataBuffer: Buffer, filename: string) {
    const s3 = new S3();
    const uploadResult = await s3
      .upload({
        Bucket: this.configService.get("aws.s3BucketName"),
        Body: dataBuffer,
        Key: `${uuid()}-${filename}`,
      })
      .promise();

    console.log("Key:", uploadResult.Key);
    console.log("url:", uploadResult.Location);
  }
}
```

一旦、S3 に格納したファイルの Key や Location はコンソールに出力するだけにします。

## ファイルを受け取るコントローラーを作成する

```ts:src/file-upload/file-upload.controller.ts
import {
  Controller,
  Post,
  UseInterceptors,
  Req,
  UploadedFile,
} from '@nestjs/common';
import { FileUploadService } from './file-upload.service';
import { FileInterceptor } from '@nestjs/platform-express';
import { response } from 'express';

@Controller('files')
export class FileUploadController {
  constructor(private readonly fileUploadService: FileUploadService) {}

  @Post('upload')
  @UseInterceptors(FileInterceptor('file'))
  async uploadFile(@Req() request, @UploadedFile() file: Express.Multer.File) {

    try {
      await this.fileUploadService.uploadFile(file.buffer, file.originalname);
    } catch (error) {
      return response
        .status(500)
        .json(`Failed to upload image file: ${error.message}`);
    }
  }
}
```

## Postman でファイルをアップロードしてみる

ここまで作成したら、一度 Postman でファイルをアップロードしてみます。

`http://localhost:3000/files/upload` に POST します。

Body > form-data の KEY に `file` を指定して、 `VALUE` でファイルを選択します。

![](https://storage.googleapis.com/zenn-user-upload/9ac56d1ff413-20221217.jpg)

コンソールには以下のように表示されます。

```shell
Key: 7e81d580-b519-4a6e-a945-b4cdf1e10df1-imadamio.jpg
url: https://xxxxxx.s3.ap-northeast-1.amazonaws.com/7e81d580-b519-4a6e-a945-b4cdf1e10df1-imadamio.jpg
```

S3 のバケットを見ると、たしかに狙ったバケットにファイルが格納されていることがわかります。

![](https://storage.googleapis.com/zenn-user-upload/686626b9dc9f-20221217.jpg)

## 複数のファイルを S3 に格納してみる

[NestJS で複数の画像をアップロードして、ローカルストレージに保存する](https://zenn.dev/fjsh/articles/nestjs-save-multiple-image) では複数のファイルをローカルストレージに保存していました。

上の記事で受け取った複数ファイルを S3 に格納するようにアレンジしてみます。

## interceptor を作成する

画像ファイルのみが送られていることをチェックするための interceptor を作成します。

```ts:
import { Request } from 'express';
import { BadRequestException } from '@nestjs/common';

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

## 複数ファイルを S3 に upload するメソッドを作成する

`uploadFiles` というメソッドを追加します。

```ts:src/file-upload/file-upload.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { S3 } from 'aws-sdk';
import { v4 as uuid } from 'uuid';
import { PublicFile } from './dto/public-file';

@Injectable()
export class FileUploadService {
  constructor(private readonly configService: ConfigService) {}
  async uploadFile(dataBuffer: Buffer, filename: string) {
    const s3 = new S3();
    const uploadResult = await s3
      .upload({
        Bucket: this.configService.get('aws.s3BucketName'),
        Body: dataBuffer,
        Key: `${uuid()}-${filename}`,
      })
      .promise();

    console.log('Key:', uploadResult.Key);
    console.log('url:', uploadResult.Location);
  }

  async uploadFiles(files: Array<Express.Multer.File>): Promise<PublicFile[]> {
    const s3 = new S3();
    const publicFiles: PublicFile[] = [];
    for (const file of files) {
      const uploadResult = await s3
        .upload({
          Bucket: this.configService.get('aws.s3BucketName'),
          Body: file.buffer,
          Key: `${uuid()}-${file.originalname}`,
        })
        .promise();
      console.log(`${file.originalname} の Key: ${uploadResult.Key}`);
      console.log(`${file.originalname} の Location: ${uploadResult.Location}`);
      publicFiles.push({
        originalname: file.originalname,
        key: uploadResult.Key,
        location: uploadResult.Location,
      });
    }
    return publicFiles;
  }
}
```

## 複数ファイルを受け取るコントローラの作成

コントローラーに `uploadFiles` を追加します。

```ts:src/file-upload/file-upload.controller.ts
import {
  Controller,
  Post,
  UseInterceptors,
  Req,
  UploadedFile,
  UploadedFiles,
} from '@nestjs/common';
import { FileUploadService } from './file-upload.service';
import {
  FileFieldsInterceptor,
  FileInterceptor,
} from '@nestjs/platform-express';
import { response } from 'express';
import { imageFileFilter } from './interceptors';
import { PublicFile } from './dto/public-file';

@Controller('files')
export class FileUploadController {
  constructor(private readonly fileUploadService: FileUploadService) {}

  @Post('upload')
  @UseInterceptors(FileInterceptor('file'))
  async uploadFile(@Req() request, @UploadedFile() file: Express.Multer.File) {
    console.log('uploadFile is called', file);
    try {
      await this.fileUploadService.uploadFile(file.buffer, file.originalname);
    } catch (error) {
      return response
        .status(500)
        .json(`Failed to upload image file: ${error.message}`);
    }
  }

  @Post('uploads')
  @UseInterceptors(
    FileFieldsInterceptor([{ name: 'files', maxCount: 4 }], {
      fileFilter: imageFileFilter,
      limits: { fileSize: 1024 * 1024 * 4 },
    }),
  )
  async uploadFiles(
    @UploadedFiles()
    params: {
      files: Array<Express.Multer.File>;
    },
  ): Promise<PublicFile[]> {
    return await this.fileUploadService.uploadFiles(params.files);
  }
}

```

## 実行結果

Postman から複数ファイルを送ってみます。

![](https://storage.googleapis.com/zenn-user-upload/bab10bc9b0ba-20221217.jpg)

リクエストを投げると、S3 上には複数のファイルが格納されて、以下のようなレスポンスが返ってきました。

```json
[
  {
    "originalname": "sample1.jpeg",
    "key": "e3bce52f-ede4-441d-84f0-e48277a8b6bc-sample1.jpeg",
    "location": "https://xxx.s3.ap-northeast-1.amazonaws.com/e3bce52f-ede4-441d-dsds-e48277a8b6bc-sample1.jpeg"
  },
  {
    "originalname": "sample2.jpeg",
    "key": "3abde8f6-dd87-4cf7-bc4a-9022d2c894fe-sample2.jpeg",
    "location": "https://xxx.s3.ap-northeast-1.amazonaws.com/3abde8f6-dd87-4cf7-dscsd-9022d2c894fe-sample2.jpeg"
  },
  {
    "originalname": "sample3.jpeg",
    "key": "af3760c4-3b5a-4c72-8185-abf32afa6e3c-sample3.jpeg",
    "location": "https://xxx.s3.ap-northeast-1.amazonaws.com/af3760c4-3b5a-4c72-cds2-abf32afa6e3c-sample3.jpeg"
  }
]
```

## 参考

- [NestJS - configuration](https://docs.nestjs.com/techniques/configuration)
- [Uploading public files to Amazon S3](https://wanago.io/2020/08/03/api-nestjs-uploading-public-files-to-amazon-s3/)
- [File Upload](https://docs.nestjs.com/techniques/file-upload)
