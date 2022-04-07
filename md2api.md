# md2apiについて
Markdownからjsonを生成して, REST APIっぽくGitHub Pagesで公開した備忘録


## 経緯
[sumeshi.github.io](https://sumeshi.github.io) を改装するにあたって, ちょっと変わった作りにしたいなと思ったので


## md2api
https://github.com/sumeshi/md2api

実行したディレクトリ配下のMarkdownファイルをすべてhtmlに変換して, REST APIっぽいjsonを生成しています.  
記事タイトルはファイル名から, 概要はh1タグの直後にあるpタグから, 公開日はgitの最終変更履歴からとっています.

このあたりはMarkdownの冒頭にyamlコードを挿入するか迷ったんですが, ある程度適当に書いてもなんとかなることを優先しました.

変換されたAPIには, 下記のようにアクセスできます.

```
元ファイル: https://github.com/sumeshi/api/md2api.md

記事一覧: https://sumeshi.github.io/api/posts/
記事単体: https://sumeshi.github.io/api/posts/md2api
```


## TODO
今の所記事数が少ないので困っていないのですが, 実行速度がネックになってきたら高速な言語で書き直そうかなと思っています.
そのうち本文を形態素解析して自動タグ付けでもしよっかな〜と考えています.  
