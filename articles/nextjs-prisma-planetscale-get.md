---
title: "Next.js & Prisma でPlanetScaleのデータベースにアクセスする"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript"]
published: true
---

[PlanetScale](https://planetscale.com/)に Next.js & Prisma から接続してデータを取得するサンプルです。

## PlanetScale のセットアップ

[PlanetScale の CLI](https://github.com/planetscale/cli#installation)をインストール。

```console
brew install planetscale/tap/pscale
```

MySQL Client をインストール。

```console
brew install mysql-client
```

pscale を upgrade する場合は以下。

```console
brew upgrade pscale
```

Settings 画面に移動して、 ` Automatically copy migration data` にチェックを入れる。

![](https://storage.googleapis.com/zenn-user-upload/24477fdc5292-20220417.png)

`Migration framework` は `Prisma` を選択する。

次に `Overview` に戻り、`Connect` ボタンをクリックする。

Prisma のフォーマットでの接続情報が表示される。

```console
DATABASE_URL='mysql://xxxxx'
```

この`DATABASE_URL` は本番環境の環境変数に設定するもの。

## データベースのセットアップ

PlanetScale にログインする。

```console
pscale auth login
```

開発用のブランチを 2 つ、作成する。`your-pscale-database-name` は PlanetScale 上で作成したデータベース名を指定する。

```console
pscale branch create your-pscale-database-name initial-setup
pscale branch create your-pscale-database-name shadow
```

以下のようなメッセージが表示されて、Branch が作成されたことがわかる。

```console
Branch initial-setup was successfully created.
```

ターミナルを 2 つ開いて、PlanetScale に接続する。

```console
pscale connect your-pscale-database-name initial-setup --port 3309
```

```console
pscale connect your-pscale-database-name shadow --port 3310
```

## Next.js のセットアップ

```console
npx create-next-app@latest --ts
```

Prisma を初期化する。

```console
npx prisma init
```

開発環境の `.env` に以下の情報を記載する。

```env:.env
DATABASE_URL='mysql://root@127.0.0.1:3309/your-pscale-database-name'
SHADOW_DATABASE_URL="mysql://root@127.0.0.1:3310/your-pscale-database-name"
```

`prisma/schema.prisma` にデータベースに接続するための情報を記載する。

```
generator client {
  provider = "prisma-client-js"
}
datasource db {
  provider = "mysql"
  url = env("DATABASE_URL")
  shadowDatabaseUrl = env("SHADOW_DATABASE_URL")
  referentialIntegrity = "prisma"
}
```

## テーブルを作ってみる

`prisma/schema.prisma` にテーブルを定義してみる。

```
generator client {
  provider = "prisma-client-js"
}
datasource db {
  provider = "mysql"
  url = env("DATABASE_URL")
  shadowDatabaseUrl = env("SHADOW_DATABASE_URL")
  referentialIntegrity = "prisma"
}

model SamplePost {
  id        Int      @default(autoincrement()) @id
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  title     String   @db.VarChar(255)
  content   String
}
```

Prisma スキーマのセットアップ。

```console
npx prisma migrate dev --name init
```

`Update available `などのコメントが表示された場合は、ローカルの Prisma 環境をアップデートしておく。

```console
npm i --save-dev prisma@latest
npm i @prisma/client@latest
```

Prisma の変更を PlanetScale に反映する。

```console
pscale deploy-request create your-pscale-database-name initial-setup
```

以下のようなエラーが表示された場合：

```console
Error: Database branch main is not a production branch.
```

"Promote to production branch" をクリックする。

![](https://storage.googleapis.com/zenn-user-upload/6bf8eaa991bf-20220417.png)

もう一度コマンドを実行するとデプロイの Pull Request のような動作が完了する。

```console
pscale deploy-request create personaldev initial-setup
Deploy request #1 successfully created.
```

ここでデプロイが完了しているわけではない。

PlanetScale の `Deploy Requests` タブを開くと、マイグレーションの Pull Request のような画面が表示される。

`pscale deploy-request create` で作成した変更の一覧が表示される。ここで "Deploy changes" をクリックする。

![](https://storage.googleapis.com/zenn-user-upload/3f6b14c97fd1-20220417.png)

すると、変更がデプロイされる。 GitHub の Pull Request をマージするようなイメージでデータベースを変更できる。

## ローカル環境で PlanetScale のテーブルの情報を取得

```console
pscale connect your-database-name main --port 3309
```

以下のようなエラーが出た場合は、`schema.prisma` を修正する。

```console
error: Error validating datasource `db`:
The `referentialIntegrity` option can only be set if the preview feature is enabled in a generator block.
```

```
generator client {
  provider = "prisma-client-js"
  previewFeatures = ["referentialIntegrity"]
}
datasource db {
  provider = "mysql"
  url = env("DATABASE_URL")
  shadowDatabaseUrl = env("SHADOW_DATABASE_URL")
  referentialIntegrity = "prisma"
}

model SamplePost {
  id        Int      @default(autoincrement()) @id
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  title     String   @db.VarChar(255)
  content   String
}

```

schema の修正後、もう一度コマンドを実行する。

```console
pscale connect your-database-name main --port 3309
```

自動で開かれる（はず） Prisma Studio からデータを追加する。

![](https://storage.googleapis.com/zenn-user-upload/823c65db9c42-20220417.jpg)

Prisma Studio はデスクトップアプリケーションとしても取得できる。

→[Prisma Studio](https://www.prisma.io/studio)

## Next.js からデータベースに接続する

```console
npm install @prisma/client
prisma generate
```

[Install Prisma Client](https://www.prisma.io/docs/getting-started/setup-prisma/start-from-scratch/relational-databases/install-prisma-client-typescript-postgres)

```console
mkdir lib
touch lib/prisma.ts
```

```js:lib/prisma.ts
import { PrismaClient } from '@prisma/client'

// PrismaClient is attached to the `global` object in development to prevent
// exhausting your database connection limit.
//
// Learn more:
// https://pris.ly/d/help/next-js-best-practices

declare global {
  // allow global `var` declarations
  // eslint-disable-next-line no-var
  var prisma: PrismaClient | undefined
}
const prisma = global.prisma || new PrismaClient()

if (process.env.NODE_ENV === "development") global.prisma = prisma

export default prisma
```

[Best practice for instantiating PrismaClient with Next.js](https://www.prisma.io/docs/support/help-articles/nextjs-prisma-client-dev-practices)

```js:pages/api/post.ts
import { prisma } from '../../lib/prisma'
import { NextApiRequest, NextApiResponse } from 'next'

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { method } = req

  switch (method) {
    case 'GET':
      try {
        const posts = await prisma.samplePost.findMany()
        res.status(200).json(posts)
      } catch (e) {
        console.error('Request error', e)
        res.status(500).json({ error: '記事データの取得中にエラーが発生しました。' })
      }
      break
    default:
      res.setHeader('Allow', ['GET'])
      res.status(405).end(`Method ${method} Not Allowed`)
      break
  }
}
```

以下のようなエラーが出た場合：

```console
error - TypeError: collection is not iterable
```

prisma を最新化して `prisma generate` を実行する。

```console
npm i prisma@latest
npm i @prisma/client@latest
prisma generate
```

ここでブラウザから `http://localhost:3000/api/post` にアクセスすると、以下のようなデータが表示される。

```json
[
  {
    "id": 1,
    "createdAt": "2022-04-17T03:58:25.873Z",
    "updatedAt": "2022-04-17T03:57:18.427Z",
    "title": "サンプルのタイトル",
    "content": "これはサンプルの記事です。"
  }
]
```

## Next.js & Prisma から PlanetScale にデータを登録する方法

以下の記事に続きを書いています。

[Next.js & Prisma 環境から PlanetScale にデータを投稿する](https://zenn.dev/fjsh/articles/2bdef362c46394)

## 参考

[Deploying a PlanetScale, Next.js & Prisma App to Vercel](https://davidparks.dev/blog/planetscale-deployment-with-prisma/)
[Next-generation ORM for Node.js & TypeScript | PostgreSQL, MySQL, MariaDB, SQL Server & SQLite](https://jsrepos.com/lib/prisma-prisma-javascript-database)
[Prisma を使って Next.js から PlanetScale に繋ぐ](https://zenn.dev/kubio/articles/654aeecb1da6bd)
