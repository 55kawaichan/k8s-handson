# minikubeでのKubernetes構築

## 事前準備
- 各種ミドルウェアは事前にインストールをお願いします。

| ミドルウェア         | version  | 備考                                              |
| -------------- | -------- | ----------------------------------------------- |
| docker         | 20.10.14 | DockerDesktopをインストールする場合はインストール不要です             |
| docker-compose | 1.29.2   | DockerDesktopをインストールする場合はインストール不要です             |
| [DockerDesktop](https://www.docker.com/products/docker-desktop/)  | 4.7.1    | docker, docker-composeを個別にインストールする場合はインストール不要です |
| [kubectl](https://kubernetes.io/ja/docs/tasks/tools/install-kubectl/)        | 1.22.5   |                                                 |
| [minikube](https://minikube.sigs.k8s.io/docs/start/) | 1.25.2   |                                                 |
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

## 簡単な用語説明
- Cluster
  - Kubernetesのリソースを管理する集合体。リソースはNodeやPodが含まれる。
- Node
  - コンテナをデプロイするためのサーバー
  - 複数NodeをまとめたものがCluster
- Pod
  - 1つ以上のコンテナの集合体
  - Pod単位でコンテナが作成、開始、終了、削除する

## MySQLの起動
まずはMySQLの起動から確認します。

```
# minikube起動
minikube start

# 下記コマンドを実行するとブラウザ上でminikubeの状態を確認できます。
# 実行する場合は、別のターミナルで実行してください。開かない場合はターミナル上のURLをブラウザに入力してください。
minikube dashboard

# MySQL起動
# ※M1 Macの方はmysql-deployment.yamlのmysqlイメージを「mysql/mysql-server:5.6」に変更してください
cd manifest
kubectl apply -f mysql-deployment.yaml
kubectl apply -f mysql-service.yaml

# 起動確認
kubectl get pod

# Podの詳細確認(Podが起動しない場合などに実行します)
kubectl describe pod $(kubectl get pod で取得したNAME)

# log確認(-f を付与することでtailが可能です)
kubectl logs $(kubectl get pod で取得したNAME)

# MySQLのCLUSTER-IP確認
kubectl get svc

# minikubeのワーカーノードに接続
minikube ssh

# MySQLログイン
# windowsの場合
docker run -it --rm mysql mysql -uroot -p --password=password -h $(kubectl get svcで出力されたwordpress-mysqlのCLUSTER-IP)

# M1 Macの場合
docker run -it --rm --platform linux/x86_64 mysql mysql -uroot -p --password=password -h $(kubectl get svcで出力されたwordpress-mysqlのCLUSTER-IP)

# database作成
create database k8s_handson;

# k8s_handsonのDBが存在することを確認
show databases;

# k8s_handsonのDBがどうなるか確認します
# 一旦MySQLのPodを削除・再作成
kubectl delete -f mysql-deployment.yaml
kubectl apply -f mysql-deployment.yaml

# MySQLログイン（前述のコマンド参照）
# k8s_handsonのDBが消えていることを確認
show databases;
```

MySQLのデータはPod上に存在しますが、Podが停止した場合DBのデータも消えてしまうため、PersistentVolumeで永続化します。<br>

```
# 現在のPodを削除
kubectl delete -f mysql-deployment.yaml

# 削除できたか確認
kubectl get pod

# mysql-deployment.yamlのコメントアウトを解除
    #volumeMounts:
    #  - name: mysql-persistent-storage
    #    mountPath: /var/lib/mysql
#volumes:
#  - name: mysql-persistent-storage
#    persistentVolumeClaim:
#      claimName: mysql-pv-claim

# PersistentVolumeClaimの起動
kubectl apply -f mysql-persistentvolumeclaim.yaml

# PersistentVolumeの確認
kubectl get pvc

# MySQL起動
kubectl apply -f mysql-deployment.yaml

# 起動確認
kubectl get pod
```

起動の確認ができたら、一旦Podを削除します。
```
kubectl delete -f mysql-deployment.yaml,mysql-persistentvolumeclaim.yaml,mysql-service.yaml

```


## WordPressの起動

### WordPressのイメージビルド
Dockerfileから自作イメージを作成します。

```
# WordPressのイメージをminikube内でbuildするようにDockerクライアントの向き先（DOCKER ENDPOINT）を変更
docker context ls
eval $(minikube docker-env)

# ルートディレクトリに移動
cd ..

# WordPressのイメージをビルド
docker build -t wp-k8s-handson:minikube .
```

### Volume作成
MySQLとWordPressのPersistentVolumeClaimを作成します。
```
cd manifest
kubectl apply -f mysql-persistentvolumeclaim.yaml
kubectl apply -f wordpress-persistentvolumeclaim.yaml

# 確認
kubectl get pvc
```

### Service作成
MySQLとWordPressが通信できるようにネットワークを作成します。
```
kubectl apply -f mysql-service.yaml
kubectl apply -f wordpress-service.yaml
```

### Pod作成
deploymentリソースからPodを作成します。
```
# MySQL起動
kubectl apply -f mysql-deployment.yaml

# MySQL起動確認
kubectl get pods

# WordPress起動
kubectl apply -f wordpress-deployment.yaml

# WordPress起動確認
kubectl get pods
kubectl logs $(wordpress_pod_name)
```

### WordPressのアドレスを取得
別ターミナルを開き、下記コマンドを実行します。※確認が完了するまではターミナルを閉じないでください。
```
minikube tunnel
```

下記コマンドを実行し、LoadBalancerの EXTERNAL-IP,PORTを確認します。
```
kubectl get svc
        NAME              TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
        kubernetes        ClusterIP      10.96.0.1        <none>        443/TCP        22m
(※ココ！)wordpress         LoadBalancer   10.110.205.210   127.0.0.1     80:30871/TCP   20m
        wordpress-mysql   ClusterIP      None             <none>        3306/TCP       20m

```

取得したアドレスでブラウザからアクセスし、WordPressの初期画面が表示されれば成功です！

ハンズオンが終わったら、minikubeを削除してください。
```
minikube delete
```