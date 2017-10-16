# 第7章 CI用Pipelineの設定

この章では[『想定利用シナリオ』](#scenario) に基づいて [Jenkins Pipeline](#pipeline) を作成していきます。

大筋では『想定利用シナリオ』の各作業をJenkinsのジョブとして作成し、次にそれらをPipelineでまとめ、最終的には資産格納を契機に自動的にCIを実行できるような設定にします。

## Jenkinsのジョブの作成

第2章で紹介した[ジョブ作成手順](#job)を参考に『想定利用シナリオ』で必要なジョブを作成します。

『想定利用シナリオ』の中でJenkinsのジョブを作成する部分は具体的には以下の部分です。

（ Pullrequest または Merge を契機にJenkinsのPipelineが起動します。）

```
「GitHub Enterprise」から資産を取得しWorkSpaceに格納
↓
Markdown 構文チェック
↓
html ファイル生成（ブログ資産生成）
↓
html 構文チェック
↓
テストサーバへデプロイ
（Mergeの場合はステージングサーバへデプロイ）
↓
アタックテスト（脆弱性検査）実施
↓
成功時メール報告
↓
Mergeの場合は本番環境へデプロイ
```

資産格納先である「GitHub Enterprise」へ開発チームがPullrequestを行うのか管理者がMergeを行うかでジョブが少々異なりますが、ほぼ同じジョブを繰り返す形になります。

### 事前準備

#### Hexo資産作成と「 GitHub Enterprise 」への格納

これからJenkinsの設定を行っていきますが、前提として、 Hexo の資産が「 GitHub Enterprise 」に格納されていなければなりません。

Hexo資産の作成は[「第2章 2-3\. Hexo導入手順」](#hexo)で紹介した例を参考にしてください。

次に作成したHexo資産を「 GitHub Enterprise 」に格納します。

- リポジトリの作成

  「 GitHub Enterprise 」にHexo資産格納用のリポジトリを作成し、Hexo を導入したディレクトリをそのリモートリポジトリに します。

  手順は[「リポジトリ作成方法」](#repository)を参考に、以下の通りです。

  ```bash
  # Hexo を導入したディレクトリに入ります。
  cd <Hexoを導入したディレクトリ>

  # 「 GitHub Enterprise 」で作成した Hexo用のリポジトリをクローンします。
  git clone git@github.com:ユーザ名/リポジトリ名
  （または git clone https://github.com/ユーザ名/リポジトリ名 ）

  # 上記で上手くいかない場合は
  git init
  git remote add origin  https://github.com/ユーザ名/リポジトリ名
  ```

- 資産の格納

  Hexo 作業用ディレクトリと「 GitHub Enterprise 」の Hexo 資産格納用のリポジトリと連携ができましたら、

  資産を「 GitHub Enterprise 」へ格納します。

  ```bash

  # ローカルリポジトリのファイルをインデックスに追加
  git add .

  # コミット
  git commit -m "[コミットのコメント記入]"

  # プッシュして ローカルリポジトリをリモートリポジトリへ反映させます。
  git push origin master
  ```

  以上で「 Hexo 」で作成した静的ウェブページの資産が「 GitHub Enterprise 」のリポジトリに格納されます。

#### テストツールの実行コマンド

本ガイドではJenkinsによって各テストツールを実行させていきます。

以下、「第6章 テストツールの導入」で紹介した各テストツールの実行コマンドがJenkinsユーザから実行できるようパスが通っている前提で、Jenkinsジョブの作成を行います。

では、１つずつジョブを作成していきます。


##### ジョブ1. 資産取得（GitHub EnterpriseからWorkSpaceへ資産格納）

開発資産を管理している「GitHub Enterprise」のリポジトリから資産を取得し、Jenkinsの作業スペースであるWorkSpaceに資産を格納します。

注意点は、

- Pullrequestの場合とmergeの場合で取得するリポジトリのブランチを変えること。
- WorkSpaceの内容を常に最新の状態にするため「ビルド開始前にワークスペースを削除する」を行うこと。

です。

以下、設定手順
- JenkinsでGitHub Enterprise資産取得用のジョブを作成します。

- [ 新規ジョブ作成 ] → [ フリースタイル・プロジェクトのビルド ] で任意の名前のジョブを作成します。

- [ 設定 ] → [ ソースコード管理 ] タブ → [ Git ]を選択し、以下の項目を設定します。
  - リポジトリ：

    - リポジトリURL

      資産を取得するGitHubリポジトリのSSHを設定します。

    - 認証情報

      [「4-3\. SSHの設定」](#SSH) を参考に、認証情報を設定します。

 - ビルドするブランチ：

    - ブランチ指定子

      取得する資産が格納されているブランチを設定します。

      ※複数のブランチの設定やワイルドカードの仕様が可能。

      本書では、

      ```
      ・ 開発チームが行うPullrequestの場合 →「 */develop 」ブランチ
      ・ 管理者が行うmergeの場合 →「 */master 」ブランチ
      ```

      を設定します。

      ![Jenkins 資産取得](./image/asset_01.jpg)

- [ 設定 ] → [ ビルド環境 ] → [ ビルド開始前にワークスペースを削除する ]にチェックします。

  これはWorkSpace内に以前に取得した資産を残さず、常に最新の資産のみ保持するための措置です。

  ![Jenkins 資産取得](./image/asset_02.jpg)

  設定が完了したら、左下の保存を押下し、完了です。

次に取得したHexo資産を各種テストツールでテストしていきます。

ここでは管理しやすいように資産作成と各テストごとにJenkinsのジョブを分けて作成します。

##### ジョブ2. Markdown構文チェック

hexoで生成したMDファイルは、

`workspace/<資産取得のジョブ名>/source/_posts`に格納されます。

このMDファイルをMarkdown構文チェックツール「 Markdownlint 」で検査します。

「 Markdownlint 」の実行コマンドは `mdl < Markdown ファイル>` です。

以上を踏まえて、Jenkinsのジョブを作成します。

- [ 設定 ] → [ ビルド ] → [ ビルド手順の追加 ]を押下し、プルダウンメニューから [ シェルの実行 ]を選択し、以下を記述します。

  ```bash
  cd ../<資産取得のジョブ名>/source/_posts
  mdl *.md
  ```

  実際には「 Markdownlint 」の実行コマンド「mdl」は各自のパスの設定などにあわせて記述してください。

  【参考画像】※参考画像では、<資産取得のジョブ名>＝guide_source 実行コマンド「mdl」＝/usr/local/bin/mdl

  ![job_mdl](./image/job_mdl.jpg)

##### ジョブ3. htmlファイル生成（ブログ資産生成)

Markdown構文チェックを通過したMDファイルからhtmlファイルを生成します。

htmlファイル生成には `hexo generate` コマンドを実施します。

- 「Markdown構文チェック」と同様に、 [ シェルの実行 ]を選択し、以下を記述します。
  ```bash
  cd ../<資産取得のジョブ名>/
  hexo generate
  ```

##### ジョブ4. html構文チェック

`hexo generate` で生成したhtmlファイルはデフォルトでは

`workspace/<資産取得のジョブ名>/public/`に格納されます。

この生成したhtmlファイルの構文チェックを「HtmlHint」ツールで実施します。

- 「Markdown構文チェック」と同様に、 [ シェルの実行 ]を選択し、以下を記述します。

  ```bash
  cd ../<資産取得のジョブ名>/public
  htmlhint
  ```

##### ジョブ5. テストサーバへデプロイ

本ガイドではテスト用サーバとしてK5提供サービスの「 CF 」を利用します。

また本ガイドでのデプロイ先はすべて「 CF 」になりますので手順は同じになります。

デプロイする資産であるhtmlファイルは `/var/lib/jenkins/workspace/<資産取得のジョブ名>/public` にあります。

- 「Markdown構文チェック」と同様に、 [ シェルの実行 ]を選択し、以下を記述します。

  ```bash
  readonly APPLICATION_NAME=<アプリケーション名>
  cd ../<資産取得のジョブ名>/public
  cf api --skip-ssl-validation <APIエンドポイント>
  cf auth <ユーザーID><パスワード>
  cf target -o <組織名>
  cf target -s <スペース名>
  cf push ${APPLICATION_NAME} -p ../<資産取得のジョブ名>/public
  ```

デプロイの確認は、ジョブの [ コンソール出力 ]に表示されたCFアップロード結果のURLで確認できます。

##### ジョブ6. アタックテスト（脆弱性検査）

テストサーバーに格納したファイルに「skipfish」を利用してアタックテスト（脆弱性検査）を実施します。

- 「Markdown構文チェック」と同様に、 [ シェルの実行 ]を選択し、以下を記述します。

  ```bash
  # 結果レポート格納用のディレクトリ作成
  rm -rf results
  mkdir results

  # skipfish 実施
  cd <Skipfishインストール先ディレクトリ>

  # 環境変数に WORKSPACE を設定します
  （※$WORKSPACE = /var/lib/jenkins/workspace/このジョブ名/）
  ./skipfish -o $WORKSPACE/results <CFアップロード結果のURL>
  ```

このジョブでのワークスペースは自身のworkspaceを利用します。

このジョブのworkspace以下にresultsディレクトリが作成され、html形式の結果レポートが格納されます。

##### ジョブ7. 成功時メール報告

すべてのジョブが成功した場合に管理者へメールにて報告するためのジョブを設定します。

[「拡張E-mail通知の設定」](#extend_email)を参考に設定してください。

手順
- [ 設定 ] → [ ビルド後の処理 ] → [ ビルド後の処理の追加 ]を押下し、プルダウンメニューから [ 拡張E-mail通知 ]を選択します。

  ![job_mail01](./image/jobmail01.jpg) ![job_mail02](./image/jobmail02.jpg)

  適宜設定項目を記述すれば完了です。

以上で各ジョブが準備できました。

## Jenkins Pipeline の作成

作成したジョブをPipelineでまとめ、資産格納を契機に自動的にCIを実行できるように設定します。

ポイントは資産格納の「Pullrequest」と「Merge」を判断し、それぞれを契機にPipelineを実行させる点です。

### Pullrequest用のPipeline作成

まずは、[「Pipeline作成手順」](#pipeline)を参考に想定シナリオの「Pullrequest」のパターンを設定していきます。

手順

- [ 新規ジョブの作成 ] → [ Pipeline ]を選択します。

- [「第4章 4-2\. WebHookの設定 」](#webhook)を参考にWebHookを設定します。

- ジョブの詳細設定画面でPipelineエリアにスクリプト記述します。

  「Pullrequest用 Pipeline」のScript 記述例[]()

  ```groovy
  def p = env.payload.indexOf("action\":\"opened")
  if(p != -1){
    node{
      # 資産取得
      stage('Pullrequest資産取得'){
        build job: '< 1.資産取得のジョブ名 >'
      }

      # Markdown 構文チェック
      stage('MD構文チェック'){
        build job: '< 2.Markdown 構文チェックのジョブ名 >'
      }

      # htmlファイルを生成
      stage('html生成'){
        build job: '< 3.htmlファイル生成のジョブ名 >'
      }

      # html構文チェック
      stage('html構文チェック'){
        build job: '< 4.html構文チェックのジョブ名 >'
      }
    }

    # 管理者認証（Pipelineの一時停止）
    stage('管理者認証'){
      input '続行しても大丈夫ですか？';
    }

    node{
      # CF（開発）サーバ（テストサーバ）へデプロイ
      stage('CF開発格納'){
        build job: '<5.テストサーバへデプロイのジョブ名 >'
      }

      # アタックテスト（脆弱性検査）
      stage('脆弱性チェック'){
        build job: ' <6.アタックテスト（脆弱性検査）のジョブ名> '
      }

      # 成功時メール報告
      stage('メール報告'){
        build job: ' <7.成功時メール報告のジョブ名> '
      }
    }
  }
  ```

  Scriptの解説

  ---

  1. payloadの判定

    webhookが発生するとGitHub側からpayloadのパーラメータが送られてきます。

    このパーラメータでwebhookが発生したイベントが「pullrequest」かどうかを判定しています。

    ```groovy
    def p = env.payload.indexOf("action\":\"opened")
      if(p != -1){
      ----- 省略 -----
      }
    ```

  2. 管理者認証stage

    管理者認証というstageを「CF開発サーバ（テストサーバ）へデプロイ」する前に設定しました。

    これは、Pipelineを一旦停止し、再開には管理者の認証を必要とするステップを加えています。

    `input 'メッセージ'` と記述することで、Pipeline画面もしくはコンソール出力画面に記述したメッセージとともに 「Proceed or Abort？」と選択肢が出現するようになります。

    このStageで一旦停止した場合、メール通知する設定が行えますので、管理者の認証として利用できます。

    例：inputで一時停止した際にメールを送信

    ```groovy
    # 管理者認証（Pipelineの一時停止）
    stage('管理者認証'){
      mail (to: '<メールアドレス>',
      subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) is waiting for input",
      body: "Please go to ${env.BUILD_URL}.");
      input '続行しても大丈夫ですか？';
    }
    ```

  ---

### Merge用のPipeline作成

mergeを契機に実行されるPipelineの記述もほぼ同じです。

異なるのは以下の点です。

- payloadのパーラメータの判定方法
- masterブランチから資産取得すること
- テストサーバとしてCF(Staging）を利用すること

以下、異なる点の詳細になります。

- mergeの場合のpayloadの判定

  mergeの場合、payloadのパーラメータは以下で判断します。

  ```groovy
  def cl = env.payload.indexOf("action\":\"closed")
  def cls = env.payload.indexOf("merged\":true")

  if(cl != -1 && cls != -1){
     ----- 省略 -----
  }
  ```

- masterブランチから資産取得する

  merge用の資産取得ジョブでは必ずmasterブランチから資産を取得するように設定します。

- ステージングサーバの利用

  mergeした資産は公開用になりますので、テスト環境は本番環境と同等のステージング環境で行います。

  本ガイドでは CF（staging）としてステージングサーバを利用します。

  「merge用 Pipeline」のScript 記述例[]()

  ```groovy
  def cl = env.payload.indexOf("action\":\"closed")
  def cls = env.payload.indexOf("merged\":true")

  if(cl != -1 && cls != -1){
    node{
      stage('merge資産取得'){
        build job: 'guide_source_master'
      }

      stage('MD構文チェック'){
        build job: 'guide_md'
      }

      stage('html生成'){
        build job: 'guide_html'
      }

      stage('html構文チェック'){
        build job: 'guide_htmlhint'
      }
    }

    stage('管理者認証'){
        input '続行しても大丈夫ですか？';
    }

    node{
      stage('CF-staging格納'){
        echo '省略します'
      }

      stage('脆弱性チェック'){
        echo '省略します'
      }
    }
  }
  ```

[[第8章 デモ実行と運用へ]](demo.md)
