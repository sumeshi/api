# おうちサーバ用PCを自作した
発作的に組みたくなったのでGW中にIntel gen12 CPUとか買って組み立てた備忘録。

## 背景
特に使う予定があるわけではないけど、なんとなくパソコン組もっかなってなったので。

最近はVPSとかクラウドとか全部契約解除しちゃって全く使ってないので、  
時間のかかりそうなジョブを適当に投げて処理させるようなサーバにしようかなと思ってます。


## 構成
次の観点で構成を考えました。
1. ケースの見た目がかっちょいいこと **!important;**
2. 新しめなパーツで組むこと
3. うるさくないこと
4. メンテしなくても大丈夫そうなこと

1, 2から、最近流行ってるコンパクト目なMini-ITXに、  
3, 4から、簡易水冷とかじゃなくて大人しく空冷にしました。

その結果、以下の構成に落ち着きました。

![](https://i.gyazo.com/7f02088e23e7519787c59e1ed8bb6edd.jpg)

|種別|品名|購入場所|金額|
|:-:|:-|:-|-:|
|ケース|Cooler Master/NR200P|Amazon|12,200|
|MB|GIGABYTE/H619I|Amazon|17,440|
|CPU|Intel/Core i3-12100|Amazon|18,480|
|SSD|Crucial/P1 Series 500GB|Amazon|5,999|
|CPUファン|SCYTHE/白虎 弐|秋葉原TSUKUMO|2,886|
|電源|Corsair|秋葉原TSUKUMO|10,980|
|メモリ|Corsair/Vengeance 8GBx2|余ってたマシンから引っこ抜いた|0|

占めて¥67,885-です。


## 感想
最近は自作してもコスパ良くないと思ってましたが、7万以下で組めて非常に満足してます。  
ほんと突発的に組みたくなっただけなので、組み上がってから1週間ずっと放置してます。  

そのうち気が向いたらProxmoxでも入れて仮想化サーバにしようと思ってます。
![](https://www.proxmox.com/en/)

![](https://i.gyazo.com/d31bfcea1bee4616f8d9c9daf388aaa0.jpg)
グラボ挿してないのでスカスカです。もうちょっとちっさいケースでもよかったかな。
