---
title: "Node.jsでAPIから取得した結果をキャッシュする"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nodejs"]
published: true
---

## やりたいこと

- 別サービスの API から取得した結果をキャッシュする（メモリに残す）
- キャッシュにデータがある場合は、別サービスの API をコールせず、キャッシュからレスポンスを返す

## シチュエーション

Next.js の API を叩くと、 Next.js から [RESAS](https://resas.go.jp/#/13/13101) の API を呼び出し、都道府県の一覧と、都道府県の総人口を取得する。

何度も API を叩くと Rate Limit に引っかかるかもしれない。有料なら課金されてしまう。
それに都道府県の情報はそれほど頻繁に変わるものではないので、一度取得した結果を使い回しても問題はないはずだ。

Next.js のサーバ起動後、最初のリクエストは RESAS の API を呼び出し、その結果はキャッシュして使い回そう。

## 使ったライブラリ

[node-cache](https://www.npmjs.com/package/node-cache) を使って結果をキャッシュした。

## ソースコード

```js:lib/resas.ts
import axios from 'axios'
import NodeCache from 'node-cache'

const RESAS_API_KEY = process.env.RESAS_API_KEY
const RESAS_ENDPOINT = 'https://opendata.resas-portal.go.jp/api/v1'

type Prefecture = {
  prefCode: number
  prefName: string
}

type PopulationOfPrefecture = {
  prefecture: Prefecture
  populations: Population[]
}

type Population = {
  year: string
  value: string
}

const cache = new NodeCache({ stdTTL: 1000 * 60 * 60 * 24 })

/**
 * RESAS API から都道府県リストを取得する。
 * @return {Prefecture[]} 都道府県のリスト
 * @see [人口構成](https://opendata.resas-portal.go.jp/docs/api/v1/population/composition/perYear.html)
 */
export const getPrefList = async (): Promise<Prefecture[]> => {
  if (RESAS_API_KEY == null) throw new Error()
  if (cache.get('prefectures')) {
    console.log('キャッシュから県の情報を返します。')
    return cache.get('prefectures') as Prefecture[]
  }
  const prefecturesUrl = `${RESAS_ENDPOINT}/prefectures`
  const { data } = await axios.get(prefecturesUrl, { headers: { 'X-API-KEY': RESAS_API_KEY } })
  const prefectures: Prefecture[] = data.result
  cache.set('prefectures', prefectures)
  return prefectures
}

/**
 * 都道府県コードから都道府県を返す。
 * @param {number} prefCode 都道府県コード
 * @return {Promise<Prefecture>>} 都道府県
 */
const getPrefById = async (prefCode: number): Promise<Prefecture> => {
  const prefectures =
    cache.get('prefectures') != null
      ? (cache.get('prefectures') as Prefecture[])
      : await getPrefList()
  const pref = prefectures.filter((prefecture) => prefecture.prefCode === prefCode)
  if (pref && pref.length !== 0) {
    return pref[0]
  } else {
    throw new Error(`prefCode ${prefCode} doesn't exist.`)
  }
}

/**
 * prefCode で渡した都道府県の総人口の配列を返す
 * @param {number} prefCode RESASの都道府県APIから取得した都道府県ID
 * @return {Promise<PopulationOfPrefecture>} 都道府県の総人口推移
 */
export const getAllPopulationByPrefId = async (
  prefCode: number
): Promise<PopulationOfPrefecture> => {
  if (RESAS_API_KEY == null) throw new Error()
  const prefecture = await getPrefById(prefCode)
  if (cache.get(`population-prefId-${prefCode}`) != null) {
    console.log('キャッシュから総人口の情報を返します。')
    const cachedPrefAllPopulations = cache.get(`population-prefId-${prefCode}`) as Population[]
    return { prefecture, populations: cachedPrefAllPopulations }
  }
  const populationByPrefIdUrl = `${RESAS_ENDPOINT}/population/composition/perYear?cityCode=-&prefCode=${prefCode}`
  const { data } = await axios.get(populationByPrefIdUrl, {
    headers: { 'X-API-KEY': RESAS_API_KEY },
  })

  const filteredAllPopulations = data.result.data.filter(
    (populationData: any) => populationData.label === '総人口'
  )
  const allPopulations = filteredAllPopulations[0].data

  cache.set(`population-prefId-${prefCode}`, allPopulations)
  return { prefecture, populations: allPopulations }
}

```

キャッシュする準備をしているのは以下の部分。

```js
import NodeCache from "node-cache";

const cache = new NodeCache({ stdTTL: 1000 * 60 * 60 * 24 });
```

キャッシュがあるかを確認しているのは以下の部分。

```js
if (cache.get(`population-prefId-${prefCode}`) != null) {
  console.log('キャッシュから総人口の情報を返します。')
  const cachedPrefAllPopulations = cache.get(`population-prefId-${prefCode}`) as Population[]
  return { prefecture, populations: cachedPrefAllPopulations }
}
```

キャッシュに保存しているのは以下の部分。

```js
cache.set(`population-prefId-${prefCode}`, allPopulations);
```

## API

Next.js の `pages/api` 以下の API から呼び出して使うイメージ。

`await getAllPopulationByPrefId(1)` はサンプルのため、固定の引数を渡している。

本来は `req`からパラメータを取得しなければならないが、この記事の主題ではない。

```js:pages/api/populations.ts
import { NextApiRequest, NextApiResponse } from 'next'
import { getAllPopulationByPrefId } from '../../lib/resas'

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const populations = await getAllPopulationByPrefId(1)
  res.status(200).json(populations)
}
```

`http://localhost:3000/api/populations` のレスポンスはこうなる。

```json
{
"prefecture": {
"prefCode": 1,
"prefName": "北海道"
},
"populations": [
{
"year": 1960,
"value": 5039206
},
略
```

1 度目のレスポンスはちょっとだけ遅い。 RESAS にデータを取りに行くからだ。
2 度目からは爆速になる。

サーバー側（Next.js 側）には 2 度目以降、`キャッシュから総人口の情報を返します。` というメッセージが表示される。

都道府県情報を取得する API。

```js:pages/api/prefectures.ts
import { NextApiRequest, NextApiResponse } from 'next'
import { getPrefList } from '../../lib/resas'

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const prefectures = await getPrefList()
  res.status(200).json(prefectures)
}
```

`http://localhost:3000/api/prefectures` のレスポンスはこうなる。

```json
[
{
"prefCode": 1,
"prefName": "北海道"
},
{
"prefCode": 2,
"prefName": "青森県"
},
{
"prefCode": 3,
"prefName": "岩手県"
},
略
```

## ブラウザ側にもキャッシュするには？

API からはキャッシュを返し、ブラウザ側でもキャッシュできるようにすることで、応答をより早くできて、無駄なリクエストも減らせるのではないかと思った。
ブラウザ側では `memory-cache` を使うのが良さそうなので、別の記事で試してみる。

[How to Cache API Calls in Next.js](https://javascript.plainenglish.io/how-to-cache-api-calls-in-next-js-f4b6aefa84f1)
