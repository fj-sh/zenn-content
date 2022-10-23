---
title: "Chrome拡張をVite+TypeScript+Reactで作る"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["chrome拡張"]
published: true
---

Vite + TypeScript + React で Chrome 拡張を作る手順です。

以下のコマンドを実行します。

```console
npm init vite@latest
✔ Project name: … .
✔ Current directory is not empty. Remove existing files and continue? … yes
✔ Select a framework: › react
✔ Select a variant: › react-ts

Scaffolding project in /Users/fj/dev/searchable-bookmarks...

Done. Now run:

  npm install
  npm run dev
```

```console
npm i @crxjs/vite-plugin -D --force

# Fix the upstream dependency conflict, or retry のエラーが出る場合は、問題がないか確認した上で --force オプションをつけてインストール
# npm i @crxjs/vite-plugin -D --force
```

Chrome の API に型をつけるために[@types/chrome](https://www.npmjs.com/package/@types/chrome)をインストールします。

```console
npm i -D @types/chrome
```

`vite.config.ts` を編集します。

```ts:vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { crx, defineManifest } from '@crxjs/vite-plugin'

const manifest = defineManifest({
  manifest_version: 3,
  name: '拡張機能の名前',
  description:
    '拡張機能の説明',
  version: '1.0.0',
  icons: {
    '16': 'icon.png',
    '48': 'icon.png',
    '128': 'icon.png',
  },
  action: {
    default_popup: 'index.html',
  },
})

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [react(), crx({ manifest })],
})
```

`icons` のアイコンの画像は`public`直下に置きます。

permission の追加が必要な場合は、`vite.config.ts` の以下の部分に追加します。

[Declare permissions](https://developer.chrome.com/docs/extensions/mv3/declare_permissions/)

```ts:vite.config.ts
const manifest = defineManifest({
  manifest_version: 3,
  name: '拡張機能の名前',
  description:
    '拡張機能の説明',
  version: '1.0.0',
  icons: {
    '16': 'icon.png',
    '48': 'icon.png',
    '128': 'icon.png',
  },
  action: {
    default_popup: 'index.html',
  },
  permissions: [
    "tabs",
    "bookmarks",
    "unlimitedStorage"
  ],
})

```

`content_scripts` や `background` の追加が必要な場合も、`vite.config.ts` に追記します。
ソースコードの指定は、`src/`から始めます。suffix は `.ts`のまま指定して OK です。

```ts:vite.config.ts
const manifest = defineManifest({
  // 省略
  action: {
    default_popup: 'index.html',
  },
  permissions: ['storage', 'alarms', 'tabs'],
  background: {
    service_worker: 'src/background-script/background.ts',
    type: 'module',
  },
  content_scripts: [
    {
      matches: ['https://tinder.com/*'],
      js: ['src/content-script/index.ts'],
    },
  ],
})

```

- [Content scripts](https://developer.chrome.com/docs/extensions/mv3/content_scripts/)
- [Migrating from background pages to service workers](https://developer.chrome.com/docs/extensions/mv3/migrating_to_service_workers/)

## ブラウザで Chrome 拡張を動かしてみる

開発中は以下のコマンドで Vite を起動します。

```console
npm run dev
```

タブバーの横にある Chrome 拡張のアイコンをクリックして、「Chrome 拡張を管理」を開きます。

右上に「デベロッパーモード」というトグルボタンがあるので、こちらを「ON」にします。

「パッケージ化されていない拡張機能を読み込む」をクリックします。

開発中のプロジェクトには`dist`ディレクトリができますので、その`dist`ディレクトリを選択します。

すると拡張機能が読み込まれます。

リリースするときは以下のコマンドを実行します。

```console
npm run build
```

同じように`dist`ディレクトリを読み込めば OK です。

Chrome ウェブストアに公開するためには、[デベロッパーダッシュボード](https://chrome.google.com/webstore/developer/dashboard)から、ビルドした拡張機能を zip で圧縮してアップロードすることになります。

## Chrome Web Store に Chrome 拡張をリリースする手順

[Chrome Web Store Developer Dashboard](https://chrome.google.com/webstore/developer/dashboard)の「アイテム」メニューを開き、「新しいアイテム」ボタンをクリックします。

`npm run build` でビルドした dist フォルダを圧縮します（Mac の場合はフォルダを右クリックして「"dist"を圧縮」を押します。

zip ファイルをアップロードすると、パッケージのタイトルとパッケージの概要は manifest.json から自動で入力されます。

説明やカテゴリなどの必須項目を入力してきます。

説明はしっかり書かないとリジェクトされるので注意が必要です。

### ショップアイコン

「画像アセット」でショップアイコンを設定する際は、[PNG のサイズ変更](https://www.iloveimg.com/ja/resize-image/resize-png)というサイトが役に立ちました。

サイズを 128x128 ピクセルに合わせないと画像がアップロードできないからです。

### スクリーンショット

スクリーンショット 5 枚まで指定できます。
アップロードしたスクリーンショット（JPG）は Chrome Web Store で表示されます。

PowerPoint で説明の絵を作り、 1280 × 800 のサイズを指定して、JPG でエクスポートしました。

### プライバシーへの取り組み

以下の内容について説明文を書きます。

- 拡張機能の用途の説明
- 権限が必要な理由
- データ使用

### 決済と配布地域

以下の情報を設定します。

- 決済方法
- 公開設定

### 審査のため送信

一通りの情報を入力し終わったら、右上の「審査のため送信」をクリックします。

「審査の合格後にアプリを自動的に公開する」にチェックを入れておけば、合格後に勝手に公開してくれます。

審査は 1 日以内に終わります。
