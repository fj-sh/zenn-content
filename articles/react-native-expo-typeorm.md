---
title: "React Native + Expo でTypeORMを使って端末にデータを保存する"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react native", "expo", "typeorm"]
published: true
---

React Native + Expo のプロジェクトで TypeORM を使う手順を紹介します。

## 必要なライブラリのインストール

```shell
npm install expo-sqlite typeorm reflect-metadata
```

[expo-example](https://github.com/typeorm/expo-example)
[Expo SQLite + TypeORM](https://dev.to/jgabriel1/expo-sqlite-typeorm-4mn8)

## tsconfig.json の設定

TypeORM を使うために必要な設定を追加。

```json:tsconfig.json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "experimentalDecorators": true,
    "strictPropertyInitialization": false,
    "emitDecoratorMetadata": true,
    "strict": true
  }
}
```

## ormconfig の設定

```shell
mkdir -p lib/{databases,entities}
```

```shell
touch lib/databases/typeorm.config.ts
```

Entity を作成後、`typeorm.config.ts` は以下のように設定します。

（Entity は各自が作成したものを入れてください）

```ts:lib/databases/typeorm.config.ts
import { Category } from '../entities/Category';
import { DataSource } from 'typeorm';

export const AppDataSource = new DataSource({
  type: 'expo',
  driver: require('expo-sqlite'),
  database: 'category_memo', // データベース名
  logging: [],
  synchronize: true, // このオプションがtrueの場合、アプリ起動時にエンティティとデータベーススキーマが同期されます。
  entities: [Category], // ここにエンティティを追加する
});

```

## AppDataSource の初期化

`App.tsx` などの 1 番最初に読み込まれるコンポーネント内で以下のように初期化します。
以下の例は Main.tsx で初期化する例です。初期化処理自体は別の関数に切り出しても問題ないと思います。

```tsx:src/lib/components/Main.tsx
export default function Main({ routeIndex, categoryReordered, setCategoryReordered }: MainProps) {
  const [initialized, setInitialized] = useState(false);

  useEffect(() => {
    async function fetchData() {
      const isInitialized = AppDataSource.isInitialized;
      if (!isInitialized) {
        await AppDataSource.initialize();
      }

      setInitialized(true);
    }
    fetchData();
  }, []);

  return ()
}
```

以下のように、AppDataSource の初期化が完了したタイミングで、必要なデータを取得します。

```ts
useEffect(() => {
  if (initialized) {
    getCategories().then((storedCategories) => {
      setCategories(storedCategories);
    });
  }
}, [AppDataSource, initialized]);
```

`AppDataSource` が初期化されていないうちにデータを取得しようとして、予期せぬエラーに遭遇してしまったため、このような書き方にしています。
もっと綺麗に書く余地はありそうです。

## Entity の作成

Entity の例です。

```ts:src/lib/entities/Category.ts
import { Column, Entity, PrimaryGeneratedColumn, Unique } from "typeorm";

@Entity()
@Unique(["name"])
export class Category {
  @PrimaryGeneratedColumn()
  categoryId: number;

  @Column({ type: "integer" })
  index: number;

  @Column({ type: "varchar", length: 255 })
  name: string;

  @Column({ type: "varchar", length: 255, nullable: true })
  createdAt: Date;

  @Column({ type: "varchar", length: 255, nullable: true })
  updatedAt: Date;
}
```

## データを取得する

データを取得する関数を作ります。

```ts:src/lib/databases/categoryRepositories.ts

export async function getCategories(): Promise<Category[]> {
  try {
    return await AppDataSource.manager.find(Category, {
      order: { index: 'ASC' },
    });
  } catch (error) {
    console.error('Error getting categories:', error);
    return [];
  }
}
```

## データを保存する

データを保存する関数の例です。

```ts:src/lib/databases/categoryRepositories.ts
export async function saveCategory(category: Category): Promise<Category> {
  try {
    const existingCategory = await AppDataSource.manager.findOne(Category, {
      where: { name: category.name },
    });

    if (existingCategory != null) {
      throw new Error('Category name already exists');
    }

    return await AppDataSource.manager.save(category);
  } catch (error) {
    console.error('Error saving category:', error);
    throw error;
  }
}
```

これらの関数を使って、データを取得し、コンポーネントで描画します。
