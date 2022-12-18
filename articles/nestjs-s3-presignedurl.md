---
title: "[NestJS + AWS] Pre-Signed URL (署名付きURL) を発行してファイルをアップロードする"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs"]
published: true
---

この記事では S3 で Pre-Signed URL（署名付き URL）を発行して、ファイルをアップロードする方法と、アップロードしたファイルをダウンロードする方法を紹介します。

事前準備としての環境変数の設定やライブラリのインストールはこちら →[NestJS で AWS S3 にファイルをアップロードする](https://zenn.dev/fjsh/articles/nestjs-s3-file-upload)の記事を確認してください 🙇‍♂️

## S3 から Pre-Signed URL を発行するサービスを作る

`getPreSignedUrlForPut` でファイルを S3 に送り込むための Pre-Signed URL を発行します。

`getPreSignedUrlForGet` でファイルを S3 から取得するための Pre-Signed URL を取得します。

```ts:src/file-upload/file-upload.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { S3 } from 'aws-sdk';
import { v4 as uuid } from 'uuid';

@Injectable()
export class FileUploadService {
  constructor(private readonly configService: ConfigService) {}

  getPreSignedUrlForPut(filename: string) {
    const s3 = new S3();
    const key = `${uuid()}-${filename}`;
    const params = {
      Bucket: this.configService.get('aws.s3BucketName'),
      Key: key,
      Expires: 60 * 5,
    };

    const url = s3.getSignedUrl('putObject', params);
    return {
      key,
      preSignedUrl: url,
    };
  }

  async getPreSignedUrlForGet(key: string) {
    const s3 = new S3();
    const params = {
      Bucket: this.configService.get('aws.s3BucketName'),
      Key: key,
      Expires: 60 * 5,
    };

    const url = s3.getSignedUrl('getObject', params);
    return {
      key,
      preSignedUrl: url,
    };
  }
}
```

## Pre-Signed URL をやり取りするためのコントローラーを作る

```ts:src/file-upload/file-upload.controller.ts
import { Controller, Req, Get } from '@nestjs/common';
import { FileUploadService } from './file-upload.service';

@Controller('files')
export class FileUploadController {
  constructor(private readonly fileUploadService: FileUploadService) {}

  @Get('preSignedUrlForPut')
  async getPreSignedUrlForPut(@Req() request) {
    return this.fileUploadService.getPreSignedUrlForPut(request.query.fileName);
  }

  @Get('preSignedUrlForGet')
  async getPreSignedUrlForGet(@Req() request) {
    const key = request.query.key;
    return this.fileUploadService.getPreSignedUrlForGet(key);
  }
}
```

## Pre-Signed URL を使ってファイルをやり取りしてみる

アップロード用の URL を取得してみます。
Query Params に `fileName` でファイル名を指定すると、その `fileName` を格納するための Pre-Signed URL を発行してくれます。

返ってくる `key` は、あとでファイルを取得するために使います。

Postman で試しに使ってみましょう。

まずはファイルを PUT するための Pre-Signed URL を取得します。

![](https://storage.googleapis.com/zenn-user-upload/60f94b6f6b8b-20221218.jpg)

以下のようなレスポンスが返されます。

```json
{
  "key": "dscds2-cdsc-43ed-be6b-f9a12ccd4feb14-example.jpg",
  "preSignedUrl": "https://xxx.s3.ap-northeast-1.amazonaws.com/473cdcbb-0627-43ed-be6b-ds8jdsucsl-example.jpg?AWSAccessKeyId=Aucsd&Expires=16713341234&Signature=dsacs8%dsds%32c%3D"
}
```

CURL を使ってファイルを PUT します。

```shell
curl -X PUT --upload-file <ファイルのパス> "<preSignedUrl>"
```

次に、レスポンスの Key を使ってファイルをダウンロードしてみます。

以下のように、PUT したときに返ってきた Key を設定します。

![](https://storage.googleapis.com/zenn-user-upload/b7761e095345-20221218.jpg)

リクエストを投げると、以下のように GET 用の `preSignedUrl` が発行されます。

ブラウザ等で URL を叩くと、ファイルをダウンロードできます。
無論、AWS S3 のバケット内にもファイルは格納されています。

```json
{
  "key": "xxx-0627-xx-x-xx-example.jpg",
  "preSignedUrl": "https://xxx-xxx.s3.ap-northeast-1.amazonaws.com/473cdcbb-0627-43ed-be6b-f9a12c4feb14-example.jpg?AWSAccessKeyId=xxxxx&Expires=167133422218&Signature=xxx%xx%x%3D"
}
```

## 参考

- [Generate pre-signed Url for the file via Node.Js](https://rajputankit22.medium.com/generate-pre-signed-url-for-the-file-via-node-js-735f0b356644)
- [Pre-Signed URL で S3 に直接ファイルをアップロードする](https://qiita.com/gungungggun/items/c1dd5da5fced8c044e3c)
- [Presigned URL を利用した S3 へのファイルアップロード](https://kakehashi-dev.hatenablog.com/entry/2022/03/15/101500)
