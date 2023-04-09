---
title: "React Native + Expo のプロジェクトを始める"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react native", "expo"]
published: true
---

## Expo + TypeScript でプロジェクトを作成

[Using TypeScript](https://docs.expo.dev/guides/typescript/)

```shell
npx create-expo-app -t expo-template-blank-typescript

# ✔ What is your app named? … .
```

## ESLint の設定

```shell
npm install eslint --save-dev
```

```shell
npm init @eslint/config

✔ How would you like to use ESLint? · style
✔ What type of modules does your project use? · esm
✔ Which framework does your project use? · react
✔ Does your project use TypeScript? · Yes
✔ Where does your code run? · browser
✔ How would you like to define a style for your project? · guide
✔ Which style guide do you want to follow? · standard-with-typescript
✔ What format do you want your config file to be in? · JavaScript
Checking peerDependencies of eslint-config-standard-with-typescript@latest
The config that you've selected requires the following dependencies:

eslint-plugin-react@latest eslint-config-standard-with-typescript@latest @typescript-eslint/eslint-plugin@^5.43.0 eslint@^8.0.1 eslint-plugin-import@^2.25.2 eslint-plugin-n@^15.0.0 eslint-plugin-promise@^6.0.0 typescript@*
✔ Would you like to install them now? · Yes
✔ Which package manager do you want to use? · npm
```

`.eslintrc.js`は以下のように設定する。

```js:.eslintrc.js
module.exports = {
  env: {
    browser: true,
    es2021: true,
  },
  extends: ['plugin:react/recommended', 'standard-with-typescript', 'prettier'],
  overrides: [],
  parserOptions: {
    ecmaVersion: 'latest',
    sourceType: 'module',
    project: './tsconfig.json',
  },
  plugins: ['react'],
  rules: {
    'react/react-in-jsx-scope': 'off',
    '@typescript-eslint/explicit-function-return-type': 'off',
    '@typescript-eslint/no-misused-promises': 'off',
    '@typescript-eslint/no-floating-promises': 'off',
  },
};
```

## Prettier の設定

```shell
npm install --save-dev prettier eslint-config-prettier @trivago/prettier-plugin-sort-imports eslint-config-prettier
touch .prettierrc.json
```

`.prettierrc.json` は以下のように設定する。

```json:.prettierrc.json
{
  "arrowParens": "always",
  "bracketSpacing": true,
  "jsxBracketSameLine": false,
  "jsxSingleQuote": false,
  "quoteProps": "as-needed",
  "singleQuote": true,
  "semi": true,
  "printWidth": 100,
  "useTabs": false,
  "tabWidth": 2,
  "trailingComma": "es5",
  "importOrderSeparation": true,
  "importOrderSortSpecifiers": true
}
```

## （Optional）React Native Paper のインストール

[Getting Started](https://callstack.github.io/react-native-paper/docs/guides/getting-started/)

```shell
npm install react-native-paper react-native-safe-area-context
npx pod-install
```

`babel.config.js` を以下のように設定する。

```js:babel.config.js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ['babel-preset-expo'],
    env: {
      production: {
        plugins: ['react-native-paper/babel'],
      },
    },
  };
};
```

`App.tsx` で `<PaperProvider>` を読み込んで、React Native Paper を有効にする。

```ts:App.tsx
import { StatusBar } from 'expo-status-bar';
import { StyleSheet, Text, View } from 'react-native';
import { Provider as PaperProvider } from 'react-native-paper';

export default function App() {
  return (
    <PaperProvider>
      <View style={styles.container}>
        <Text>Open up App.tsx to start working on your app!</Text>
        <StatusBar style="auto" />
      </View>
    </PaperProvider>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
    alignItems: 'center',
    justifyContent: 'center',
  },
});
```

## シミュレーターで動作確認する

```shell
npx expo start
```

## 実機で動作確認する

`--tunnel`オプションをつけて起動する。

```shell
npx expo start --tunnel
```

起動時に表示される QR コードを実機のカメラで読み込めば OK。

端末に [Expo Go](https://apps.apple.com/jp/app/expo-go/id982107779)はインストールしておく。
