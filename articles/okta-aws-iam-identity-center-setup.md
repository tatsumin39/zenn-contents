---
title: "OktaとAWS IAM Identity CenterのSSO・Provisioning設定をまとめてみた"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["okta", "aws"]
published: true
published_at: 2026-07-12 17:00
publication_name: joug
---
こんにちは！たつみんです！

ひさびさにAWS環境をゼロから作成する機会があり、OktaからSSOやProvisioningを実現するためにAWS IAM Identity Centerの設定をしました。

その際、以前と少し変わっている箇所があったのでまとめてみます。

私が以前所属していた企業で執筆した以下の記事の最新版という位置付けです。

https://blog.cloudnative.co.jp/18900/

## 3行まとめ

- OktaとAWS IAM Identity Centerの連携手順は、大枠では以前と同じだよ
- SAMLメタデータやSCIMエンドポイントURLは、特別な理由がなければデュアルスタックでよさそうだよ
- Oktaから同期しただけではAWSアカウントは使えないので、グループ・AWSアカウント・許可セットの紐付けまで必要だよ

## 以前の記事から変わっていたところ

以前の記事では、AWS IAM Identity Center方式とAWS Account Federation方式を比較しながら、どちらを選ぶべきかを中心にまとめていました。

今回はあらためてAWS IAM Identity Center方式で設定してみたところ、考え方自体は大きく変わっていないものの、実際に手を動かすといくつか読み替えが必要な箇所がありました。

特に気になったのは以下の点です。

- Okta側の画面構成が変わっており、以前の記事で触れていたSign onタブ中心の説明から、AuthenticationタブやProvisioningタブを見ながら設定する流れに変わっていました。
- AWS IAM Identity Center側で、SAMLメタデータやSCIMエンドポイントURLにデュアルスタックとIPv4専用の選択肢が表示されるようになっていました。
- OktaでユーザーをアサインしてSSOできる状態にしただけでは、AWSアカウントにはアクセスできません。AWS IAM Identity Center側でグループ、AWSアカウント、許可セットを紐付けるところまで実施して初めて利用できる状態になります。
- AWS access portal上では、AWSアカウントと許可セットを選択して利用する形になるため、Okta側のグループ設計とAWS側の許可セット設計をあわせて考える必要があります。

そのため本記事では、単にSSOとProvisioningの設定手順をなぞるだけではなく、「Oktaで作成したユーザーやグループがAWS IAM Identity Centerでどのように扱われ、最終的にAWSアカウント利用につながるのか」まで含めて整理します。

## OktaとAWS IAM Identity Centerの関係性

OktaとAWS IAM Identity Centerを連携する場合、Okta側で管理しているユーザーやグループをAWS IAM Identity Centerへ同期し、その後AWS IAM Identity Center側でAWSアカウントと許可セットを割り当てます。

この関係性を理解しておくと、SSOやProvisioningの設定で「どこまでがOkta側の役割で、どこからがAWS側の役割なのか」が整理しやすくなります。

ざっくりとした流れは以下の通りです。

1. OktaでAWSを利用するユーザーを作成し、アクセス権限の単位に合わせてグループを作成し、所属させます。
2. OktaからAWS IAM Identity CenterへユーザーやグループをProvisioningします。
3. AWS IAM Identity Center上で、同期されたユーザーまたはグループに対してAWSアカウントと許可セットを割り当てます。
4. ユーザーはOktaからSSOし、割り当てられたAWSアカウントと許可セットを選択して利用します。

| 要素 | 役割 |
| --- | --- |
| Oktaユーザー | 利用者のIDです。Oktaで認証されます。 |
| Oktaグループ | 利用者をまとめる単位です。役割・権限単位などで作成します。 |
| IAM Identity Centerユーザー | OktaからProvisioningされたAWS側のユーザーです。 |
| IAM Identity Centerグループ | OktaグループからProvisioningされたAWS側のグループです。 |
| AWSアカウント | 実際にアクセスしたいAWS環境です。開発・検証・本番などの単位になります。 |
| 許可セット | AWSアカウント内で利用できる権限の定義です。AdministratorAccessやReadOnlyAccessなどを割り当てます。 |


