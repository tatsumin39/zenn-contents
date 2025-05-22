---
title: "ポーリング方式からPubSubHubbubへ：YouTubeの動画更新を効率的に取得する技術"
emoji: "⚡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["websub", "youtube","typescript"]
published: true
published_at: 2025-05-23 08:00
---
## 概要
以前作成した以下の記事で、YouTubeのライブ配信や動画投稿をDiscordにリアルタイムで通知するシステムを構築しました。
https://zenn.dev/tatsumin/articles/stream-notifications-bot

この仕組みでPostgreSQLにはかなりのデータが保存されてきており、そのデータを利用してさまざまな見せ方ができるのではないかと考えて、Webアプリとして[Streamer Now](https://streamer-now.com)を作成し公開しました。

これまでの仕組みではRSSフィードを定期的にチェックするポーリング方式で行なっていたため、YouTube Feedへのアクセス負荷を考慮する必要がありました。
具体的には1分ごとにポーリングする場合は70チャンネルも登録できないという課題がありました。
そこで、[Google PubSubHubbub Hub](https://pubsubhubbub.appspot.com/)を利用してYouTubeの動画投稿や配信情報をリアルタイムで取得する方法を実装しました。

### Google PubSubHubbub Hubのメリット

RSSフィードをポーリングするよりも以下のようなメリットがあります。

- リアルタイムで通知を受け取れる
- ポーリングするよりもサーバー負荷が少ない
- 削除された動画の情報も取得できる
- スケーラビリティが高く、多数のチャンネルを監視できる

## サンプルコード
以下のレポジトリにサンプルコードを公開しています。

https://github.com/tatsumin39/youtube-pubsubhubbub-example

## PubSubHubbubの仕組み

PubSubHubbub（WebSub）は「Publish-Subscribe」パターンを実装したプロトコルです。以下の3つの役割があります。

1. **Publisher（発行者）**: コンテンツを提供するサービス（YouTubeなど）
2. **Hub**: 購読者に通知を配信するサーバー（Google PubSubHubbub Hub）
3. **Subscriber（購読者）**: 通知を受け取るサービス（あなたのアプリケーション）

この仕組みにより、Subscriberはコンテンツの更新を常にチェックする必要がなく、更新があった時だけHubから通知を受け取ることができます。

## 処理の流れ
サンプルコードでは以下のような処理の流れになります。

1. チャンネルIDを指定してsubscribeToPubSubHub関数を実行
2. subscribeToPubSubHub関数内でPubSubHubbub HubにチャンネルIDを指定してsubscribeを実行
3. webhookエンドポイントでsubscribe時の確認リクエストを処理や動画更新通知を受け取る

### メイン関数

チャンネルIDを指定し、subscribeToPubSubHub関数を実行し、処理結果をログに表示しています。

```ts
import { subscribeToPubSubHub } from './subscribe'

// 使用例
async function main() {
  try {
    // 特定のチャンネルをPubSubHubに登録
    const channelId = 'UCxxxxxxxxxxxxxxxxxxxxxxxx' // 実際のチャンネルIDに置き換え
    console.log(`チャンネルID: ${channelId} をPubSubHubに登録します`)
    
    const response = await subscribeToPubSubHub(channelId)
    
    if (response.status === 202) {
      console.log('✅ 登録リクエストが受理されました')
    } else {
      console.error('❌ 登録に失敗しました:', response.status)
    }
  } catch (error) {
    console.error('エラーが発生しました:', error)
  }
}

main()
```

### PubSubHubBubに登録する関数

PubSubHubbubに登録するための情報を揃えてリクエストを送信します。
`hub.lease_seconds`が有効期限です。後述する考慮ポイントで説明しますが、定期的に再登録を行う必要があります。

```ts
/**
 * YouTubeチャンネルをPubSubHubBubに登録する関数
 * @param channelId YouTubeチャンネルID
 * @returns レスポンス
 */
export async function subscribeToPubSubHub(channelId: string) {
  const hubUrl = 'https://pubsubhubbub.appspot.com/subscribe'
  const topicUrl = `https://www.youtube.com/xml/feeds/videos.xml?channel_id=${channelId}`
  const callbackUrl = `${process.env.YOUR_BASE_URL}/api/webhook/pubsub`

  const params = new URLSearchParams({
    'hub.callback': callbackUrl,
    'hub.lease_seconds': '864000', // 10日間
    'hub.mode': 'subscribe',
    'hub.topic': topicUrl,
    'hub.verify': 'sync'
  })
  
  const response = await fetch(hubUrl, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded'
    },
    body: params.toString()
  })
  
  console.log('Response status:', response.status)
  
  return response
}
```

### Webhookエンドポイント

ここでは、動画更新通知を受け取るためのwebhookエンドポイントを作成します。
ローカル環境で動作確認する場合は、ngrokなどのツールを利用してローカルサーバーのURLを公開しておくと便利です。

動画が公開されたり配信予定が公開されるとPOSTリクエストが送信されてくるので、それを受け取って処理を行います。サンプルコードではログを表示するだけにしています。

```ts
import { NextRequest, NextResponse } from 'next/server'
import xml2js from "xml2js"

/**
 * PubSubHubBubからの確認リクエストを処理するハンドラー
 */
