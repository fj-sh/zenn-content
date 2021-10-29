---
title: "Rubyのハッシュとシンボル"
emoji: "📌"
type: "tech"
topics: ["ruby"]
published: true
---

ハッシュはキーと値の組み合わせでデータを管理するオブジェクトです。
ハッシュを作成する場合は、ハッシュリテラルという構文を使います。

```ruby
{ 'japan' => 'yen', 'us' => 'doller', 'india' => 'rupee'}
```

## ハッシュから値を取り出す

```ruby
currencies = { 'japan' => 'yen', 'us' => 'doller', 'india' => 'rupee'}
p currencies['japan'] # yen
```

## ハッシュにキーを追加する

```ruby
currencies['dorakue'] = 'gold'
p currencies # {"japan"=>"yen", "us"=>"doller", "india"=>"rupee", "dorakue"=>"gold"}
```

## ハッシュを使った繰り返し処理

```ruby
currencies.each do |key, value|
  puts "#{key}: #{value}"
end
```

## ハッシュの要素の削除

```ruby
currencies.delete('dorakue')
```

## シンボル

シンボルとは文字列にコロン（`:`）を前に置いて、任意の名前を定義するものです。

```ruby
:シンボルの名前
```
の形でシンボルを使います。

```ruby
:apple.class  # Symbol
```

シンボルはRubyの内部で整数として管理されます。
2つの値が同じかどうか調べる場合、文字列よりも高速に処理できます。

シンボルは「同じシンボルであれば同じオブジェクトである」という特徴があるため、同じシンボルのobject_idは同一になります。

```ruby
p :banana.object_id # 1020508
p :banana.object_id # 1020508
p :banana.object_id # 1020508

p 'banana'.object_id # 60
p 'banana'.object_id # 80
p 'banana'.object_id # 100
```

シンボルはイミュータブルなので破壊的な変更は不可能です。


## ハッシュのキーにシンボルを使う

Rubyでは、ハッシュのキーには文字列ではなくシンボルが好まれます。

```ruby
currenceis = { :japan => 'yen', :us => 'doller', :india => 'rupee'}
```

上記のハッシュは以下のように書き換えることができます。

```ruby
currencies = { japan: 'yen', us: 'doller', india: 'rupee' }
# {:japan=>"yen", :us=>"doller", :india=>"rupee"}
```

キーも値もシンボルの場合は以下のように書きます。

```ruby
currencies = { japan: :yen, us: :doller, india: :rupee }
# {:japan=>:yen, :us=>:doller, :india=>:rupee}
```