:::message alert
OktaでAWS IAM Identity Centerアプリにユーザーやグループをアサインしただけでは、AWSアカウントへのアクセス権は付与されません。OktaからIAM Identity Centerへユーザー・グループを同期したうえで、AWS側でAWSアカウントと許可セットを割り当てる必要があります。
:::

実際の運用では、個別ユーザーに直接AWSアカウントを割り当てるよりも、OktaグループをIAM Identity Centerグループとして同期し、そのグループに対してAWSアカウントと許可セットを割り当てるほうが、利用者の追加・削除をOktaグループのメンバー変更だけで反映しやすくなります。

例えば、以下のようなイメージです。

| Oktaグループ | IAM Identity Centerグループ | AWSアカウント | 許可セット |
| --- | --- | --- | --- |
| AWS-Developers | AWS-Developers | 開発アカウント | PowerUserAccess |
| AWS-ReadOnly | AWS-ReadOnly | 開発 / 検証 / 本番アカウント | ReadOnlyAccess |
| AWS-Administrators | AWS-Administrators | 管理アカウント | AdministratorAccess |

このようにしておくと、利用者の追加・削除はOktaグループのメンバー管理で行い、AWS側ではグループ単位でAWSアカウントと許可セットを管理できます。

## 設定方法

ここではAWS IAM Identity Centerのインスタンス作成から実施していますが、すでにAWS IAM Identity Centerを利用中の場合は適宜読み替えてください。

### 事前準備

この記事ではIAM Identity Centerにプッシュするために、あらかじめ以下の3つのOktaグループを作成し、ユーザーを所属させておきます。

- AWS-Developers
- AWS-ReadOnly
- AWS-Administrators

### SSO設定

1. Browse App CatalogからAWS IAM Identity Centerを検索し**Add integration**をクリックします。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-01.png)
    
2. Application labelはそのままとし、Doneをクリックします。
3. Authenticationタブに移動し、SAML Signing Certificates内のアクションからView IdP metadataをクリックします。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-02.png)
    
4. 別タブで以下のような内容が表示されます。`<md:EntityDescriptor` から `</md:EntityDescriptor>`までをコピーし、テキストエディタ等に貼り付け、**metadata.xml**としてファイル保存します。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-03.png)
    
5. AWSマネジメントコンソールからIAM Identity Centerに移動し、有効にするをクリックします。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-04.png)
    
6. AWSリージョンの確認画面で問題がなければそのまま有効にするをクリックします。リージョンを変更したい場合は右上から行います。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-05.png)
    
7. ダッシュボードが表示されるので、左側のメニューから設定をクリックします。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-06.png)
    
8. アイデンティティソースのアクションからアイデンティティソースを変更をクリックします。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-07.png)
    
9. 外部 ID プロバイダーを選択し、次へをクリックします。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-08.png)
    
10. サービスプロバイダーのメタデータが表示されます。
    
    
    :::message
    今回の検証では、デュアルスタック側の値を利用して問題なく設定できました。IPv4専用にしたい理由がなければ、デュアルスタックで進めてよさそうです。
    :::

    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-09.png)
    
11. Okta Admin Console側の操作として、作成済みのAWS IAM Identity CenterのAuthenticationタブでSign-on settingsのEditボタンをクリックし、AWS SSO ACS URLとAWS SSO issuer URLにAWS側で表示された値を反映します。
    - AWS SSO ACS URL：**IAM Identity Center Assertion Consumer Service (ACS) の URL**
    - AWS SSO issuer URL：**IAM Identity Center 発行者 URL**
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-10.png)
    
    
    :::message
    Okta側の項目名にはAWS SSOという表記が残っていますが、現在のAWS IAM Identity Centerを指しているものとして進めます。また、AWS SSO ACS URL1からAWS SSO ACS URL17までの入力欄があります。これはIAM Identity Centerを複数のリージョンに複製した場合に利用します。
    :::

    
