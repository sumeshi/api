# 話さないのなら、君のメモリーをVolatilityで調べるしかなさそうだな。
イヤだ！それだけはやめてくれ……。

## はじめに

インシデント対応で証拠を保全するとき、メモリも一緒に取得することは多いと思います。  
ただ、取得したはいいものの「で、ここから何を見るの？」となることもしばしば。

メモリには、ディスク上のファイルやログだけでは追えない情報も含まれています。  
実行中のプロセスやネットワーク接続、ファイルキャッシュ、レジストリ、アプリケーションが扱っていたデータの断片など、調査対象はさまざま。

ファイルの一覧を開いて中身を読むようにはいかないので、ツールを使ってバイナリの中から必要な情報を探していく感じ。なかなか難しめ。

## メモリとは

本記事ではRAM=..., メモリ=...として扱う。

RAMは、情報を長期間保存しておくための媒体ではない。OSやアプリケーションの動作に合わせて内容が書き換わり、プロセスの終了やメモリ領域の再利用によって、それまでの情報は失われていく。

また、RAMは揮発性なので、電源を切ると内容を保持できない。再起動後もOSやアプリケーションによる利用が始まるため、再起動前の実行状態を調べたいなら、その前に保全しておく必要がある。

### メモリに記録される情報

メモリには、プログラムのコード、スタックやヒープ、ロードされたDLL、プロセスやソケットの管理構造などが置かれる。これらを解析すると、取得時点でどのようなプロセスが動き、どのコマンドラインで起動され、どこと接続していたのかを調べられる。

ファイルやレジストリのキャッシュが残っていることもある。例えば、ログを消去する前のレコードやEVTXの一部がメモリに残っていれば、ディスク側から追えなくなったイベントログを回収できる場合がある。

不要になったメモリ領域も、解放された瞬間に必ずゼロクリアされるわけではない。上書きされるまでの間、終了済みプロセスのデータなどが残ることがある。

例えば、Volatilityの`psscan`は、カーネルがメモリを割り当てるpool領域からプロセスの管理構造を探すため、終了済みプロセスの痕跡を見つける場合もある。`netscan`もネットワークの管理構造を走査して探すので、出力された接続がすべて取得時点で有効だったとは限らない。

部分的な上書きで管理構造が崩れ、URLやパスなどの文字列だけが残ることもある。次の調査対象を探す手掛かりにはなるが、その断片だけでは元のプロセスや使用時刻を特定できない場合もある。何と関連付けられるかによって、結果から言えることが変わる。

### 物理メモリと仮想メモリ

一口にメモリといっても、プロセスから見える仮想メモリと、RAM上の物理的な配置は異なる。Windowsはページと呼ばれる単位でメモリを管理し、ページテーブルを使って仮想アドレスと物理アドレスを対応付けている。そのため、プロセスから連続して見える領域でも、物理メモリでは離れた場所に配置されていることがある。

この違いは、検索や復元の結果にも影響する。例えば、文字列が物理的に離れたページに分かれていると、物理メモリを先頭から検索するだけでは見つからないことがある。ツールでページの対応を解釈し、プロセスの仮想メモリとして検索すれば見つかる場合がある。

逆に、すでにプロセスから参照されなくなった断片は、物理メモリを直接走査した方が見つかることもある。どちらか一方で済むものでもない。

また、すべての仮想ページが常にRAMに載っているわけではない。Windowsはメモリの使用状況に応じて一部のページを `pagefile.sys` などへ退避するため、物理メモリイメージだけではプロセスの全データが揃わないことがある。可能ならそれらのファイルも合わせて保全したい。

## メモリの保全

### 保全時の注意点

ライブ取得には時間がかかります。体感では1GBあたり1分くらいかな。バカデカメモリの場合は注意。実際の所要時間は、取得ツールや保存先の書き込み速度などでも変わります。

