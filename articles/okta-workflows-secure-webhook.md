---
title: "SmartHRとkickflowの事例で学ぶ、Okta WorkflowsのWebhookセキュリティ"
emoji: "🪄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["okta", "oktaworkflows", "HMAC", "security"]
published: true
published_at: 2024-07-17 08:00
publication_name: joug
---
こんにちは！たつみんです🖐️

今日は、Okta Workflowsを使ったWebhookのセキュリティについてご紹介します。

Okta WorkflowsとSaaS連携をする際にWebhookを利用することが多いと思います。この時、Okta WorkflowsではWebhookを受け取るためにAPI Endpointカードを利用しますが、発行されたInvoke URL宛のリクエストをすべて受け入れてしまいます。
これには、悪意のある攻撃者が正当なリクエストを模倣した送信を行えてしまうというセキュリティリスクがあります。これを放置すると、システムの安全性が大きく損なわれる可能性があります。

このセキュリティリスクに対して、連携するSaaS側がシークレットトークンもしくはHMACに対応している場合、セキュリティを強化することが可能です。この記事では、シークレットトークンに対応するSmartHRと、HMACに対応するkickflowを例に、Okta Workflows側でどのようにInvoke URL宛のリクエストを検証できるかをご紹介します。

## この記事で伝えたいこと
- シークレットトークン検証は対応が容易だが、セキュリティとしては少し不安が残るよ
- HMAC検証は対応が複雑だが、セキュリティとしては非常に安心できるよ
- 適切な検証方法を選択することで、Webhookのセキュリティを大幅に向上させることができるよ

## Okta WorkflowsのAPI Endpointカードについて
あらかじめOkta WorkflowsでAPI Endpointカードを設定し、Invoke URLを取得しておきます。
この時に以下のようにAliasが空欄となっている場合はflowを保存してください。
![](/images/okta-workflows-secure-webhook/image01.png)
再度API endpoint settingsを確認するとAliasが反映され有効なInvoke URLが取得できます。

## シークレットトークンによる検証方法
ここではSmartHRを例に説明をします。
関連ドキュメントは以下です。
- [Webhook連携を設定する](https://support.smarthr.jp/ja/help/articles/5260914962073/)
- [SmartHR Webhook の仕様について](https://developer.smarthr.jp/api/about_webhook)

### SmartHR側の設定
以下のように設定します。
![](/images/okta-workflows-secure-webhook/image02.png)
1. URLにOkta Workflowsで設定したAPI EndpointカードのInvoke URLを入力します。
2. Webhook の用途・説明を設定します。
3. シークレットトークンを自動生成ボタンをクリックし生成します。
4. 送信のトリガーとなるイベント、送信する内容、通知の無効化を必要に応じて設定します。

### Okta Workflows側の設定
以下が全体図です。とてもシンプルです。
![](/images/okta-workflows-secure-webhook/image03.png)
1. API Endpointカードのheadersに `x-smarthr-token` を設定します。
2. Continue ifカードで、1. で取り出した `x-smarthr-token` とSmartHR側の設定で生成されたシークレットトークンを比較します。

### 実行結果の例
実行結果の可視化のために上記のOkta Workflowsの設定のContinue ifカードの後にComposeカードを設定し実際の挙動を確認してみます。

#### 成功例
ヘッダー経由で取得したx-smarthr-tokenと、入力した値が一致することで後続の処理であるComposeカードが実行されています。
![](/images/okta-workflows-secure-webhook/image04.png)

#### 失敗例
以下ではContinue ifでx-smarthr-tokenとは違う値を入力しているため後続の処理であるComposeカードは実行されていません。
![](/images/okta-workflows-secure-webhook/image05.png)

## HMACによる検証方法
ここではkickflowを例に説明します。
関連ドキュメントは以下です。
- [Webhook APIを設定する](https://support.kickflow.com/hc/ja/articles/360048308493-Webhook-API%E3%82%92%E8%A8%AD%E5%AE%9A%E3%81%99%E3%82%8B)
- [Webhook API](https://developer.kickflow.com/webhook/)
- [Webhook APIについてのFAQ](https://developer.kickflow.com/faq/#webhook-api%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6%E3%81%AEfaq)

### kickflow側の設定
以下のように設定します。
![](/images/okta-workflows-secure-webhook/image06.png)
1. 名前を設定します。
2. リクエスト先URLにOkta Workflowsで設定したAPI EndpointカードのInvoke URLを入力します。
3. シークレットに十分な長さの英数字のランダムな文字列を作成し入力します。
4. 送信対象、送信するイベントを必要に応じて設定します。

### Okta Workflows側の設定
以下が全体図です。エスケープ文字の対応のために必要とするカードが多くあります。
![](/images/okta-workflows-secure-webhook/image07.png)
1. API Endpointカードのheadersに `x-kickflow-signature` を設定します。
2. Replaceカードで、bodyから `&` を `\u0026` に変換します。
3. Replaceカードで、2. で変換したbodyから `<` を `\u003c` に変換します。
4. Replaceカードで、3. で変換したbodyから `>` を `\u003e` に変換します。
5. HMACカードで、algorithmを `sha256` 、keyを kickflow側の設定で設定した値、dataを4. で変換した値、digestを `hex` を指定します。
6. Concatenateカードで、`sha256=` と5. の結果を結合します。
7. Continue ifカードで、1. で取り出した `x-kickflow-signature` と6. で生成された値を指定し比較します。

### 実行結果の例
実行結果の可視化のために上記のOkta Workflowsの設定のContinue ifカードの後にComposeカードを設定し実際の挙動を確認してみます。

#### 成功例
ヘッダーの `x-kickflow-signature` とHMACで計算され値が一致することで後続の処理であるComposeカードが実行されています。
![](/images/okta-workflows-secure-webhook/image08.png)

#### 失敗例
以下ではHMACカードでkeyをkickflow側の設定3. で設定した値とは違う値を入力しているため後続の処理であるComposeカードは実行されていません。
![](/images/okta-workflows-secure-webhook/image09.png)

## Okta Workflowsのサンプル
以下に今回作成したFlowを格納しています。ダウンロードし、インポートすることで利用できます。
それぞれの環境変数を更新することで基礎的な検証が行えるかと思います。詳しくはこちらのGitHubリポジトリをご覧ください。
https://github.com/tatsumin39/zenn-contents/tree/main/sample/oktaworkflows

## まとめ
この記事では、Okta Workflowsを利用してセキュアにWebhookを受信する方法をSmartHRとkickflowの2つの事例を通じて確認しました。それぞれの特徴をまとめると以下のようになります。
- シークレットトークン: 対応は容易ですが、トークンを直接ヘッダーでやりとりするため、漏えい時のリスクが高いです。
- HMAC: 対応は複雑ですが、ヘッダーでハッシュ値をやりとりするため、より強固なセキュリティが確保できます。

これらの検証方法はSaaS側が対応している必要があります。対応していないSaaSでは、Webhookの設定で検証方法がない場合もあります。また、対応している場合でも、ドキュメントの記載が不十分で具体的にどのように検証可能かがわかりにくいこともあります。

この記事がWebhookセキュリティを見直すきっかけとなれば幸いです。