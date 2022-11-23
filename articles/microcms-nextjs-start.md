---
title: "microCMSのAPIから取得したデータをNext.jsで表示する"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["microcms"]
published: falsetrue
---

microCMS の API からコンテンツデータを受け取り、ブラウザに表示するまでの手順を紹介します。

フロントエンドで使うフレームワークは Next.js とします。

## microCMS のユーザー登録

[microCMS](https://microcms.io/)でまずはアカウントを作成します。

- サービス名
- サービス ID

は任意で OK です。

- ブログ
- お知らせ
- バナー

などの API がテンプレートとして用意されているので、ここでは「ブログ」を選択します。

![](https://storage.googleapis.com/zenn-user-upload/7de0fcebe4a0-20221119.jpg)

「ブログ」を選択すると、WordPress の管理画面のような、ブログ投稿ができる画面が自動で作成されます。

![](https://storage.googleapis.com/zenn-user-upload/44e77d3e9c41-20221119.jpg)

ひと記事分がデフォルトで作成されていて、右上にある「API プレビュー」というボタンを押すと、API からどんな値が返ってくるかを確認できます。

```json
{
  "id": "xxxx",
  "createdAt": "2022-11-19T05:55:27.667Z",
  "updatedAt": "2022-11-19T05:58:58.277Z",
  "publishedAt": "2022-11-19T05:55:27.667Z",
  "revisedAt": "2022-11-19T05:58:58.277Z",
  "title": "（サンプル）まずはこの記事を開きましょう",
  "content": "<h2 id=\"hdb41525ba7\">ブログテンプレートから作成されました🎉</h2><p>ブログテンプレートからAPIを作成しました。<br>おつかれさまでした🎉<br></p><h2 id=\"hf45076424a\">APIプレビューを試そう🚀</h2>",
  "eyecatch": {
    "url": "https://images.microcms-assets.io/assets/87c8da978fa34b1993ed7cd703c5946b/99e392a883cb4c8a95ffdd1049503e82/blog-template.png",
    "height": 630,
    "width": 1200
  },
  "category": {
    "id": "xxxx",
    "createdAt": "2022-11-19T05:55:25.938Z",
    "updatedAt": "2022-11-19T05:55:25.938Z",
    "publishedAt": "2022-11-19T05:55:25.938Z",
    "revisedAt": "2022-11-19T05:55:25.938Z",
    "name": "更新情報"
  }
}
```

この記事ではテンプレートで初めに作成された API をブラウザに描画するまでを扱います。

記事の一覧の表示であったり、詳細画面の表示等は別の記事でやります。

## Next.js のプロジェクトを作成する

まずは Next.js のプロジェクトを作成します。

```console
$ npx create-next-app@latest --typescript
✔ What is your project named? … .
✔ Would you like to use ESLint with this project? … No / Yes

$ mkdir src
$ mv pages styles src
$ touch .env .env.development.local
$ npm i microcms-js-sdk
```

環境ファイルには以下の変数を定義しておきます。

```.env.development.local
API_KEY=XXX
SERVICE_DOMAIN=xxxx
ENDPOINT=yyy
```

それぞれの値は以下で置き換えてください。

- `API_KEY`: 左側のメニューの 権限管理 > x 個の API キー で取得できる値
- `SERVICE_DOMAIN`: API の URL`https://xxxx.microcms.io/api/v1/yyy`の `xxxx` の部分。
- `ENDPOINT` は `https://xxxx.microcms.io/api/v1/yyy` の `yyy` の部分。

API のエンドポイントは右上の「API 設定」 > 基本情報 から確認できます。

API キーは左メニュー下の「権限管理」 > API キー から確認できます。

なお、Next.js の環境変数は以下のような扱いになっています。

| ファイル名         | 読み込まれるタイミング |
| ------------------ | ---------------------- |
| `.env.local`       | 毎回                   |
| `.env.development` | `next dev` 時のみ      |
| `.env.production`  | `next start` 時のみ    |

引用元：[Next.js で環境変数（env）を使いこなすための記事](https://zenn.dev/aktriver/articles/2022-04-nextjs-env)

## API クライントを作る

`src/lib/microcms-client.ts` を作成し、API クライアントを作成します。

```ts:src/lib/microcms-client.ts
import { createClient } from "microcms-js-sdk";

const API_KEY = process.env.API_KEY;
const SERVICE_DOMAIN = process.env.SERVICE_DOMAIN;
const ENDPOINT = process.env.ENDPOINT;

export const getEnv = () => {
  if (
    API_KEY === undefined ||
    SERVICE_DOMAIN === undefined ||
    ENDPOINT === undefined
  ) {
    throw new Error(
      `環境変数が設定されていません。 API_KEY:${API_KEY}, SERVICE_DOMAIN: ${SERVICE_DOMAIN}, ENDPOINT: ${ENDPOINT}`
    );
  }
  return {
    apiKey: API_KEY,
    serviceDomain: SERVICE_DOMAIN,
    endpoint: ENDPOINT,
  };
};

export const client = createClient({
  serviceDomain: getEnv().serviceDomain,
  apiKey: getEnv().apiKey,
});
```

## レスポンスのインターフェースを定義する

API から返ってきた値に型をつけます。
今回はデフォルトのブログ API に型をつけていますが、カスタム API を作成した場合は、カスタムした内容に合わせて型を作ってください。

```ts:src/types/blog-response.ts
export interface BlogResponse {
  contents: [BlogContents];
  totalCount: number;
  offset: number;
  limit: number;
}

export interface BlogContents {
  id: string;
  createdAt: string;
  updatedAt: string;
  publishedAt: string;
  revisedAt: string;
  title: string;
  content: string;
}

export interface EyeCatch {
  url: string;
  height: string;
  width: string;
}

export interface Category {
  id: string;
  createdAt: string;
  updatedAt: string;
  publishedAt: string;
  revisedAt: string;
  name: string;
}
```

## コンテンツの本文とタイトルを表示してみる

まずは microCMS から取得したコンテンツをズラッと表示させてみます。

```ts:src/pages/index.tsx
import {BlogContents, BlogResponse} from "../types/blog-response";
import {client, getEnv} from "../lib/microcms-client";
import {GetStaticProps} from "next";

interface Props {
  blogContents: BlogContents[]
}

export default function Home({blogContents}: Props) {
  return (
    <>
      <div>
        {blogContents && blogContents.map((blogContent, index) => (
          <div key={blogContent.id}>
            <h2 key={`${blogContent.id}-title-${index}`}>タイトル：{blogContent.title}</h2>
            <div key={`${blogContent.id}-content-${index}`} dangerouslySetInnerHTML={{__html: `${blogContent.content}`}} />
          </div>
        ))}
      </div>
    </>
  )
}

export const getStaticProps: GetStaticProps = async () => {
  const data: BlogResponse = await client.get({
    endpoint: getEnv().endpoint
  })

  return {
    props: {
      blogContents: data.contents
    }
  }
}
```

`npm run dev` で起動して `http://localhost:3000` を開くと、以下のような画面が表示されます。

![](https://storage.googleapis.com/zenn-user-upload/69d4a97b2a29-20221123.jpg)

CSS を当てていないのでただの文字の羅列になっていますが、ひとまず API からデータを取得して描画するまでできました。
