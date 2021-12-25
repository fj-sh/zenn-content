---
title: "Azure で Kubernetes を動かす｜まずはログインまで"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Azure]
published: false
---

## 開発環境の準備

### Azure CLI コマンドをインストール

```console
$ brew update && brew install azure-cli
```

### Azure にサインイン

```console
$ az login
```

ブラウザが開かれるので、ログイン。

### kubectl コマンドのインストール

kubectl コマンドは、Kubernetes クラスターの状態を確認したり、構成を変更するためのもの。

```console
$ brew install kubernetes-cli
```

### az コマンドと kubectl コマンドの違い

Kubernetes クラスター内で動くコンテナーアプリケーションを操作するのが kubectl コマンド。
Azure を使って Kubernetes クラスターそのものの構築や削除などを行うのが az コマンド。

## コンテナーイメージのビルド

Azure Container Registry は Azure が提供するコンテナーイメージの共有サービス。

ACR を使ってコンテナーイメージのビルドと共有を行う。

### レジストリの作成

レジストリ名が利用可能化調べる。

```console
$ az acr check-name -n myFirstACRRegistry
```

レジストリ名をシェル環境変数 `ACR_NAME` に設定する。

```console
$ ACR_NAME=myFirstACRRegistry
```

コンテナーレジストリを作成する、Azure のリソースグループ名を設定する。

```console
$ ACR_RES_GROUP=$ACR_NAME
```

リソースグループを作成する。

```console
$ az group create --resource-group $ACR_RES_GROUP --location japaneast
```

ACR のレジストリを作成する。

```console
az acr create --resource-group $ACR_RES_GROUP --name $ACR_NAME --sku Standard --location japaneast
```

`Dockerfile` があるディレクトリを指定してイメージをビルドする

```console
$ az acr build --registry $ACR_NAME --image image-name:v1.0 directory-name/
```

イメージを確認する。

```console
az acr repository show-tags -n $ACR_NAME --repository image-name
```

## Azure で Kubernetes クラスターを作成する

ACR で作成したコンテナーイメージは AKS で作成した Kubernetes クラスター上で pull して動かす。

ACR と AKS 間で認証を行う。

ACR のリソース ID を取得する。

```console
$ ACR_ID=$(az acr show --name $ACR_NAME --query id --output tsv)
```

サービスプリンシパル名をシェル環境変数 `SP_NAME` に設定する。

```console
$ SP_NAME=sample-acr-service-principal1130
$ SP_PASSWORD=$(az ad sp create-for-rbac --name $SP_NAME --role Reader --scopes $ACR_ID --query password --output tsv)
$ APP_ID=xxxx-xxxx-xxxx
```

↑ の `APP_ID` で指定している ID は Azure ポータルの ホーム > アプリの登録 のアプリケーション ID。

クラスターを作成する。

```console
$ AKS_CLUSTER_NAME=AKSCluster
$ AKS_RES_GROUP=$AKS_CLUSTER_NAME
$ az group create --resource-group $AKS_RES_GROUP --location japaneast
```

`$ az aks get-versions -l japaneast` で利用できる Kubernetes バージョンを確認し、 AKS で Kubernetes クラスターを作成する。

```console
$ az aks create \
--name $AKS_CLUSTER_NAME \
--resource-group $AKS_RES_GROUP \
--node-count 1 \
--kubernetes-version 1.19.11 \
--node-vm-size Standard_DS1_v2 \
--generate-ssh-keys \
--service-principal $APP_ID \
--client-secret $SP_PASSWORD
```
