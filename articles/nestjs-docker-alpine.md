---
title: "NestJSをDocker（Alpine）で動かす"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs"]
published: true
---

## NestJS のプロジェクトを作成する

任意のディレクトリで以下のコマンドを実行します。

```shell
npm i -g @nestjs/cli
nest new .
```

プロジェクトを作成したら、とりあえず動かしてみます。

```shell
npm run start:dev
```

以下のコマンドを実行すると `Hello World!` が返ってきます。

```shell
curl http://localhost:3000/
```

## Dockerfile を用意する

```shell
touch Dockerfile
```

```docker:Dockerfile
FROM node:19.8.1-alpine3.17 as base

WORKDIR /app
RUN chown -R node:node /app

RUN apk add --no-cache bash tini tzdata \
    && ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime \
    && echo "Asia/Tokyo" > /etc/timezone \
    && apk del tzdata \
    && rm -rf /var/cache/apk/*

ENV TZ=Asia/Tokyo

FROM base as development

COPY --chown=node:node package*.json ./

RUN npm i

COPY --chown=node:node . .

USER node

EXPOSE 8080

ENTRYPOINT ["tini", "--", "npm", "run", "start:dev"]

FROM base as build

COPY --chown=node:node package*.json ./
COPY --chown=node:node --from=development /app/node_modules ./node_modules
COPY --chown=node:node . .

RUN npm run build

ENV NODE_ENV=production

RUN npm i --omit=dev

USER node

FROM base as production

ENV NODE_ENV=production

COPY --chown=node:node --from=build /app/node_modules ./node_modules
COPY --chown=node:node --from=build /app/dist ./dist

EXPOSE 8080

USER node

ENTRYPOINT ["tini", "--", "node", "dist/main.js"]
```

## Docker で NestJS を動かす

開発用のイメージをビルドします。

```shell
docker build . --target development -t nestapp:dev
```

コンテナを立ち上げます。

```shell
docker run -p 3000:3000 nestapp:dev
```

`Hello World` が返ってくることを確認します。

```shell
curl http://localhost:3000
```

本番環境用のコンテナもビルドしてみます。

```shell
docker build . --target production -t nestapp:prod
```

コンテナを立ち上げます。

```shell
docker run -p 3000:3000 nestapp:prod
```

同じく `curl` で Hello World が返ってくることを確認します。

```shell
curl http://localhost:3000
```

次に、 docker compose で Docker コンテナを動かしてみます。

```shell
touch docker-compose.yml
```

`docker-compose.yml` は以下のようにします。

```yml:docker-compose.yml
version: "3.8"

services:
  app:
    build:
      context: .
      target: development
    volumes:
      - .:/app
    ports:
      - "3000:3000"
```

以下のコマンドで docker compose を立ち上げます。

```shell
docker compose up
```

`curl` で Hello World が返ってくることを確認します。

```shell
curl http://localhost:3000
```

最後に、`trivy` を使って脆弱性がないか確認します。

```shell
brew install trivy
```

以下のコマンドを実行します。

```shell
trivy image nestapp:prod
```

結果、脆弱性がないことが確認できました。

```shell
nestapp:prod (alpine 3.17.2)

Total: 0 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 0, CRITICAL: 0)
```

インターネット上では、「安易に alpine を使うのをやめて、 slim や distroless を使え」という記事が多かったのですが、 `slim` や `distroless` でビルドすると、どうしても `trivy` の脆弱性検査に引っかかってしまったので、結局 `alpine` をベースにしました(2023 年 3 月時点)
