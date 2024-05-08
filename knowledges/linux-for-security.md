# セキュリティ研究者向けLinuxディストリビューションまとめ
個人的にいい感じだなと思ったやつ。偏見あり。使ってないやつも一応メモ的に。


## ペネトレーションテスト目的
### Kali Linux (オススメ度: ★★★★★)
ペネトレーションテストに特化したDistribution  
前はデスクトップ環境がGnomeかなんかでハチャメチャに重かったんですが、今はかなりいい感じです。

Debianベースなので変な癖もないし、ペネトレ目的でなくてもデフォでだいたいなんでもできます。  
こだわりがなければとりあえずこれだけ使っておけば問題ないです。

[https://www.kali.org/get-kali/](https://www.kali.org/get-kali/)


### Parrot Security OS (オススメ度: ★★★★☆)
Kaliと同じくペネトレーションテストに特化したDistribution  
Kaliが重かった時代はこれをメインに使ってました。見た目もmacOSっぽくてGood

こちらもDebianベース

[https://www.parrotsec.org/](https://www.parrotsec.org/)


### BlackArch Linux (オススメ度: ★★★☆☆)
ペネトレーションに特化したDistribution  
ファイルサイズがデカすぎる。Archベースなので使えるソフトウェアはダントツで多いです。  
KISSの原則に反しているので個人的にはあまり好きではない。  

[https://blackarch.org/](https://blackarch.org/)


## フォレンジック目的
### Tsurugi Linux (オススメ度: ★★★★★)
インシデントレスポンス・デジタルフォレンジックに重きをおいたDistribution  
汎用的に使えるTsurugi Linux [LAB]と、保全に特化したTsurugi Acquireの2種類がある。

名前が日本風でいいよね。[LAB]はツールがいっぱい入ってて結構使いやすいです。  
その分容量が結構デカいのと、ミラーサーバがEU付近のものしかなくて、日本からだとダウンロードに冗談みたいな時間がかかります。

私の作ったツールがデフォルトでインストールされてます。嬉しいね。  
[https://twitter.com/sum3sh1/status/1492030729194438657](https://twitter.com/sum3sh1/status/1492030729194438657)

- [sumeshi/evtx2es](https://github.com/sumeshi/evtx2es)  
- [sumeshi/mft2es](https://github.com/sumeshi/mft2es)  

[https://tsurugi-linux.org/](https://tsurugi-linux.org/)


### CAINE (オススメ度: ★★★★☆)
C.A.IN.E. (Computer Aided Investigative Environment) の通り、PCを対象とした調査支援環境です。  
Live起動して保全する用って感じ。  
[https://www.caine-live.net/](https://www.caine-live.net/)

保全用ドキュメントが公開されたりしていて結構いい感じです。  
[使ってみた](https://sumeshi.github.io/posts/tools/caine)


### PARADIN (オススメ度: ★★★★☆) )
非商用であれば無償で使えます。デザインもかっちょいいし結構いい感じ。  
これもLive起動して保全する用ですね。  
[https://sumuri.com/software/paladin/](https://sumuri.com/software/paladin/)


### SIFT Workstation (オススメ度: ★★☆☆☆)
インシデントレスポンス・デジタルフォレンジックに重きをおいたDistribution  
ダウンロードするのにSANSのアカウントが必要です。

イメージはあんまり更新されないです。  
同ページ内にある「UbuntuにSIFTをインストールする方法」でツール一式ダウンロードしたほうがいいかもね。知らんけど。

[https://www.sans.org/tools/sift-workstation/](https://www.sans.org/tools/sift-workstation/)


### DEFT Linux (オススメ度: -) 
たぶんもう配布されてないです。  
フォレンジック用ツールではよく名前が上がってますが、今ならTsurugiかCAINEでいいんじゃないかなと思います。  


## 匿名活動目的
### Tails (オススメ度: ★★★★★)
匿名性に重きをおいたDistribution

私はTor接続専用マシンとして使っています。  
基本的にメモリ上にしかデータを保管せず、シャットダウン時にすべてのデータを削除する徹底ぶり。すごい。  

デフォルト設定ではそんなにセキュアなわけではないのでちょっと工夫する必要あり。  
悪いことはダメです。ダメゼッタイ。

[https://tails.boum.org/](https://tails.boum.org/)


### Whonix (オススメ度: ★★★★☆)
匿名性に重きをおいたDistribution  
Whonix-GatewayとWhonix-Workstationがセットで配布されてる。

Whonix-Gatewayは、tor接続用のプロキシとして使うことができる。  
WindowsなりUbuntuなり、その端末から出る通信すべてをtor経由にしたい場合に、このOSを通すと便利。

Whonix-Workstationはよくわかりません。  
Gatewayと組み合わせていい感じに使ってねってことなのかもしれないけど、特段使う理由がないし使ってる人も見たことない。

[https://www.whonix.org/](https://www.whonix.org/)

参考資料: [ITショシンシャの苦悩 - Whonix-Gatewayを構築してみた件](https://no-voids.hatenablog.com/entry/20210926)


### Qubes OS (オススメ度: ★★☆☆☆)
何でもかんでもvmでやって環境隔離することでセキュリティを担保しましょうよ、という思想のOS。  
インターネットに接続するのにもそれ用のvmがあってそれと接続して...という感じ。  

しばらく使ってみましたが普段遣いはマジ面倒です。でも本気でセキュリティを考えたいならおすすめ。  
Xenベースでスペックにはかなり余裕をもたせる必要があるので、デスクトップとかで使うのがいいです。

[https://www.qubes-os.org/](https://www.qubes-os.org/)


### Linux Kodachi (オススメ度: -) 
使ったことない。匿名性に重きをおいたOSらしい。  
[https://sourceforge.net/projects/linuxkodachi/](https://sourceforge.net/projects/linuxkodachi/)


### Subgraph OS (オススメ度: -) 
使ったことない。匿名性に重きをおいたOSらしい。  
全然更新されてないけどこれはプロジェクト頓挫してるのか...？  
[https://subgraph.com/](https://subgraph.com/)


## マルウェア解析目的
### REMnux (オススメ度: ★★☆☆☆)
マルウェア解析に重きをおいたDistribution  
アイコンがかっこいいです。あまりこれを使うケースがない。  
[https://remnux.org/](https://remnux.org/)


## 結論
正直Kali Linux 1個あればあんまり困らない。  
あとは保全用のLive起動ディスク1個くらい持ってけばいい気がする。それかWindows FEを作っといたほうがいいかもね。  
