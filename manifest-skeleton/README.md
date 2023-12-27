# rhf2021-cicd-manifest
# !!デモを行う場合必ずリポジトリをFolkすること!!
![folk](https://gitlab.com/jpishikawa/rhf2021-cicd-manifest/-/raw/main/folk_repo.png)

### 環境準備とOperatorのインストール
---
RHPDSにてOpenShift環境を構築  
Operator Hubより"OpenShift Pipelines"と"OpenShift GitOps"をインストール

### デモで利用するリポジトリ
---
本デモでは以下2種類のリポジトリを使用する。
* rhf2021-cicd-manifest: 使用する各種マニフェストファイルを含むリポジトリ  
  https://gitlab.com/jpishikawa/rhf2021-cicd-manifest

* rhf2021-cicd-app: Patient Health Recordsのアプリケーションリポジトリ  
  https://gitlab.com/jpishikawa/rhf2021-cicd-app

### リポジトリのクローン
---
作成したクラスタに対しocコマンドが実行可能な環境で以下を実行。
```
# マニフェストのみ。アプリ側は不要
git clone https://gitlab.com/jpishikawa/rhf2021-cicd-manifest.git
```

### 実行環境の準備
---
```
cd rhf2021-cicd-manifest/

# プロジェクトの作成と変更
oc new-project app-develop
oc label namespace app-develop argocd.argoproj.io/managed-by=openshift-gitops

oc new-project app-staging
oc label namespace app-staging argocd.argoproj.io/managed-by=openshift-gitops

oc new-project app-production
oc label namespace app-production argocd.argoproj.io/managed-by=openshift-gitops

# app-developのコンテナイメージを他のプロジェクトからPull出来るように設定
oc policy add-role-to-user \
    system:image-puller system:serviceaccount:app-staging:default \
    --namespace=app-develop
    
oc policy add-role-to-user \
    system:image-puller system:serviceaccount:app-production:default \
    --namespace=app-develop

# SonarQubeインストール
oc project app-develop
cd tekton/preparations/
oc apply -f sonarqube-install.yaml

# Tekton用PVC作成
oc apply -f tekton-pvc.yaml

# GitLab用Secret作成
export GITLAB_USER=自分のユーザー名
export GITLAB_TOKEN=自分のトークン
cat gitlab-auth.yaml | envsubst | kubectl apply -f -
oc secrets link pipeline gitlab-token

# GitLab用Secret作成（webhook）
export SECRET_TOKEN=Webhook用トークン
oc create secret generic gitlab-webhook --from-literal=secretkey=${SECRET_TOKEN}
```

### Argo CDへのリポジトリ追加
---
```
# ログインパスワードの取得
oc get secret openshift-gitops-cluster -n openshift-gitops -o "jsonpath={.data['admin\.password']}"|base64 --decode
```
RouteでURLを確認し、Argo CDコンソールにアクセス。adminでログイン  
設定画面にてConnect repo using HTTPSを選択し、マニフェストリポジトリの情報と認証情報を追加

### Argo CD CRの作成
---
```
cd ../../argocd/

# AppProjectの作成
cat dev-project.yaml | envsubst | kubectl apply -f -
cat stg-project.yaml | envsubst | kubectl apply -f -
cat prod-project.yaml | envsubst | kubectl apply -f -

# Applicationの作成 (一度CIを実行するまでは内部レジストリにイメージが無いためアプリはデプロイされません)
cat dev-app.yaml | envsubst | kubectl apply -f -
cat stg-app.yaml | envsubst | kubectl apply -f -
cat prod-app.yaml | envsubst | kubectl apply -f -
```

### Tekton Pipelines CRの作成
---
```
cd ../tekton/tasks/

# Taskの作成
oc apply -f .

# Pipelineの作成
cd ../pipelines/
oc apply -f rhf-pipeline.yaml
```

### Tekton Triggers CRの作成
---
```
cd ../triggers/

# Triggers用のSA作成
oc apply -f sample-sa.yaml

# EventListner/TriggerBinding/TriggerTemplate の作成
oc apply -f health-triggerbinding.yaml  
oc apply -f health-triggertemplate.yaml
oc apply -f health-eventlistner.yaml  

# EventListner用Routeの作成
oc apply -f el-route.yaml
```

### GitLab Webhookの設定
---
EventListnerのRouteのURLをコピー

rhf2021-cicd-appリポジトリにて Settings -> Webhook を選択し、上記のURLを設定  
Secret tokenにはWebhook用のSecret作成時に使用した値を入れる

Push eventsにチェックを入れ、対象のブランチをmasterとし設定を保存

作成したWebhookにて、Test -> Push events を実行し、パイプラインが実行されるか確認

### デモでの操作
---
rhf2021-cicd-appリポジトリを開き、
/site/public/index.html を選択し
WebIDEにて74行目以下のコメントアウトを外す
```
# before
      <!--
      <div class="box">
        <div class="map" id='map'></div>
      </div>
      -->

# after
      <div class="box">
        <div class="map" id='map'></div>
      </div>
```

OpenShiftコンソールのパイプライン実行のログからパイプライン実行状況を確認  
scan-appが終わったらSonarQubeにアクセス(admin/admin)し静的診断の結果などを見せる

アプリケーションにログインした際にマップが追加されていることを確認する
