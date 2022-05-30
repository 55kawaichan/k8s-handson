# EKSでのKubernetes構築
[Amazon EKS の開始方法 – AWS Management Console と AWS CLI](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/getting-started-console.html)に沿って構築を行います。構築にはマネジメントコンソールとeksctlの2通りの方法がありますが、今回は段階を踏んで構築したいため、マネジメントコンソールから作成します。

### 事前準備
下記ツールのインストールおよび設定を行います。
- [aws CLI](https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/getting-started-install.html)
- [Amazon EKS クラスター の IAM ロール作成](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/service_IAM_role.html)

### Amazon EKS クラスターの作成
- CloudFromationから、VPCを作成します。
  - テンプレートURL：https://amazon-eks.s3.us-west-2.amazonaws.com/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml
  - stack名：my-eks-vpc-stack
  - リージョン：ap-northeast-1
- [Amazon EKSコンソール](https://ap-northeast-1.console.aws.amazon.com/eks/home?region=ap-northeast-1#/clusters)を開きます。
  - クラスターを追加→作成をクリックします。
  - クラスター設定
    - クラスター名を「my-cluster」とし、クラスターサービスロールに「Amazon EKS クラスター の IAM ロール作成」で作成したロール名が設定されていることを確認します。
  - ネットワーキングを指定
    - VPCが作成したVPCであることを確認します。
    - クラスターエンドポイントアクセスを「パブリックおよびプライベート」に変更します。
    - その他設定はデフォルトとします。
  - 適宜ログ設定等を行い作成します。（作成には10分ほどかかります）
- kubectlが作成したクラスターに接続できるように設定します。
  ```
  # kubeconfigに作成したクラスター情報を追加
  aws eks update-kubeconfig --region ap-northeast-1 --name my-cluster
  # 設置ができたか確認
  kubectl get svc
  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
  kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   10m
  ```
  - ※minikube-eksとの切り替えは以下のコマンドで行います。
  ```
  # kubectlで選択できる向き先の一覧
  kubectl config get-contexts
  # 切り替え
  kubectl config use-context $CLUSTER_NAME
  ```
### マネージド型ノードの作成
#### Pod実行ロールの作成
- [IAMロール](https://us-east-1.console.aws.amazon.com/iamv2/home#/roles)より、ロールを作成をクリックします。
- カスタム信頼ポリシーを選択し、[手順](https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/getting-started-console.html#:~:text=Fargate%20%E2%80%93%20Linux-,Managed,-nodes%20%E2%80%93%20Linux)のjsonをコピペします。
- アタッチするポリシーを選択します。
  - AmazonEKSWorkerNodePolicy
  - AmazonEC2ContainerRegistryReadOnly
  - AmazonEKS_CNI_Policy
- ロール名を「myAmazonEKSNodeRole」とし、ロールを作成します。

#### ノード作成
- [Amazon EKS コンソール](https://console.aws.amazon.com/eks/home#/clusters)を開き、作成したクラスターを選択します。
- [設定] タブ - [Compute] (コンピューティング) タブ - ノードグループを追加 をクリックします。
- ノードグループの設定 で、各項目を入力します。
  - 名前：my-nodegroup
  - ノードIAMロール：myAmazonEKSNodeRole（ポッド実行ロールの作成で作成したロール）
  - ノードグループのコンピューティング設定
    - ディスクサイズ：60GiB
- 以降の項目はデフォルトでノードを作成します。

## アプリケーションのデプロイ
### Volumeの作成
- マニフェストからPersistent Volume Claimを作成します。
```
# pvcの作成
kubectl apply -f mysql-persistentvolumeclaim.yaml,wordpress-persistentvolumeclaim.yaml

# ステータスの確認
kubectl get pvc
```
### ECRへのDockerImage登録
wordpressのイメージは自作イメージ想定なので、DockerfileよりイメージをビルドしECRにプッシュします。EKSではECRに登録したwordpressのイメージを参照します。
- ECRのコンソールを開き、リポジトリを作成します。
- ページ上部の「プッシュコマンドを表示」から、イメージをプッシュします。
```
# ECRにログイン
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin $(アカウント情報)

# Dockerビルド
docker build -t wp-k8s-handson:minikube .

# TAG作成
docker tag wp-k8s-handson:minikube $(アカウント情報)/wp-k8s-handson:minikube

# イメージのプッシュ
docker push $(アカウント情報)/wp-k8s-handson:minikube
```

### Serviceの作成
mysql-service.yamlのclusterIP: none をコメントアウトし、コマンドを実行します。
```
kubectl apply -f mysql-service.yaml,wordpress-service.yaml
```

### Pod作成
deploymentリソースからPodを作成します。wordpress-deployment.yaml のimageはECRに登録したものに変更しておきます。
```
kubectl apply -f mysql-deployment.yaml
kubectl apply -f wordpress-deployment.yaml
# 確認
kubectl get deployment
kubectl get pvc
kubectl get svc
```
- EC2のコンソールを開き、各種設定がなされているか確認してみましょう。
- kubectl get svc で表示されるEXTENAL IPにWebブラウザでアクセスし、WordPressの初期画面が表示されれば成功です。

※セキュリティグループの設定でアクセス制限を行なってください。
