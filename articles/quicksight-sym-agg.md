---
title: "LAC-A関数を用いてQuickSightで対象集計のようなものを行う"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["QuickSight"]
published: false
---

# 目標

Googleの対称集計にあるサンプルと同じような集計をQuickSightで行う。

# 背景

1対多のテーブルのレコードを重複なく集計するのは難しい。Lookerでは対称集計というものがあるようだがQuickSightでは（ググっている限り）対称集計がサポートされていない。とはいえ[LAC-A関数を使えば対称集計のように使えそう](https://aws.amazon.com/jp/blogs/news/create-advanced-insights-using-level-aware-calculations-in-amazon-quicksight/)なので集計を行う。

# 使用するもの

Googleの[対称集計](https://cloud.google.com/looker/docs/best-practices/understanding-symmetric-aggregates?hl=ja)にある[orders](/images/quicksight-sym-agg/orders.csv)と[order_items](/images/quicksight-sym-agg/order_items.csv)のデータ。

# データセットの準備

1. ダウンロードしておいたcsvファイルをorders, order_itemsとしてインポートする。
![make-dataset1](/images/quicksight-sym-agg/make-dataset1.png)

1. ordersとorder_itemsを `order_id` でjoinしてsample_ordersとして新しいデータセットを作成する。
![make-dataset2](/images/quicksight-sym-agg/make-dataset2.png)

このままだとtotal列にレコードの重複があり、そのまま和などをとってはいけないことがわかる。
![make-dataset3](/images/quicksight-sym-agg/make-dataset3.png)

# レコードを重複しないように集計する

total列にはレコードの重複があるため例えば `sum(total)` などの集計は意味をもたない。今回の例は `quantity * unit_price` を新しい計算フィールドとして用意すれば回避できるが、勉強のために、あくまでtotal列を使って集計する。

# LAC-A関数

ビジュアルに用いられているfieldに加えて任意のfieldを加えてGroupByな集計を行うことができる関数群。複数レコードが存在しても、その中で `max` などをとることで対称集計のようにふるまうことができる。[blog記事](https://aws.amazon.com/jp/blogs/news/create-advanced-insights-using-level-aware-calculations-in-amazon-quicksight/)をみてもLAC-Aは、より抽象的な概念で対称集計のためにある関数というわけではない。
正直blogをみても挙動のイメージがつかないので手を動かしながら確認していく。

## 合計の売上金額の集計

そのまま `sum(total)` を計算しても重複するレコードがあるため上手くいかない。各 `order_id` において `total` がひとつだけ計算されるようにすればよいので計算フィールド `total_sales`というカラムをで `sum(max(total, [{order_id}]))` と作成する。そうすれば各 `order_id` において集約されるので `50.36 + 24.12 + 50.36 = 124.84` と正しい値になる。
![total-sales1](/images/quicksight-sym-agg/total-sales1.png)
もちろん、各 `order_id` において集約することが重要なので `max` でなくても `sum(min(total, [{order_id}]))` で計算フィールドを設定しても正しい値になる。

## `order_date` ごとの売り上げ集計

先ほど使った計算フィールド `sum(max(total, [{order_id}]))` を使って `order_date` でのピボット集計を行う。きちんと `order_id` ごとに正しく売上が計算されていることがわかります。
![total-sales2](/images/quicksight-sym-agg/total-sales2.png)

## `order_id` ごとの集計

先ほど使った計算フィールド `sum(max(total, [{order_id}]))` を使って `order_id` でのピボット集計を行う。ピボットと計算フィールドで`order_id`が重複していますが、きちんと `order_id` ごとに正しく売上が計算されていることがわかります。
![total-sales3](/images/quicksight-sym-agg/total-sales3.png)

これまで合計金額を計算してきたが平均なども同様に集計できる。計算フィールドは`avg(min(total, [{order_id}]))`として設定すれば良い

## 平均売上金額(注文単位)
![avg-sales1](/images/quicksight-sym-agg/avg-sales1.png)

## `order_date` ごとの平均売上金額(注文単位)
![avg-sales2](/images/quicksight-sym-agg/avg-sales2.png)

# 注意

これらの計算が上手くいくのは `order_id` と `total` が1対1に対応しているためです。1対1対応なので `max` や `min` で集約することができます。LAC-A関数は対称集計を行うために用意された関数というわけではないので1対1対応であることを要請しません。そのため1対1対応でなくても全く問題なく値を返すことには注意が必要です。

# まとめ

対称集計を使わなくて良いならば使わないのがベストかと思います。今回の例でいえば `quantity * unit_price` で新しいカラムを作るのがベストだと思います。使用に注意は必要ですが便利な機能なので使っていきたいところです。
