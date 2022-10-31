---
title: "NestJS+Nginx+MySQLの開発環境をDocker化する"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs", "docker"]
published: true
---

## NestJS のセットアップ

```shell
npm i -g @nestjs/cli
nest new nestjs-docker

# Which package manager would you ❤️  to use? (Use arrow keys)
# ❯ npm

cd nestjs-docker
```

参考：[First steps](https://docs.nestjs.com/first-steps)

## 必要なファイル、ディレクトリの作成

Nginx や MySQL の設定ファイルを置くために docker ディレクトリを作成します。

```shell
mkdir -p docker/db/data
```

```shell
touch docker/{entrypoint-nginx.sh,vhost.template}
```

```shell
touch docker/db/my.cnf
```

Docker の起動に必要なファイルを作成します。

```shell
touch {Dockerfile,Dockerfile-nginx,Dockerfile-prod,docker-compose.yml,.dockerignore}
```

環境変数ファイルを作成します。

```shell
cat <<EOF > .env.sample
DATABASE_NAME=nestdb
DATABASE_USER=nestuser
DATABASE_PASSWORD=nestpass
DATABASE_ROOT_PASSWORD=nestrootpass
DATABASE_PORT=3306
NGINX_SERVER_NAME=localhost
NEST_HOST=nest
NEST_PORT=3000
NGINX_MAX_BODY=100M
NGINX_PORT=80
EOF
```

`.env` は Git で管理しないので、`.env.sample` を作ってコピーします。

```shell
cp .env.sample .env
```

`.gitignore` に `.env` と後に Docker から使う MySQL のデータボリュームを追加します。

```shell
echo "\n.env" >> .gitignore
echo "docker/db/data" >> .gitignore
```

## Dockerfile

`Dockerfile` に以下をコピー＆ペーストします。

```docker:Dockerfile
FROM node:16-alpine3.15

RUN apk add --update python3 make g++\
   && rm -rf /var/cache/apk/*

WORKDIR /app
RUN chown -R node:node /app
USER node

ENV NODE_ENV development

COPY --chown=node:node package.json package-lock.json ./
RUN npm ci

COPY --chown=node:node . .


EXPOSE 3000

CMD [ "npm", "run", "start:dev" ]
```

`Dockerfile-nginx` に以下をコピー＆ペーストします。

```docker:Dockerfile-nginx
FROM nginx:alpine

COPY ./docker/entrypoint-nginx.sh /

RUN set -ex && \
	apk add --no-cache bash && \
	chmod +x /entrypoint-nginx.sh

COPY ./docker/vhost.template /etc/nginx/conf.d/vhost.template

CMD ["/entrypoint-nginx.sh"]
```

`Dockerfile-prod` に以下をコピー＆ペーストします。

```docker:Dockerfile-prod
FROM node:16-alpine3.15

WORKDIR /app
RUN chown -R node:node /app
USER node

ENV NODE_ENV production

COPY --chown=node:node package.json package-lock.json ./

# install dev dependencies too
RUN npm ci

COPY --chown=node:node . .
RUN set -x && npm run build

EXPOSE 3000

CMD [ "node", "dist/main.js" ]
```

`.dockerignore` に以下をコピー＆ペーストします。

```txt:.dockerignore
node_modules/
```

## docker-compose.yml

`docker-compose.yml` に以下をコピー＆ペーストします。

```yml:docker-compose.yml
version: '3'
services:
  nest:
    build:
      context: .
      dockerfile: Dockerfile # 本番環境は Dockerfile-prod を指定する
    container_name: nest
    depends_on:
      - db
    volumes:
      - ./src:/app/src
      - .env:/app/.env

  nginx:
    build:
      context: .
      dockerfile: Dockerfile-nginx
    container_name: nest-nginx
    env_file:
      - .env
    depends_on:
      - nest
    environment:
      - NGINX_SERVER_NAME=${NGINX_SERVER_NAME}
      - NEST_HOST=${NEST_HOST}
      - NEST_PORT=${NEST_PORT}
      - NGINX_MAX_BODY=${NGINX_MAX_BODY}
      - NGINX_PORT=${NGINX_PORT}
    ports:
      - ${NGINX_PORT}:${NGINX_PORT}

  db:
    image: mysql:5.7
    container_name: nest-db
    env_file:
      - .env
    environment:
      MYSQL_DATABASE: ${DATABASE_NAME}
      MYSQL_USER: ${DATABASE_USER}
      MYSQL_PASSWORD: ${DATABASE_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DATABASE_ROOT_PASSWORD}
    ports:
      - "${DATABASE_PORT}:3306"
    volumes:
      - ./docker/db/data:/var/lib/mysql
      - ./docker/db/my.cnf:/etc/mysql/conf.d/my.cnf
```

## my.cnf の作成

`docker/db/my.cnf` に以下をコピー＆ペーストします。

```txt:my.cnf
[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_bin
default-time-zone='Asia/Tokyo'

[client]
default-character-set=utf8mb4
```

## Nginx の設定

`docker/vhost.template` に以下をコピー＆ペーストします。

```txt:vhost.template
server {
	listen ${NGINX_PORT};
	server_name ${NGINX_SERVER_NAME};
	root /app/public;
	client_max_body_size ${NGINX_MAX_BODY};

	location / {
		# try_files $uri =404;
		proxy_pass  http://${NEST_HOST}:${NEST_PORT};
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Host $server_name;
		proxy_set_header X-Forwarded-Proto https;
	}
}
```

`docker/entrypoint-nginx.sh` に以下をコピー＆ペーストします。

```shell:entrypoint-nginx.sh
#!/bin/bash

vars=$(compgen -A variable)
subst=$(printf '${%s} ' $vars)
envsubst "$subst" < /etc/nginx/conf.d/vhost.template > /etc/nginx/conf.d/default.conf
nginx -g 'daemon off;'
```

## Docker Compose を起動

`docker compose up` で Docker を起動してみます。

```shell
docker compose up
```

動作確認ができたら `control + c` で Docker Compose を停止して、コンテナを削除します。

```shell
docker compose down
```

Docker をバックグラウンドで起動します。

```shell
docker compose up -d
```

curl で `localhost` から GET できることを確認します。

```shell
curl localhost

# Hello World!%
```

ここまでで NestJS を Docker で動かす準備ができました。

次は NestJS と TypeORM を組み合わせてマイグレーションを実行していきます。

## 参考

- [Docker で Node.js を動かすときのベストプラクティス](https://blog.shinonome.io/nodejs-docker/)
- [nestjs-boilerplate](https://github.com/Vivify-Ideas/nestjs-boilerplate)