12. AWS側に戻り、IdP SAML メタデータに作成しておいた**metadata.xml**ファイルをアップロードし、次へをクリックします。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-11.png)
    
13. 確認および確定画面で「承諾」と入力し、アイデンティティソースを変更をクリックします。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-12.png)
    
14. 正常に変更しました。と表示されればSSO設定は完了です。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-13.png)
    

### **Provisioning設定**

1. IAM Identity Centerの設定に移動し、「自動プロビジョニング」の有効にするをクリックします。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-14.png)
    
2. 以下のようにSCIMエンドポイントURLとアクセストークンが表示されます。
    
    
    :::message
    デュアルスタックとIPv4 のみの2つのURLが表示されますが、デュアルスタックではIPv4 と IPv6 の両方に対応しているため特別な理由がなければデュアルスタックのURLを利用します。
    :::

    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-15.png)
    
3. Okta Admin Console側の操作として、作成済みのAWS IAM Identity CenterアプリのProvisioningタブに移動します。Enable API Integrationを有効にし、Base URLとAPI TokenにAWS側で表示された値を入力します。
    
    
    :::message
    以前の記事ではSCIMエンドポイントURLの末尾スラッシュを削除して`/v2`にする必要がありましたが、今回の検証ではAWS側に表示されたURLをそのまま入力して問題ありませんでした。
    :::

    
    Test API Credentialsをクリックし、successfullyと表示されたことを確認してSaveをクリックします。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-16.png)
    
4. To AppでCreate Users、Update User Attributes、Deactivate Usersそれぞれを有効化し、Saveをクリックします。
    
    
    :::message alert
    ここでは検証環境のためDeactivate Usersも有効化しています。本番環境では、ユーザー無効化時の影響を確認したうえで有効化すると無難です。
    :::

    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-17.png)
    

ここまでで、OktaからIAM Identity Centerへユーザーを同期する準備ができました。次に、SSOとProvisioningが動作していることを確認します。

## SSOとProvisioningの動作確認

OktaからAWS IAM Identity Centerにユーザーをアサインします。

![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-18.png)

Okta Dashboardで「AWS IAM Identity Center」をクリックすると、以下のようにAWS access portalが表示され、「アカウント」と「アプリケーション」のタブが確認できます。

これで基本的なSSOとProvisioningが動作している確認はできましたが、AWSアカウントやアプリケーションと紐付けていないためユーザーは何もすることができません。

![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-19.png)

## AWSアカウントとの紐付け

AWS上のリソースを利用するには、OktaとAWS IAM Identity Centerの関係性でも触れたとおり、OktaからProvisioningされたIAM Identity CenterユーザーをAWSアカウントと紐付ける必要があります。

ここでは個別ユーザーではなく、OktaグループをPush GroupsでAWS IAM Identity Centerへ同期し、IAM Identity Centerグループに対してAWSアカウントと許可セットを割り当てます。

1. Okta Admin Console側の操作として、作成済みのAWS IAM Identity CenterのPush Groupsタブで必要なグループを指定します。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-20.png)
    
2. AWS IAM Identity Centerのグループ一覧にOktaでプッシュしたグループが表示されています。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-21.png)
    
3. ひとつのグループを選択し、AWSアカウントと許可セットを選択し、割り当てをクリックします。必要な分だけこの操作をします。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-22.png)
    
4. あらためてOkta Dashboardで「AWS IAM Identity Center」をクリックし、AWS access portalにアクセスすると以下のように3つのAWSアカウントが表示されています。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-23.png)
    
5. 任意のAWSアカウントをクリックすると、利用可能な許可セットが表示されます。利用したい許可セットをクリックすると、対象のAWSアカウントへアクセスできます。
    
    ![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-24.png)
    

## CLIでの利用について

