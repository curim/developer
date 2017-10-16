# 第8章 デモ実行と運用

前章までで構築したCI環境をもとにCI(継続的インテグレーション)を実施します。


[「想定利用シナリオ」](#scenario)をもとに作業し、運用していきます。


------------------------------------------------------------------------------------

## 8-1. 「想定利用シナリオ」の実行と運用

概要の[シナリオ運用概要図](#ci_02)どおりに作業を進めます。

### 開発資産取得

> ![scenario01](./image/scenario01.jpg)

各開発チームが作業を行う前にGitHub Enterpriseのリポジトリのmasterブランチから資産を取得します。

手順は[「第3章 3-2.PullRequest と Merge 」](#git_function)で紹介したとおりです。

ここでは、

- 資産管理のリモートリポジトリ名を「CI_Guide」
- リポジトリ「CI_Guide」のmasterブランチには開発資産がすでに格納されている前提
- 開発作業用のブランチ名を「team_A」

として以下を実行します。

- リポジトリのクローン

  開発チームの仮想サーバへ資産管理のリモートリポジトリ「CI_Guide」をクローンします。

  `git clone https://github.com/ユーザ名/CI_Guide.git`

- ブランチの作成

  次に開発チームの作業用ブランチ「team_A」を作成します。

  `git checkout -b team_A`

- 開発作業

  作成したブランチに入り作業します。

### ブランチをPush

> ![scenario02](./image/scenario02.jpg)

開発チームでの作業が完了したら、リモートリポジトリ「CI_Guide」へPushします。

手順は同じく[「第3章 3-2.PullRequest と Merge 」](#git_function)で紹介したとおりです。

想定利用シナリオでは以下を実行します。

- ファイルのPush

  「ci_test.md」ファイルを作成し、リモートリポジトリへPushします。

  ```
  # 作成したファイルをインデックスに追加。
  git add ci_test.md

  # コミットしメッセージ「開発チームＡ」をつけます。
  git commit -m "開発チームＡ"

  # 作業用ブランチ「team_A」へプッシュします。
  git push -u origin team_A
  ```

- Pushの結果確認  

  正しくPushできたかGitHub Enterpriseの画面で確認します。

  【リモートリポジトリ「CI_Guide」画面その１】
  > ![scenario_push](./image/scenario_push.jpg)

  赤枠①「Your recently pushed branches」に最近pushしたブランチが表示されます。

  赤枠② [Branch:master]をプルダウンすると、

  赤枠③ 開発チーム用に作成した「team_A」ブランチが表示されます。

  赤枠③の「team_A」ブランチを選択して画面を遷移させます。

  【リモートリポジトリ「CI_Guide」画面その２】
  > ![scenario_branch](./image/scenario_branch.jpg)

  赤枠④にコミットする際につけたメッセージ「開発チームＡ」が正しく表示されています。

  ブランチにPushしたファイルは「ci_test.md」は「souce/\_posts」フォルダに格納されました。


### Pull Requestを契機にJenkinsの作動

ブランチに正しく資産が格納できましたら、レビュー依頼 Pull Requestを作成します。

想定利用シナリオではこのPull Requestを契機にJenkinsが実行されます。

> ![scenario03](./image/scenario03.jpg)

想定利用シナリオでは以下を実行します。

- シナリオ①「GitHubでPullRequest（レビュー依頼）」

  前掲【リモートリポジトリ「CI_Guide」画面その２】の赤枠⑤からPull Requestを作成します。

  手順は第3章の[「PullRequest 手順」](#pullreq)の通りです。

  > ![scenario_pullreq](./image/scenario_pullreq.jpg)

- シナリオ②「PullRequestを契機にwebhook」

  PullRequestを作成するとJenkinsの[『「Pullrequest用 Pipeline」のScript 記述例』](#script_pullreq)で作成したPipelineが作動します。

  本ガイドでは「pullrequest_teamA」という名で作成し実行しました。

  Jenkins画面で確認します。

  > ![scenario_input](./image/scenario_input.jpg)

  「管理者認証」のステージで一時停止していることが確認できます。

  メール通知を設定していれば、メールを確認後、この画面から認証をしてPipelineを続行させます。

  全てのステージが完了すれば以下のような画面になります。

  > ![scenario_result](./image/scenario_result.jpg)


- シナリオ③ CIの結果を通知

  Pipelineの結果は設定したメールにて管理者に報告されます。

  管理者はソースのレビュー結果とPipelineで実施された各種テストの結果を検討します。


- シナリオ④デプロイ　⑤実際の資産の確認

  デプロイの確認は、ジョブの [ コンソール出力 ]に表示されたCFアップロード結果のURLで確認できます。

  > ![scenario_hexo](./image/scenario_hexo.jpg)

  上記画像では開発チームＡが作成した「ci_test.md」ファイルからhtmlファイルが生成され、正しく表示されていることが分かります。

### Mergeを契機にJenkinsの作動

管理者はソースのレビュー結果とPipelineで実施された各種テストの結果を検討し、さらにデプロイされた資産の確認を行い、全て問題なければ開発資産の統合を行います。

> ![scenario04](./image/scenario04.jpg)

- シナリオ⑥ masterブランチへMerge

  GitHub Enterpriseの対象リポジトリの画面からmergeを行います。

  手順は第3章の[「Merge 手順」](#merge)の通りです。

  > ![scenario_merge](./image/scenario_merge.jpg)

  画面の[Merge Pull Request]を押下すると資産が統合され、以下のように結果が表示されます。

  > ![scenario_merge_success](./image/scenario_merge_success.jpg)

- シナリオ⑦ 「Merge を契機にwebhook

  MergeされるとJenkinsの[『「Merge用 Pipeline」のScript 記述例』](#script_merge)で作成したPipelineが作動します。

  本ガイドでは「guide_merge」という名で作成し実行しました。  

  > ![scenario_pipeline_merge](./image/scenario_pipeline_merge.jpg)

- シナリオ⑧～⑩

  PullRequestの場合とほぼ同じです。

- シナリオ⑪ CF（本番）へデプロイ

  本ガイドではCF(Staging)デプロイまでをJenkinsで自動化しました。

  公開用サーバであるCF（本番）は安全面等を考慮し手動でデプロイする運用として想定しました。

  以上で想定利用シナリオの 「 チーム開発で静的ウェブページを作成し、公開するまでCIを実施する 」 ことが完了しました。

## おわりに

本ガイドでは「FUJITSU Cloud Service K5」を利用したCI環境の構築について紹介しました。

本ガイドを参考に、プロジェクトの環境や状況にあわせてCI環境をK5上に構築してください。
