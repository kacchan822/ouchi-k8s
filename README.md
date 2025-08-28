# ouchi-k8s

## 概要

このリポジトリは、おうちk8s（[MicroK8S](https://microk8s.io/)）上で[ArgoCD](https://argo-cd.readthedocs.io/en/stable/)を使ったGitOpsを実現するためのマニフェストを管理するものです。

## 主な機能

このリポジトリを使って、以下のアプリケーションやツールをKubernetesクラスタに導入・管理できます。

*   **ArgoCD**: Gitリポジトリとクラスタの状態を同期させるGitOpsのコアコンポーネントです。このリポジトリではArgoCD自身もArgoCDで管理（Self-Managed）します。
*   **Cert-Manager (設定)**: クラスタのTLS証明書を自動で管理するための設定（`ClusterIssuer`など）をデプロイします。
*   **NFS Subdir External Provisioner**: NFSサーバーを利用して、動的にPersistentVolumeをプロビジョニングするためのストレージプロビジョナーです。
*   **Portainer**: Kubernetesクラスタを管理するためのWeb UIです。
*   **n8n**: ワークフロー自動化ツールです。
*   **PostgreSQL**: n8nのための専用データベースです。

## ディレクトリ構造

```
.
├── LICENSE
├── README.md
├── apps                  # 個別のアプリケーションのArgoCD Applicationマニフェスト
│   ├── cert-manager-config # Cert-Managerの設定（ClusterIssuerなど）
│   ├── n8n.yaml            # n8nのApplication定義
│   └── postgres.yaml       # PostgreSQLのApplication定義
├── argocd                # ArgoCDのApp of Appsパターンのためのルート定義
│   ├── project.yaml      # ArgoCDのProject定義
│   └── root.yaml         # すべてのアプリケーションを束ねるApp of Appsのルート
└── cluster               # クラスタ全体のコンポーネント
    ├── addons            # ArgoCDやPortainerなどのアドオンのApplication定義
    └── core              # ArgoCD自体のインストール用マニフェスト (Kustomize)
```

-   `argocd/`: ArgoCDのApp of Appsパターンを実現するためのルートとなるマニフェストが格納されています。`root.yaml`が全体のエントリーポイントです。
-   `cluster/`: Kubernetesクラスタの基本的なアドオンを管理します。
    -   `core/`: ArgoCD自体のインストール用マニフェストです。
    -   `addons/`: PortainerやNFSプロビジョナーなど、クラスタ全体で利用するアドオンのArgoCD Applicationマニフェストです。
-   `apps/`: n8nやPostgreSQLなど、ユーザーが利用するアプリケーションのArgoCD Applicationマニフェストです。

## セットアップ手順

MicroK8Sのセットアップが完了し、`metallb`と`cert-manager`のプラグインが有効化されている状態からの手順です。

### 前提条件

*   MicroK8Sがインストールされ、クラスタが起動していること。
*   MicroK8Sの以下のプラグインが有効になっていること。
    *   `dns`
    *   `ha-cluster` (シングルノードでも有効化を推奨)
    *   `helm3`
    *   `storage`
    *   `metallb` (IPアドレス範囲を設定済みであること)
    *   `cert-manager`
*   永続化ボリューム(PV)のために、NFSサーバーが利用可能であること。

### 1. リポジトリのフォークとクローン

まず、このリポジトリをご自身のGitHubアカウントにフォークし、ローカル環境にクローンします。

### 2. 設定ファイルの編集

クローンしたリポジトリで、ご自身の環境に合わせていくつかのファイルを編集します。

#### a. ArgoCDの参照元リポジトリの変更

ArgoCDが参照するGitリポジトリのURLを、あなたがフォークしたリポジトリのURLに変更します。

**対象ファイル:** `argocd/root.yaml`

```yaml
# argocd/root.yaml

...
    source:
      # このURLをフォークしたリポジトリのURLに変更
      repoURL: https://github.com/YOUR_USERNAME/ouchi-k8s.git
...
```
※ `cluster-addons` と `apps` の2箇所を変更してください。

#### b. NFSプロビジョナーの設定

NFSサーバーのIPアドレスと共有ディレクトリのパスを、ご自身の環境に合わせて設定します。

**対象ファイル:** `cluster/addons/nfs-client-provisioner.yaml`

```yaml
# cluster/addons/nfs-client-provisioner.yaml

...
    helm:
      values: |
        nfs:
          # ご自身のNFSサーバーのIPアドレスに変更
          server: 192.168.228.250
          # ご自身のNFSサーバーのパスに変更
          path: /volume1/ouchi-k8s-nfs
...
```

### 3. Kubernetes Secretの作成

`n8n`と`PostgreSQL`は、パスワードなどの機密情報をKubernetesのSecretを介して取得します。サンプルファイルを元に、ご自身の環境用のSecretマニフェストを作成し、クラスタに適用してください。

まず、`n8n`用のネームスペースを作成します。
```bash
microk8s kubectl create ns n8n
```

次に、サンプルをコピーしてSecretを作成します。

**PostgreSQL用Secret:**
```bash
# postgres-secrets.yaml.example をコピー
cp apps/postgres-secrets.yaml.example apps/postgres-secrets.yaml
```
`apps/postgres-secrets.yaml` を開き、`YOUR_POSTGRES_PASSWORD` を安全なパスワードに変更してください。

**n8n用Secret:**
```bash
# n8n-secrets.yaml.example をコピー
cp apps/n8n-secrets.yaml.example apps/n8n-secrets.yaml
```
`apps/n8n-secrets.yaml` を開き、`YOUR_ENCRYPTION_KEY` を安全なキー（32文字以上推奨）に変更してください。

作成したSecretをクラスタに適用します。
```bash
microk8s kubectl apply -f apps/postgres-secrets.yaml
microk8s kubectl apply -f apps/n8n-secrets.yaml
```
**注意:** これらのSecretファイルは機密情報を含むため、Gitにはコミットしないでください。（`.gitignore` に追加済みです）

### 4. 変更のコミットとプッシュ

編集したファイルをコミットし、ご自身のリポジトリにプッシュします。

```bash
git add .
git commit -m "Configure for my environment"
git push origin main
```

### 5. ArgoCDのインストール

まず、ArgoCD自体をクラスタにインストールします。

```bash
# ArgoCD用のネームスペースを作成
microk8s kubectl create ns argocd

# Kustomizeを使ってArgoCDのベースマニフェストを適用
microk8s kubectl apply -k cluster/core/argocd/overlays/default
```

### 6. App of Appsの適用

最後に、すべてのアプリケーションを管理するArgoCDのルートアプリケーションを適用します。これにより、ArgoCDがGitリポジトリを監視し始め、自動的にすべてのアプリケーションとアドオンをデプロイします。

```bash
microk8s kubectl apply -f argocd/root.yaml
```

### 7. デプロイの確認

ArgoCDのUIにアクセスして、アプリケーションが同期されていく様子を確認できます。

```bash
# ArgoCDのUIにポートフォワード
microk8s kubectl port-forward -n argocd svc/argocd-server 8080:443

# 初期パスワードを取得
ARGOCD_PASSWORD=$(microk8s kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD password: ${ARGOCD_PASSWORD}"
```
ブラウザで `https://localhost:8080` を開き、ユーザー名 `admin` と上記のパスワードでログインしてください。
