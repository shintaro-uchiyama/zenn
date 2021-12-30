---
title: "Microserviceでの認証・認可ちょっと考えてみた"
emoji: "🌟"
type: "tech"
topics: ["microservice", "認証", "認可"]
published: false
---
# 概要

MSWでモックした状態だけど
なんとなく動く画面ができてきたので
認証・認可あたりをゴリッと実装していきたい

けど、ガチャガチャ書く前にちょっと頭を整理してみる

基本はこの素敵記事で勉強させていただきました🙇‍♂️  
https://please-sleep.cou929.nu/microservices-auth-design.html

# 認証

セキュリティは奥が深いので基本認証は他の素敵なサービス利用して実施したい
Firebase Auth, Cognito, Auth0とか色々あるけど
とりあえず昔ローカルのEmulator動かすとこまでやったのでFirebase Auth使う

## 処理の流れ

ブラウザでGoogleアカウントとかGitHubアカウントでログインしてもらう

そんで取得したトークンからEmail取得してユーザー情報取得
アクセストークンとリフレッシュトークンをユーザーに付与する

アクセストークンは短く1時間、リフレッシュトークンは1週間くらい有効にしとく

ザクっと処理の流れはこんな感じ

![token](/images/microservice_auth_design/access_refresh_token.png)

## 鍵のローテーション

トークンの暗号化、復号に使用する鍵セットをローテーション
退職者の人がこっそり秘密鍵パクって、なんでもできるなんで事態を避けたい

とはいえ、新しい鍵セットを作成してから
リフレッシュトークンが有効な期間（1週間）は古い鍵セットは消せない
すぐ古いの消しちゃうと、ちょっと前に認証成功した人のトークン復号出来なくて未ログインになっちゃう

![token](/images/microservice_auth_design/key_rotation.png)

# 認可

認可はRBAC(Role-based access control)で行う    
認証時に発行したJWT形式のAccessTokenにPermissionを付与して  
各マイクロサービスでそのPermissionの存在判定を行う

## 処理の流れ

各マクロサービス内でPermissionの存在判定は  
文字列のベタ書きじゃなくてGolangとかのmodにして外部から適切に定数で取得できるようにしたい...  

![authorization](/images/microservice_auth_design/authorization.png)

# まとめ

頭が少し整理できたので、がつっと実装していく💪🏻