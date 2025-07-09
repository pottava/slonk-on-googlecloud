# Slurm on GCE

作業される方には以下の権限が設定されていることを確認してください。

- IAM 設定権限 iam.serviceAccounts.setIamPolicy
- IAP で保護されたトンネル ユーザー roles/iap.tunnelResourceAccessor
- Service Networking Admin roles/servicenetworking.networksAdmin

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
git clone https://github.com/pottava/slonk-on-googlecloud.git
```

### 1.3. 設定値の変更

`slonk-on-googlecloud/gce/a3mega-deployment.yaml` ファイルの以下変数を指定、または書き換えてください。

- bucket: Terraform の状態を管理するためのバケットを指定してください
- deployment_name: IaC ファイルを格納するフォルダ名として一意となる名前をつけてください
- project_id: Google Cloud のプロジェクト ID
- network_name_system, subnetwork_name_system: 必要あればネットワーク名を変更してください
- slurm_cluster_name: 作成するクラスタごとに一意となる名前をつけてください
- a3mega_reservation_name: 予約名を指定してください
- a3mega_cluster_size: 常時起動する A3 Mega VM の数を指定してください

## 2. GKE クラスタの作成

### 2.1. 認証を通す

```bash
gcloud auth login
gcloud auth application-default login
```

プロジェクト ID を環境変数に設定し

```bash
export PROJECT_ID=
```

CLI のデフォルト プロジェクトとして指定します。

```bash
gcloud config set project "${PROJECT_ID}"
```

### 2.2. 利用サービスの有効化

```bash
gcloud services enable compute.googleapis.com servicenetworking.googleapis.com storage.googleapis.com file.googleapis.com
```

## 3. Slurm クラスタの作成

### 3.1. バケットの作成

Terraform の状態をリモートで保存するためのバケットを作成します。任意の名前をつけて

```bash
export BUCKET_NAME=
```

バケットを作成します。

```bash
gcloud storage buckets create "gs://${BUCKET_NAME}" --default-storage-class STANDARD --location asia-northeast1 --uniform-bucket-level-access
gcloud storage buckets update "gs://${BUCKET_NAME}" --versioning
```

### 3.2. クラスタや周辺サービスのセットアップ

Terraform がなければ、先に [Terraform をインストール](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) します。

Packer がなければ、先に [Packer をインストール](https://developer.hashicorp.com/packer/tutorials/docker-get-started/get-started-install-cli) します。

その上で `gcluster` コマンドで環境を構築しましょう。

```bash
./gcluster deploy -d slonk-on-googlecloud/gce/a3mega-deployment.yaml slonk-on-googlecloud/gce/a3mega-blueprint.yaml --auto-approve
```

もし後からコンピューティング ノードの数を増やしたり、クラスターに新しいパーティションを追加したりする場合は、以下のコマンドでクラスタを差分更新できます。

```bash
./gcluster deploy -d slonk-on-googlecloud/gce/a3mega-deployment.yaml slonk-on-googlecloud/gce/a3mega-blueprint.yaml --only primary,cluster --auto-approve -w
```

### 3.3. SSH 接続

接続先を確認し

```bash
gcloud compute ssh $( gcloud compute instances list --filter "name ~ login" --format "value(name)" ) --zone asia-northeast1-b --tunnel-through-iap
```

Slurm の挙動を確認してみます。

```bash
sinfo
srun -p debug df -h
```

実際に GPU が利用できるか、または実際の計算が投入できるかなどを試してみてください。

```bash
srun -N 1 enroot import docker://nvcr.io#nvidia/pytorch:24.04-py3
```

## 5. 環境のシャットダウン

`blueprint_name` を指定して、環境を破棄します。

```bash
./gcluster destroy <your_blueprint_name> --auto-approve
```
