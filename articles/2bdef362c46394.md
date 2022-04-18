---
title: "Next.js & Prisma 環境から PlanetScale にデータを投稿する"
emoji: "📌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript"]
published: true
---

[Next.js & Prisma で PlanetScale のデータベースにアクセスする](https://zenn.dev/fjsh/articles/733a1334ffc1c8) からの続きです。

前回の記事では、Next.js & Prisma から PlanetScale のデータベースに接続して、データを取得してみるところまでやりました。

この記事では、Next.js から Prisma を使って、 PlanetScale にデータを INSERT するところまでやってみます。

## Prisma の基本的なコード

前回の記事の簡単な振り返りです。

Prisma のインスタンスを取得するためのコード。

```js:lib/prisma.ts
declare global {
  // allow global `var` declarations
  // eslint-disable-next-line no-var
  var prisma: PrismaClient | undefined
}

export const prisma =
  global.prisma ||
  new PrismaClient({
    log: ['query'],
  })

if (process.env.NODE_ENV === 'development') global.prisma = prisma
```

Prisma のスキーマ定義。

```prisma:prisma/schema.prisma
// This is your Prisma schema file,
// learn more about it in the docs: https://pris.ly/d/prisma-schema


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

`.env` には以下の情報を指定しています。

```
DATABASE_URL='mysql://root@127.0.0.1:3309/personaldev'
SHADOW_DATABASE_URL="mysql://root@127.0.0.1:3310/personaldev"
```

`127.0.0.1` を `DATABASE_URL` にしているのになぜ、PlanetScale 上のデータベースに接続されるのかが不思議だったのですが、どうも `pscale` でログインして、データベースに接続した状態で `127.0.0.1` にクエリを投げると、PlanetScale にフォワードしてくれるっぽいです。

```console
pscale auth login
```

でログインして、

```console
pscale connect <データベース名> <ブランチ名> --port 3309
```

でコネクションが確立されるみたいです（確証はなし）

[PlanetScale quickstart guide](https://docs.planetscale.com/tutorials/planetscale-quick-start-guide)

さて、ここまで書いてきた内容を活用して、Next.js & Prisma からデータを登録してみます。 CRUD の CREATE の部分です。

## Prisma からデータを投稿する

Prisma を使っている部分はこちらです。 `title` と `content` を `SamplePost` というテーブルに登録します。

```js:lib/blog-post.ts
import { prisma } from './prisma'

export const blogPost = async (title: string, content: string) => {
  const createPost = await prisma.samplePost.create({
    data: {
      title,
      content,
    },
  })
  return createPost
}

```

この `lib/blog-post.ts` はブラウザからは呼べないので、Next.js の API 経由で呼び出します。

```js:pages/api/create-post.ts
import type { NextApiRequest, NextApiResponse } from 'next'
import { blogPost } from '../../lib/blog-post'

interface BlogsNextApiRequest extends NextApiRequest {
  body: {
    title: string
    content: string
  }
}

export default function handler(req: BlogsNextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') {
    res.status(405).send({ message: 'Only POST requests allowed' })
    return
  }

  const title = req.body.title
  const content = req.body.content

  console.log('body', title, content)
  if (title != null && content != null) {
    blogPost(title, content)
      .then((createdPost) => {
        console.log(`ID:${createdPost.id} | ${createdPost.title} を作成しました。`)
      })
      .catch((error) => {
        console.error(error)
      })
  }
  res.status(200).json({ name: 'サンプルの投稿が完了' })
}

```

この API を呼び出すページを作ってみます。

```js:pages/blog.tsx
import React, { useRef } from 'react'

const Blog = () => {
  const inputTitle = useRef<HTMLTextAreaElement>(null)
  const inputPost = useRef<HTMLTextAreaElement>(null)
  const handlePost = (event: React.MouseEvent<HTMLButtonElement>) => {
    event.preventDefault()
    const title = inputTitle.current?.value
    const content = inputPost.current?.value
    fetch('/api/create-post', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ title, content }),
    })
  }
  return (
    <div className={'flex justify-center'}>
      <form>
        <div className={'flex flex-col justify-center items-center'}>
          <label htmlFor={'title'}></label>
          <textarea
            ref={inputTitle}
            className={
              'w-96 mt-16 mb-5 px-3 py-3  bg-gray-200 border rounded-lg text-lg text-gray-700 focus:outline-none rounded-2xl resize-none focus:border-blue-500 focus:shadow-outline'
            }
            placeholder={'タイトル'}
            spellCheck={false}
            id={'title'}
            autoComplete={'off'}
          />
          <label htmlFor={'body'}></label>
          <textarea
            ref={inputPost}
            className={
              'w-96 h-96 mt-5 mb-5 px-3 py-3  bg-gray-200 border rounded-lg text-lg text-gray-700 focus:outline-none rounded-2xl resize-none focus:border-blue-500 focus:shadow-outline'
            }
            placeholder={'記事'}
            spellCheck={false}
            id={'title'}
            autoComplete={'off'}
          />
        </div>
        <div className={'flex items-end'}>
          <button
            className="w-32 mt-2 bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded ml-auto"
            onClick={handlePost}
          >
            送信
          </button>
        </div>
      </form>
    </div>
  )
}

export default Blog
```

画面で見るとこんな感じです。

![](https://storage.googleapis.com/zenn-user-upload/770fdda1c60c-20220418.jpg)

タイトルと本文を投稿してみます。

![](https://storage.googleapis.com/zenn-user-upload/fadea977caa4-20220418.jpg)

ローディングだったり、投稿後の入力のクリアなどはしていません。サンプルなので。

ここで、PlanetScale の Console を使って、投入したデータを確認します。

![](https://storage.googleapis.com/zenn-user-upload/39dac079ef47-20220418.png)

同じデータは[Prisma Studio](https://www.prisma.io/studio)でも確認できます。

Prisma Studio の UI はこんな感じです。

![](https://storage.googleapis.com/zenn-user-upload/d6a60ba68dc4-20220418.jpg)

手元で確認したい場合は、Prisma Studio を使うのが便利そうです。

## 参考

[CRUD - Prisma](https://www.prisma.io/docs/concepts/components/prisma-client/crud#create)
