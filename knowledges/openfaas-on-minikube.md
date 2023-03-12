# MinikubeにOpenFaaSをデプロイする
Ubuntu20.04, OpenFaaS使ってみたかったので。

## 概要
おうちサーバでFaaSして～って思ってたけど案外選択肢が少ない。気がする。  
思いつくのは下記のツール類。

1. [OpenFaaS](https://www.openfaas.com/)
2. [LocalStack](https://localstack.cloud/)
3. [Knative](https://knative.dev/docs/)
4. [serverless framework](https://github.com/serverless/serverless)

なんか1が一番シンプルそう。  
あんまり複雑な構成にするとWebサーバ立てたほうが楽そうで本末転倒なので。


## 環境
Proxmox上に構築したUbuntu 20.04のVM  
CPUは2Core, RAM2GBを割り当てました。

前提条件としてdockerをインストール、一般ユーザ権限で起動できるようにしてあります。


## 手順
[minikube start](https://minikube.sigs.k8s.io/docs/start/)

Installation - Linux, x86-64, Stable, Debian packageに沿って作業します。

### kubectlのインストール
```.bash
sudo snap install kubectl --classic
```

### Minikubeのインストール
```.bash
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
$ sudo dpkg -i minikube_latest_amd64.deb

$ minikube start
$ minikube status
```

### OpenFaaSのnamespaceを設定
```.bash
$ git clone https://github.com/openfaas/faas-netes
$ cd faas-netes
$ kubectl apply -f namespaces.yml
$ kubectl get namespaces
```

### OpenFaaSにBasic認証を設定
```.bash
$ kubectl -n openfaas create secret generic basic-auth --from-literal=basic-auth-user={{ユーザ名}} --from-literal=basic-auth-password={{パスワード}}
```

### OpenFaaSのデプロイ
```.bash
$ kubectl apply -f ./yaml/
```

### minikubeのポートフォワーディング
minikubeのIPを確認
```.bash
$ minikube ip
{{IPアドレス}}
```

ホストの80番ポートへのアクセスをminikubeの31112ポートへフォワーディングする
```.bash
$ sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination {{IPアドレス}}:31112
$ sudo iptables -t nat -A POSTROUTING -p tcp -d {{IPアドレス}} --dport 31112 -j MASQUERADE
$ sudo iptables -A FORWARD -p tcp -d {{IPアドレス}} --dport 31112 -j ACCEPT
```

## ダッシュボードの確認
http://{{ホストのIPアドレス}}:80 にアクセスする


## 参考
- [OpenFaas を Kubernetes 上にデプロイする](https://al-batross.net/2021/04/06/openfaas-deploy-on-kubernetes/)
