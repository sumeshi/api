# 話さないのなら、君のメモリーをVolatilityで調べるしかなさそうだな。
イヤだ！それだけはやめてくれ……。

## はじめに

インシデント対応で証拠保全をするとき、メモリ（RAM）も一緒に取得することが多いと思います。  
ただ、取得したはいいものの「で、ここから何を見るの？」となりがち。

メモリには、ディスク上のファイルやログだけでは追えない情報が含まれています。  
実行中のプロセスやネットワーク接続、ファイルキャッシュ、レジストリ、アプリケーションが扱っていたデータの断片など、調査対象はさまざま。

ログを読むようにはいかないので、ツールを使ってバイナリの中から必要な情報をDigっていく感じ。なかなか難しめ。


## メモリとは

メモリとひとくちにいってもいろいろある。  
RAM上の物理メモリを指す場合もあれば、プロセスから見える仮想メモリを指す場合もあるのだが、OSやアプリケーションが動作中のコードやデータを置く場所、としてざっくり読んで貰えれば。

メモリは、情報を長期間保存しておくための媒体ではなく、OSやアプリケーションの動作に合わせて内容が書き換わり、プロセスの終了やメモリ領域の再利用によって、それまでの情報は失われていく。
また、電源供給が絶たれればデータはすべて吹き飛ぶ（揮発性）ので、侵害されたコンピュータの電源を落としてしまうと情報は永遠に失われてしまう。可能なら起動したままにしておきたい。

そのため、動作中のコンピュータででRAMの内容を保全し、そのイメージを解析するのが一般的である。このメモリの内容を取得してファイルに保存したものを、メモリイメージあるいはメモリダンプと呼ぶ。  

メモリ効率の問題から、一部のデータはディスクに書き出される（pagefile.sysなど）ため、それも補助的な調査対象になる。


### メモリに残る情報

メモリには、プログラムのコード、スタックやヒープ、ロードされたDLL、プロセスやソケットの管理構造などが置かれる。これらを解析すると、取得時点でどのようなプロセスが動き、どのコマンドラインで起動され、どこと通信接続していたのかを調べられる。

また、プロセスが掴んでいたファイルやレジストリのキャッシュが残っていることもある。例えば、ログを消去する前のレコードやEVTXの一部がメモリに残っていれば、ディスク側から追えなくなったイベントログを回収できる場合がある。

不要になったメモリ領域も、解放された瞬間に必ずゼロクリアされるわけではない。上書きされるまでの間、終了済みプロセスのデータ（URLやパスなど）が一部残存することもある。


### 物理メモリと仮想メモリ

一口にメモリといっても、プロセスから見える仮想メモリと、RAM上の物理的な配置は異なる。Windowsはページと呼ばれる単位でメモリを管理し、ページテーブルを使って仮想アドレスと物理アドレスを対応付けている。そのため、プロセスから連続して見える領域でも、物理メモリでは離れた場所に配置されていることがある。

この違いは、検索や復元の結果にも影響する。例えば、文字列が物理的に離れたページに分かれていると、物理メモリを先頭から検索するだけでは見つからないことがある。　　

そんなときは、ツールでページの対応を解釈し、プロセスの仮想メモリとして検索すれば見つかる場合もある。  
逆に、すでにプロセスから参照されなくなった断片は、物理メモリを直接走査した方が見つかることもあるので、どちらか一方が優れているとかそういう話でもない。


## メモリの保全

### 保全時の注意点

ライブ取得には時間がかかります。体感では1GBあたり1分くらいかな。バカデカメモリの場合は注意。

