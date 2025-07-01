# Slurm on GKE

## 1. Cluster Toolkit による環境定義

### 1.1. Cluster Toolkit のインストール

Go 環境がなければ、先に [Go をインストール](https://go.dev/doc/install) します。

その上で

```bash
git clone https://github.com/GoogleCloudPlatform/cluster-toolkit.git
cd cluster-toolkit
make
./gcluster --version
```

### 1.2. テンプレートのダウンロード

```bash
curl -so gke.yaml https://raw.githubusercontent.com/pottava/slonk-on-googlecloud/refs/heads/main/gke/cluster.yaml
```

### 1.3. 設定値の変更

`gke.yaml` ファイルの以下変数を指定、または書き換えてください。

- blueprint_name, deployment_name: あなたのものだとわかる一意の名前をつけてください
- project_id: Google Cloud のプロジェクト ID
- authorized_cidr: 環境構築中のクライアント IP アドレス（例: curl -s httpbin.org/ip | jq -r '.origin' ）

## 2. GKE クラスタの作成

### 2.1. 認証を通す

```bash
gcloud auth login
gcloud auth application-default login
```

プロジェクト ID を環境変数に設定し、CLI のデフォルト プロジェクトとしても指定します。

```bash
export PROJECT_ID=
gcloud config set project "${PROJECT_ID}"
```

### 2.2. ユーザーへの権限設定

管理者の場合は `Kubernetes Engine Admin` ロールが必要になります。

```bash
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member "user:$(gcloud config get-value core/account)" --role "roles/container.admin"
```

一般ユーザーには DNS エンドポイントへの接続許可のため `Kubernetes Engine Developer` ロールが必要です。

```bash
export PROJECT_ID="$( gcloud config get-value project )"
gcloud projects add-iam-policy-binding "${PROJECT_ID}" --member "user:$(gcloud config get-value core/account)" --role "roles/container.developer"
```

### 2.3. 利用サービスの有効化

```bash
gcloud services enable compute.googleapis.com container.googleapis.com servicenetworking.googleapis.com file.googleapis.com
```

## 3. GKE クラスタの作成

### 3.1. クラスタや周辺サービスのセットアップ

Terraform がなければ、先に [Terraform をインストール](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) します。

その上で `gcluster` コマンドで環境を構築しましょう。

```bash
./gcluster deploy gke.yaml
```

### 3.2. GKE クラスタの設定変更

（本来 Terraform / Cluster Toolkit で IaC を徹底すべきですが、一部未対応なものに対してのワークアラウンドです）  
GKE の名前を環境に設定しつつ

```bash
export CLUSTER_NAME=
```

クラスタの接続形式をよりセキュアなものに変更（DNS エンドポイントを有効化）します。

```bash
gcloud container clusters update "${CLUSTER_NAME}" --region "asia-southeast1" --enable-dns-access --no-enable-private-endpoint --no-enable-master-authorized-networks
```

イメージ ストリーミングも有効化しておきましょう。

```bash
gcloud container clusters update "${CLUSTER_NAME}" --region "asia-southeast1" --enable-image-streaming
```

### 3.3. kubectl 構成情報の更新

`gke-gcloud-auth-plugin` がなければツールをインストールします。

https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#install_plugin

```bash
gcloud container clusters get-credentials "${CLUSTER_NAME}" --location "asia-southeast1" --dns-endpoint
kubectl cluster-info
```

### 3.4. テスト実行

```bash
kubectl create -f g2-*/primary/nvidia-smi-job-*.yaml
kubectl create -f g2-*/primary/shared-fs-job-*.yaml
```

実行状況をみて、処理が完了 (Complete) したら

```bash
kubectl get jobs -w
```

結果を確認してみましょう。

```bash
echo -e "\n>>> NVIDIA-SMI\n"
kubectl logs "job/$( kubectl get jobs | grep '^nvidia-smi-job-' | awk '{print $1}' )"
echo -e "\n\n>>> Shared FS\n"
kubectl logs "job/$( kubectl get jobs | grep '^shared-fs-job-' | awk '{print $1}' )"
```

## 4. Slinky のインストール

### 4.1. Slurm のためのリソース設定

名前空間の作成

```bash
git clone https://github.com/pottava/slonk-on-googlecloud.git
kubectl apply -f slonk-on-googlecloud/gke/resources/namespace.yaml
```

Slurm 向け PVC などの作成

```bash
kubectl apply -f slonk-on-googlecloud/gke/resources
```

### 4.2. Slurm Operator インストール

Helm がなければ、先に [Helm をインストール](https://helm.sh/ja/docs/intro/install/) します。

その上で [公式の手順](https://github.com/SlinkyProject/slurm-operator/blob/v0.3.0/docs/quickstart.md) に沿って `Slurm Operator` までをインストールしましょう。

### 4.3. Slurm クラスタのインストール

設定テンプレートをダウンロードし、

```bash
curl -L https://raw.githubusercontent.com/SlinkyProject/slurm-operator/refs/tags/v0.3.0/helm/slurm/values.yaml -o values-slurm.yaml
```

values-slurm.yaml を編集します。

- [L.21](https://github.com/SlinkyProject/slurm-operator/blob/v0.3.0/helm/slurm/values.yaml#L21) nameOverride に GKE クラスタの名前を入れます

ログインノードを有効化し

- [L.310](https://github.com/SlinkyProject/slurm-operator/blob/v0.3.0/helm/slurm/values.yaml#L310) enabled を true に変更
- [L.350](https://github.com/SlinkyProject/slurm-operator/blob/v0.3.0/helm/slurm/values.yaml#L350-L351) rootSshAuthorizedKeys にあなたの公開鍵を指定

NVIDIA の Slurm プラグイン、Pyxis を使えるようにしておきましょう。  
[こちら](https://github.com/SlinkyProject/slurm-operator/blob/v0.3.0/docs/pyxis.md#configure)に沿って設定ファイルを変更します。

そして GPU ノードが Slurm のパーティションとして認識されるよう、以下の設定を `nodesets:` の一つとして [L.487](https://github.com/SlinkyProject/slurm-operator/blob/v0.3.0/helm/slurm/values.yaml#L486) あたりに追加します。

```yaml
# l4 ノードプール用の NodeSet
- name: "l4"
  enabled: true
  # GPU に合わせたコンテナイメージを指定
  image:
    repository: "nvcr.io/nvidia/cuda"
    tag: "12.9.1-cudnn-runtime-ubuntu22.04"
  # GKE ノードのラベルに合致させる
  nodeSelector:
    cloud.google.com/gke-nodepool: "l4"
  # GPU ノードの Taint を許容する
  tolerations:
    - key: "nvidia.com/gpu"
      value: "present"
      operator: "Exists"
      effect: "NoSchedule"
  # コンテナが要求するリソース
  resources:
    limits:
      nvidia.com/gpu: "1"
  # マウントオプション
  volumeClaimTemplates: []
  extraVolumeMounts:
    - name: training-data
      mountPath: /training-data
    - name: shared
      mountPath: /shared
  extraVolumes:
    - name: training-data
      persistentVolumeClaim:
        claimName: slurm-training-pvc
    - name: shared
      persistentVolumeClaim:
        claimName: filestore-pvc
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Retain
  useResourceLimits: true
  partition:
    enabled: true
    config:
      Default: "NO"
      State: "UP"
      MaxTime: "UNLIMITED"
```

設定を環境へ反映します。

```bash
helm install slurm oci://ghcr.io/slinkyproject/charts/slurm --version 0.3.0 --namespace slurm --values values-slurm.yaml
```

`slurm-login-xxx` Pod が起動するのを待ちましょう。

```bash
kubectl --namespace slurm get pods -w
```

### 4.4. SSH 接続

接続先を確認し

```bash
SLURM_LOGIN_IP="$(kubectl get services -n slurm -l app.kubernetes.io/instance=slurm,app.kubernetes.io/name=login -o jsonpath="{.items[0].status.loadBalancer.ingress[0].ip}")"
```

SSH の設定にホストを追加します。

```bash
cat << EOF >> ~/.ssh/config
Host slurm
  HostName ${SLURM_LOGIN_IP}
  User root
  Port 2222
EOF
```

接続してみましょう。

```bash
ssh slurm
```

Slurm の挙動を確認してみます。

```bash
sinfo
srun grep PRETTY /etc/os-release
srun --container-image=alpine:latest grep PRETTY /etc/os-release
```

実際に GPU が利用できるか、または実際の計算が投入できるかなどを試してみてください。

```bash
srun --partition l4 --nodes 1 --gpus 1 nvidia-smi
```

## 5. 環境のシャットダウン

`blueprint_name` を指定して、環境を破棄します。

```bash
./gcluster destroy <your_blueprint_name>
```
