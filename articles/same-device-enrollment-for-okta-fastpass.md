---
title: "Okta Verifyの同一デバイス登録機能を試してみた！：2024年6月のリリースノート"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["okta", "fastpass", "mfa"]
published: true
publication_name: joug
---
こんにちは！JOUG運営メンバーのたつみんです！

OktaではAuthenticatorとしてOkta Verifyを利用することが多いと思います。
今日は2024.06のリリースノートでEarly Access(早期アクセス)として以下の記述がありましたので、その検証結果を残しておきます。

> **Same-device enrollment for Okta FastPass**
On orgs with Okta FastPass, the Okta Verify enrollment process has been streamlined:
> - Users can initiate and complete enrollment on the device they’re currently using. Previously, two different devices were required to set up an account.
> - Users no longer need to enter their org URL during enrollment.
> - The enrollment flow has fewer steps.
>
> This feature is supported on Android, iOS, and macOS devices. To enable it, go to Admin ConsoleSettings and turn on Same-Device Enrollment for Okta FastPass.

## 概要
リリースノートの簡単な要約としては、これまでOkta Verifyの登録をする際に2つの異なるデバイスを使用する必要がありました。今回、ユーザーが使用しているデバイスで登録が行えるようになりました。

これにより、ユーザーアクティベーション時にモバイル版Okta Verifyの登録が不要となり、シームレスな登録が可能になります。

モバイル版Okta Verifyの登録ができない場合、以下のブログのように複雑なポリシー設計と煩雑なセットアップが必要でした。
[Oktaユーザーアクティベーション時にモバイル版Okta VerifyをセットアップをせずにOkta FastPassを使いたい時の対応方法](https://blog.cloudnative.co.jp/17539/)

## 留意点

なお、この機能はAndroid、iOS、macOSのみ対応ですが、現在プレビュー版のOkta Verifyバージョン5.1.3ではWindowsにも対応しています。正式リリースを待ちましょう。


## 設定方法

### Early Access（早期アクセス）機能の有効化

1. Okta管理画面のSetting > Featuresにアクセスします。
2. **Same-Device Enrollment for Okta FastPass**を有効化します。
   ![](/images/same-device-enrollment-for-okta-fastpass/image01.png)

### Okta Verify Enrollment optionsの調整
Okta管理画面>AuthenticatorsからOkta VerifyのEditからEnrollment optionsを以下のように**Any method**となっているかを確認します。
これにより現在対応ができないWindows端末の場合にQRコードによる登録を許容することができます。
![](/images/same-device-enrollment-for-okta-fastpass/image02.png)

## ユーザーアクティベーション時の挙動
実際にテストユーザーを作成し挙動を確認してみます。
1. アクティベーションメールや一時パスワードを利用しサインインを開始します。
2. Okta Verifyのセットアップを求めらられるので**セットアップ**をクリックします。
![](/images/same-device-enrollment-for-okta-fastpass/image03.png)
3. **Okta Verifyの設定**をクリックします。
![](/images/same-device-enrollment-for-okta-fastpass/image04.png)
   - ここで従来のモバイル版Okta Verifyでの登録を行うためのリンクも用意されています。これはOkta Verify Enrollment optionsの調整で設定したためです。
4. Okta Verifyが起動するので**アカウントの追加**をクリックします。
![](/images/same-device-enrollment-for-okta-fastpass/image05.png)
   - ここではすでにOktaのテナントURLが入力された状態となっています。
5. 残りの操作はこれまでと同様に実施します。
6. 登録後に3.の画面に戻ってきますが変化がないため、**サインインに戻ります**をクリックします。
7. サインイン画面で**Okta FastPassでサインインする**をクリックし、Okta Dashboardにアクセスできることを確認します。

## まとめ
このEarly Access(早期アクセス)によってOktaアカウントのアクティベーション時に複数の端末を行き来しなくてよくなりました。ユーザーにとっても説明をする情報システム部門にとっても摩擦のないスムースなオペレーションができることは素晴らしいなと思います。
一つ残念なのは、登録後に自動遷移しない点です。Okta Dashboardまで自動遷移すれば、作業がスムーズに完了すると思います。

Windows版Okta Verifyもすぐに対応バージョンにアップデートされると思います。そうなればすべてのプラットフォームで悩みの種が一つ減りますね。