また、前述の通りメモリの取得中もOSやアプリケーションは動いているため、イメージの先頭と末尾で状態が食い違うこともある。  
この不整合は[Memory Smear](https://www.nist.gov/glossary-term/39326)と呼ばれます。


### メモリ外の保全対象

Windowsの場合、下記のようなファイルとしてメモリの一部あるいは全部が書き出される可能性がある。欠損部分を復元できる場合があるため、可能なら一緒に保全しておくこと。

| ファイル | 含まれる情報 |
| --- | --- |
| `C:\pagefile.sys` | メモリから退避されたページ |
| `C:\swapfile.sys` | Windowsが退避したアプリケーションのメモリなど, Win8/WinSrv2012以降のみ |
| `C:\hiberfil.sys` | 休止状態やFast Startupで保存された状態 |
| `C:\Windows\MEMORY.DMP` | クラッシュダンプ |

メモリと保全タイミングがズレると、ツールが誤ったプロセスの解釈を行う場合がある。近いタイミングで取得できるツールを使うこと。


### 取得ツール

メモリを取得するツールは様々あるが、インシデント対応であれば [Magnet RESPONSE](https://www.magnetforensics.com/resources/magnet-response/) のような収集ツールを使うとよい。メモリだけでなく、pagefile.sysや揮発性の高いデータ、主要なアーティファクトもまとめて保全してくれる。

![[Pasted image 20261003164427.png]]

![[Pasted image 20261003164513.png]]

![[Pasted image 20261003164705.png]]

以前は [FTK Imager](https://www.exterro.com/digital-forensics-software/ftk-imager) も有用な選択肢だったのだが、一時期うまくメモリが取得できないなどの不具合があり（現在は解消）評判が落ちてしまった。悲しいね。業界的にもMagnetのほうが最近は人気がある気がする。

![[Pasted image 20261003164749.png]]

日本国内においては、[CDIR-Collector](https://github.com/CyberDefenseInstitute/CDIR) もよく使われている。メモリと主要なアーティファクトをまとめて保全できる。pagefile.sysは収集対象外だが、あまり気にしなくていいという方はこっちのほうが普及はしている。

![[Pasted image 20261003164931.png]]

CDIR-Collectorの取得項目は`cdir.ini`で設定する。RAMも取得するなら`MemoryDump = true`を指定しておき、cdir-collector.exeをカチカチすると保全できる。

どのツールを使う場合も、対象のWindowsのビルドやCPUアーキテクチャ、ドライバ読み込みの制約を含めて事前に試しておくこと。  
また、解析対象が複数台ある場合、取得後にファイルサイズがおかしくないか、プロセス一覧が読めるか程度は確認しておきたい。あとから取り直すというのは難しい。


### 保全時の記録

後から取得時の内容を確認できるよう、次の情報は残しておきたい。

- 対象ホスト名
- 取得開始・終了時刻
- ツール名とバージョン、実行コマンドや収集設定
- 出力先、出力形式、取得ファイルのハッシュ
- エラーや読み取り失敗を含む取得ログ

Magnet RESPONSEの場合、これらの情報はログとして出力してくれるので安心。


### 休止ファイルの展開

ディスクから`hiberfil.sys`を保全できた場合は、[Hibernation Recon](https://arsenalrecon.com/products/hibernation-recon/faqs)などで展開して解析する。

休止ファイル内のメモリデータは圧縮されているため、まず解析に使える形へ再構成する。

```powershell
> .\HibRec.exe /HiberFil=C:\Cases\hiberfil.sys
```

初回起動時にアクティベーション画面が出てくるが、Cancelで閉じるとFree Modeで起動できる。

主な出力は下記。

| 出力 | 内容と使い道 |
| --- | --- |
| `ActiveMemory.bin` | 保存されていたメモリを展開・再構成したもの。対応するメモリ解析ツールへ渡す |
| `DecompressedSlackLegacy.bin` / `DecompressedSlackModern.bin` | 現在の有効な保存データの外に残る領域（slack）を展開したもの。文字列検索やカービングの対象にする |
| `HibRec.log` | 処理内容やエラーを確認するためのログ |

slackには以前の休止処理などのデータが残る場合があるので、`ActiveMemory.bin`と同じ時点の状態として混ぜずに調べる。  
Fast Startup設定時に保存されたhiberfil.sysは、ユーザーセッション全体を含まないため、ライブ取得したRAMと同じ範囲を調べられるわけではない。ハイバネーションのタイミングによっては、過去のメモリ状態と現在のメモリ状態2つを手に入れられるので、点ではなく線の解析ができる。


## 解析の準備

### 調査目的

何を調べたいかによって調査項目は異なる。  
たとえば、すでに見つかっているマルウェアの痕跡がないかを調べたいなら、特徴的な文字列やIOCでバイナリに検索かけても良いし、通信先などの情報を知りたいならプロセスの構造を分析する。

| 調査目的 | 作業内容                  | 主なツール                              |
| ------------------------------ | ------------------- | ---------------------------------- |
| 既知のドメイン、パス、コマンド、特徴的な文字列を検索したい | 文字列検索               | Strings、bstrings、ripgrep、ripgrep-all、langscan |
| 含まれるURLやメールアドレスを列挙したい         | パターン検索 | bulk_extractor |
| 特定のバイト列や複数条件に一致する領域を探したい | ルール検索 | YARA |
| プロセス情報や通信先を調査したい | OSの管理構造に沿った解析 | Volatility、MemProcFS |
| メモリ上のファイルを抽出したい | ファイルオブジェクト、キャッシュの解析 | Volatility、MemProcFS |
| 管理構造が失われたファイルを抽出したい | カービング | foremost、scalpel、PhotoRec、bulk_extractor-rec |

本稿では、バイナリとしての解析、OSの管理構造に沿った解析の2軸で解説する。  
どちらかをやればいいというものではなく、見つけた情報は適宜整理して別の調査に役立てるとよい。


### 環境構築

解析対象とOSは合わせておくと吉。WindowsならWindowsがいい。  
文字列周りはLinuxのが触りやすい場合もあるので、[SIFT Workstation](https://www.sans.org/tools/sift-workstation) あたりを使っても良い。


## 解析手法1: バイナリ解析

バイナリの構造を眺めるというよりは、正体不明のドデカバイナリから可読文字列をダンプして、とりあえずわかることを列挙する、的なアプローチ。

簡単にできるが、それがどういう性質のもので、調査において何を示すのか？というのは前後の文字列から仮説を立てるしかない。  
マルウェアの名前が見つかった！と思っても、よくよく見ると定義ファイルのキャッシュかなにかだったりすることもよくある。多少疑ってかかるくらいの勢いで読もう。

### 文字列抽出

兎にも角にも、文字列を抽出しなければ話にならん。

ASCIIおよびUnicodeの文字列が既定で抽出される、[Sysinternals Strings](https://learn.microsoft.com/ja-jp/sysinternals/downloads/strings) がおすすめ。
最小連続文字列長 `-n` は 6 とか 8 ぐらいでよいと言われている。何も出てこなければ適宜調整。

```powershell
> .\strings64.exe -n 8 memory.raw > memory-strings-ascii-and-unicode.txt
```

Linuxを使っていて、GNU Stringsなどでやるなら、必ず文字コードを意識して出力すること。

```bash
$ strings -a -n 8 -t x memory.raw > memory-strings-ascii.txt
$ strings -a -n 8 -t x -e l memory.raw > memory-strings-utf16le.txt
```

`-t x` を付けておくと、文字列が見つかったファイルオフセットも残る。あとで元のメモリイメージへ戻って周辺を確認したいときに便利。
Sysinternals Stringsでは`-o`でオフセットを付けられる。GNU Stringsの`-e l`は入力をUTF-16LEとして読む指定で、日本語なども含めたUnicode文字列を網羅的に抽出できるわけではない。

#### 出力の圧縮

メモリ全体から文字列を抽出すると、テキストファイルだけで数GBを超える。x100台とかになると手元の機器に置いとくのは厳しいとなるかも。テキストは圧縮がよく効くので、gzipに流したっていい。

```bash
$ strings -a -n 8 -t x memory.raw | gzip -c > memory-strings-ascii.txt.gz
```

Windowsの標準環境にはgzipコマンドがないので、出力後に [7-Zip](https://www.7-zip.org/) などでgzip圧縮するとよい。

```powershell
> .\strings64.exe -n 8 memory.raw > memory-strings.txt
> .\7z.exe a -tgzip memory-strings.txt.gz memory-strings.txt
```

#### 進捗表示

メモリ程度のstringsで必要になることはあまりないが、進捗がわからないと落ち着かないという人は [`Pipe Viewer`](https://www.ivarch.com/programs/pv.shtml) を使うと良い。

Linuxなら `pv` から入力ファイルを読み込ませると、処理済みサイズ、転送速度、進捗率、残り時間の目安を確認できる。

```bash
$ pv memory.raw | strings -a -n 8 -t x > memory-strings-ascii.txt
```

Windowsの標準環境にはそんなものない。追加ツールを入れないなら諦めよう。

### 文字列検索

既知の不審なドメインやファイル名などのIOC（侵害の痕跡を探すための指標）がある場合は、その値を検索する。以下の`strings.txt`は、前の手順で抽出したテキストファイルに置き換える。
grepでもいいが、[ripgrep](https://github.com/burntsushi/ripgrep) のほうが爆速。

```bash
$ rg -i -F 'malicious.example.com' strings.txt
```

よく使うオプションは下記。

| オプション | 説明 |
| --- | --- |
| `-i` | 大文字・小文字を区別しない |
| `-F` | 正規表現ではなく固定文字列として検索する。ドメインの`.`などもそのまま扱う |
| `-f FILE` | 検索パターンをファイルから1行ずつ読み込む。IOCをまとめて検索するときに便利 |
| `-A NUM` | 一致した行の後ろを指定行数表示する |
| `-B NUM` | 一致した行の前を指定行数表示する |
| `-C NUM` | 一致した行の前後をそれぞれ指定行数表示する |
| `-o` | 行全体ではなく、一致した部分だけを表示する |

例えば、IOCを1行に1件ずつ書いた`ioc.txt`があるなら、下記のように前後3行と合わせて確認できる。  
ただし、物理メモリ上で近くにある文字列が、同じプロセスや同じ時点のデータとは限らないので注意。

```bash
$ rg -i -F -f ioc.txt -C 3 strings.txt
```

gzip圧縮している場合は、zgrepを使えばよい。[ripgrep-all](https://github.com/phiresky/ripgrep-all) なら、圧縮ファイルもripgrepと同じ使い心地で検索できる。

```bash
$ rga -i -F 'malicious.example.com' strings.txt.gz
```


#### langscan

[langscan](https://github.com/sumeshi/langscan) を使うと、UTF-8テキストから英語以外の文字列を検索できる。必要に応じてUTF-8へ変換してから渡す。

```bash
$ langscan strings-utf8.txt
```

キリル文字だけひっかけたいな～というときには次のようにかけばよい。

```bash
$ langscan --lang ru strings-utf8.txt
```

exeファイル内に多言語対応処理が書かれている関係でノイズが非常に多くなったりもする。  
後述のように、プロセス単位でダンプされたメモリなど狭い範囲で使うか、ざっと絞り込みたいときに使えば良い。


#### bstrings

[bstrings](https://github.com/EricZimmerman/bstrings) を使えば、よく使われる正規表現パターンで文字列の検索が可能。
パターン一覧は以下のように確認する。

```powershell
> .\bstrings.exe -p
```

お好みのものを検索すればよい。  
バイナリに対しても、テキストに対しても検索できる。

```powershell
> .\bstrings.exe -f strings.txt --lr email
```

よく使うのは下記辺り。

| パターン名     | 説明                                   |
| --------- | ------------------------------------ |
| b64       | Base64文字列                            |
| bitcoin   | BitCoinウォレットアドレス                     |
| bitlocker | BitLocker回復キー                        |
| cc        | クレジットカード番号                           |
| email     | メールアドレス                              |
| guid      | GUID                                 |
| ipv4      | IPアドレスバージョン4                         |
| ipv6      | IPアドレスバージョン6                         |
| mac       | MACアドレス                              |
| reg_path  | レジストリハイブに関連するPath                    |
| sid       | Microsoft Security Identifiers (SID) |
| unc       | UNC Path                             |
| url3986   | RFC 3986準拠URL                        |
| win_path  | Windows Path                         |
| zip       | ZIPコード                               |


#### bulk_extractor

IOCがまだ分からない段階では、[bulk_extractor](https://github.com/simsong/bulk_extractor)でURL、メールアドレスなどを一括抽出できる。ファイルシステムを解釈せず入力をバイト列として走査するため、メモリイメージも入力にできる。

```bash
$ bulk_extractor -o ./bulk memory.raw
```

主な出力は下記。詳しくは [forensics.wiki](https://forensics.wiki/bulk_extractor/) を見るとよい。
出力されるファイルは、有効にしたスキャナや実際に見つかったデータで変わる。

| 出力ファイル           | 説明                                 |
| ---------------- | ---------------------------------- |
| ccn.txt          | クレジットカード番号                         |
| domain.txt       | インターネットドメイン                        |
| email.txt        | メールアドレス                            |
| ip.txt           | IPアドレス                             |
| telephone.txt    | 米国および国際電話番号                        |
| url.txt          | URL                                |
| url_searches.txt | インターネット検索された用語。結構役に立つ率が高い。         |
| wordlist.txt     | 抽出されたすべての単語リスト。パスワードクラックなどに有用。     |
| zip.txt          | ZIPファイルに関する情報。Office形式などもzipなので有用。 |

こうやって見つけた情報はまた別のところでも使うのでちゃんと整理しておきましょうネ。うまいこと一般化すれば強力な武器にもなる。


### YARA検索

単純な検索では引っかからない場合や、マルウェアファミリはわかってるんだけどどう検索していいかわからない場合は[YARA](https://github.com/VirusTotal/yara) を使えばいい。

`{malware-family} yara rule` とかでググるとたくさん出てくる。必要に応じてカスタムしながらつかうこと。ベーシックなルールセットはこれ。[yara-rules/rules](https://github.com/yara-rules/rules)

Rust実装の [YARA-X](https://github.com/virustotal/yara-x) のほうが最近は開発が盛ん。ただし、一部のモジュールに依存したルールなどは動かないので注意。


### ファイルカービング

ファイルヘッダの構造や、レコードの特徴からファイルあるいはその断片を探すカービングをしてもよい。
ただし、ファイル全体が読み込まれていない、仮想的には連続する領域に記録されていても、物理的には領域が離れているなどの要因で、不完全な復元となるケースが多い。画像ファイルの一部でも出てきたら儲けたな、ぐらいの期待度で。

#### foremost

[foremost](https://github.com/korczis/foremost)は、ヘッダやフッタなどの特徴を使ってファイルを回収するツール。たとえばJPEGとPNGを探すなら、形式を絞って次のように実行する。

```bash
$ foremost -t jpg,png -i memory.raw -o out
```


#### scalpel

[scalpel](https://github.com/sleuthkit/scalpel)も、ファイルの特徴を定義してカービングするツール。foremostベースにカスタムされたやつ。

```bash
$ scalpel -c scalpel.conf -o scalpel-out memory.raw
```

配布されている`scalpel.conf`を作業用にコピーし、探したい形式の定義だけコメントを外して使う。
既定の設定ではすべて無効になっている。


#### PhotoRec

[PhotoRec](https://photorec.io/) は前述の2つより開発が盛ん。    
アイコンが怪しいが、[Autopsy](https://sleuthkit.org/autopsy/docs/user-docs/4.20.0/photorec_carver_page.html) にもモジュールが取り込まれている実績もある。対応フォーマットは400種類以上あるとか。

qphotorec_win.exeを使うとGUI版を起動できる。


#### bulk_extractor-rec

[Bulk Extractor with Record Carving](https://www.kazamiya.net/bulk_extractor-rec)は、bulk_extractorにレコード回収用のスキャナを追加したもの。EVTXのファイルやチャンク、MFTレコード、USNジャーナルなどを対象にできる。ファイル全体が残っていなくても、レコード単位なら回収できるかもしれない。

BE Viewerを開いて、Toolsからrun Bulk Extractorをクリックするとファイル選択ができる。  
PhotoRecなどで抽出できなかったイベントログなどもかなり引っかかる。


## 解析手法2: 管理構造の解析

単純に文字列抽出とかではなく、OSの管理構造に沿って解析する方法。  

あやしいプロセスの名前などが分かっている場合はその実行有無を調べればいいし、候補がなければプロセス一覧、コマンドライン、通信先などをひととおり出力しておいて後でゆっくり読めばいい。

一般的には、[MemProcFS](https://github.com/ufrisk/memprocfs)とVolatilityが有名。

SANSの [Memory Forensics Cheat Sheet](https://www.sans.org/posters/memory-forensics) も参考になるので読んでも良い。

### MemProcFS

メモリから回収できるファイルやアーティファクトを横断的に見たい場合は、MemProcFSのforensic modeが便利。

#### マウント

Windowsでは [Wiki](https://github.com/ufrisk/MemProcFS/wiki) に従ってDokanyなどを準備し、空いているドライブ(一般的には `M:`)へマウントする。

```powershell
> .\MemProcFS.exe -device C:\Cases\memory.raw -mount M -forensic 4
```

![[Pasted image 20261003172559.png]]

起動時に`-forensic`をつけてフォレンジックモードを有効にすることで、同じ入力イメージ・設定・MemProcFSバージョンでの解析結果を再現しやすくなる。マウント後に有効化すると、キャッシュや処理順序の違いから差異が出る可能性がある。

| モード | 意味                                                   |
| --- | ---------------------------------------------------- |
| 1   | インメモリで動くSQLite、ファイルとしては残らない                          |
| 2   | MemProcFS終了時に削除されるSQLite、ファイルとして残すが自動削除              |
| 3   | MemProcFS終了時も保持されるSQLite、ファイルとして残す                   |
| 4   | MemProcFS終了時も保持されるSQLite、ファイルとして残す（固定名: vmm.sqlite3） |

データベースの保存場所は`M:\forensic\database.txt` にかかれている。
私の場合は下記だった。

```
C:\Users\example\AppData\Local\Temp\vmm.sqlite3
```

解析の進捗状況は `M:\forensic\progress_percent.txt` に記録されるので、100になるまで待ってから、結果を確認する。

同時期に取得したpagefileがあるなら、対応する番号で追加できる。
デフォルトのWindows10では、各ページファイルにインデックス番号が振られており、pagefile.sysは0, swapfile.sysは1が割り当てられている。
ページファイルを追加・変更した環境では番号が異なる場合があるので、対象環境の構成も確認する。

```powershell
> .\MemProcFS.exe -device C:\Cases\memory.raw -pagefile0 C:\Cases\pagefile.sys -pagefile1 C:\Cases\swapfile.sys -mount M -forensic 4
```

この2つの起動例は、pagefileの有無に応じて選ぶ。

| フォルダ     | 説明                                                                                                                                                                                                  |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| conf     | MemProcFSの状態と構成設定                                                                                                                                                                                   |
| forensic | フォレンジック関連情報、めちゃ重要。                                                                                                                                                                                  |
| misc     | その他プラグインに関連する項目。物理アドレスから仮想アドレスを検索する[phys2virt](https://github.com/ufrisk/MemProcFS/wiki/FS_Phys2Virt)や、Bitlockerキーを復元する[bitlocker](https://github.com/ufrisk/MemProcFS/wiki/FS_BitLocker)などが置かれている。 |
| name     | プロセス情報（名前ごと）                                                                                                                                                                                        |
| pid      | プロセス情報（プロセスIDごと）                                                                                                                                                                                    |
| py       | Pythonプラグイン関連。[pypykatz regsecrets](https://github.com/ufrisk/MemProcFS-plugins)プラグインとか入れるとここに表示される。                                                                                                |
| registry | レジストリハイブ。ページングなどによって破損している場合もある。                                                                                                                                                                    |
| sys      | OS、ユーザー、プロセス、ネットワークなど、システム全体の情報。 |
| vm       | Hyper-Vベースの技術（VM, Sandbox, WSL2, VMware/VirtualBox）などが検知されるとそれを解析することができる                                                                                                                           |

主要なところだけさらっと。

#### sys

システムに関する情報がいろいろある。まずはここを見ると良い。

![[Pasted image 20261003182724.png]]

タイムゾーンやバージョン情報、コンピュータ名などなど。
あとは proc/proc.txt あたりを見るとプロセスツリーが見れる。

![[Pasted image 20261003182836.png]]

同様に、users/users.txtでユーザ一覧、tasks/tasks.txtでタスク一覧、net/netstat.txtで通信状況などが見れる。一通り見てからアタリをつけて掘り下げ調査をやるといい感じ。

#### forensic

フォレンジック用にわかりやすくまとめたデータが置かれている。

![[Pasted image 20261003174827.png]]

とりあえず見るべきは `csv`, `files`, `ntfs` あたりかな。

[csv](https://github.com/ufrisk/MemProcFS/wiki/FS_Forensic_CSV) には名前の通り様々な視点で整理したファイルがcsvで置かれている。Timeline Explorerなどが見やすい。
`findevil` や `yara` などの情報もカバーされているので、別ディレクトリをわざわざ見なくてもよい。

たとえば、`M:\forensic\files\files.txt`なら復元候補のファイル一覧を見ることができる。
気になるものがあれば復元しよう。
ファイルの実体は `files` から持ってくれば良い。Windowsのフォルダ構造そのままなので使いやすい。NTFSの構造上、サイズの小さいファイルはMFT側に直接記録されていることがある。その場合は `ntfs` 側にあるかもしれない。

![[Pasted image 20261003180220.png]]

#### name / pid

どちらもプロセスに関する情報で、それがプロセス名別かPID別かというだけ。nameの方は末尾にプロセスIDがついているのでそっちのが見やすいかも。

![[Pasted image 20261003180638.png]]

![[Pasted image 20261003180652.png]]

基本情報は名前のとおり。name-longがプロセス名、pidがプロセスID、ppidは親のプロセスID、time-createはプロセス起動時間、win-cmdlineは起動時コマンド、win-environmentは環境変数。気になるなら見てみれば良い。

プロセス内で重要なのだと、`files/handles` とか？プロセスが開いているファイルハンドルを手掛かりに再構築されたファイル。
`files/modules` はメモリ上のモジュールから再構築されたexeやdllなど。`files/vads` はVAD（Virtual Address Descriptor）を手掛かりに再構築されたファイル。VADは、プロセスの仮想メモリ領域と、その保護属性や対応するファイルなどを管理する構造である。

どの場所から回収した場合も、ファイル全体がメモリに残っているとは限らない。パーサで読めるかを確認し、欠損やエラーも記録しておく。

.evtxで検索するとイベントログが見つかったりする。ディスク側で削除されていても一部イベントの復元ができるかも。見つけたら[調査してみるとよい](https://sumeshi.github.io/posts/knowledges/windows-eventlog-analysis-101) 。
![[Pasted image 20261003181311.png]]

![[Pasted image 20261003181432.png]]

#### registry

名前の通りレジストリハイブ。ハイブファイルも置かれてるし、パースされたテキストもある。

![[Pasted image 20261003182638.png]]

個人的にはRegistryExplorerとかで見るほうが見やすい。が、壊れていることもよくある。
![[Pasted image 20261003182416.png]]

レジストリへの変更はメモリ上のハイブに反映され、トランザクションログ（`.LOG1`、`.LOG2`）を使いながらディスクへ書き戻される。そのため、メモリのほうがディスク上にあるものよりも新しい場合もある。見る価値はある。

### Volatility

メモリ調査といえばこれ！のイメージだが、マジで血管がブチ切れるぐらい時間がかかる。ざっと眺めるならMemProcFSのが早い。

とはいえ、こちらのほうがプラグインが充実しているので、処理かけてからご飯食べに行くとかそういう気持ちで。

[vol-rs](https://github.com/daffainfo/vol-rs) という高速なRust実装版もある。CTFとかならこういうの使ってもいいかもね。まだ成熟したプロダクトではないので、実務で使う場合は評価検証が必要だと思う。

Volatility 2と3がよく使われるが、最近のOSなら3でいい。たまに古代の発掘品とかが来たときは2じゃないと動かないときもある。どちらも用意しておくとよい。

また、Volatilityは慣れないと非常に使いづらい（し、3系はインストールも大変）。[Volatility Workbench](https://www.osforensics.com/tools/volatility-workbench.html) や[KaniVola](https://github.com/4n6ist/KaniVola) などのラッパーを使うと非常に楽。コマンドをカチャカチャ打つほうが気持ちいいのはわかるのだが、実際インシデント対応してるときにそんな暇はないことがほとんど。

3系ならVolatility Workbenchがすごく使いやすい
![[Pasted image 20261003183952.png]]

2系ならKaniVolaがすごく使いやすい
![[Pasted image 20261003184427.png]]

以下はVolatility 3をコマンドラインから実行する前提として解説する。Volatility Workbenchを使っている場合はコマンドに対応するプラグインを選択すればいい感じにやってくれる。

コマンド例の`vol3.py`はVolatility 3を起動するコマンドとして表記している。pipでインストールした環境なら`vol`、ソースから実行するなら`python vol.py`など、自分の環境に合わせて読み替える。プラグイン名やオプションもバージョンによって変わることがあるので、`-h`で確認する。

#### シンボル情報

Volatility 3にはWindowsのシンボルを自動解決する仕組みがあるので、あまり意識しなくても使えるが、MSのサーバに取りに行かなければならないのでインターネット接続が必要である。

私はオフライン絶対至上主義なので、あらかじめシンボル情報をローカルに準備しておく。
[JPCERT/CC - オフラインでVolatility 3を実行する方法](https://blogs.jpcert.or.jp/ja/2021/08/volatility3_offline.html) が非常に参考になる。

これを自動化するようなスクリプトを書いておくと実際対応するときに助かる。

一方、Volatility 2では `--profile` の指定が必要。間違ったシンボル情報を使うと、一見うまくパースできているように見えて壊れていることがあるので注意。

#### 基本情報

まずは`windows.info`でOSやカーネルの情報が読めるかを確認する。ここで失敗する場合は、取得形式、イメージの欠損、シンボルの取得状況などを確認してから先へ進む。

```bash
$ vol3.py -q -f memory.raw windows.info > info.txt
```

#### プロセス一覧

実行中のプロセスと親子関係、実行時コマンドラインを保存する。
プロセス名だけ見ても怪しいかどうか判別は難しい。実行パス、親プロセス、起動引数、実行ユーザーなどの観点を組み合わせて確認する。

正規名に似たプロセスや、`C:\Users\Public`などの[悪用されがちなフォルダ](https://attack.mitre.org/techniques/T1074/001/)から起動された実行ファイルは掘り下げるべき。

```bash
$ vol3.py -q -f memory.raw windows.pslist > pslist.txt
$ vol3.py -q -f memory.raw windows.pstree > pstree.txt
$ vol3.py -q -f memory.raw windows.cmdline > cmdline.txt
```

`pslist`はOSが管理するプロセスのリストをたどり、`pstree`は同じ列挙結果を親子関係で表示する。`pstree`も別の方法で隠蔽プロセスを探しているわけではない。

また、親プロセスがすでに終了していたりPIDが再利用されていたりすると、誤った対応付けになることもある。
作成時刻・終了時刻やプロセスオブジェクトの位置も併せて確認する。

`psscan`は、カーネルがメモリを割り当てるpool領域を走査してプロセスの構造体を探すため、終了済みあるいは隠蔽されたプロセスも検出できる場合があるが、時間がかかる。裏で実行しながらコーヒー飲んでpslistとかを眺めておけば良い。

```bash
$ vol3.py -q -f memory.raw windows.psscan > psscan.txt
```

psxviewを使うと、複数のプロセス列挙方法の結果を突合してくれる。
pslistにはないけどpsscanにはあった、とか。

```bash
$ vol3.py -q -f memory.raw windows.malware.psxview > psxview.txt
```

ただし、列挙方法ごとに終了済みプロセスの扱いも違うので、結果が食い違うだけで隠蔽と決めつけない。PID（プロセスID）、作成・終了時刻、プロセスオブジェクトの位置を突き合わせて候補を絞る。

#### プロセスの詳細

候補をPID `4240`に絞れたら、そのプロセスが何を読み込み、何を参照していたかを調べる。ハンドルはファイル、レジストリキー、他のプロセスなどのオブジェクトを参照するための識別子で、調査対象を広げる手掛かりになる。

```bash
$ vol3.py -f memory.raw windows.cmdline --pid 4240
$ vol3.py -f memory.raw windows.dlllist --pid 4240
$ vol3.py -f memory.raw windows.handles --pid 4240
```

`dlllist`で読み込まれたDLLのパスや配置を確認し、`handles`で参照しているファイルやレジストリキーなどを確認する。カーネルのモジュールを列挙する`windows.modules`とは、調べる対象が異なる。

例えばコマンドラインに一時ディレクトリ上のスクリプトがあれば、そのファイル名を検索や回収の対象にする。ハンドルに文書のパスがあれば、関連するファイルやアクセスの痕跡をディスク側でも確認する。ハンドルの存在だけでは、ファイルの全内容を読んだことや外部へ送信したことまでは分からない。

#### コマンド履歴

プロセスの起動引数は`cmdline`、コンソールに残っている入力履歴は`cmdscan`で調べる。たとえば`cmd.exe`を起動した後に何を入力したか知りたい場合は、こちらも試しておく。

```bash
$ vol3.py -q -f memory.raw windows.cmdscan > cmdscan.txt
```

取得できるのは対応するコンソールの履歴構造がメモリに残っている範囲なので、すべてのシェル操作を復元できるわけではない。PowerShellの履歴やスクリプトの実行内容は、PSReadLineの履歴ファイルやPowerShellログなども照合する。

#### ネットワーク接続

ネットワークの管理構造を調べるなら`netscan`を使う。

```bash
$ mkdir -p out
$ vol3.py -q -f memory.raw windows.netscan > netscan.txt
```

見つかったPIDをプロセス一覧や作成時刻と照合し、そのプロセスを追加で調べる。終了済み・解放済みの構造が残る場合もあるので、`State`や`Created`も読む。`Created`はネットワークオブジェクトの作成時刻を示す。

また、`netscan`から通信内容や転送量は得られない。  
接続先との実際のやり取りを確認するなら、メモリだけでなくプロキシ、DNS、ファイアウォール、EDRなどの記録へ調査を広げる必要がある。

#### 不審な実行領域

`malfind`は、VADの属性などから、不審な実行可能領域を探す。ファイルをディスクに残さず、別のプロセスのメモリ上でコードを実行するような痕跡を探すときに使える。

```bash
$ vol3.py -f memory.raw windows.malware.malfind --pid 4240
```

出力されたアドレスや保護属性、先頭部分のバイト列・逆アセンブル結果を見て、追加で解析する領域を選ぶ。JITコンパイルなど正規の処理でも候補が出るため、ヒットした領域の内容や関連するモジュールも確認する。

こちらも `--dump` で抽出できる。


#### プロセス内の検索

文字列検索でドメインが見つかったものの、どのプロセスと関係するか分からない場合は、同じ文字列をYARAルールにしてプロセスの仮想メモリを検索できる。

```bash
$ vol3.py -f memory.raw windows.vadyarascan --yara-file ioc.yar
```

`vadyarascan`は、VADをたどってプロセスの仮想メモリ領域を検索する。  
この結果は「そのプロセスの読み取れた領域にIOCがあった」という根拠になるが、それがどう使われていたかは他のアーティファクトと突合して見ていく必要がある。


#### PEとメモリの抽出

実行ファイルの解析に渡したいのか、ヒープなども含めて文字列を探したいのかで、抽出方法を変える。

```bash
# プロセスの実行イメージをPEとしてダンプ
$ vol3.py -f memory.raw -o pe windows.pslist --pid 4240 --dump

# 読み取り可能なプロセスメモリを抽出し、アドレスとの対応を保存する
$ vol3.py -q -f memory.raw -o pages windows.memmap --pid 4240 --dump > memmap-4240.txt

# malfindによって検出された領域を抽出する
$ vol3.py -f memory.raw -o suspicious windows.malware.malfind --pid 4240 --dump
```

ダンプしたPEは通常、もとのPEファイルと同一ではないので、ハッシュ値を求めてVirusTotalなどで検索することはできない。
実行することも難しい。 ~~やるならIAT再構築などをする必要があるが、ここでは取り扱わない。~~

が、前述の文字列抽出やYARAによるスキャンによって解析のヒントが得られることもある。

[FLOSS](https://github.com/mandiant/flare-floss) は、実行ファイルのコードを解析し、難読化されていた文字列や、実行時に組み立てられる文字列の抽出を試みるツール。  
通常のstringsでは出てこない文字列を探したいときに有用。

入力には、回収したPEを指定する。次の`recovered.exe`は、その回収ファイルの例。

```bash
$ floss recovered.exe
```


#### ファイルの回収

メモリには、プロセスが使用したファイルやOSのキャッシュも残る。調査対象のファイル名やパスが分かれば回収を試し、得られた内容をその形式に対応するパーサへ渡せる。

`filescan`は、Windowsがファイルを管理する`FILE_OBJECT`を走査する。まずファイル名から候補を探し、そのオブジェクトに対応する内容を`dumpfiles`で回収する。

```bash
$ vol3.py -q -f memory.raw windows.filescan > filescan.txt
```

次はWindows 10 / 11を対象に、見つかった`FILE_OBJECT`の仮想アドレスを指定する例である。アドレスは実際の出力に置き換える。

```bash
$ vol3.py -f memory.raw -o cache windows.dumpfiles --virtaddr 0xffff800012345670
```

`--virtaddr`と`--physaddr`は、それぞれ`FILE_OBJECT`の仮想アドレスと物理アドレスを受け取る。

プロセスとの関連が分かっていれば、PIDを指定して回収候補を絞ることもできる。

```bash
$ vol3.py -f memory.raw -o out/cache windows.dumpfiles --pid 4240
```

`filescan`で名前が見つかっても、ファイル内容がしっかり残っているとは限らない。
欠落しているならば部分的な復元として文字列検索したりいろいろすればよい。


## 調査結果の整理

メモリフォレンジックをしていると、ファイルがグチャグチャになりがちなので、フォルダをいい感じに整理しておくとよい。
MemProcFSであれば、タイムラインとして整理された情報を出力してくれるので一通りコピーしておく。

整理は本当に大変なので、面倒なら[AIにぶん投げても良い](https://sumeshi.github.io/posts/works/dont-make-ai-your-forensic-analyst)と思う。

ツールによって結果の食い違いが発生することもままある。  
誤りと決めつけずに、出力時の機能がメモリの何をみてその結果を出力したのか？というのを意識しながら検証すること。列挙する管理構造、終了済みオブジェクトの扱い、欠損ページの処理など。。


## おわりに

メモリフォレンジックでは、ディスクフォレンジックと比較して破損したデータを扱う機会が非常に多くなります。それをどう活かすかというのは中身を見てアタリをつける経験とセンスになってくるのでなんとも言えないのですが、困ったらとりあえず文字列として解釈してみるとか、GIMPに突っ込んで画像として見てみるとか、マジに困ったらAIを頼ってもいいと思います。

メモリフォレンジックなんて人間がやることではない。
