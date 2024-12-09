---
title: "YouTubeのShort動画を簡単に判定する方法"
emoji: "🚦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["node", "youtube"]
published: true
published_at: 2024-12-10 08:00
---
## 概要
以前作成した以下の記事で、YouTubeのライブ配信や動画投稿をDiscordにリアルタイムで通知するシステムを構築しました。
[YouTube通知Botを作る！Node.jsとPostgreSQLでほぼリアルタイム通知](https://zenn.dev/tatsumin/articles/stream-notifications-bot)
しかし、登録しているチャンネルでショート動画の投稿が増えてきたため、通知や管理のためにショート動画を判定するロジックを追加することにしました。

## YouTubeのショート動画の特徴
YouTubeのショート動画は以下の特徴を持っています。
- 60秒以内
- 正方形または縦長である必要があります。
- 動画タイトルもしく説明に `#Shorts` を含める(デスクトップOSからの投稿の場合)

例えば以下の動画がショート動画に該当します。

https://www.youtube.com/shorts/VJJPjVj6pNw


## YouTubeショート動画を判定する方法
YouTube Data APIではショート動画を直接判定する方法が提供されていないため、別のアプローチを検討しました。
上記の条件をコードに落とし込むことも可能ですが、より簡単で確実な方法として、リダイレクト判定を活用しました。

### 判定ロジック
- ショート動画用のURL `https://youtube.com/shorts/<videoId>` にアクセスします。
- リダイレクトされる先が `https://youtube.com/watch?v=<videoId>` であれば通常動画。
- リダイレクトされずにそのまま表示される場合はショート動画と判定します。


以下は、Node.jsでこの判定を行う関数の例です.

### ショート動画判定用のNode.js関数

```js
import fetch from 'node-fetch';

/**
 * YouTube動画がShort動画か通常動画かを判定する
 * @param {string} videoId - 判定対象の動画ID
 * @returns {Promise<boolean>} - Short動画の場合はtrue、それ以外はfalse
 */
export async function isShort(videoId) {
  const url = `https://youtube.com/shorts/${videoId}`;
  try {
    // リダイレクトを追跡する設定
    const response = await fetch(url, { method: 'HEAD', redirect: 'follow' });
    // レスポンスURLが/shorts/を含む場合はShort動画
    if (response.url.includes('/shorts/')) {
      return true; // Short動画
    }
    // それ以外の場合は通常動画
    return false;
  } catch (error) {
    console.error('⛔️ isShort関数内でエラーが発生:', error.message);
    return false; // 判定できない場合は通常動画として扱う
  }
}
```

### 使用例

これを以下のように呼び出すことで戻り値によってショート動画か通常動画かを扱えるようになりました。

```js
// 使用例: 判定結果を取得して用途に応じた処理を行う
const type = await isShort(videoId) ? 'short' : 'video';
if (type === 'short') {
  console.log('ショート動画です。');
} else {
  console.log('通常動画です。');
}
```

## まとめ
この関数を活用することで、ショート動画を区別して以下のような用途に対応可能です。

- Discord通知メッセージの内容をショート動画か通常動画かで切り替える
- データベースに保存する際、動画のタイプをステータスとして管理
- WebアプリのUIでショート動画と通常動画を異なる形式で表示

リダイレクトを利用したシンプルな判定方法を取り入れることで、効率的にショート動画を扱えるようになりました。ぜひ参考にしてみてください！

