# CAINE Linux
C.A.IN.E. (Computer Aided Investigative Environment)

## 概要
Computer Aided Investigative Environment の通り、PCを対象とした調査支援環境です。  
フォレンジック用のLinuxはいくつかありますが、明確にそれを目的として活発に開発が続いているのは現在これくらいかなと思います。  
[https://www.caine-live.net/](https://www.caine-live.net/)

この手のセキュリティ系LinuxはDebianを元にしているものが多いですが、CAINEはUbuntuベースです。  

関連記事: [セキュリティ研究者向けLinuxディストリビューションまとめ](https://sumeshi.github.io/posts/knowledges/linux-for-security)


## インストール
CAINE 13.0 "WARP" を対象とします。  
ダウンロードしたらハッシュ値をチェックします。サイトが改ざんされていたら元も子もないですが一応。  

```
› certutil -hashfile caine13.0.iso sha256
SHA256 hash of caine13.0.iso:
6d25180757d6a8a71e98706009d7a9ba3613131727fc96c2037d78bbd4c8ce3a
CertUtil: -hashfile command completed successfully.
```

インストールしてもよいですが、保全対象上のマシン等でLiveで起動する方が多いかなと思います。  
今回はVMで起動します。  
![caine-1](https://github.com/sumeshi/api/assets/35072092/07bd4398-83cd-4124-8276-41a30774d3ec)

DEFTもそうですが、デスクトップ画像がガビガビなのはなぜなんでしょうか。(古いPCでも動作させるため？)  

## 保全
公式でイメージファイルの保全方法が公開されています。  
[Imaging with CAINE](https://www.caine-live.net/page8/CAINE%20Imaging%20Instructions%20(March%202021)%20-%20External.pdf)

