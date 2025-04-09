## ✅ Argo CD + Express API チュートリアル

📁 **パス前提**：`~/dev/k8s-ubuntu-kind-api-03-argo-cd`

---

### 📌 チュートリアルの流れ

1. **リポジトリ準備とコードの確認**
2. **Dockerfileの作成とイメージビルド**
3. **ECRへのPush（Dockerイメージ）**
4. **Argo CDのインストールと設定**
5. **Argo CDアプリケーションの作成**
6. **Express API の自動デプロイ**

---

### 🔧 前提環境

- **Ubuntu 22.04 LTS**
- **Docker**, **kind**, **kubectl**, **AWS CLI**, **Argo CD**
- GitHubリポジトリ: `https://github.com/kurosawa-kuro/k8s-api-sample-3000`
- **ECR** にExpress API Docker イメージをプッシュ
- **Kubernetes クラスタ**（kindまたはEKSなど）

---

### 1️⃣ **準備作業: リポジトリの確認とクローン**

まず、APIコードリポジトリをクローンし、コードを確認します。

```bash
# リポジトリのクローン
git clone https://github.com/kurosawa-kuro/k8s-api-sample-3000.git
cd k8s-api-sample-3000

# 必要なパッケージをインストール
npm install
```

---

### 2️⃣ **Dockerイメージの作成とECRへのプッシュ**

#### 2.1 **Dockerfileの確認とイメージビルド**

```bash
# Dockerイメージをビルド
docker build -t k8s-api-sample:latest .
```

#### 2.2 **ECRにログインとリポジトリ作成**

```bash
# AWS CLIでECRにログイン
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 503561449641.dkr.ecr.ap-northeast-1.amazonaws.com

# ECRリポジトリの作成
aws ecr create-repository --repository-name k8s-api-sample --region ap-northeast-1
```

#### 2.3 **イメージタグ付けとECRにプッシュ**

```bash
# イメージにタグ付け
docker tag k8s-api-sample:latest 503561449641.dkr.ecr.ap-northeast-1.amazonaws.com/k8s-api-sample:latest

# イメージをECRにプッシュ
docker push 503561449641.dkr.ecr.ap-northeast-1.amazonaws.com/k8s-api-sample:latest
```

---

### 3️⃣ **Argo CDのインストールと設定**

#### 3.1 **Argo CDのインストール**

```bash
# Argo CDインストール
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# サービスのポートフォワード
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

#### 3.2 **管理者パスワード取得**

```bash
# 初期パスワードの取得
kubectl -n argocd admin initial-password -o jsonpath='{.status.initialPassword}'
```

Web UIにアクセスし、ログインします。

---

### 4️⃣ **Argo CD アプリケーションの作成**

#### 4.1 **GitHubリポジトリの追加**

```bash
# Argo CDにGitリポジトリを追加
argocd repo add https://github.com/kurosawa-kuro/k8s-ubuntu-kind-api-03-argo-cd.git --username <GitHubのユーザー名> --password <GitHubのパスワード>
```

#### 4.2 **アプリケーション作成**

```bash
# アプリケーション作成
argocd app create k8s-api-sample \
  --repo https://github.com/kurosawa-kuro/k8s-ubuntu-kind-api-03-argo-cd.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

---

### 5️⃣ **アプリケーションの同期とデプロイ**

#### 5.1 **同期とデプロイの実行**

```bash
# アプリケーションの同期
argocd app sync k8s-api-sample
```

Web UIまたはCLIで同期状況を確認できます。

---

### 6️⃣ **動作確認**

デプロイ後、Express APIが正常に動作するかを確認します。

```bash
# Podログの確認
kubectl logs -l app=k8s-api-sample

# API確認
curl http://<Ingressの外部URL>/posts
```

正常に動作すれば、レスポンスが返ってきます。

---

### ✅ まとめ

- **GitHubリポジトリ**：コードとDockerfileが含まれているリポジトリを利用
- **ECRにDockerイメージ**：ビルドしたDockerイメージをECRにプッシュ
- **Argo CDの設定**：GitHubリポジトリをArgo CDで管理し、アプリケーションを自動デプロイ
- **動作確認**：APIの動作を確認

これで、Argo CDを使ったExpress APIのデプロイメントが完成です。この流れを繰り返すことで、Argo CDを活用した継続的デリバリーの練習ができます。
