---
title: "DynamoDBのテーブル設計どうやってるのか雰囲気掴みたい"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア topics: []
published: false
---

# 概要

これまでほぼRDBMSしか触ってなかったので  
転職の合間で少し時間あるこの時期にDynamoDBの公式ドキュメント触って  
スケーラブルでマネジドなNoSQLちょっと語れる男になりたい  
https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Introduction.html

# 大事なこと

- RDBMSはそこまで使い方見えてなくてもテーブル設計できる
    - DynamoDBではどのキーで検索してどういう情報取得するか見えてないと設計できない
        - 検索の柔軟性がないので事前に使われ方がわかってないといけないのだ
- テーブルも正規化してたくさん作るんじゃなくてなるべく少なく。もはや1個がベスプラの世界線
    - 各itemはPK(Partition Key)とSK(Sort Key)またはPKと呼ばれるPrimaryな項目が必須
    - このPrimaryな項目はテーブル内で一意である必要がある
- インデックスっていう概念がテーブル設計において重要になってくる
    - この辺は後編で扱う

# テーブル設計してみる

これ読んだ自分の理解をメモっとくだけ    
https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/bp-relational-modeling.html

「お客様が従業員に注文する」みたいなよくあるケース

## アクセスパターンの洗い出し

例えばこんな感じのデータ取得がしたいとする

1. 従業員IDを元に従業員詳細を取得
2. 従業員名を元に従業員詳細を取得
3. お客様の注文一覧をお客様IDと日付から取得
3. お客様の注文一覧を日付から取得

## エンティティを定義

アクセスパターンを踏まえて  
大元テーブルを作成するためにエンティティを定義

PK(Partition Key)にはデータが分散するような項目を指定  
とりあえずは従業員やお客様のID指定が無難かなぁ     
SK(Sort Key)にはPKに加えて検索したい条件&ソートが上手く効く項目を指定するっぽい  
（この辺は何個かケースこなさないと上手く設計できなさそう・・・）

1. Employee - PK: EmployeeID, SK: EmployeeName
3. Customer - PK: CustomerID, SK: AccountRepID
4. Order - PK OrderID, SK: CustomerID

## 大元になるテーブル作成

エンティティで定義したPK, SKを元にitem作成して  
各項目で必要なデータもAttributesとして考えてく

| Partition Key             | Sort Key                       | Attributes                      |                      |                           |
|---------------------------|--------------------------------|---------------------------------|----------------------|---------------------------|
| (EmployeeID)<br>EMPLOYEE1 | (EmployeeName)<br>John Smith   | (Email)<br>xxx@example.com      | (Country)<br>Japan   |                           |
|                           | (SalesPeriod)<br>Quota-2017-Q1 | (OrderTotals)<br>50000          |                      |                           |
| (CustomerID)<br>CUSTOMER1 | (CustomerName)<br>Taro Tanaka  | (Country)<br>Japan              |                      |                           |
| (OrderID)<br>ORDER1       | (CustomerID)<br>CUSTOMER1      | (StatusDate)<br>OPEN#2018_08_11 | (OrderTotal)<br>1200 | (SalesRepID)<br>EMPLOYEE1 |

RDBMSとは正反対な設計の流れ  
スキーマを定義して正規化してくんじゃなくて  
一つのテーブルでよろしくデータ突っ込めるように頑張ってく感じ

### 集約

2行目にあるように特定期間の合計値を格納するのは`集約`と呼ばれるテクニックらしい  
4行目にあるようにOrderが入ってItemが追加されたことを契機に  
Amazon DynamoDB Stream発動してLambdaが計算して2行目の合計値を更新する  
こういうテクニック系は知らないとわからんわな・・

## アクセスパターンを実現するためにインデックス作成

### Global Secondary Index

作成した元テーブルのPKでは検索できないような条件がある場合  
Global Secondary Indexを指定して検索できるようになるのだ イメージとしては別のテーブルが作成されるような感じっぽい

例えば従業員の名前から従業員詳細取りたいとか  
お客様の特定期間における購入情報を知りたいとか

| Partition Key                | Sort Key                        | Attributes                 |
|------------------------------|---------------------------------|----------------------------|
| (EmployeeName)<br>John Smith | (StatusDate)<br>OPEN#2018_08_11 | (Email)<br>xxx@example.com |
| (CustomerID)<br>CUSTOMER1    | (StatusDate)<br>OPEN#2018_08_11 | (OrderTotal)<br> 1200      |

ちなみに作成できるGSIには上限があるので  
ひとつのGSIで複数の検索条件を実現できるようにするのを  
`GSI Over Loading`というらしい

### GSIシャーディング

ある特定期間の全体売り上げみたい  
とかいう時に使えるテクニックらしい

| Partition Key           | Sort Key                        | Attributes                |                        |
|-------------------------|---------------------------------|---------------------------|------------------------|
| (Coefficient)<br>[0..N] | (StatusDate)<br>OPEN#2018_08_11 | (CustomerID)<br>CUSTOMER1 | (OrderTotals)<br>50000 |
| (Coefficient)<br>[0..N] | (StatusDate)<br>OPEN#2018_08_11 | (ProductID)<br>PRODUCT1   | (OrderQuantity)<br>50  |

こうするとよしなにpartition分散したデータ格納になって  
ほっとパーティションを生み出さず効率よくデータが取れる！

ちなみにNの算出は計算式があってそれに準ずるが良いらしい

# その他特質すべきパターン

## 時系列データ

基本テーブル数を少なく、1つにするぐらいがいいらしいけど  
時系列データに関してはテーブル分割するとうまいことできる

ちょっと描くのめんどくさくなってきたのでリンクだけ貼りますが  
https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/bp-time-series.html

古いテーブルを別出しすることで  
トラフィックボリュームの多い最近のデータに必要なリソースを割り当て  
古いデータにはコストをかけないというテクニックがあるらしい

# まとめ

パーティションキー単位でストレージに保存されて  
ソートキーでソートされた順で近くに保存されてる

上記特性を鑑みて  
各パーティションに分散してデータ取得のリクエストが行くようにするのが素敵らしい  
下手な設計すると毎度同じパーティションにリクエストが行くホットパーティションが発生して  
レイテンシーが高く高コストな感じになっちゃうみたい

今回ドキュメント読んだだけなので  
実際に作ってみてどれくらいのレイテンシーやコストになるのか検証したい