# CAINE を用いたディスクの保全
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

本手順では `msuhanov/ntfs-samples/ntfs.raw` を使います。64GB(圧縮時80MB) と小さいので扱いやすいです。  
[DFIR検証用イメージファイル](https://sumeshi.github.io/posts/knowledges/dfir-images)

### 下準備
CAINEのタイムゾーンをJSTに合わせます。  
また、保全の際に実施した操作と時刻を記録しておきます。  
このあたりは、デジタルフォレンジック研究会が公開している [「証拠保全ガイドライン」](https://digitalforensic.jp/home/act/products/home-act-products-df-guideline-8th/) が詳しいです。  

### マウント
Mounter (画面右下にある緑色のHDDアイコン) をクリックして保全対象をマウントします。  
このMounterを使うとReadOnlyでマウントされるようなので安全です。  

保全対象は Test_volume です。  
![caine-2](https://github.com/sumeshi/api/assets/35072092/429f61aa-d25c-49b9-9e65-9234855e7dfc)

イメージの保存先の設定もします。  
保全イメージよりも大きなディスク(128GB) を用意してパーティショニングしたら、  
Mounterのアイコンを右クリックしてWritableマウントモードに切り替えます。  
![caine-3](https://github.com/sumeshi/api/assets/35072092/6d71e375-2c09-4c37-a74e-3293aba53c49)

これ以降マウントするディスクはWritableになるので注意です。  
先程と同様にディスクを選択してOKを押すと、Writableでマウントされていることがわかります。  
![caine-4](https://github.com/sumeshi/api/assets/35072092/eace4815-c4b8-476f-924e-b6b4eb326a45)

### 保全
Guymanager を使ってイメージの保全を行います。  
保全対象の `/dev/sdb` を右クリックしてAcquire imageを選択  
![caine-5](https://github.com/sumeshi/api/assets/35072092/ad07a657-9724-434f-a095-089fb174b316)

色々設定できますが、ほぼデフォルトで行きます。  
E01形式で、2GBごとに分割保存される設定です。  
![caine-6](https://github.com/sumeshi/api/assets/35072092/3a6a58b7-e481-4b52-9ec8-fc755b7fea13)

Startを押すとディスクの保全が開始され、Progressが表示されています。  
![caine-7](https://github.com/sumeshi/api/assets/35072092/02757b2d-b0e0-41ef-bda8-1a532cb982ce)

### 確認
保全が終了すると、指定したディスクに `.E01` 形式のファイルと `.info` 形式のファイルが保全されていることが確認できます。  
![caine-8](https://github.com/sumeshi/api/assets/35072092/a7c97d5b-e367-4a4a-af9c-4dd24dc2345e)

.infoファイルには保全に利用したGuymanagerのバージョンや詳細情報、保全したイメージのハッシュ値などが記録されています。

## おわりに
CAINEを使うことで、GUIで簡単にディスクの保全ができました。  
また、デフォルトでReadOnlyになっているなどの必須機能についても搭載されており、実際のフォレンジックで使われることを想定して開発されたものであることがわかります。  

個人的には結構使いやすいと思うので、ご家庭に一本はLive起動USBを用意しておいてもいいと思います。  
お わ り
