# 第4章 JenkinsとGitHub Enterpriseの連携

本ガイドでは構成管理ツール 「GitHub Enterprise」とCIツール「Jenkins」を連携させることによってCI環境を構築していきます。

「GitHub Enterprise」と「Jenkins」の連携に必要な設定として「WebHookの設定」と「SSHの設定」を行います。
連携を行うことで「 GitHub Enterprise 」のリポジトリに行われたイベントを「Jenkins」へ通知し、そのイベントをトリガーに「Jenkins」のジョブを起動させることができます。


以下では「GitHub Enterprise」と「Jenkins」の連携に必要な「WebHookの設定」と「SSHの設定」の手順を紹介します。

--------------------------------------------------------------------------------------------------------------------------------

## 4-1. GitHub EnterpriseのJenkinsへのトリガー

本ガイドでは「 GitHub Enterprise 」のリポジトリに行われた「PullRequest」と「Merge」をトリガーとしてJenkinsのジョブを実行させます。

Jenkinsでは「PullRequest」で起動するジョブと「Merge」で起動するジョブを用意し、
次の項で紹介する WebHookの機能で「 GitHub Enterprise 」から「Jenkins」へ送られてくる情報を判断し、該当するジョブを実行します。

具体的なジョブは第7章で作成します。

ここではJenkinsの起動に「PullRequest」と「Merge」をトリガーとして利用することを押さえて下さい。

## 4-2. WebHookの設定

WebhookとはPushやPullRequestなどのイベントによりGitHub Enterpriseのリポジトリに変化があったことを連携するURLへ通知する機能です。

Payloadというパラメータでイベントに関する詳細情報を渡すことができます。

連携先をJenkinsにすることによって、GitHub Enterpriseのイベント情報を契機(トリガー)にJenkinsのジョブを実行することが可能となります。

GitHub Enterpriseと連携先であるJenkinsの両方に設定が必要です。

-----------------------------------------------------------------------------------------------------------------

### Jenkins 側での設定

Jenkins側での設定については、ジョブ作成時に設定が必要になります。

以下、設定手順

1. 認証トークンの設定
  - ジョブの[ 設定 ] → [ build triggers ] → [ リモートからビルド ] にチェックします。
  - チェック後、[ 認証トークン ] に任意のトークンを記述します。( 例: pullrequest )

  > ![認証トークン](./image/WebHook1.png)

2. Payload

  「Payload 」とは WebHook の契機となったイベントの詳細情報が入っているパラメーターです。

  Jenkins側では、このパラメータを受け取ることで、ジョブのトリガーとして利用することが可能になります。

  さらに詳細情報を読み取ることで発生したイベントが「PullRequest」なのか「Merge」なのかを判別することも可能になります。

  - ジョブの[ 設定 ] → [ general ] → [ ビルドのパラメータ化 ] にチェックします。
  - チェック後、[ パラメータの追加 ] → [ 文字列 ] を選択します。
  - [名前]に "payload" と記述してください。

  > ![buildのパラメータ化](./image/WebHook2.png)

### GitHub Enterprise側での設定

GitHub Enterprise側では、連携先URLとWebhookを作動させるイベントを設定します。

以下、設定手順

1. Webhook設定画面の表示
  - 対象のリポジトリの [ Settings ] → [ Hooks&Service ] を選択します。
  - [ Add webhook ]ボタンを押下します。

  > ![Add webhookl](./image/WebHook3.png)

  下記設定画面に遷移したら赤①から③を設定します。

  【Webhooks/Add Webhooks 設定画面】
  > ![Add webhook2](./image/WebHook7.png)

1.  Payload URL 設定

  上記【Webhooks/Add Webhooks 設定画面】の赤①の欄に以下を設定します。

  http://[USER_ID]:[API_TOKEN]@[JENKINS_HOST]/job/[JOB_NAME]/buildWithParameters?token=[TOKEN_NAME]

  ```
  Payload URL 設定
  【USER_ID】：Jenkinsにログインするためのユーザ名
  【API_TOKEN】：Jenkinsのアカウントごとに発行されるトークン(後述)
  【JENKINS_HOST】：JenkinsのHOST名
  【JOB_NAME】：WebHookで連携するJenkinsで作成したジョブ名
  【TOKEN_NAME】：連携するJenkinsのジョブで設定した認証トークン

  ```
  例)`http://admin:xxxxx@jenkins.example.com/job/sample/buildWithParameters?token=pullrequest`

  【API_TOKENの取得方法】
  ---
    API TokenはJenkins側で取得します。

    Jenkinsのセキュリティ機能が有効になっている場合、WebHookを介してビルドを実行するには「ユーザーID」と「API Token」が必要になります。

    ※API Tokenとはログイン時に用いる「パスワード」ではなく、アカウントごとに発行される32文字のランダムな英数字列です。

    以下、取得手順

    1. [ Jenkinsの管理 ] → [ ユーザーの管理 ] を選択します。
    > ![API Token](./image/WebHook4.png)

    2. ユーザーID を選択し、[ 設定 ]マーク を押下します。
    > ![API Token](./image/WebHook5.png)

    3. [ APIトークンの表示 ] で取得することができます。
    > ![API Token](./image/WebHook6.png)

  ---

