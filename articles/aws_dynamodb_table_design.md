---
title: "DynamoDBのテーブル設計どうやってるのか雰囲気掴みたい"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア  
topics: ["dynamodb", "aws", "gsi"]
published: true
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
- テーブルも正規化してたくさん作るんじゃなくてなるべく少なく
    - もはや1個がベスプラの世界線
- インデックスっていう概念を利用して検索性を高くできる
    - ただインデックスはなるべく少なくしないと高コストになる

# テーブル設計してみる

これ読んだ自分の理解をメモっとくだけ    
https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/bp-relational-modeling.html

「お客様が従業員に注文する」みたいなよくあるケース

## アクセスパターンの洗い出し

例えばこんな感じのデータ取得がしたいとする

1. 従業員IDを元に従業員詳細を取得
2. 従業員名を元に従業員詳細を取得
3. 特定のお客様の注文一覧をお客様IDと日付から取得
3. お客様全体の注文情報を日付から取得

## エンティティを定義

アクセスパターンを踏まえて  
大元テーブルを作成するためにエンティティを定義

1. Employee - PK: EmployeeID, SK: EmployeeName
3. Customer - PK: CustomerID, SK: AccountRepID
4. Order - PK OrderID, SK: CustomerID

ここで定義したPK(Partition Key)の単位でデータが保存されるらしい  
このPKを均等に分散してデータアクセスするとコスト効率とレイテンシーが低いみたい  
同じPKの箇所に大量にデータアクセスがくるとコストも高くレイテンシーも高くなるみたい  
（とりあえずは従業員やお客様のID指定が無難かなぁ）

SK(Sort Key)にはPKに加えて検索したい条件を指定する  
この項目で前方一致とか範囲指定もできる

## 大元になるテーブル作成

エンティティで定義したPK, SKを元にitem（レコード）作成して  
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
Global Secondary Indexを指定して検索できるようになるのだ

イメージとしては別のテーブルが作成されるような感じっぽい

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

ちなみにNの算出は以下のような計算式があって  
それに準じて算出するのが良いらしい

```
ItemsPerRCU = 4KB / AvgItemSize
PartitionMaxReadRate = 3K * ItemsPerRCU
N = MaxRequiredIO / PartitionMaxReadRate
```

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

まだまだ奥は深そうだけど  
なんとなくドキュメント読んで雰囲気を掴んだ

結構癖が強めだが  
なんといっても使用しないとお金かからない  
サーバーレス構成の利点はでかいので  
今Privateで作ってるシステムのストレージとして検討してみる！！