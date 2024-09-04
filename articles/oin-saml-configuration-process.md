---
title: "Okta Integration Network(OIN)にSaaSを登録した話"
emoji: "⚙️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["okta"]
published: true
published_at: 2024-09-06 08:00
publication_name: joug
---
こんにちは！たつみんです🖐️

今回は、私が所属しているSaaS企業で、Okta Integration Network (OIN) にSAML連携を公開した際の備忘録と裏話をお届けします。サクッと読める内容なので、「OIN登録ってこんな感じなんだ」と軽い気持ちで読んでいただければと思います。

## OINとは？
OktaでSaaSのSAML設定やSCIM設定などの連携をテンプレート化したものです。
このテンプレートにより、Oktaとの連携が数ステップで完了できるようになります。
企業や組織は、セキュアかつ簡単にSaaSアプリをOktaに統合できます。

## OINにSAML連携を登録するために必要なもの
- Okta Developer Edition テナント
- Okta Browser Plugin
- Customer configuration document guidelines 公開用のオンラインストレージ

## OINへの登録から公開までの流れ
基本的な流れとしては、以下のガイドに従い作成し、申請を行います
[Submit an SSO integration with the OIN Wizard](https://developer.okta.com/docs/guides/submit-oin-app/saml2/main/)

### 登録の大まかな流れ
1. Okta Developer Edition テナントを作成
2. Okta管理画面からApplication > Your OIN Integrationsにアクセスし、ウィザードに従って設定
3. 英語でCustomer configuration document guidelinesを作成し、オンラインで公開
4. SAML連携アプリケーションを作成し、セルフテストを実施
5. **Okta (本国)** へ申請
6. フィードバックで修正点があれば修正後、再度審査依頼
7. 審査OKの連絡後、公開日を調整
8. Okta Japan担当者とプレスリリースに関して打ち合わせ
9. OIN公開を確認し、プレスリリース公開

## Your OIN Integrations での操作
Your OIN Integrationsは非常に使いやすいので、簡単にご紹介します。
画面は以下の3つのステップで構成されており、順番に進めるだけで申請まで完了できます。
- Select your protocol
- Configure your integration
- Test Integration experience


1. **Select your protocol**内で「Security Assertion Markup Language (SAML)」を選択します。
![](/images/oin-saml-configuration-process/image01.png)
2. **Configure your integration**内でOINカタログ上に表示されるプロパティを設定します。
![](/images/oin-saml-configuration-process/image02.png)
3. 続いてIntegration variablesで変数の定義を行います。
![](/images/oin-saml-configuration-process/image03.png)
4. SAMLプロパティを定義します。ここでは3. で定義した変数をorg.<変数名>で活用できます。
![](/images/oin-saml-configuration-process/image04.png)
5. **Test Integration experience**内のTest integrationでOkta担当者が利用する審査時に利用するテスト用アカウントを指定します。
![](/images/oin-saml-configuration-process/image05.png)
6. IdP-initiatedやSP-initiated、Just-In-Time provisioningに対応するか選択します。SP-initiatedの場合は、テスト開始用のURLを指定します。
![](/images/oin-saml-configuration-process/image06.png)
7. Generate Instanceからアプリケーションを作成し、Add To Testerをクリックします。
![](/images/oin-saml-configuration-process/image07.png)
8. OIN Submission Tester (SSO only)でRun Testを実行します。
![](/images/oin-saml-configuration-process/image08.png)
9. テスト結果が全てクリアしたらSubmit integrationをクリックし、申請を行います。
![](/images/oin-saml-configuration-process/image09.png)


### Run Testの挙動
Run Testを実行すると自動的にシークレットブラウザが起動し、Okta Browser Pluginによりテストが進行します。私たちのSaaSはSP-initiatedとJust-In-Time provisioningに対応していたため、それぞれのテストを実行しました。テスト結果は自動的にJSONファイルとして作成され、エクスポートすることが可能です。


## 苦労した点
当初、別システム「OIN Manager」から申請を行う必要がありましたが、作業開始後に「Okta Developer Edition」から申請する方式に切り替わりました。このため、混乱が一部ありましたが、Oktaの最新の申請プロセスに従えばスムーズに進められるはずです。
また、セルフテストでうまくいかない点もありましたが、Okta Japanのエンジニアにサポートしていただき、無事に進行できました。


## まとめ
Okta Integration NetworkへのSAML連携登録は、Your OIN Integrationsの直感的な操作により、スムーズに進められます。SaaSをOktaと連携させる際は、このプロセスが役立つかもしれません。今後OINにSaaSを登録する方の参考になれば幸いです。
