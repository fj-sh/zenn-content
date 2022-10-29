---
title: "npmでパッケージ管理するための基本知識"
emoji: "📚"
type: "tech"
topics: ["nodejs"]
published: true
---

npm（Node Package Manager）の略で、 Node.js のデフォルトのパッケージ管理ツールです。
npm は依存パッケージとバージョンの管理を自動化します。 Isaac Z. Schlueter によって開発されました。

## package.json

パッケージのメタデータを管理するのが package.json と呼ばれるJSONファイルです。
`npm init`で作成します。 `-y` を指定すれば、　CLIからの質問を省略してすぐに package.json を作れます。

```
$ npm init -y
```

以下のような package.json が作られます。

```json:package.json
{
  "name": "npm-package",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

### main

package.json の`"main":`にはパッケージのエントリポイントとなるJavaScriptファイルを指定します。ファイルはパッケージルートからの相対パスを指定します。

インストールしたパッケージを利用する際、パッケージ名を指定して `require()`（または `import` ）しますが、このときは `main` で指定したモジュールの exports オブジェクトが返されます。

ためしにエントリポイントとなる index.js を作成し、 `require()` してみます。

```console
$ tree -L 2
.
└── npm-package
    ├── index.js
    └── package.json
```

```node:npm-package/index.js
'use strict'
module.exports = 'Hello from my package'
```

REPLで`require()`してみます。

```
$ node 
> require('./npm-package')
'Hello from my package'
```

### license, private

license にはパッケージ利用にあたってのライセンスを指定します。
パッケージを非公開としたい場合は`private: true`を合わせて指定します。

```json:package.json
{
  "license": "UNLICENSED",
  "private": true
}
```

### dependencies

パッケージの依存先パッケージを管理するための項目です。
`dependencies`に依存パッケージを追加するには、`npm install <パッケージ名>`というコマンドを使用します。

コマンド例は以下です。

```console
$ npm install rimraf
$ node
> require('rimraf')
[Function: rimraf] { sync: [Function: rimrafSync] }
```

`npm install`を実行したら、以下のようなことが起こります。

- package.jsonの`dependencies`に`rimraf`が追加される
- node_modulesというディレクトリが作られ、その中に依存パッケージの実体が配置される

依存関係の構造は依存ツリーと呼ばれます。
依存ツリーは`npm ls`で可視化できます。

```console
$ npm ls --all
npm-package@1.0.0 /Users/fj/dev/js-sandbox/node/npm-package
└─┬ rimraf@3.0.2
  └─┬ glob@7.1.7
    ├── fs.realpath@1.0.0
    ├─┬ inflight@1.0.6
