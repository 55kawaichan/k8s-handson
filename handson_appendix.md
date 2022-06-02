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
####  ConfigMapでDDLを参照する

#### initContanerで初期データを配置する

#### 画像ファイルをinit-images-jobを作成してPVに保存する