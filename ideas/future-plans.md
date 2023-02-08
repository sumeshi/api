# 今後の方針
今後やりたいこととか。随時更新

## このサイトについて
雑多に色々情報をメモしていこうかなと思っています。  
QiitaとかZennとかNoteとかあんまし肌に合わなかったのである程度情報が溜まったら [dev.to](https://dev.to/sumeshi) にでも公開しよっかな。英語の勉強も兼ねて。

セキュリティ関連の自分用メモとか、趣味開発についてのメモを書いていく予定です。  
このサイト自体が自分の作ったツールの実験場なので、そのうちブラッシュアップして公開できたら嬉しいかな。今のとこ [md2api](https://sumeshi.github.io/posts/md2api) くらいしかまともなツールないけど。

## 今取り組んでること
### sumeshi.github.io の開発
ちまちま作ってるけどあんまり進んでないかも。  
とりあえず色々思いついたことを文章にして思考を整理しています。

~~あとなぜかsitemap.xmlがGoogle様に認識されない。~~  
なんか原因わかりました。lastmodの項目、ISOフォーマットなら大丈夫って聞いてたのにダメらしい。  
次のようなフォーマットにするといいらしいです。クソボケがよ

```
2022-01-01
```

### 公開してるツール群のメンテ
\*2esとか、ntfs\*とか  
動作条件Python3.7以上とかにしてたけど、後方互換性保つと新しい文法使えないしもういいよね。  
古いバージョン使いたい人はフォークしてるだろうから知りません。近く3.9以上にすると思います。  

### 簡易全文検索エンジンの開発
Elasticsearchの簡易版っぽいのを目指してます。

もともと学部生のときに自然言語処理とかやってたこともあって、  
多少ノウハウがあるのでやってみよっかなって感じです。

[sumeshi.github.io](https://sumeshi.github.io) の検索機能を実現するついでに機能自体を独立させてライブラリにしちゃおって思ってます。

### ファストフォレンジック用ツールの開発
[GitHub](https://github.com/sumeshi) で公開してるツール群の機能を統合して可視化までやってくれるような感じのを考えてます。
大学院のときにやってた研究を個人の趣味として完成させてみたいなと思って考えてますが、いろんなことやってるせいでちっとも進んでないです。

### ログ可視化ツールの開発
ファストフォレンジック用ツールの開発にあたって、ログを視覚的にわかりやすく表示したいので [d3.js](https://d3js.org/) あたりで作ろうかなと思ってます。
イメージとしては [mermaid.js](https://mermaid-js.github.io/mermaid/#/) っぽいやつを。

### マルウェア解析/インシデント分析プラットフォームの構築
模擬ネットワークっぽいものを作って、その上で悪いプログラムとか挙動を分析できるようにしたいなと考えてます。  
サンドボックスの複数環境版的な。

[AutomatedLab](https://automatedlab.org/en/latest/) 使いたいからという気持ちが強いです。  
まだ何も進んでないけど。とりあえず[基盤として使う予定のサーバは組みました](https://sumeshi.github.io/posts/ideas/homebuilt-pc-2022-05)。

### NASのお掃除プログラムの開発
一応家にNASがあるんですが、今の所データの放り込み先ぐらいにしか活用できてません。  
最近のやつだとDockerでコンテナとか立てられるらしいけど、そんなに高性能でもないのでどうしようかなぁって考えてます。

大量に保存してる写真とか動画とか音楽とか一時的にローカルに保存した論文のPDFとか無料配布されてた本とかたくさんあるので始末に負えません。  
何が入ってるかわかんない状態でインターネットに出すのは怖いので、ローカルにGitLabでも立ててデータ整理用スクリプトの管理とCIによる実行とかしようかなって考えてます。  

GiteaとDroneの組み合わせがナウでヤングな感じらしいので、そっちにするかも。go言語製で軽いし。  
一人で使うにはGitLabはオーバースペックすぎるのもありますしね。

### クローラの開発
Slackとかにセキュリティ関連のニュース記事を投げるクローラでも作ろうかなと思っています。  
NASのお掃除プログラムのところにも書いたように、クローリングスクリプト+CIで定期実行させておけばいいかなぁと。

セキュリティのニュースをまとめてるところって結構少なくて方方を回るのが面倒なんですよね。  
~~規約的にクローリングを禁止してるサイトもあるけど、だったらWeb APIなりRSSを公開しろよクソボケと毎日思っています。~~

うまくいったら公開するかもね。

## やりたいこと
色々あるけど、最近Pythonばっか書いててつまらないのでRustとかでガッツリ開発したいと考えてます。
あと、プログラミングとかのコミュニティ的なものには属してないんですが、一緒に開発できるメンバーでも探そうかなと思ってます。モチベ高めの人で。

だいたいそんな感じです。