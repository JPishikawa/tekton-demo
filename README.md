###### tags: `Tekton`
# OpenShift Pipelines demo

## 環境準備とOperatorのインストール
* RHPDSにてOpenShift環境を構築
* 上記実施後にOperator Hubより"OpenShift Pipelines"をインストール
* GitHubとDockerHubを扱うためアカウントが無い場合事前に作成

## リポジトリの取得
本デモでは以下2種類のリポジトリを使用する。
**tekton-demo**: 使用するtektonのCRを含むリポジトリ
https://github.com/JPishikawa/tekton-demo

**mkdocs-test**: pipelineの実行契機となるリポジトリ※
https://github.com/JPishikawa/mkdocs_test
※中身は何でも良いため上記以外のリポジトリでも問題なし

### tekton-demoリポジトリのクローン
作成したクラスタに対し、ocコマンドが実行可能な環境で行うことを推奨。
```
git clone https://github.com/JPishikawa/tekton-demo.git
```

### mkdocs-testリポジトリのフォーク
pushやmergeを行うため、デモ専用のリポジトリを作成しておく。
自身のGitHubアカウントにログインしておき、以下にアクセス。
https://github.com/JPishikawa/mkdocs_test
画面右上のForkを押下。
Fork後に以下のように自身のアカウント配下にmkdocs-testリポジトリがあることを確認。
/[個人のアカウント名]/mkdocs-test
 
## DockerHubで新規リポジトリを作成
画面右上のCreate Repositoryから新規のリポジトリを作成。（Private推奨）

## プロジェクトの作成と環境構築

### デモで使用するプロジェクトの作成
```
oc new-project demo-cicd
```
### SonarQubeのインストール
```
oc apply -f ./preparation/sonarqube-install.yaml
```

### DockerHub認証情報のSecretを作成
自身のDockerHubの認証情報をBase64でエンコード
```
echo -n "id:password" | base64
```
config.jsonを作成(authの値を上記の出力結果に変更)
```
{
	"auths": {
		"https://index.docker.io/v2/": {
			"auth": "xxxxx"
		}
	}
}
```
上記ファイルを元にSecretを作成
```
oc create secret generic dockerhub-cred \
    --from-file=.dockerconfigjson=./config.json \ 
    --type=kubernetes.io/dockerconfigjson
```
ServiceAccount pipelineにSecretを紐付け
```
oc secrets link pipeline dockerhub-cred
```

### GitリポジトリからのWebhook認証用Secretを作成
後ほど作成するWebhook用にSecretを作成
```
oc create secret generic github-webhook --from-literal=secretkey=tekton-demo
```

### tkn CLIのインストール（本デモではオプション）
Linuxの場合は以下。その他は[こちら](https://github.com/tektoncd/cli)を参照
```
# Get the tar.xz
curl -LO https://github.com/tektoncd/cli/releases/download/v0.20.0/tkn_0.20.0_Linux_x86_64.tar.gz
# Extract tkn to your PATH (e.g. /usr/local/bin)
sudo tar xvzf tkn_0.20.0_Linux_x86_64.tar.gz -C /usr/local/bin/ tkn
```

## デモ
### Tekton Pipelines
#### TaskRun
あらかじめ登録されているClusterTaskに対し、TaskRunを作成し正しく実行されるか確認する。
```
cd ./tekton-pipelines/

# git clone元レポジトリの変更
sed -i -e "s%https://github.com/JPishikawa/mkdocs_test.git%https://github.com/[ユーザー名]/mkdocs_test.git%g" gitclone-taskrun.yaml

# TaskRunリソースの作成
oc create -f gitclone-taskrun.yaml

# 実行状況の確認
oc get taskrun
```
適宜OpenShiftコンソールで実行状況を確認


#### PipelineRun
イベント契機ではなく手動でPipelineRunリソースを作成することでPipelineを実行。
```
# SonarSqan Taskのインストール
oc apply -f sonarqube-scanner-task.yaml

## tknコマンドの場合
# tkn hub install task sonarqube-scanner --version 0.1

# 実行するPipelineの作成
oc apply -f test-pipeline.yaml

# PipelineRunリソースの作成
oc create -f test-pipelinerun.yaml

# 実行状況の確認
oc get taskrun
```
適宜OpenShiftコンソールで実行状況を確認

### Tekton Triggers
#### 各種リソースの作成
```
cd ../tekton-triggers

# コンテナイメージ保存先を変更
sed -i -e "s%jpishikawa/mkdocs-test%[ユーザー名]/[リポジトリ名]%g" test-triggertemplate.yaml

oc apply -f .
```
#### Webhookエンドポイントの作成
```
# EventListnerのService名を確認
oc get service

# Routeの作成
oc expose service [EventListnerService]

# エンドポイントを確認
oc get eventlistner
```

#### Webhookの設定
最初にフォークしたリポジトリにアクセスし、"Settings"→"Webhooks"から新規のWebhookを設定。
Payload URLには先ほど作成したWebhookのエンドポイントのURLを設定、
Content typeをapplication/json 、
Secretには認証に使用するシークレットキー（tekton-demo）を入力

設定が完了したら"Add webhook"ボタンを押下。

#### PullRequestの作成とマージ
developブランチにて適当なファイル（test、等）を新規追加しcommitする。
developからmasterへのPullRequestを作成し、merge。

#### 実行状況の確認
ここまでの操作が正しく実施出来ていればWebhookを契機とした新たなPipelineRunが作成されているはず。
適宜実行状況を確認し、完了したらDockerHubにてGitのcommitIDがタグとして付与されたイメージを確認する。

## Clean Up
フォークしたGitリポジトリやDockerHubのリポジトリは適宜削除
