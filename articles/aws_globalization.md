---
title: "SaaS on AWSのGlobalization対応てみんなどうしてるんやろ"  
emoji: "🌊"  
type: "tech" # tech: 技術記事 / idea: アイデア  
topics: []  
published: false  
---

# 概要

ふとSaaSを海外に展開するときAWS環境どうするやろと思った  
ので。。。実装はしてないけど色んなサイト探検したことをメモっとく

フロントエンドの多言語対応やDBの時刻をUTCで保存するなど  
ふわっと海外イメージしてたけど
インフラレベルまでしっかり考えたことなかったので改めて・・・

# そもそも

こんな感じでとりあえず東京リージョンで作って  
そのまま海外からも使ってもらうと以下2点あたりが突っ込まれそう

1. 地理的に遠いリージョンのLatencyが高くなる問題
2. データをそもそも国外に置かないで問題

↓イメージ的にはこんな感じ  
![cross_region_problem](/images/aws_globalization/cross_region_problem.png)

# AWS のMulti-region構成

`AWS Multi-region`とかで調べると  
DR(Disaster Recovery)の文脈だけど記事がたくさん出てきた  

- https://aws.amazon.com/jp/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iv-multi-site-active-active/
- https://www.slideshare.net/AmazonWebServices/architecture-patterns-for-multiregion-activeactive-applications-arc209r2-aws-reinvent-2018

この辺の記事見てると  
以下パターンが考えられるみたい

1. Read local, write global
   1. 読み込みは近いリージョン・書き込みは特定のリージョン
   1. ![cross_region_problem](/images/aws_globalization/read_local_write_global.png)
1. Read local, write partitioned
   1. 読み込みは近いリージョン・書き込みは初回書き込んだリージョンに書く
   2. それをホームリージョンとして、ユーザーが旅行して別のリージョンいっちゃってもホームに書き込む
   3. こうすれば前パターンのように常に特定リージョンに大量書き込みリクエストが来てパンクするみたいな事象も避けられる
   4. でもこれそれぞれがそれぞれのリージョンに書き込んで、他のリージョンに全部Replicationするの・・？
      1. Globalな情報取るときは各リージョンのDB情報全部取らなきゃいけないってこと・・・？
   5. ![cross_region_problem](/images/aws_globalization/read_local_write_partitioned.png)
1. Read local, write local
   1. 読み込み・書き込み共に近いリージョンでやっちゃう
      1. 具体的にはDynamoDBとかAurora(MySQL)の機能を使ってlocalでの書き込みをGlobalに伝搬させるみたい
   1. ![cross_region_problem](/images/aws_globalization/read_local_write_local.png)

DynamoDB使うなら`Read local, write local`が良さそう  
Aurora(MySQL)のGlobal DB機能使えば同じくいけそうだが  
transactionはったらlatencyとかどうなるの？とか御しきれる自信がない

`Read local, write partitioned`を採用したら各リージョンに書き込まれたデータを  
他の全リージョンにReplicationして制御したりとめちゃ大変そうなんで  
大人しく`Read local, write global`して他リージョンにReplicationしてRead localしてもらうのが良いかな

...というか冷静に考えると  
これでLatency対策は考慮できるが  
自国にデータは置いてほしいというCompliance問題の対策にはならない...

グローバルで共通に使うデータ関連はこの辺に照らし合わせて検討すりゃ良さそう

# SaaSのMulti-region展開

ちょっと方向性変えて`SaaS Multi-region`とかで調べると  
この辺の記事が出てきた  

- https://aws.amazon.com/jp/blogs/apn/architecting-multi-region-saas-solutions-on-aws/
- https://d1.awsstatic.com/events/reinvent/2019/Multi-region_SaaS_Staying_agile_in_a_multi-geography_model_DEM143.pdf
- https://www.europeclouds.com/blog/using-the-cloud-to-build-multi-region-architecture

この辺の画像が全てを表してる気がする  
![cross_region_problem](/images/aws_globalization/distributed_on_boarding.png)

1. Landing pageでユーザーがどの辺の人か選択させて  
2. それ以降は選択リージョンにRoutingする

てな流れが書いてある  
個人的には辿り着きたいページに行くまでに1個ページ挟まるだけで邪魔くさく感じちゃいますが  
とにかく大事なのはユーザーにどのリージョン（どの辺の人）かを選択させて  
洗濯した先でにRouting, データ保存含めてやっちゃう

懸念としては

1. ユーザーがリージョン変えたいときにデータのmigration必要あり  
2. データの分析のためには全リージョンデータ収集用のリージョンが必要
   1. 全データ一つのリージョンに集約して良いのか問題は別でありますが...

# ザクっとこんな感じ？

この二つ調べた結果こんな感じかな・・・

## 基本的なRead/Write

基本最寄り（自身の選択した）リージョンでRead/Writeさせる
これで最初記載した以下懸念の対策になる

1. 地理的なLatency問題
2. 自国（選択したリージョン）にデータを置きたい

![cross_region_read_write](/images/aws_globalization/cross_region_read_write.png)

## Global共通利用するデータのRead/Write

`Read local, write global`パターン  
Dynamo DBのGlobal Tableのように各リージョンに書いたものが  
Globalに展開されるものは最寄りリージョンで書けばいいが  
他は単一リージョンに書いて他リージョンはReplicationさせとくのがシンプルかな・・・  
（リージョン跨いだ書き込み通信によるLatency問題は目をつぶる）
（本当に共通で利用したいならDynamoDB検討）

![cross_region_read_write_main](/images/aws_globalization/cross_region_read_write_main.png)

# まとめ

まずは日本で成功したから調べろや感もあるけど  
ちょっと会話に出てきてあまりちゃんと調べたことなかったので興味本位で調べてみた

色々調べたけど完全なる正解は多分ない  
なるべく複雑にならないように、そもそもの仕様を整理することが何より大事だと思った
RDSで`Read local, write local`みたいな構成も本当に必要かい？的な議論は十分にしたほうが良さそう

いつか海外展開のプロジェクトに携われたら  
もっと調べて手を動かして検証していきたい