# pfSenseを使ったProxmoxのVLAN設定
おうちサーバ用に構築したProxmoxのVLAN設定について

## 概要
Proxmox上に立てる仮想マシンのセグメントを分けて管理したい。  
具体的にはこんな感じで。

![](https://i.gyazo.com/ede80b9cf39c61f905adc300e7364234.png)

192.168.0.0/24 配下のProxmoxサーバ上でVLANを使うことで、192.168.10.0/24 と 192.168.20.0/24 の2つ(以上)にネットワークを区切る。  
そうしたい理由としては、仮想マシン側から 192.168.0.0/24 のネットワークにアクセスしてほしくなかったり、192.168.0.10/24 と 192.168.0.20/24 間で通信してほしくないから。


## 手順
大きく分けると次のような感じ。

1. Proxmox側のネットワーク設定
2. pfSenseのインストール
3. pfSenseの設定
4. 仮想マシン側の設定

### 1. Proxmox側のネットワーク設定
- サイドバーから Datacenter -> pve をクリック
- pveのサイドバーから System -> Networkをクリック

デフォで設定されてるvmbr0というのがあると思うので、下記のように設定する。VLAN-awareがキモ。  
IPアドレスとかNetwork Deviceは適当に読み替えること。

![image](https://user-images.githubusercontent.com/35072092/179372959-8e638aa8-4224-4167-81d9-51e0249fa827.png)

### 2. pfSenseのインストール
https://www.pfsense.org/

公式サイトからISOが落とせるので、Proxmoxにダウンロードして仮想マシンを立てる。スペックは適当。  
起動すると色々聞かれるけど、基本的にはいはい言ってればよい。

### 3. pfSenseの設定
インストール後、rebootが終わると初期ネットワーク設定を聞かれる。  
今回は次の3セグメントを設定した。

WAN: 192.168.0.0/24  
LAN1: 192.168.0.10/24
LAN2: 192.168.0.20/24

ネットワーク設定が終わったら次のコマンドで一旦pfSenseのFW設定を無効化する。  
```
$ pfctl -d
```

そうするとWEBUIにアクセスできるので、よしなにFW設定をする。  
とりあえず最低限セグメント間のアクセスをBlockにしておけばOK。

```
LAN1:
  Source: LAN1 net -> Destination: WAN net
  Source: LAN1 net -> Destination: LAN2 net

LAN2:
  Source: LAN2 net -> Destination: WAN net
  Source: LAN2 net -> Destination: LAN1 net
```

設定が終わったらApply Configurationみたいなボタンを押すと設定が反映される。  
必要に応じてWEBUIへのアクセスを許可しておくこと。

参考: https://devblog.lac.co.jp/entry/20220413


### 4. VMの追加
ここまでの設定が終われば、あとは各セグメントにVMを追加していけばよい。  
ネットワークアダプタの設定時に VLAN Tag を設定してあげれば自動でそのセグメントに設定ができる。

![image](https://user-images.githubusercontent.com/35072092/183535336-9538c08f-fc48-4ccf-bcab-b3214c01c7f3.png)

上記の画像は192.168.20.0/24配下に追加したときの例。


### 所見
VirtualBoxみたいな仮想SWとか追加して楽にできるかな～と思ったけど全然そんなことなかった。  
何年か前にESXi+vyosでやったときも同じようなことした記憶があるので、ネットワークちゃんと勉強しないといけなさそう。

pfSenseで一台仮想マシン専有してるとかちょっとアホっぽいので物理SWに変えるとか、なんか考えようかな。

お わ り
