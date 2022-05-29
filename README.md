# k8sハンズオン
- [こちらのリンク](https://kubernetes.io/ja/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/)をベースにハンズオンを行います。
- 各種ミドルウェアは事前にインストールをお願いします。
  - Docker
  - kubectl
    - [こちらのリンク](https://kubernetes.io/ja/docs/tasks/tools/install-kubectl/)を参考にkubectlをインストールしてください
  - minikube
    - [こちらのリンク](https://minikube.sigs.k8s.io/docs/start/)を参考にminikubeをインストールしてください

## Kubernetesの基礎
Kubernetesは、Kubernetes MasterとKubernetes Node の2種類のノードから成り立っています。

| Node              | 説明                                    |
| ----------------- | ------------------------------------- |
| Kubernetes Master | APIエンドポイントの提供、コンテナのスケジューリング・スケーリングを担う |
| Kubernetes Node   | 実際にコンテナが起動するノード。いわゆるDockerホスト         |

- マニフェストファイル
  - YAML形式が一般的です。
  - kubectlでマニフェストファイルの情報をもとに Kubernetes Masterが持つAPIにリクエストを送り、Kubernetes Masterに「リソース」の登録を行います。
- リソース
  - Kubernetesを操作するために登録します。
  - 今回のハンズオンでは、mysqlとwordpressに対して3種類のリソースに対応したマニフェストファイルを作成します。

| リソース    | 説明 |
| --- | --- |
| Deployment    | コンテナの実行に関するリソース |
| Service    | コンテナを外部公開するようなエンドポイントを提供するリソース |
| PersitentVolumeClaim    | 永続化領域を利用するためのリソース |
- [TODO]構成図を書く

## [TODO]Podやレプリカセット等の説明

## mysqlの起動
チュートリアルのページではMySQLとWordpressのマニフェストファイルを作成し、一気に起動していますが、まずはMySQLの起動から確認します。

```
# minikube起動
minikube start
minikube dashboad

# MySQL起動
cd manifest
kubectl apply -k ./

# 起動確認
kubectl get pods
kubectl logs wordpress-mysql

# ポートフォワード
kubectl port-forward wordpress-mysql(pod id) 13306:3306

# mysqlログイン
 mysql -s -uroot -p -h 127.0.0.1 --port=13306 
```

MySQLのデータはPod上に存在しますが、Podが停止した場合DBのデータも消えてしまうため、PersistentVolumeで永続化します。<br>

```
# manifest/kustomization.yamlのコメントアウトを解除
  #  - mysql-persistentvolumeclaim.yaml

# manifest/mysql-deployment.yamlのコメントアウトを解除
    #volumeMounts:
    #  - name: mysql-persistent-storage
    #    mountPath: /var/lib/mysql
#volumes:
#  - name: mysql-persistent-storage
#    persistentVolumeClaim:
#      claimName: mysql-pv-claim

# 現在のPodを削除
kubectl delete pod wordpress-mysql(pod id)

# 再起動
cd manifest
kubectl apply -k ./

# PersistentVolumeの確認
kubectl get persistentvolume
```

## WordPressの起動
MySQLと同様に、ファイルをmanifestに移動し、以下のコマンドを実行してください。
- wordpress-deployment.yaml
- wordpress-persistentvolumeclaim.yaml
- wordpress-service.yaml
- Dockerfile

```
# manifest/kustomization.yamlのコメントアウトを解除
#  - wordpress-deployment.yaml
#  - wordpress-service.yaml
#  - wordpress-persistentvolumeclaim.yaml

# wordpressのイメージをminikube内でbuildするようにDockerクライアントの向き先（DOCKER ENDPOINT）を変更
docker context ls
eval $(minikube docker-env)

# wordpressのイメージをビルド
docker build -t wordpress-test:minikbe

# WordPress, MySQL起動
cd manifest
kubectl apply -k ./

# 起動確認
kubectl get pods
kubectl logs wordpress

# WordPress Serviceのアドレスを取得
minikube service wordpress --url
```
取得したアドレスでブラウザからアクセスし、WordPressの初期画面が表示されれば成功です！

MySQLとWordPressの接続はそれぞれServiceリソースが行なっています。Serviceのマニフェストファイルの説明をします。

## [TODO]EKSでの起動
[Amazon EKS の開始方法 – AWS Management Console と AWS CLI](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/getting-started-console.html)に沿って構築を行います。

### 事前準備
下記ツールのインストールおよび設定を行います。
- [aws CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [IAM からの必要なアクセス許可](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonelastickubernetesservice.html)
- [Amazon EKS クラスター の IAM ロール作成](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/service_IAM_role.html)

### Amazon EKS クラスターの作成


## [TODO]Appendix
### Self-Healing
### Scaleout
