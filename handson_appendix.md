# Appendix

## Deployment
- strategy
- imagePullPolicy
### Self-Healing

### Scaleout
https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/autoscaling.html
- Cluster Autoscaler
- Karpenter

### 初期データの登録方法
- ConfigMapでDDLなどは参照可能とした

- initContanerでpublicディレクトリの内容を配置した（initial.sqlもちょっと大きかったのですが、まぁサンプルということで同様としています）

- 画像ファイルはinit-images-jobを作成してPVに保存したい