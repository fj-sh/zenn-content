---
title: "JavaScript を使って Azure Cosmos DB の Cassandra API を呼び出す"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["azure"]
published: true
---

Cosmos DB の Cassandra API を使って、データを保存する手順です。

迷わなそうなところはサクサクと書きます。

初期準備の手順は以下です。

- サービスから Cosmos DB を選択
- Azure Cosmos DB アカウントを作成
- Cassandra API を選択

「Azure Cosmos DB アカウントの作成 - Cassandra」の画面でサブスクリプションとリソースグループを適当に選択し、

- アカウント名：cassandra-sandbox
- 場所：東アジア（East Asia）
- 容量モード：Serverless

とします。バックアップなどの設定は今回は一番安そうなやつを適当に選びました。

Azure Cosmos DB アカウントに `cassandra-sandbox` が作成されます。

## KeySpace を作る

まずは Azure 上で KeySpace を作成します。

> RDBMS でいうところのデータベース。キースペースは Cassandra において、一番大きいデータのまとまりです。すべてのデータはどれかひとつのキースペースに所属します。キースペースの主な役割はどのようにデータの複製を配置するか(レプリケーション)を決定することです。
> [第 3 回　 CQL(Cassandra Query Language)の基礎](https://enterprisezine.jp/article/detail/7146)

Azure Cosmos DB アカウント上で、上の手順で作成した`cassandra-sandbox` という Cassandra DB を開きます。

概要 - ＋テーブルの追加

をクリックすると、右側に「Add Table」というページが出てきます。

ここで Keyspace を作成できます。

Keyspace name は `sandbox`

CREATE TABLE sandbox.`user`

```
(user_id int, user_name text, user_city text, PRIMARY KEY (user_id))
```

としました。

`cassandra-driver` をインストールします。

```console
npm i cassandra-driver
```

Azure CosmosDB の Cassandra に接続する JavaScript のコードは以下です。

必要な情報は Azure Cosmos DB アカウントのページの左メニュー「接続文字列」を開けばわかります。

```js
const cassandra = require("cassandra-driver");

process.env.NODE_TLS_REJECT_UNAUTHORIZED = "0";

const callCassandra = async () => {
  const keySpace = "sandbox"; // 概要 - テーブルの追加 で作成した keySpace
  const username = "sandbox-user"; // 接続文字列の ユーザー名
  const password = "password=="; // 接続文字列の プライマリ パスワード
  const contactPoints = "[ユーザー名].cassandra.cosmos.azure.com:10350"; // 接続文字列 の コンタクトポイント
  const localDataCenter = "East Asia"; // 概要ページの「読み取りの場所・書き取りの場所」に書いてある

  let authProvider = new cassandra.auth.PlainTextAuthProvider(
    username,
    password
  );

  let client = new cassandra.Client({
    contactPoints: [contactPoints],
    keyspace: keySpace,
    localDataCenter: localDataCenter,
    authProvider: authProvider,
    sslOptions: {
      secureProtocol: "TLSv1_2_method",
      rejectUnauthorized: false,
    },
  });
  await client.connect();

  let query = `CREATE TABLE IF NOT EXISTS sandbox.user (user_id int PRIMARY KEY, user_name text, user_city text)`;
  await client.execute(query);
  console.log("テーブルを作成しました。");

  const arr = [
    `INSERT INTO  ${keySpace}.user (user_id, user_name , user_city) VALUES (1, 'AdrianaS', 'Seattle')`,
    `INSERT INTO  ${keySpace}.user (user_id, user_name , user_city) VALUES (2, 'JiriK', 'Toronto')`,
    `INSERT INTO  ${keySpace}.user (user_id, user_name , user_city) VALUES (3, 'IvanH', 'Mumbai')`,
    `INSERT INTO  ${keySpace}.user (user_id, user_name , user_city) VALUES (4, 'IvanH', 'Seattle')`,
    `INSERT INTO  ${keySpace}.user (user_id, user_name , user_city) VALUES (5, 'IvanaV', 'Belgaum')`,
    `INSERT INTO  ${keySpace}.user (user_id, user_name , user_city) VALUES (6, 'LiliyaB', 'Seattle')`,
    `INSERT INTO  ${keySpace}.user (user_id, user_name , user_city) VALUES (7, 'JindrichH', 'Buenos Aires')`,
    `INSERT INTO  ${keySpace}.user (user_id, user_name , user_city) VALUES (8, 'AdrianaS', 'Seattle')`,
    `INSERT INTO  ${keySpace}.user (user_id, user_name , user_city) VALUES (9, 'JozefM', 'Seattle')`,
  ];

  for (const element of arr) {
    await client.execute(element);
  }

  console.log("データを入力しました。");

  query = `SELECT * FROM ${keySpace}.user`;
  const resultSelect = await client.execute(query);

  for (const row of resultSelect.rows) {
    console.log(
      "Obtained row: %d | %s | %s ",
      row.user_id,
      row.user_name,
      row.user_bcity
    );
  }

  client.shutdown();
};

callCassandra()
  .then(() => {
    console.log("Cassandra との通信が完了しました。");
  })
  .catch((err) => {
    console.log(err);
  });
```

実行結果は以下のようになります。

```console
テーブルを作成しました。
データを入力しました。
Obtained row: 5 | IvanaV | undefined
Obtained row: 6 | LiliyaB | undefined
Obtained row: 7 | JindrichH | undefined
Obtained row: 2 | JiriK | undefined
Obtained row: 9 | JozefM | undefined
Obtained row: 3 | IvanH | undefined
Obtained row: 4 | IvanH | undefined
Obtained row: 1 | AdrianaS | undefined
Obtained row: 8 | AdrianaS | undefined
Cassandra との通信が完了しました。
```

入力されたデータは Azure Portal 上でも確認できます。

![](https://storage.googleapis.com/zenn-user-upload/303b8d937a55-20220109.jpg)

### 参考

CosmosDB の Cassandra API にデータを挿入しようとしても、以下のようなエラーが出てうまくいかないケースもあるかと思います。

```console
NoHostAvailableError: All host(s) tried for query failed. First host tried, 52.174.23.201:10350: OperationTimedOutError: The host 52.175.25.211:10350 did not reply before timeout 12000 ms
```

- `localDataCenter`が設定されていないケース
- `process.env.NODE_TLS_REJECT_UNAUTHORIZED = "0";`

などを疑ってください。

[ネイティブ SDK パッケージを使用して Azure 上の Cassandra DB に接続する](https://docs.microsoft.com/ja-jp/azure/developer/javascript/how-to/with-database/use-cassandra-as-cosmos-db)