3. Content type 設定

  前掲の【Webhooks/Add Webhooks 設定画面】の赤②で設定を行います。

  "Content type"はどのような形式でPayloadを表現するかを決めます。

  選択肢に
  - application/json
  - application/x-www-urlencoded

  が用意されておりますが、Jenkinsで対応している「application/x-www-urlencoded」を指定します。<br>

  > ![Content type](./image/webhook_contentType.jpg)


4.  Which events would you like to trigger this webhook?

  前掲の【Webhooks/Add Webhooks 設定画面】の赤③で設定を行います。

  "Which events would you like to trigger this webhook?"ではWebHookを起動させるタイミングを指定できます。

  設定は任意で、選択肢として以下が用意されています。

  - 「Just the push event.」

      push時のみWebHookを起動します。

  - 「Send me everything.」

      全てのイベントに対してWebHookを起動します。

  - 「Let me select individual events.」

      選択した項目に対してWebHookを起動します。

  本書ではpullrequest時のみWebHookを起動させるため、『Let me select individual events.』を選択し、PullRequestを選択します。

  > ![Content type](./image/webhook_trigger.jpg)

  『Let me select individual events.』の選択肢は以下の通りです。

  今回は「Pull request」と「merge」でジョブを分けますので、「Pull request」にだけチェックをつけます。

  はじめからチェックがついている「Push」のチェックは外してください。

  > ![GitHub WebHook ](./image/WebHook8.png)

## 4-3. SSHの設定

JenkinsとGitHub Enterpriseを接続するために必要な設定です。

※ここでは、JenkinsにSSH接続するために必要な秘密鍵と公開鍵が作成されていることを前提としています。

### Jenkins の設定

Jenkins 側では、ジョブ作成時にSSH用の秘密鍵を認証情報として設定する必要があります。※設定が必要なジョブは、GitHubから資産を取得するジョブのみになります。

以下、設定手順

- [ ソースコード管理 ] → [ Git ] にチェックします。
- チェック後、[ リポジトリURL ] に連携したいGitHubのリモートリポジトリURLを入力します。
- [ 認証情報 ]にある[ 追加 ]を押下し、出力される[ Jenkins ]を選択します。

【ソースコード管理画面】

> ![認証トークン](./image/SSH3.png)

選択後、[ 認証情報の追加 ] 画面がポップアップします。

【認証情報の追加画面】

> ![Jenkins SSH](./image/SSH4.png)

以下の項目を設定します。
- Domain<br>

  設定内容：グローバルドメイン(固定)

- 種類<br>
  設定内容：SSHユーザ名と秘密鍵

- スコープ<br>
  設定内容：グローバル<br>
  ※詳細は選択肢の右にあるヘルプアイコンから確認してください。

- ユーザー名<br>
  設定内容：Jenkinsがインストールされているサーバのユーザ名<br>

- 秘密鍵<br>
  設定内容：３つの選択肢から秘密鍵の参照元を選びます。

  - 「Jenkinsマスター上の~/.sshから」

    Jenkinsマスター上にある.sshファイルに格納されている秘密鍵を参照します。

  - 「Jenkinsマスター上のファイルから」

    Jenkinsマスター上にあるファイル名を設定し、参照します。

  - 「直接入力」

    直接、秘密鍵の情報を記述し、設定します。

  ※本書では「直接入力」を選択します。

以上の項目を設定したら、一番下の左にある"追加"を押下し、設定完了です。


### GitHub Enterpriseの設定

GitHub側では、GitHubの個人設定からSSH用の公開鍵を設定する必要があります。

以下、設定手順

 [ 右上のプロフィールアイコン ] → [ Settings ] → [ SSH & GPG keys ] を選択します。

> ![GitHub SSH](./image/SSH1.png)

[ New SSH Key ]ボタンを押下し、以下の項目を設定します。

- Title

  設定内容：登録する SSH key の名称を任意で設定します。

- Key

  設定内容：Jenkinsに接続するための公開鍵情報を設定します。

> ![ GitHub SSH ](./image/SSH2.png)

設定後、[ Add SSH key ] を押下し、設定完了です。

[[第5章 CFの説明へ]](cf.md)