export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url)
  const challenge = searchParams.get('hub.challenge')
  
  if (challenge) {
    console.log('PubSubHubbub 購読確認リクエスト受信')
    return new NextResponse(challenge, {
      status: 200,
      headers: { 'Content-Type': 'text/plain' },
    })
  }
  
  return new NextResponse('Bad Request', { status: 400 })
}

/**
 * YouTubeからの動画更新通知を処理するハンドラー
 */
export async function POST(request: NextRequest) {
  // リクエストボディを取得
  const body = await request.text()
  
  // XMLをパース
  const parsedPubSub = await xml2js.parseStringPromise(body)

  // フィードにentryが含まれていない場合（動画削除など）
  if (!parsedPubSub?.feed?.entry) {
    console.log('entry が存在しない更新通知を受信')
    return new NextResponse('OK', { status: 200 })
  }

  // 動画情報を取得
  const entry = parsedPubSub.feed.entry[0]
  
  const videoData = {
    videoId: entry["yt:videoId"][0],
    title: entry.title[0],
    published: entry.published[0],
    channelId: entry["yt:channelId"][0]
  }

  console.log('新しい動画が公開されました:', videoData)
  
  // ここで動画情報を保存したり、通知を送ったりする処理を実装
  
  return new NextResponse('OK', { status: 200 })
}
```

動画の公開や更新時に送信されるリクエストについては以下のドキュメントを参考にしてください。

https://developers.google.com/youtube/v3/guides/push_notifications?hl=ja

また、ドキュメントには記載されていませんが動画データが削除された場合は、以下のような通知が送信されます。

```xml
<?xml version='1.0' encoding='UTF-8'?>
<feed xmlns:at="http://purl.org/atompub/tombstones/1.0" xmlns="http://www.w3.org/2005/Atom">
  <at:deleted-entry ref="yt:video:DummyId" when="2025-05-15T09:30:45.123456+00:00">
    <link href="https://www.youtube.com/watch?v=DummyId"/>
    <at:by>
      <name>サンプルチャンネル</name>
      <uri>https://www.youtube.com/channel/DummyChannelID</uri>
    </at:by>
  </at:deleted-entry>
</feed>
```

## 実装時の考慮ポイント

###  PubSubHubbubの有効期限と再登録
Google PubSubHubbub Hubは登録時に有効期限が設定されるため、定期的に再登録をする必要があります。
チャンネル管理用のテーブルを作成しておき、チャンネルIDと有効期限を保存し、有効期限が切れる前に更新するようなロジックを実装するとよいでしょう。

```sql
CREATE TABLE channel_subscriptions (
  channel_id TEXT PRIMARY KEY,
  subscription_expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

### 詳細な動画情報の取得とデータベースへの保存
PubSubHubbubから送信されるデータから取得できる項目には配信予定時刻などがないため、videoIdからYouTube Data APIの[Videos: list](https://developers.google.com/youtube/v3/docs/videos/list?hl=ja)を利用して詳細情報を取得しデータベースに保存しておくと良いでしょう。

一度送信された動画情報でPubSubHubbubから再度送信されることがあります。そのため、データベースに保存済みの場合はYouTube Data APIの実行を省略し、更新を行うロジックは別とするほうが効率的です。

### 動画情報の更新を取得する際の最適化
作成したWebアプリでは配信前から配信中などのような状態変化を検知するために、以下の戦略を採用しています：

1. 配信1時間前からYouTube Data APIを定期的に実行
2. 配信中の動画についても視聴者数や配信終了を検知するためにリクエストを実施
3. YouTube Data APIの[Videos: list](https://developers.google.com/youtube/v3/docs/videos/list?hl=ja)を使用（一度に50件まで動画IDを指定可能）

この最適化により、早めに配信が始まった場合や予定時間が後ろ倒しになった場合にも対応できます。また、常に配信中の動画と配信予定の動画が合わせて50件以内であれば、API消費量を抑えることができます。

## エラーハンドリングとロギング

実運用では以下のようなエラーハンドリングとロギングを実装することをお勧めします：

1. PubSubHubbub Hubへのリクエスト失敗時のリトライ機構
2. Webhook受信時のエラーハンドリング
3. 通知処理の非同期実行によるWebhookレスポンスの高速化
4. 重要なイベントのロギングと監視

## まとめ
Google PubSubHubbub Hubを利用することで、RSSフィードをポーリングする際に感じていた課題の多くが解決でき、現在では300以上のチャンネルの更新をリアルタイムで受け取ることができています。

また、RSSフィードをポーリングする際には更新頻度が低いチャンネルについてはポーリング間隔を長くするなどの調整が必要でしたが、PubSubHubbubを利用することでポーリング間隔にとらわれずに更新を受け取れるようになり、チャンネル管理も容易になりました。

今後YouTubeのAPIクォータを考慮したシステム設計をする際には、ぜひPubSubHubbubの活用をご検討ください。

## 参考文献
- [YouTube API - 通知について](https://developers.google.com/youtube/v3/guides/push_notifications?hl=ja)
- [WebSub（旧PubSubHubbub）の仕様](https://www.w3.org/TR/websub/)
- https://zenn.dev/meihei/articles/01cd06f729056a