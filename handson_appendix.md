# Appendix

## Deployment
- strategy
  - Podの更新戦略を設定します。
    - Recreate
      - 既存の全てのPodは新しいPodが作成されるときに削除されます。
    - RollingUpdate
      - DeploymentはローリングアップデートによりPodを更新します。ローリングアップデートの処理をコントロールするためにmaxUnavailableとmaxSurgeを指定できます。
- imagePullPolicy
  - https://kubernetes.io/ja/docs/concepts/configuration/overview/#%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A%E3%82%A4%E3%83%A1%E3%83%BC%E3%82%B8
### Scaleout
EKSは、2つのオートスケーリング製品をサポートしています。各種デプロイ方法は、以下のAWSドキュメントを参照してください。
https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/autoscaling.html
- Cluster Autoscaler
  - https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md
- Karpenter
  - https://karpenter.sh/v0.10.1/

### 初期データの登録方法
####  ConfigMapでDDLを参照する
ConfigMapとは、設定情報などのKey-Valueで保持できるデータを保存しておくリソースです。Key-Valueといっても、nginx.confなどの設定ファイル自体も保存可能です。

##### 使用方法
deploymentのVolumesにConfigMapを追記します
```yaml
volumes:
  - name: init-sql
    configMap:
      name: sql
```

以下のコマンドを実行し、ConfigMapを適用します。
```
kubectl create configmap sql --from-file=$(DDLを参照したいディレクトリ)
kubectl get configmap sql -o yaml
```

#### Jobリソースを利用して画像ファイルをPersistentVolumeClaimに保存する
コンテナを利用して一度限りの処理を実行させるリソース「Job」のマニフェストファイルを作成し、PersistentVolumeClaimと紐づけることで、初期データ等の登録が可能となります。
https://kubernetes.io/docs/concepts/workloads/controllers/job/

