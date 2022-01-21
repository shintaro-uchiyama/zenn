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

# Latency対策

この辺の記事読んでると  

- https://aws.amazon.com/jp/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iv-multi-site-active-active/
- https://www.slideshare.net/AmazonWebServices/architecture-patterns-for-multiregion-activeactive-applications-arc209r2-aws-reinvent-2018

以下パターンが考えられそう

1. Read local, write local
   1. 読み込み・書き込み共に近いリージョンでやっちゃう
   1. ![cross_region_problem](/images/aws_globalization/read_local_write_local.png)
1. Read local, write global
   1. 読み込みは近いリージョン・書き込みは特定のリージョン
   1. 上の例で言えば書き込みは全て東京リージョンに来る的な
   1. ![cross_region_problem](/images/aws_globalization/read_local_write_global.png)
1. Read local, write partitioned
   1. 読み込みは近いリージョン・書き込みは初回書き込みリージョン
   1. それをホームリージョンとして、旅行して位置ずれた時もホームに書き込む
   1. こうすれば常に特定のリージョンに大量書き込みリクエストが来てパンクするみたいな事象も避けられる
   1. ![cross_region_problem](/images/aws_globalization/read_local_write_partitioned.png)