```

### devDependencies

devDependencies には開発時のみ必要なパッケージを書きます。

`eslint` や `jest`など、本番環境で使わないパッケージは `devDependencies` に書きます。

devDependencies に追加するには以下のように、`-D` か `--save-dev` オプションをつけて npm を実行します。

```shell
npm install -D A
```

### バージョンの指定方法

package.json の devDependencies を見ると、バージョンに `^`（キャレット）がついていたりします。
これらの意味を説明します。

```json
{
  "devDependencies": {
    "@types/aws-lambda": "^8.10.108",
    "@typescript-eslint/eslint-plugin": "^5.41.0",
    "@typescript-eslint/parser": "^5.41.0",
    "esbuild": "^0.15.12",
    "eslint": "8.22.0",
    "eslint-config-google": "^0.14.0",
    "nodemon": "^2.0.20",
    "ts-node": "^10.9.1"
  }
}
```

ちなみにパッケージのバージョンの区切りは`メジャー.マイナー.パッチ`という意味です。

参考：[semver(1) -- The semantic versioner for npm](https://www.npmjs.com/package/semver/v/4.3.6)


#### チルダ「~」

一番末尾のバージョンに変更があるパッケージがあった場合のみ更新します。

例：`~1.2.3` なら `1.2.3` 以上 `1.3.0` 未満のバージョンに依存する

#### キャレット（^）

一番左端のバージョンは固定して、それ以外のバージョンで新しいものがあれば更新します。

例：`^1.2.3` は `1.2.3` 以上 `2.0.0` 未満のバージョンに依存する。


#### 全てのバージョンを許可する（*アスタリスク）

```
"vue": *
```

#### ビタっとバージョンを指定する

`""`で囲めば、バージョンを固定できます。

```
"eslint": "8.22.0"
```


### package-lock.json

package-lock.json には`npm instal <パッケージ名>`によってインストールされたパッケージの情報が、間接的に依存するものも含めてすべて記述されます。

package-lock.jsonによって、アプリケーションの依存の状態を完全に再現させることができます。

#### `npm ci`

`npm ci`はnode_modulesがあればそれを削除し、package-lock.json の情報に基づいてパッケージをインストールします。

#### `npm install`

`npm install`はnode_modulesを削除せず、必要なパッケージのうち未インストールのものだけをインストールします。

#### `npm uninstall`

`npm install <パッケージ名>`でインストールしたパッケージをアンインストールするには`npm uninstall <パッケージ名>`を実行します。


### npx

2017年に公開された node バージョン 5.2.0 以降では `npx` コマンドが使えます。
`npx <コマンド名>` はそれがローカルまたはグローバルにインストールされたnpmパッケージのコマンドにあれば実行し、なければnpmレジストリから該当するパッケージをインストールして実行します。

インストールされたパッケージはキャッシュに保存され、2回目以降の `npx <コマンド名>`は高速に動作します。

npx が導入される前はローカルインストールしたパッケージのコマンドを scripts を使わずに実行する方法がなかったため、グローバルインストールによってグローバルにPATHを通すことが好まれる場面もありました。

ですが、　npx の導入以後、`npm -g`　`npm --global` へのインストールは可能な限り使わない傾向が強まりました。

グローバルインストールを使うと、パッケージへの依存に関する情報をコードで共有できなくなるからです。


## Yarn

Yarn は2016年にFacebook社からリリースされたパッケージ管理ツールです。
パッケージインストールの速さとyarn.lockファイルが魅力でした。

インストールの速さは並列処理とキャッシュによって実現されます。
yarn.lock は package-lock.json に相当するものです。

Yarn リリース当初 npm には package-lock.json がなかったという経緯があります。

### npm と Yarn のコマンドの対応関係

|コマンドの機能| npm | Yarn |
| --- | --- | --- |
|package.json の作成| `npm init`| `yarn init` |
|依存パッケージの一括インストール| `npm install` | `yarn install` |
|package-lock.jsonに従った依存パッケージの一括インストール| `npm ci` | `yarn install --frozen-lockfile`|
|npm パッケージのインストール（dependencies）| `npm install <パッケージ名>`| `yarn add <パッケージ名>`|
|npm パッケージのインストール（devDependencies）| `npm install <パッケージ名> --save-dev` | `yarn add <パッケージ名> --dev`|
|npm パッケージのグローバルインストール|`npm install <パッケージ名> --global`|`yarn global add <パッケージ名>`|
|npm パッケージのアンインストール|`npm uninstall <パッケージ名>`|`yarn remove <パッケージ名>`|
|バージョンの揚げ方を指定したバージョンの更新| `npm version patch|minor|major`|`yarn version --patch|--minor|--major`|
|具体的なバージョンを指定したバージョンの更新|`npm version <新しいバージョン>`|`yarn version --new-version <新しいバージョン>`|
|scripts のスクリプトの実行|`npm run <スクリプト名> [--オプション]`|`yarn[ run] <スクリプト名> [オプション]`|
|インストールしたnpmパッケージの実行可能ファイルの実行|`npx <コマンド名> [オプション]`|`yarn[ run] <コマンド名> [オプション]`|
|npmパッケージの公開|`npm publish`|`yarn publish`|


すでにnpmを使っているプロジェクトをYarnに移行したい場合は`yarn import`コマンドを実行することでpackage-lock.jsonからyarn.lockを生成してくれます。


## Pnpm 

高速、かつディスク容量効率が良いパッケージマネージャーとして、[Pnpm](https://pnpm.io/ja/)を使っているプロジェクトも増えてきているようです。
npm、yarn、pnpm で比較すると、pnpm が最も歴史が浅く、新しいです。