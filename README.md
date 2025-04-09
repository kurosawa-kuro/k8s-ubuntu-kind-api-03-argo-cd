以下に、**先ほどのチュートリアルをリファクタリング（改善）** した形でまとめます。前回の内容を大きく変えず、「ECR 認証の有効期限」「Argo CD 同期操作」「ディレクトリ構造の明示」「長期運用での Docker Registry Secret 使用例」などの改善点を取り込んでいます。  

---

# 🧪 改訂版チュートリアル：EC2 上で最小構成 Express API を Kubernetes（kind）＋ Argo CD でデプロイ

このチュートリアルでは、**EC2上に kind クラスタを構築し、ECR からイメージを Pull して動かす Express API** を **Ingress + Argo CD** でデプロイ・GitOps 化するまでの流れを解説します。  

## 事前の注意点

1. **ECR 認証の有効期限**  
   - ECR からイメージを Pull する際の認証トークンは **最大12時間** 程度で失効します。  
   - チュートリアル内では簡便さを優先して `containerdConfigPatches` に直接トークンを埋め込んでいますが、長期運用する場合は **「Docker Registry Secret」** を利用してイメージPullするほうが推奨されます。後述の [長期運用向け：Docker Registry Secret](#長期運用向けdocker-registry-secret) を参考にしてください。

2. **Argo CD の同期操作**  
   - チュートリアル終盤で Argo CD を用いて GitOps 化します。  
   - 初回のデプロイや設定更新の際、`argocd app sync` や `argocd app get` などCLI操作を行う場合があります。アプリが自動同期設定になっていても、一度は手動で同期させると状況が把握しやすいでしょう。

3. **EC2 セキュリティグループの設定**  
   - ポート **80**（Ingress用）と **3000**（任意でAPIのNodePortなど確認する場合）を開放しておきます。  
   - Argo CDのWeb UIはポートフォワードでアクセスする例を記載していますが、公開したい場合は別途Ingress設定を行う必要があります。

---

## ディレクトリ構成サンプル

参考までに、Gitリポジトリを下記のように構成すると想定しやすいです。

```bash
k8s-ubuntu-kind-api-01-2-basic-ingress-argo-cd/
├── k8s
│   ├── app.yaml         # Argo CD Application (GitOps設定ファイル)
│   ├── deployment.yaml  # Express APIのDeployment
│   ├── service.yaml     # Service
│   └── ingress.yaml     # Ingress
├── kind-cluster.yaml    # kindクラスタ定義(ECRトークン埋め込み)
└── README.md            # チュートリアル内容(本ドキュメント)
```

---

## 0. 前提環境

| 要素         | 内容                                                                 |
|--------------|----------------------------------------------------------------------|
| OS           | Ubuntu 22.04（EC2）                                                 |
| ストレージ   | 最低 30GB（`/var/lib/docker` 領域確保のため）                         |
| 事前準備     | `Docker`, `kind`, `kubectl`, `AWS CLI`, `ECR 認証済み`, `APIイメージPush済み` |
| ECRリポジトリ| `503561449641.dkr.ecr.ap-northeast-1.amazonaws.com/k8s-api-sample`   |
| ポート開放   | EC2 セキュリティグループにて **ポート80, 3000** を開放                 |

---

## 1. kindクラスタ構築（ECR連携込み）

> **注意:** これは短期デモ向けのやり方で、ECRトークンが失効すると Pull に失敗する可能性があります。

```bash
cd ~/dev/k8s-ubuntu-kind-api-01-2-basic-ingress-argo-cd

# ECRトークン取得（12時間ほどで失効します）
ECR_TOKEN=$(aws ecr get-login-password --region ap-northeast-1)

cat <<EOF > kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.auths."503561449641.dkr.ecr.ap-northeast-1.amazonaws.com"]
        username = "AWS"
        password = "$ECR_TOKEN"
EOF

# kindクラスタを作成
kind create cluster --config kind-cluster.yaml
```

---

## 2. Ingress Controller のインストール

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.5/deploy/static/provider/kind/deploy.yaml

# Node に ingress-ready ラベルを付与
kubectl label node kind-control-plane ingress-ready=true

# Pod が Ready になるまで待機
kubectl wait --namespace ingress-nginx \
  --for=condition=Ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=180s
```

---

## 3. Express API 用 マニフェスト適用

あなたのリポジトリに配置した `k8s/` ディレクトリのマニフェストを適用します。

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
```

- `deployment.yaml` 内のイメージ指定が下記のように **ECRリポジトリ** を指していることを確認してください。
  ```yaml
  containers:
  - name: express-api
    image: 503561449641.dkr.ecr.ap-northeast-1.amazonaws.com/k8s-api-sample:latest
  ```

---

## 4. 動作確認（Ingress 経由）

```bash
# ローカル（EC2 インスタンス内）からの場合
curl http://localhost/

# 別のPCからEC2のパブリックIP経由でアクセスするなら
curl http://<EC2のパブリックIP>/

# => {"message":"Hello World!"}

# Pod のログ確認例
kubectl logs -l app=express-api
# => "Express server is running ..."
```

もしこの段階で `ImagePullBackOff` になるようであれば、ECRの認証トークンが切れている可能性があります。クラスタを削除 → トークン再取得 → 再作成すると解消します。

---

## 5. Argo CD インストール（Kindクラスタ内）

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Argo CD Server ポッドが Ready になるまで待機
kubectl wait --namespace argocd \
  --for=condition=Ready pod \
  --selector=app.kubernetes.io/name=argocd-server \
  --timeout=180s
```

---

## 6. Argo CD CLI セットアップ

```bash
VERSION=$(curl --silent "https://api.github.com/repos/argoproj/argo-cd/releases/latest" \
          | grep '"tag_name":' | sed -E 's/.*"v([^"]+)".*/\1/')
curl -sSL -o argocd \
  "https://github.com/argoproj/argo-cd/releases/download/v$VERSION/argocd-linux-amd64"

chmod +x argocd
sudo mv argocd /usr/local/bin/
```

---

## 7. Argo CD ログイン／初期パスワード

```bash
# Argo CD Server へのポートフォワード（別ターミナル推奨）
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 初期パスワードを取得 (adminユーザ)
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d

# 取得したパスワードを使用してCLIログイン
argocd login localhost:8080
```

> ブラウザで Argo CD UI を見たい場合は、`https://localhost:8080` にアクセスします（初回は自己署名証明書の警告あり）。  

---

## 8. Argo CD アプリ定義 & デプロイ

### 8.1 `k8s/app.yaml` (Application 定義)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: express-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: "https://github.com/kurosawa-kuro/k8s-ubuntu-kind-api-01-2-basic-ingress-argo-cd.git"
    targetRevision: HEAD
    path: k8s
  destination:
    server: "https://kubernetes.default.svc"
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 8.2 適用と同期確認

```bash
kubectl apply -f k8s/app.yaml

# Argo CD にアプリが登録されているか確認
argocd app list

# 状態確認
argocd app get express-api

# 必要に応じて手動同期
argocd app sync express-api
```

---

## 9. 動作最終確認

```bash
# 再度Inrgess経由で確認
curl http://<EC2のパブリックIP>/
# => {"message":"Hello World!"}

# Argo CD UI でアプリの状態が「Synced」「Healthy」ならOK
```

---

## 🔒 長期運用向け：Docker Registry Secret

もし短期デモでなく **長期運用** を想定する場合、**ECRトークンの期限問題** を避けるため、Kubernetes の `Secret` を作成し、Deployment がそれをPull Secretとして参照するのが一般的です。  

### 手順の例

1. **ECRトークンを取得**  
   ```bash
   aws ecr get-login-password --region ap-northeast-1 \
   | kubectl create secret docker-registry ecr-regcred \
       --docker-server=503561449641.dkr.ecr.ap-northeast-1.amazonaws.com \
       --docker-username=AWS \
       --docker-password=- \
       --docker-email=none
   ```

2. **Deployment で Pull Secret を指定**  
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: express-api
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: express-api
     template:
       metadata:
         labels:
           app: express-api
       spec:
         imagePullSecrets:
           - name: ecr-regcred        # <--ここでSecretを参照
         containers:
         - name: express-api
           image: 503561449641.dkr.ecr.ap-northeast-1.amazonaws.com/k8s-api-sample:latest
           ports:
             - containerPort: 3000
   ```
3. **定期的に Secret を更新**  
   - トークンは12時間ほどで無効化されるため、サブスクリプトで定期的に再生成するか、`aws ecr get-login-password` を利用した自動更新をCIに仕込むなどで対応可能です。  
   - `Argo CD` 経由での GitOps 運用をするなら、このSecretは `Encrypted Secret` や外部Secret管理ツール（Sealed Secrets, External Secrets Operator など）を併用することも多いです。

---

## ✅ まとめ

- **このリファクタリング版チュートリアル** では、前回の内容に以下の注意点・改善を盛り込みました。  
  - **ECR トークン失効対策の解説**  
  - **Argo CD の手動同期操作例**  
  - **マニフェスト配置ディレクトリ構成例**  
  - **Ingress / EC2 セキュリティグループ周りの確認**  

- 基本的なセットアップフローは変わりませんが、**数日以上継続稼働** させたいなら必ず **Docker Registry Secret** を検討ください。  
- Argo CD の UI と CLI を併用することで、より本格的な GitOps 運用を行えます。

これでより高い再現性と運用しやすさを両立できるはずです。もし他に気になる箇所や追加要望があれば、お気軽にお知らせください。  