取得中もOSやアプリケーションは動くため、イメージの先頭と末尾で状態が食い違うこともある。この不整合は[Memory Smear](https://www.nist.gov/glossary-term/39326)と呼ばれます。

### RAMの外に残るメモリ由来のデータ

OSや仮想化基盤がメモリの内容をディスクへ書き出していれば、ライブメモリを取得できなかった場合でも、その一部を調べられる可能性がある。

| データ | 含まれ得る情報 | 調査時に確認する点 |
| --- | --- | --- |
| `pagefile.sys` | RAMから退避されたページ | 領域は再利用される。RAMイメージと取得時点・ページファイル番号が対応するか |
| `swapfile.sys` | Windowsが別途退避したアプリケーションのメモリなど | `pagefile.sys`とは用途や管理方法が異なる |
| `hiberfil.sys` | 休止状態やFast Startupで保存された状態 | Fast Startupでは主にカーネルセッションを保存し、ユーザーセッション全体は保存しない |
| クラッシュダンプ | 障害時などに保存されたメモリ | Complete、Kernel、Smallなど、ダンプの種類で含まれる範囲が異なる |
| VMのスナップショット | 仮想マシンのメモリ状態 | `.vmem`ファイルなど。メモリを含む設定で取得されたか。解析に必要な付随ファイルも揃っているか |

ディスクも保全できるなら、これらのファイルも一緒に確保しておきたい。ただし、違う時点で取得したRAMとpagefileを解析ツールに組み合わせて与えると、無関係なデータをプロセスの内容として解釈してしまうおそれがある。取得元と取得時点も記録しておく。

### 取得ツールの選定

RAMのみを取得するなら、[Magnet DumpIt](https://www.magnetforensics.com/resources/magnet-dumpit-for-windows/)や[Belkasoft Live RAM Capturer](https://belkasoft.com/ram-capturer)のような専用ツールが扱いやすい。

一方、インシデント対応でRAMに加えてpagefileやvolatile data、主要なシステムアーティファクトもまとめて保全したい場合は、[Magnet RESPONSE](https://www.magnetforensics.com/resources/magnet-response/)のような収集ツールも便利。Magnet RESPONSEはRAM取得にDumpItを利用し、pagefileや各種IRデータを一連のcollectionとして保存できる。

![[Pasted image 20261003164427.png]]

![[Pasted image 20261003164513.png]]

![[Pasted image 20261003164705.png]]

以前は [FTK Imager](https://www.exterro.com/digital-forensics-software/ftk-imager) も有用な選択肢だったのだが、一時期うまく取得できないなどの不具合があり（現在は解消）評判が落ちてしまった。悲しいね。業界的にもMagnetのほうが最近は人気がある気がする。

![[Pasted image 20261003164749.png]]

初動対応向けの[CDIR-Collector](https://github.com/CyberDefenseInstitute/CDIR)も選択肢になる。RAM、`$MFT`、`$UsnJrnl`、Prefetch、イベントログ、レジストリなどをまとめて保全できる。pagefile.sysは収集対象外だが、そもそもファストフォレンジック向けツールなのであまり気にしなくていいという方はこれ。関連ファイル一式をUSB媒体などに配置し、管理者権限で`cdir-collector.exe`を実行する。

![[Pasted image 20261003164931.png]]

CDIR-Collectorの取得項目は`cdir.ini`で設定する。RAMも取得するなら`MemoryDump = true`を指定しておく。既定のRAM取得には[WinPmem](https://github.com/Velocidex/WinPmem)を使うため、収集ツールと、その内部で動くメモリ取得ツールの両方を確認しておくとよい。

cdir-collector.exeをカチカチすると保全できる。

どのツールを使う場合も、対象のWindowsのビルドやCPUアーキテクチャ、ドライバ読み込みの制約を含めて事前に試しておきたい。取得後はログやファイルサイズを確認し、解析環境でOS情報やプロセス一覧を読めるところまで確かめる。

### 保全時に記録すべき情報

当然のことだが、証拠保全用のツール自体もメモリを使用し、対象システムの状態を変えてしまう。  
後から取得作業の痕跡を区別できるよう、次のような情報は残しておきたい。

- 対象ホストとOSの情報
- 取得開始・終了時刻とタイムゾーン
- ツール名とバージョン、実行コマンドや収集設定
- 出力先、出力形式、取得ファイルのハッシュ
- エラーや読み取り失敗を含む取得ログ

また、保存先には、十分な容量と書き込み速度のある外部媒体などを用意する。システムドライブへ巨大なイメージを書き込むと、削除済みファイルのデータが残っていた領域を上書きする可能性があり、後でディスクを調べる場合の回収にも影響する。

取得したファイルのハッシュ値も取っておくと良い。可能なら2種類以上のアルゴリズムで。

## 解析の流れ

### 環境構築

解析対象とOSは合わせておくと吉。  
WindowsならWindowsがいいね。

それでも俺はLinuxが...という人は[SIFT Workstation](https://www.sans.org/tools/sift-workstation) を使っても良い。あまり使いやすいとは思わないが、strings, volatilityくらいならまぁなんとかなる。

あとは、SANSの [Memory Forensics Cheat Sheet](https://www.sans.org/posters/memory-forensics) も参考になる。

### 調査目的を定める

何を調べたいかによって調査方法は異なる。  

たとえば、すでに見つかっているマルウェアの痕跡がないかを調べるなら、特徴的な文字列などを検索かけても良いし、通信先などの情報を知りたいならプロセスの構造を分析する。

| 調べたいこと                         | 方法                  | 主なツール                              |
| ------------------------------ | ------------------- | ---------------------------------- |
| 既知のドメイン、パス、コマンド、特徴的な文字列が残っているか | 文字列検索               | Strings、bstrings、ripgrep           |
| URLやメールアドレスを候補として列挙したい         | 特徴的なデータの一括抽出        | bulk_extractor                     |
| 特定のバイト列や複数の条件に一致する領域を探したい      | バイナリ検索、ルールによる検索     | YARA、MemProcFS Search              |
| プロセス、ソケット、ハンドルの関係を知りたい         | Windows内部構造の解析      | Volatility 3、MemProcFS             |
| キャッシュ上のファイルを回収したい              | ファイルオブジェクトやキャッシュの解析 | Volatility 3の`dumpfiles`、MemProcFS |
| 管理構造が失われたファイルやレコードを探したい        | シグネチャなどを使ったカービング    | foremost、scalpel、bulk_extractor    |

本稿では、解析手法としてバイナリとして解析する方法、OSの管理構造に沿って解析する方法を紹介するが、どちらかをやればいいというものではなく、見つけた情報は適宜整理して別の調査に役立てる。
## 解析手法1: バイナリとして解析する

バイナリの構造を眺めるというよりは、正体不明のドデカバイナリから可読文字列をダンプして、とりあえずわかることを列挙する、的なアプローチ。

簡単にできるが、それがどういう性質のもので、調査において何を示すのか？というのは前後の文字列から仮説を立てるしかない。  
マルウェアの名前が見つかった！と思っても、よくよく見ると定義ファイルのキャッシュかなにかだったりすることもよくある。多少疑ってかかるくらいの勢いで読もう。

### 文字列を抽出する

兎にも角にも、文字列を抽出しなければ話にならん。

AsciiおよびUnicodeが規定で出力される、[Sysinternals Strings](https://learn.microsoft.com/ja-jp/sysinternals/downloads/strings) がおすすめ。
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

#### 出力を圧縮する

メモリ全体から文字列を抽出すると、テキストファイルだけで数GBを超える。x100台とかになると手元の機器に置いとくのは厳しいとなるかも。テキストは圧縮がよく聞くので、gzipに流したっていい。

```bash
$ strings -a -n 8 -t x memory.raw | gzip -c > memory-strings-ascii.txt.gz
```

Windowsの場合、相当するコマンドはないので出力後に [7-Zip](https://www.7-zip.org/) などでgzip圧縮するとよい。

```
> .\string64.exe -n 8 memory.txt > memory-strings.txt
> .\7z.exe a -tgzip memory-strings.txt.gz memory-strings.txt
```

#### 進捗を表示する

メモリ程度のstringsで必要になることはあまりないが、進捗がわからないと落ち着かないという人は [`Pipe Viewer`](https://www.ivarch.com/programs/pv.shtml) を使うと良い。

Linuxなら `pv` を間に入れておくと、処理済みサイズ、転送速度、進捗率、残り時間の目安を確認できる。

```bash
$ strings -a -n 8 -t x memory.raw | pv > memory-strings-ascii.txt
```

Windowsにはそんなものない。諦めよう。

### 検索する

既知の不審なドメインやファイル名などのIOCがある場合は、その値を検索する。
grepでもいいが、[ripgrep](https://github.com/burntsushi/ripgrep) のほうが爆速。

```bash
$ rg -i -F 'malicious.example.com' strings.txt
```

よく使うオプションは下記。

| オプション | 説明  |
| ----- | --- |
| -i    |     |
| -F    |     |
| -f    |     |
| -A    |     |
| -B    |     |
| -C    |     |
| -o    |     |

gzip圧縮している場合は、zgrepを使えばよい。が、[ripgrep-all](https://github.com/phiresky/ripgrep-all) のが早い。

```bash
$ rga -i -F 'malicious.example.com' strings.txt.gz
```

オプションは一緒。

#### flossでより高度な文字列抽出を行う

```
$ floss memory.raw
```
https://github.com/mandiant/flare-floss

#### langscanで異言語検索を行う

[langscan](https://github.com/sumeshi/langscan) を使うと、UTF-8テキストから英語以外の文字列を検索できる。

```bash
$ langscan strings-utf8.txt
```

キリル文字だけひっかけたいな～というときには次のようにかけばよい。

```bash
$ langscan --lang ru strings-utf8.txt
```

とはいえ、exeファイル内に多言語対応処理が書かれている関係でノイズが非常に多くなったりもする。後述のように、プロセス単位でダンプされたメモリなど狭い範囲で使うか、ざっと絞り込みたいときに使えば良い。

#### bstringsでパターン検索を行う

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

#### bulk_extractorでパターン抽出を行う

IOCがまだ分からない段階では、[bulk_extractor](https://github.com/simsong/bulk_extractor)でURL、メールアドレスなどを一括抽出できる。ファイルシステムを解釈せず入力をバイト列として走査するため、メモリイメージも入力にできる。

```bash
$ bulk_extractor -o ./bulk memory.raw
```

主な出力は下記。詳しくは [forensics.wiki](https://forensics.wiki/bulk_extractor/) を見るとよい。

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

こうやって見つけた情報はまた別のところでも使うのでちゃんと整理しておきましょうネ。
場合によってはあなたの強力な武器にもなります。

### ファイルカービングを行う

ファイルヘッダの構造や、レコードの特徴からファイルあるいはその断片を探すカービングをしてもよい。
ただし、ファイル全体が読み込まれていない、仮想的には連続する領域に記録されていても、物理的には領域が離れているなどの要因で、不完全な復元となるケースが多い。画像ファイルの一部でも出てきたら儲けたな、ぐらいの期待度で。
#### foremost によるカービング

[foremost](https://github.com/korczis/foremost)

#### scalpel によるカービング

[scalpel](https://github.com/sleuthkit/scalpel)

#### PhotoRecによるカービング
カービング用ソフトの [PhotoRec](https://photorec.io/) を使っても良い。[Autopsy](https://sleuthkit.org/autopsy/docs/user-docs/4.20.0/photorec_carver_page.html) にもモジュールが取り込まれているほど優秀。対応フォーマットは400種類以上あるとか。
#### Bulk Extractor with Record Carving によるカービング

[Bulk Extractor with Record Caving](https://www.kazamiya.net/bulk_extractor-rec) 

### ルール検索をする
### YARA

単純な検索では引っかからない場合や、マルウェアファミリはわかってるんだけどどう検索していいかわからない場合は[YARA](https://github.com/VirusTotal/yara) を使えばいい。

`{malware-family} yara rule` とかでググるとたくさん出てくる。必要に応じてカスタムしながらつかうこと。

Rust実装の [YARA-X](https://github.com/virustotal/yara-x) のほうが最近は開発が盛ん。ただし、一部のプラグインに依存したルールなどは動かないので注意。

ベーシックなのはこれ。[yara-rules/rules](https://github.com/yara-rules/rules)

## 解析手法2: OSの管理構造に沿って解析する

単純に文字列抽出とかではなく、OSの管理構造に沿って解析する方法。  

あやしいプロセスの名前などが分かっている場合はその実行有無を調べればいいし、候補がなければプロセス一覧、コマンドライン、通信先などをひととおり出力しておいて後でゆっくり読めばいい。

一般的には、[MemProcFS](https://github.com/ufrisk/memprocfs)とVolatilityが有名。

### MemProcFS

メモリから回収できるファイルやアーティファクトを横断的に見たい場合は、MemProcFSのforensic modeが便利。

#### イメージのマウント

Windowsでは [Wiki](https://github.com/ufrisk/MemProcFS/wiki) に従ってDokanyなどを準備し、空いているドライブ(一般的には `M:`)へマウントする。

```powershell
> .\MemProcFS.exe -device C:\Cases\memory.raw -mount M -forensic 4
```

![[Pasted image 20261003172559.png]]

`--forensic` をつけてフォレンジックモードを有効にすることで、同じMemProcFSバージョンであれば結果が再現可能になる。つけないと処理順序の違いから差異が出る可能性がある。

| モード | 意味                                                   |
| --- | ---------------------------------------------------- |
| 1   | インメモリで動くSQLite、ファイルとしては残らない                          |
| 2   | MemProcFS終了時に削除されるSQLite、ファイルとして残すが自動削除              |
| 3   | MemProcFS終了時も保持されるSQLite、ファイルとして残す                   |
| 4   | MemProcFS終了時も保持されるSQLite、ファイルとして残す（固定名: vmm.sqlite3） |
データベースの保存場所は`M:\forensic\database.txt` にかかれている。
渡しの場合は下記だった。

```
C:\Users\example\AppData\Local\Temp\vmm.sqlite3
```

解析の進捗状況は `M:\forensic\progress_percent.txt` に記録されるので、100になるまで待ってから、結果を確認する。

同時期に取得したpagefileがあるなら、対応する番号で追加できる。
デフォルトのWindows10では、各ページファイルにインデックス番号が振られており、pagefile.sysは0, swapfile.sysは1が割り当てられている。

```powershell
> .\MemProcFS.exe -device memory.raw -pagefile0 C:\Cases\pagefile.sys -pagefile1 swapfile.sys -mount M -forensic 4
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
| sys      | ファイルシステムの様々な情報が記載されている。                                                                                                                                                                             |
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

たとえば、files.txt なら復元可能なファイルの一覧を見ることができる。
気になるものがあれば復元しよう。
ファイルの実体は `files` から持ってくれば良い。Windowsのフォルダ構造そのままなので使いやすい。NTFSの構造上、サイズの小さいファイルはMFT側に直接記録されていることがある。その場合は `ntfs` 側にあるかもしれない。

![[Pasted image 20261003180220.png]]
#### name / pid

どちらもプロセスに関する情報で、それがプロセス名別かPID別かというだけ。nameの方は末尾にプロセスIDがついているのでそっちのが見やすいかも。

![[Pasted image 20261003180638.png]]

![[Pasted image 20261003180652.png]]

プロセス内で重要なのだと、`files/handles` とか？開いているプロセスハンドルから再構築されたファイル。
`files/modules` はメモリ断片から再構築されたexe, dll, sysなど。`files/vads` は仮想アドレスディスクリプタから再構築されたファイル

.evtxで検索するとイベントログが見つかったりする。ディスク側で削除されていても一部イベントの復元ができるかも。見つけたら[調査してみるとよい](https://sumeshi.github.io/posts/knowledges/windows-eventlog-analysis-101) 。
![[Pasted image 20261003181311.png]]

![[Pasted image 20261003181432.png]]

あとは名前のとおり。name-longがプロセス名、pidがプロセスID、ppidは親のプロセスID、time-createはプロセス起動時間、win-cmdlineは起動時コマンド、win-environmentは環境変数。気になるなら見てみれば良い。
#### registry

名前の通りレジストリハイブ。ハイブファイルも置かれてるし、パースされたテキストもある。

![[Pasted image 20261003182638.png]]

個人的にはRegistryExplorerとかで見るほうが見やすい。が、壊れていることもよくある。
![[Pasted image 20261003182416.png]]

レジストリへの変更は、.LOGに一時記録されたうえでシャットダウン時などに統合されるので、メモリのほうがディスク上にあるものよりも新しい場合もある。見る価値はある。

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


#### シンボル情報の解決

Volatility 3にはWindowsのシンボルを自動解決する仕組みがあるので、あまり意識しなくても使えるが、MSのサーバに取りに行かなければならないのでインターネット接続が必要である。

私はオフライン絶対至上主義なので、あらかじめシンボル情報をローカルに準備しておく。
[JPCER/CC - オフラインでVolatility 3を実行する方法](https://blogs.jpcert.or.jp/ja/2021/08/volatility3_offline.html) が非常に参考になる。

これを自動化するようなスクリプトを書いておくと実際対応するときに助かる。

一方、Volatility 2では `--profile` の指定が必要。間違ったシンボル情報を使うと、一見うまくパースできているように見えて壊れていることがあるので注意。

以下はVolatility3をコマンドラインから実行する前提として解説する。Volatility Workbenchを使っている場合はコマンドに対応するプラグインを選択すればいい感じにやってくれる。

#### プロセス一覧を見る

実行中のプロセスと親子関係、実行時コマンドラインを保存する。
プロセス名だけ見ても怪しいかどうか判別は難しい。実行パス、親プロセス、起動引数、実行ユーザーなどの観点を組み合わせて確認する。

正規名に似たプロセスや、C:\Users\Publicなどの[悪用されがちなフォルダ](https://attack.mitre.org/techniques/T1074/001/)から起動された実行ファイルは掘り下げるべき。

```bash
$ vol3.py -q -f memory.raw windows.pslist > pslist.txt
$ vol3.py -q -f memory.raw windows.pstree > pstree.txt
$ vol3.py -q -f memory.raw windows.cmdline > cmdline.txt
```

-scan系のプラグインはメモリ全体をスキャンして構造体を解析するため、終了済みあるいは隠蔽されたプロセスも検出できるが、時間がかかる。裏で実行しながらコーヒー飲んでpslistとかを眺めておけば良い。

また、親プロセスがすでに終了していたりPIDが再利用されていたりすると、誤った対応付けになることもある。
作成時刻・終了時刻やプロセスオブジェクトの位置も併せて確認する。

```bash
$ vol3.py -q -f memory.raw windows.psscan > psscan.txt
$ vol3.py -q -f memory.raw windows.cmdscan > cmdscan.txt
```

psxviewを使うと、複数のプロセス列挙方法の結果を突合してくれる。
pslistにはないけどpsscanにはあった、とか。

```bash
$ vol3.py -q -f memory.raw windows.malware.psxview > psxview.txt
```

### 不審なPIDのコマンドライン、DLL、ハンドルを調べる

候補をPID `4240`に絞れたら、そのプロセスが何を読み込み、何を参照していたかを調べる。ハンドルはファイル、レジストリキー、他のプロセスなどのオブジェクトを参照するための識別子で、調査対象を広げる手掛かりになる。

  

```bash

vol -f memory.raw windows.cmdline --pid 4240

vol -f memory.raw windows.dlllist --pid 4240

vol -f memory.raw windows.handles --pid 4240

vol -f memory.raw windows.malware.malfind --pid 4240

```

  

例えばコマンドラインに一時ディレクトリ上のスクリプトがあれば、そのファイル名を検索や回収の対象にする。ハンドルに文書のパスがあれば、関連するファイルやアクセスの痕跡をディスク側でも確認する。ハンドルの存在だけでは、ファイルの全内容を読んだことや外部へ送信したことまでは分からない。

  

`malfind`は、VADの属性などから、不審な実行可能領域を探す。JITコンパイルなど正規の処理でも候補が出るため、ヒットした領域の内容や関連するモジュールも確認する。空の結果も、コードインジェクションがなかったことの証明にはならない。

  

MemProcFSを使う場合は、同じPIDのディレクトリからコマンドラインや実行パスを読める。

  

```powershell

Get-Content M:\pid\4240\win-cmdline.txt

Get-Content M:\pid\4240\win-path.txt

```

  

### ネットワーク接続とプロセスを照合する

  

ネットワークの管理構造を調べるなら`netscan`を使う。JSONへ保存しておくと、接続先やPIDを変えながら検索するたびに、メモリ全体を再走査せずに済む。

  

```bash

vol -q -f memory.raw -r json windows.netscan > out/netscan.json

```

  

例えばDuckDBなら、JSONを直接読み込み、調査対象のIPアドレスを絞れる。

  

```sql

SELECT PID, Owner, ForeignAddr, ForeignPort, State, Created

FROM read_json_auto('out/netscan.json')

WHERE ForeignAddr = '203.0.113.10';

```

  

見つかったPIDをプロセス一覧や作成時刻と照合し、そのプロセスを追加で調べる。終了済み・解放済みの構造が残る場合もあるので、`State`や`Created`も読む。`Created`はネットワークオブジェクトの作成時刻であり、最後にパケットを送った時刻ではない。

  

また、`netscan`から通信内容や転送量は得られない。接続先との実際のやり取りを確認するなら、プロキシ、DNS、ファイアウォール、EDRなどの記録へ調査を広げる。


### YARAでIOCをプロセスの領域へ関連付ける

文字列検索でドメインが見つかったものの、どのプロセスと関係するか分からない場合は、同じ文字列をYARAルールにしてプロセスの仮想メモリを検索できる。次を`ioc.yar`として保存する。

```yara
rule case_domain
{
    strings:
        $domain = "c2.example.org" ascii wide nocase
    condition:
        $domain
}
```

`ascii wide`でASCIIとUTF-16LE形式の英数字文字列を対象にし、`nocase`で大文字・小文字を区別しない。Volatility 3の`vadyarascan`は、VAD（プロセスの仮想メモリ領域を管理する構造）をたどって検索する。対象のPIDが分かっていれば、次のように指定する。

```bash
vol -f memory.raw windows.vadyarascan --pid 4240 --yara-file ioc.yar
```

PIDが分からなければ`--pid 4240`を外し、列挙できたプロセスのVADを走査する。対象が増える分、時間もかかる。結果のPIDと仮想アドレスから、候補となるプロセスや領域を絞れる。

この結果は「そのプロセスの読み取れた領域にIOCがあった」という根拠になる。通信した証拠にするには、ブラウザの表示内容、キャッシュ、セキュリティ製品の検知パターンなどとして保持されていた可能性も考え、ネットワークの管理構造や外部の通信ログと照合する。

YARAは、文字列以外のバイト列や複数条件も記述できる。ただし、ディスク上のPEを前提とするルールは、メモリへ展開された配置では一致しないことがある。ルールの想定する入力と、検索する範囲を合わせて使う。

  

### 追加解析用にPEやプロセスメモリを抽出する

  

実行ファイルの解析に渡したいのか、ヒープなども含めて文字列を探したいのかで、抽出方法を変える。Volatility 3では、例えば次のように指定する。

  

```bash

mkdir -p out/pe out/pages out/suspicious

  

# プロセスのメイン実行イメージをPEとして再構成する

vol -f memory.raw -o out/pe windows.pslist --pid 4240 --dump

  

# 読み取り可能なプロセスメモリを抽出し、アドレスとの対応も保存する

vol -q -f memory.raw -o out/pages windows.memmap --pid 4240 --dump > out/memmap-4240.txt

  

# malfindが候補に挙げた領域を抽出する

vol -f memory.raw -o out/suspicious windows.malware.malfind --pid 4240 --dump

```

  

`pslist --dump`で得るPEと、`memmap --dump`で得るメモリ領域の集合は用途が異なる。後者を調べるときは、保存した対応表の`Virtual`や`Offset in File`を使い、出力ファイルの位置から元の仮想アドレスへ戻れるようにする。

  

再構成したPEが解析可能なら、[FLOSS](https://github.com/mandiant/flare-floss)で難読化・復号文字列の抽出を試せる。FLOSSはPE内のコードを解析するため、任意のプロセスメモリ全体をそのまま渡す用途には向かない。メモリ上の再配置や欠損があると、PEの解析やディスク上のファイルとのハッシュ照合にも影響する。

  

古い解説にある`procdump`は、Volatility 2のプラグイン名である。Volatility 3では、欲しいデータに対応するプラグインと`--dump`を選ぶ。

  

MemProcFSで公開される各プロセスの`memory.vmem`は仮想アドレス空間をファイルとして見せたもので、未割り当て領域も含めると巨大になる。普通の小さなダンプだと思って丸ごとコピーする前に、必要な領域や抽出方法を確認する。

  

### Volatility

メモリには、プロセスが使用したファイルやOSのキャッシュも残る。調査対象のファイル名やパスが分かれば回収を試し、得られた内容をその形式に対応するパーサへ渡せる。
### Volatility 3でキャッシュ上のファイルを回収する

`filescan`は、Windowsがファイルを管理する`FILE_OBJECT`を走査する。まずファイル名から候補を探し、そのオブジェクトに対応する内容を`dumpfiles`で回収する。

```bash

vol -q -f memory.raw windows.filescan > out/filescan.txt

rg -i -F '.evtx' out/filescan.txt

```

次はWindows 10 / 11を対象に、見つかった`FILE_OBJECT`の仮想アドレスを指定する例である。アドレスは実際の出力に置き換える。

  

```bash

mkdir -p out/cache

vol -f memory.raw -o out/cache windows.dumpfiles --virtaddr 0xffff800012345670

```

  

`--virtaddr`と`--physaddr`は、それぞれ`FILE_OBJECT`の仮想アドレスと物理アドレスを受け取る。古いWindowsや別の列挙方法の出力を使う場合も、どちらのアドレスかを確認して指定する。stringsで得たファイル内の位置を、そのまま渡すオプションではない。

  

プロセスとの関連が分かっていれば、PIDを指定して回収候補を絞ることもできる。

  

```bash

vol -f memory.raw -o out/cache windows.dumpfiles --pid 4240

```

  

`filescan`で名前が見つかっても、ファイル内容まで残っているとは限らない。`dumpfiles`が出力するのはキャッシュなどから回収可能な内容であり、欠落した範囲や複数種類のキャッシュ由来の出力を含む場合がある。出力サイズだけで完全性を判断せず、対象形式のパーサで読める範囲とエラーを確認する。

### HibRec
Hibernation Recon
https://arsenalrecon.com/products/hibernation-recon/faqs


## 調査結果の整理
### CSVやタイムラインを保存する

forensic modeの解析が完了したら、結果を作業ディレクトリへコピーしておく。マウントを終了しても参照でき、同じCSVをチーム内で共有できる。

```powershell
New-Item -ItemType Directory -Force .\out\memprocfs-csv | Out-Null
Copy-Item M:\forensic\csv\*.csv .\out\memprocfs-csv\
```

`process.csv`、`net.csv`、`timeline_all.csv`などをTimeline ExplorerやDuckDBへ渡し、プロセス名、PID、接続先、時刻で絞る。CSVの時刻はUTCなので、イベントログやEDRの表示時刻と比較するときはタイムゾーンを揃える。

タイムラインの各行は、プロセスの作成、レジストリキーの更新、ファイルの時刻など、異なる意味を持つ。時刻順に並んだというだけで因果関係を決めず、行の種類と元データへ戻って解釈する。

### ログやディスクの痕跡と照合する

調査結果には、入力イメージのハッシュ、ツールのバージョン、コマンド、出力先に加え、根拠となったPIDやアドレスも残す。後から「どのメモリから、どの処理で取り出した結果か」を確認できるようにする。

例えば「不審なプロセスが外部と通信した可能性がある」という仮説なら、次のように確認対象を増やせる。

| メモリで得た手掛かり            | 次に照合するもの                                     |
| --------------------- | -------------------------------------------- |
| プロセス名、パス、作成時刻、コマンドライン | Security 4688、Sysmon 1、EDRのプロセス記録、Prefetchなど |
| 接続先IPと所有プロセス          | EDRのネットワーク記録、プロキシ、DNS、ファイアウォールのログなど          |
| スクリプト名やコマンド断片         | 回収したスクリプト、PowerShellログ、関連するファイルのメタデータなど      |
| キャッシュから回収したEVTXやレジストリ | ディスク側から保全した同じアーティファクト                        |
ログが記録されるかは設定に依存する。また、PIDはホストをまたいで一意ではなく、同じホストでも再利用される。ホスト、起動セッション、時刻を揃えて照合する。

VolatilityとMemProcFSの結果が食い違った場合も、すぐ片方を誤りと決めない。列挙する管理構造、終了済みオブジェクトの扱い、欠損ページの処理が違えば結果は変わる。対象のアドレスや抽出データを比較し、対応するWindowsクラッシュダンプなら[WinDbg](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/)でカーネル構造を確認する方法もある。

## おわりに

メモリ調査では、見つけた手掛かりに応じて次の操作を選ぶ。ドメインが見つかればプロセスの領域との関連を調べ、プロセスが絞れればコマンドラインや接続先を確認し、キャッシュにファイルが残っていれば対応するパーサへ渡す。

取得したメモリをプロセス一覧だけで終わらせず、文字列、管理構造、回収ファイルまで使うと、追加調査の候補が増える。そこで得た情報をログやディスクの痕跡と照合し、対象環境で何が起きたのかを確かめていく。
