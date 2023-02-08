# セキュリティ研究者向けLinuxディストリビューションまとめ
個人的にいい感じだなと思ったやつ。偏見あり。使ってないやつも一応メモ的に。

## Kali Linux
ペネトレーションテストに特化したDistribution  
前はデスクトップ環境がGnomeかなんかでハチャメチャに重かったんですが、今はかなりいい感じです。

Debianベースなので変な癖もないし、ペネトレ目的でなくてもデフォでだいたいなんでもできます。  
こだわりがなければとりあえずこれだけ使っておけば問題ないです。

[https://www.kali.org/get-kali/](https://www.kali.org/get-kali/)


## Parrot Security OS
Kaliと同じくペネトレーションテストに特化したDistribution  
Kaliが重かった時代はこれをメインに使ってました。見た目もmacOSっぽくてGood

こちらもDebianベース

[https://www.parrotsec.org/](https://www.parrotsec.org/)


## Tails
匿名性に重きをおいたDistribution

私はTor接続専用マシンとして使っています。  
基本的にメモリ上にしかデータを保管せず、シャットダウン時にすべてのデータを削除する徹底ぶり。すごい。  

デフォルト設定ではそんなにセキュアなわけではないのでちょっと工夫する必要あり。  
悪いことはダメです。ダメゼッタイ。

[https://tails.boum.org/](https://tails.boum.org/)


# Whonix
匿名性に重きをおいたDistribution  
Whonix-GatewayとWhonix-Workstationがセットで配布されてる。

Whonix-Gatewayは、tor接続用のプロキシとして使うことができる。  
WindowsなりUbuntuなり、その端末から出る通信すべてをtor経由にしたい場合に、このOSを通すと便利。

Whonix-Workstationは存在理由が不明。  
Gatewayと組み合わせていい感じに使ってねってことなのかもしれないけど、特段使う理由がないし使ってる人も見たことない。

[https://www.whonix.org/](https://www.whonix.org/)

参考資料: [ITショシンシャの苦悩 - Whonix-Gatewayを構築してみた件](https://no-voids.hatenablog.com/entry/20210926)


## Tsurugi Linux
インシデントレスポンス・デジタルフォレンジックに重きをおいたDistribution  

名前が日本風でいいよね。ツールがいっぱい入ってて結構使いやすいです。  
その分容量が結構デカいのと、ミラーサーバがEU付近のものしかなくて、日本からだとダウンロードに冗談みたいな時間がかかります。

私の作ったツールがデフォルトでインストールされてます。嬉しいね。  
[https://twitter.com/sum3sh1/status/1492030729194438657](https://twitter.com/sum3sh1/status/1492030729194438657)

- [sumeshi/evtx2es](https://github.com/sumeshi/evtx2es)  
- [sumeshi/mft2es](https://github.com/sumeshi/mft2es)  

[https://tsurugi-linux.org/](https://tsurugi-linux.org/)


## SIFT Workstation
インシデントレスポンス・デジタルフォレンジックに重きをおいたDistribution  
ダウンロードするのにSANSのアカウントが必要です。

あんまり更新されないし見た目も古臭いので私はそんなに好きじゃないです。  
使い勝手も微妙なので、同ページ内にある「UbuntuにSIFTをインストールする方法」でツール一式ダウンロードしたほうがいいかもね。知らんけど。

[https://www.sans.org/tools/sift-workstation/](https://www.sans.org/tools/sift-workstation/)


## REMnux
マルウェア解析に重きをおいたDistribution  
アイコンがかっこいいです。使い勝手は微妙。

[https://remnux.org/](https://remnux.org/)


## Qubes OS
何でもかんでもvmでやって環境隔離することでセキュリティを担保しましょうよ、という思想のOSっぽい。  
思想はすごく好きだけどUIがダサいので使ったことない。

普段遣いPCで悪いことする人にはかなりよさそう。  
あとスペックにはかなり余裕をもたせる必要がありそうなのでデスクトップとかがいいのかな。

[https://www.qubes-os.org/](https://www.qubes-os.org/)


## Subgraph OS
匿名性に重きをおいたOSらしい。使ったことない。  
全然更新されてないけどこれはプロジェクト頓挫してるのか...？

[https://subgraph.com/](https://subgraph.com/)


## 後で書く
- deft linux
- blackarch linux
- kodachi
