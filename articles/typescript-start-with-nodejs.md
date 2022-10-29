---
title: "TypeScript + Node.js + ESLint の環境をとにかく早く構築する"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nodejs"]
published: true
---

TypeScript + Node.js の開発環境をてっとり早く作りたい方のための記事です。

- Node.js
- npm
- [gibo](https://github.com/simonwhitaker/gibo)

はインストール済として進めます。

```shell
$ node -v
v16.15.0

$ npm -v
8.5.5

$ gibo -v
gibo 2.2.7
```

## プロジェクトディレクトリを作成

```shell
project_name=sample-proj && mkdir ${project_name} && cd ${project_name} && mkdir src && touch src/index.ts
```

```shell
cat <<EOF > src/index.ts
console.log('TS Project start!')
EOF
```

## Git の環境の整理

```shell
git init && gibo dump node > .gitignore
```

## package.json の生成と TypeScript のインストール

```shell
npm init -y && npm i typescript @types/node
```

## `tsconfig.json` の作成

```shell
npx tsc --init --rootDir src --outDir build \
--esModuleInterop --resolveJsonModule --lib es6 \
--module commonjs --allowJs true --noImplicitAny true
```

## `ts-node`と`nodemon`をインストール

```shell
npm install --save-dev ts-node nodemon
```

```shell
cat <<EOF > nodemon.json
{
  "watch": ["src"],
  "ext": ".ts,.js",
  "ignore": [],
  "exec": "ts-node ./src/index.ts"
}
EOF
```

`package.json` に追記。

```json:package.json
  "scripts": {
    "start:dev": "nodemon",
  },
```

## `ESLint`をインストール

```shell
npx eslint --init

✔ How would you like to use ESLint? · style
✔ What type of modules does your project use? · esm
✔ Which framework does your project use? · none
✔ Does your project use TypeScript? · No / Yes
✔ Where does your code run? · browser
✔ How would you like to define a style for your project? · guide
✔ Which style guide do you want to follow? · google
✔ What format do you want your config file to be in? · JavaScript
Checking peerDependencies of eslint-config-google@latest
Local ESLint installation not found.
The config that you've selected requires the following dependencies:

@typescript-eslint/eslint-plugin@latest eslint-config-google@latest eslint@>=5.16.0 @typescript-eslint/parser@latest
✔ Would you like to install them now? · No / Yes
✔ Which package manager do you want to use? · npm
```

```shell
cat <<EOF > .eslintignore
node_modules
dist
EOF
```

`package.json` に追記。

```json:package.json
  "scripts": {
    "start:dev": "nodemon",
    "lint": "eslint . --ext .ts",
    "eslint:fix": "eslint --fix"
  },
```

## TypeScript をビルドする

`esbuild`をインストールします。

```
npm install -D esbuild
```

package.json に以下を追記します。

```json:package.json
"scripts": {
  "prebuild": "rm -rf dist",
  "build": "esbuild src/index.ts --bundle --minify --sourcemap --platform=node --target=es2020 --outfile=dist/index.js",
},
```

## VSCode と WebStorm の設定

```shell
mkdir .vscode && touch .vscode/settings.json
```

```shell
cat <<EOF > .vscode/settings.json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true
}
EOF
```

```shell
mkdir -p .idea/jsLinters && touch .idea/prettier.xml && touch .idea/jsLinters/eslint.xml
```

```shell
cat <<EOF > .idea/jsLinters/eslint.xml
<?xml version="1.0" encoding="UTF-8"?>
<project version="4">
  <component name="EslintConfiguration">
    <option name="fix-on-save" value="true" />
  </component>
</project>
EOF
```

## `ESLint: TypeError: this.libOptions.parse is not a function` が出た場合

`package.json` の ESLint のバージョンを `8.22.0` に変更して、 `npm install` を実行します。

```diff json:package.json
-    "eslint": "^8.26.0",
+    "eslint": "8.22.0",
```

キャレット（`^`）は一番左端のバージョンは固定して、それ以外のバージョンで新しいものがあれば更新する、という意味です。

[npm でパッケージ管理するための基本知識](/npm-fundamentals)

## ESLint の `max-len: 80` を緩和する

行の最大文字数が 80 文字である、と ESLint に怒られてしまうことがあるかもしれません。
80 文字は少なすぎるので、最大文字数を 120 文字にしました。

以下のように`'rules':` に `'max-len': ['error', {'code': 120}],`を設定すれば OK です。

```js:.eslintrc.js
module.exports = {
  'env': {
    'browser': true,
    'es2021': true,
  },
  'extends': [
    'google',
  ],
  'parser': '@typescript-eslint/parser',
  'parserOptions': {
    'ecmaVersion': 'latest',
    'sourceType': 'module',
  },
  'plugins': [
    '@typescript-eslint',
  ],
  'rules': {
    'max-len': ['error', {'code': 150}],
  },
};
```

## 参考

- [Node.js QuickStart](https://basarat.gitbook.io/typescript/nodejs)
- [How to Setup a TypeScript + Node.js Project](https://khalilstemmler.com/blogs/typescript/node-starter-project/)
- [TypeScript + Node.js プロジェクトのはじめかた 2020](https://qiita.com/notakaos/items/3bbd2293e2ff286d9f49)
- [How to use ESLint with TypeScript](https://khalilstemmler.com/blogs/typescript/eslint-for-typescript/)
- [Set up Node.js project with Typescript, ESLint, and Prettier](https://blog.tericcabrel.com/set-up-a-nodejs-project-with-typescript-eslint-and-prettier/)
