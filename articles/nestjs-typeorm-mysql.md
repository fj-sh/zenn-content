---
title: "NestJS+TypeORMでローカルで動かしたMySQLに接続する"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs"]
published: false
---

[NestJS を Docker（Alpine）で動かす](https://zenn.dev/fjsh/articles/nestjs-docker-alpine)の続きです。

ローカル環境で MySQL を立ち上げて、NestJS + TypeORM で接続します。

## ローカル環境で MySQL を立ち上げる

1. my.cnf ファイルを作成し、MySQL の文字コード設定を UTF-8 に設定します。この例では、my.cnf という名前のファイルを作成します。

```shell
touch my.cnf
```

```txt:my.cnf
[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
default-time-zone='Asia/Tokyo'
sql_mode=''

[client]
default-character-set=utf8mb4
```

2. .env フィアルを作成し、MySQL の環境変数を設定します。

```
MYSQL_DATABASE=mydb
MYSQL_USER=myuser
MYSQL_PASSWORD=mypassword
MYSQL_ROOT_PASSWORD=myrootpassword
```

3. docker-compose.yml を修正し、 MySQL サービスを定義します。

```yml:docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      target: development
    container_name: app
    volumes:
      - .:/app
    ports:
      - "3000:3000"
    depends_on:
      - db
    env_file:
      - .env
  db:
    image: mysql:8.0
    container_name: db
    env_file:
      - .env
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./my.cnf:/etc/mysql/conf.d/custom.cnf

volumes:
  mysql_data:
```

`docker compose up` を実行すると、 MySQL のコンテナが作成されます。

## TypeORM から MySQL に接続する

```shell
npm install --save @nestjs/typeorm typeorm mysql
```

```shell
npm i @nestjs/config
```

### `process`, `__dirname` について、対応するファイルは tsconfig.json に含まれていませんと警告が出た場合

tsconfig.json ファイルを開いて、compilerOptions セクションに esModuleInterop と resolveJsonModule, "types": ["node"] を追加します。

```json:tsconfig.json
{
  "compilerOptions": {
    ...
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "types": ["node"],
    ...
  },
  ...
}
```

以下のコマンドで `@types/node` をインストールします。

```shell
npm install --save-dev @types/node
```