許可セットとともに表示されている「アクセスキー」をクリックすると、以下のようにAWS CLIで利用するための一時的な認証情報や設定コマンドが表示されます。

ここで表示される情報はIAMユーザーの長期アクセスキーではなく、AWS IAM Identity Center経由で取得する一時的な認証情報です。利用者は表示されたコマンドをコピーすることで、選択したAWSアカウントと許可セットの権限でCLI操作できます。

![](/images/okta-aws-iam-identity-center-setup/okta-aws-iam-identity-center-setup-25.png)

## 設定時に気になったところ

### デュアルスタックとIPv4専用のどちらを選ぶか

SAMLメタデータやSCIMエンドポイントURLでは、デュアルスタックとIPv4専用の選択肢が表示されます。今回の検証ではデュアルスタック側の値を利用して問題なく設定できました。

特別なネットワーク要件がなければ、デュアルスタックで進めてよさそうです。

### SCIMエンドポイントURLの末尾スラッシュ

以前の記事では、Okta側のBase URLに入力する際にSCIMエンドポイントURLの末尾スラッシュを削除し、`/v2/` ではなく `/v2` とする必要がありました。

今回の検証では、AWS IAM Identity Center側に表示されたSCIMエンドポイントURLをそのままOkta側のBase URLに入力して、Test API Credentialsがsuccessfullyとなることを確認できました。

このあたりはOkta側やAWS側の画面・仕様変更で挙動が変わる可能性があるため、実際の設定時にはTest API Credentialsで確認するのがよさそうです。

### OktaでアサインしただけではAWSアカウントは使えない

OktaからAWS IAM Identity Centerにユーザーをアサインすると、SSO自体はできます。ただし、AWSアカウントや許可セットを割り当てていない場合、AWS access portal上で利用できるAWSアカウントは表示されません。

SSO設定、Provisioning設定、Push Groups、AWSアカウントと許可セットの割り当てまでを一連の流れとして確認するのがよさそうです。

### アクセスキーはIAMユーザーの長期アクセスキーではない

AWS access portalで表示されるアクセスキーは、IAMユーザーに発行する長期アクセスキーではなく、AWS IAM Identity Center経由で取得する一時的な認証情報です。

名前だけ見ると少し紛らわしいですが、CLI利用時にはこの違いを意識しておくとよさそうです。

## まとめ

数年ぶりにOktaとAWS IAM Identity Centerを連携してみましたが、SSOとProvisioningの基本的な考え方は以前と大きく変わっていませんでした。

一方で、Okta側の画面構成やAWS側のデュアルスタック / IPv4専用の選択肢など、実際の設定画面は以前の記事から変わっている箇所がありました。

また、今回あらためて整理しておきたいと感じたのは、OktaでユーザーをアサインしてSSOできる状態にすることと、AWSアカウントを実際に利用できる状態にすることは別という点です。

OktaからAWS IAM Identity CenterへユーザーやグループをProvisioningしたうえで、AWS IAM Identity Center側でAWSアカウントと許可セットを割り当てる必要があります。

個別ユーザーに直接AWSアカウントを割り当てるよりも、OktaグループをPush GroupsでAWS IAM Identity Centerへ同期し、グループ単位でAWSアカウントと許可セットを割り当てるほうが運用しやすいと思います。

利用者の追加や削除はOktaグループ側で行い、AWS側ではグループとAWSアカウント、許可セットの対応関係を保つ形にすると、Okta管理者とAWS管理者の役割分担も整理しやすくなります。

以前の記事ではAWS IAM Identity Center方式とAWS Account Federation方式の比較を中心に書きましたが、今回はAWS IAM Identity Center方式を前提に、現在の画面でどのように設定し、どこまで実施すればAWSアカウントを利用できる状態になるのかを確認しました。

これからOktaとAWS IAM Identity Centerを連携する場合は、SSO設定、Provisioning設定、グループのPush、AWSアカウントと許可セットの割り当てをひとつの流れとして捉えると理解しやすいと思います。