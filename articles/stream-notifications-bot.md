---
title: "YouTube通知Botを作る！Node.jsとPostgreSQLでほぼリアルタイム通知"
emoji: "📥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nodejs", "PostgreSQL", "youtubeapi", "discord", "flyio"]
published: true
---
## はじめに
このプロジェクトは、[以前作成したGAS](https://zenn.dev/tatsumin/articles/youtube-to-discord-notifier)での実装をnode.jsとPostgreSQLに移植し、機能追加を行いました。


## 概要
Discord Botを使用してYouTubeのライブ配信や動画の通知を行うものです。指定されたチャンネルのRSSフィードを監視し、新しい動画が投稿されたり、ライブ配信が開始されたりした際に、Discordチャンネルに通知を送信します。
また、スラッシュコマンドによりライブ配信中の動画情報を取得したり、絵文字リアクションでリマインダー登録をすることができます。


## どのようなことが実現できるか？
### 通知
以下のように配信状況や動画投稿が通知されます。
![](/images/stream-notifications-bot/image01.png)


### リマインダー登録
ライブ配信予定の投稿に対して絵文字 :remind: でリアクションをすることでライブ配信予定の5分前にDiscord BotからDMにて通知が届きます。
なお、ライブ配信予定が変更になった場合は新しい配信予定時刻に基づきリマインダー設定が更新されます。
リマインダー登録のイメージ
![](/images/stream-notifications-bot/image02.png)

BotからDMによるリマインダー通知のイメージ
![](/images/stream-notifications-bot/image03.png)

### liveコマンド
`/live` コマンドを実行すると、現在ライブ配信中の情報が表示されます。
![](/images/stream-notifications-bot/image04.png)

### upcomingコマンド
`/upcoming` コマンドを実行すると、現在時刻から15分以内に開始予定のライブ配信情報が表示されます。
以下は直近15分以内に開始予定のライブ配信情報がなかった場合の表示内容です。
![](/images/stream-notifications-bot/image05.png)

`/upcoming 60` のようにオプションとして任意の分数を指定することが可能です。
以下は60分以内に開始予定のライブ配信情報が1件あった場合の表示内容です。
![](/images/stream-notifications-bot/image06.png)

### reminderlistコマンド
`/reminderlist` コマンドを実行すると、登録した有効なリマインダーが表示されます。
もちろんリマインダーを登録したユーザー自身の内容だけが表示されます。
![](/images/stream-notifications-bot/image07.png)

### 管理者向け機能
BotとのDMで簡易なSQLであれば実行できます。
主にライブ配信予定やライブ配信中ステータスの動画が非公開となった場合に`video_data`テーブルの`live`カラムが正しく更新されない場合があります。
そのような場合にこの機能を利用し、対象データの特定と更新もしくは削除を行うことができます。
また、新しいYouTubeチャンネルを登録したい場合などもこの機能を利用します。
![](/images/stream-notifications-bot/image08.png)


## コードはこちら
セットアップ方法などもこちらに記載しています。
https://github.com/tatsumin39/stream-notifications-bot


## 苦労したポイント
### YouTube Feedへのアクセス負荷
GASと比べて実行間隔を細く設定できるため1分間隔としましたがFeed取得対象とするYouTubeチャンネルが増えたことでFeed取得エラーが発生することが多くなりました。
明確な制限値は公開されていませんが、同一のシステムもしくはIPからおおよそ1日あたり10万回アクセスがあるとエラーが発生する可能性が高いようでした。

そのため、Feed取得をライブ配信中心のチャンネルは1分間隔、動画配信中心のチャンネルは10分間隔と処理を分ける実装とすることで調整を行いました。

### リマインダー管理
リマインダーの実装にはnode-scheduleライブラリを利用しています。リマインダーJobが設定された後にアプリケーションを再起動した場合にJobが削除されてしまいます。
そのため実際に数時間後、数日後のリマインダーであってもすぐにはJobを登路せずに一度`reminder`テーブルに必要な情報を格納しています。
Job登録はリマインダー発動の直前としています。また、Job登録後に再起動があっても対応できるようにアプリケーション起動時は登録済みのJobがある場合は再登録する処理を実装しています。


## fly.ioについて
node.jsで実装する方針とした後にデプロイ先としていくつか検討しましたが、無料で利用できることや比較的わかりやすい点から[fly.io](https://fly.io/)を採用しました。それに伴いデータベースはPostgreSQLで構成しています。


## 今後のアップデート構想
ユーザー向けの機能としてはTwitch対応を検討しています。YouTubeとは考え方が変わる部分もあるため対応は当分先になりそうです。

管理者向けには現在の簡易的なメンテナンス機能をBotとのDMで提供していますが、よりよい体験のためにはWebアプリとして開発をしたほうがいいのではないかと考えています。