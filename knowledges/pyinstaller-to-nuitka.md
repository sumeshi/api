# PyInstallerはキツいのでNuitkaを使う
よく忘れるので備忘録

## Nuitkaとは
[https://nuitka.net/](https://nuitka.net/)

Pythonスクリプトを実行ファイル形式にすることができるソフトウェアです。  
今までPyInstallerというのを使っていたんですが、かなりしんどいので乗り換えました。 

GitHubで公開してるソフトもそのうち切り替えていく予定です。
- [ntfsdump](https://github.com/sumeshi/ntfsdump)
- [ntfsfind](https://github.com/sumeshi/ntfsfind)

~~最初からバイナリ化することを見据えているならそもそもPythonで書かなきゃいいんですが、作りたいツールがいっぱいあるせいで私はついPythonを使ってしまいます。~~

## PyInstallerの問題
下記は私が気になっていた点です

### なんかpathを通すのがうまくいかないことがある
引数でimportのpathを通してあげないと上手くいかないことがよくあります。  
そもそもsrcレイアウトがpythonだと少数派っぽいのであんまり配慮されてないことがよくあるみたいですね。

あと動的リンクされてるライブラリとかを使おうとすると **hidden-import** オプションとかで設定してあげなきゃいけなくて面倒なことが結構ありました。

### 起動時にマルウェア判定されがち
デフォルトで入ってるbootloaderでビルドすると割りと検知されてウザいです。  
ローカルでビルドし直すと直るっぽいんですが、CIでいちいちそんなことやりたくないです。

## Nuitkaつよ
Nuitkaはpythonのソースを一旦C言語にトランスパイルしてビルドしてくれるみたいです。  
そのためPyInstallerと比較して多少パフォーマンスが良いらしいですね。

例えば、**main.py** を実行ファイルにしたい場合、次のようにビルドできます。
```
$ nuitka --follow-imports --onefile main.py
```

CIで実行するときには途中で必要モジュールのダウンロードをして良いか聞かれるので、下記のオプションを有効にしてあげるといいっぽいです。
```
--assume-yes-for-downloads
```

[ntfsdump](https://github.com/sumeshi/ntfsdump) をNuitkaに乗り換えようとしてるんですけどなんかうまくいかなくて頭抱えてます。  
ローカルだとうまくいくのに！なぜ！！

よくわからないのでひとまず後回しにします。

## 参考文献
- [PyInstaller より圧倒的に優れている Nuitka の使い方とハマったポイント](https://blog.tsukumijima.net/article/python-nuitka-usage/)
- [[Python] Pyinstallerで実行ファイルがマルウェアに分類されてしまったときの対策](https://gamingpc.one/dev/python-pyinstaller/)

## おまけ
身も蓋もないけど、最初からNimとかで書けば良い気がする  
実行時コンパイラのない言語なんてサイテー！
