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

こんな感じでとりあえず東京リージョンで作る

![cross_region_problem](/images/aws_globalization/cross_region_problem.png)

ただこのままだと以下2点あたりが突っ込まれそう  

1. 地理的に遠いリージョンのLatencyが高くなる問題
2. データをそもそも国外に置かないで問題