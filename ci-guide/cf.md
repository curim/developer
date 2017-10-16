# 第5章 CF環境構築

本ガイドではテスト用サーバ、公開用サーバ としてK5が提供するサービス「 CF 」を利用します。

「 CF 」はオープンソースの「Cloud Foundry」をベースとしたアプリケーション実行環境サービスです。

ご利用内容等の詳細は公式サイト[「FUJITSU Cloud Service K5　CF」](http://jp.fujitsu.com/solutions/cloud/k5/function/paas/cf/ )をご覧ください。

-----------------------------------------------------------------------------------------------------

## 5-1. CF導入方法（ビルドパック）および環境設定方法

[「 CF 使用手順チュートリアル 」](https://cf-docs.jp-east-1.paas.cloud.global.fujitsu.com/ja/manual/tut/tut/topics/preface.html)の「契約者のサービス利用開始の流れ」を参照し、ビルドパックを選択して配備、ユーザ登録を行って下さい。

本ガイドで使用するビルドパックは「Staticfile」です。

## 5-2. CFコマンドインストール

仮想マシン CentOS 7 にCFコマンドをインストールします。

[「CF 使用手順チュートリアル」](https://cf-docs.jp-east-1.paas.cloud.global.fujitsu.com/ja/manual/tut/tut/topics/preface.html)の「開発者のアプリケーション開発の流れ＞事前設定＞CFコマンドの事前設定」を参照し、インストールして下さい。

## 5-3. CFアップロード手順

CFへ資産をアップロードする手順を説明します。

本ガイドで構築するCIではJenkinsによって開発資産をCFへデプロイします。その基本となる手順です。

CFコマンドに関しては[「リファレンス CF コマンド」](https://cf-docs.jp-east-1.paas.cloud.global.fujitsu.com/ja/manual/ref/ref/topics/c-cf-cli.html)をご参考下さい。

仮想マシンCentOS7での作業です。

```bash
#アップロードするファイルがあるディレクトリへ移動
cd <ディレクトリ名>

#CFエンドポイントを設定
cf api --skip-ssl-validation <APIエンドポイント>
※<APIエンドポイント>：https://api.***.paas-cf.cloud.global.fujitsu.com ***部分はリージョン識別子

#ログイン
cf login -u <ユーザ名>@<契約番号> -o <契約番号> -s <スペース名>
(パスワードを入力)

# アップロード
cf push <APPLICATION_NAME>
※アップロードされたファイルは表示されたURLで確認できます。

#ログの表示
cf logs <APPLICATION_NAME> --<スペース名>

#ログアウト
cf logout

```

[[第6章 テストツールの導入へ]](test-tools.md)
