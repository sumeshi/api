# pfSenseを使ったProxmoxのVLAN設定
おうちサーバ用に構築したProxmoxのVLAN設定について

## 概要
Proxmox上に立てる仮想マシンのセグメントを分けて管理したい。  
具体的にはこんな感じで。

![](https://i.gyazo.com/ede80b9cf39c61f905adc300e7364234.png)

192.168.0.0/24 配下のProxmoxサーバ上でVLANを使うことで、192.168.10.0/24 と 192.168.20.0/24 の2つ(以上)にネットワークを区切る。  
そうしたい理由としては、仮想マシン側から 192.168.0.0/24 のネットワークにアクセスしてほしくなかったり、192.168.0.10/24 と 192.168.0.20/24 間で通信してほしくないから。

FW設定とか頑張ってもできると思うけど、構成が変わるたびにわちゃわちゃするよりはこっちのが楽だよね。


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

ネットワーク設定が終わったら次のコマンドでpfSenseのWEBUIにアクセスできるようにする。

```
$ pfctl -d
```

FW設定でうまいことアレをあれする

実はまだ途中なので後で書く

参考: https://devblog.lac.co.jp/entry/20220413
