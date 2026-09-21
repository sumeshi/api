# NTFS脊髄剣
Windowsイメージファイルから直接ファイルをぶっこ抜く方法。ボボボボボボボボボボボ

## はじめに

デジタルフォレンジックでは、調査時の操作によって調査対象環境を変更（汚染）してしまわないように、ディスクイメージという形式でデータを保全することが一般的です。

ところが、イメージは基本的にシステム容量とほぼ同じサイズになるため、1TBのPCを保全すると1TBかそれに近いサイズのバカデカファイルが出来上がります。

調査時にはこれをマウントして、内部構造を確認できるソフトウェアでファイルを取り出して...ということをするのですが、めんどくさい。

Windowsのシステムを対象とする場合、一般的には、[FTK Imager](https://www.exterro.com/digital-forensics-software/ftk-imager) とか [Arsenal Image Mounter](https://arsenalrecon.com/products/arsenal-image-mounter) でやることが多いです。

![ftkimager](https://github.com/user-attachments/assets/7da6f2ca-9b57-49e0-b720-22b8fcdf3859)

これらのツールはGUIで非常に使いやすい一方、大量のイメージを機械的に処理することには向いていません。  
調査対象が数十台とかあると非常にしんどいので、イメージファイルから直接ファイルを検索してぶっこ抜くツールをつくりました。

https://github.com/sumeshi/ntfsdump

https://github.com/sumeshi/ntfsfind


## 対応イメージファイル

対応しているイメージ形式は下記の通りです。  
自動識別機構を入れているので、使用時に意識することはありませんが。

- `dd`(raw)
- `E01`(EnCase)
- `VHD` / `VHDX`(`AVHDX` 差分ディスクを含む)
- `VMDK`(スプリットエクステント、スナップショット差分チェーンを含む)
- VMwareのVMディレクトリ、`.vmx`、`.vmsd` ファイル

ファイルシステムは **NTFS**、パーティションテーブルは GPT / MBR 両方に対応しています。

`v3.2.0` 以前では、仮想マシンイメージが差分スナップショットで構成されている場合、一度RAWディスクに変換してから使う必要がありました。が、現在は基本的に不要です。

```
> VBoxManage clonehd --format raw vmdiskimage.vmdk imagefile.raw
```


## インストール方法

### コンパイル済みバイナリ

Windows, Linux(Ubuntu) 用のバイナリを GitHub で公開しています。  
遷移先のページからバイナリをダウンロードして実行してください。

- [ntfsdump - Releases](https://github.com/sumeshi/ntfsdump/releases)
- [ntfsfind - Releases](https://github.com/sumeshi/ntfsfind/releases)

Python コードを実行ファイルにコンパイルする [**Nuitka**](https://nuitka.net/) を利用しているため、一部の **アンチウイルスソフトウェアによって検知** される場合があります。

気になる方は後述のPyPIからインストールするか、隔離された仮想環境上で実行してください。


### PyPI からのインストール

Python3.13 以上をサポートしています。
インストールする際は下記のコマンドを実行してください。

```bash
$ pip install ntfsdump ntfsfind
```


## ファイルの検索

[`ntfsfind`](https://github.com/sumeshi/ntfsfind) は、ディスクイメージ上のMFTレコードを直接解析してファイルパスを検索するツールです。  
たとえば、`.evtx` 拡張子のファイルを探したいならこう。

```powershell
> ntfsfind.exe .\example.E01 ".*\.evtx"
/Windows/System32/winevt/Logs/Setup.evtx
/Windows/System32/winevt/Logs/Microsoft-Windows-WindowsUpdateClient%4Operational.evtx
/Windows/System32/winevt/Logs/Microsoft-Windows-Winlogon%4Operational.evtx
/Windows/System32/winevt/Logs/Microsoft-Windows-WindowsBackup%4ActionCenter.evtx
/Windows/System32/winevt/Logs/Microsoft-Windows-TerminalServices-LocalSessionManager%4Admin.evtx
/Windows/System32/winevt/Logs/Microsoft-Windows-TerminalServices-LocalSessionManager%4Operational.evtx
/Windows/System32/winevt/Logs/Microsoft-Windows-PrintService%4Admin.evtx
...
```

検索クエリには正規表現を使用できます。  
パスセパレータはWindows / Linuxともに / へ正規化されます。


### メタデータフィルタ

MFTレコードが持つメタデータで条件を絞り込むこともできます。

```powershell
# 2025年以降に作成された1MB以上の evtx,exe
> ntfsfind.exe .\example.E01 --extension evtx,exe --size ">=1MB" --created ">=2025-01-01"
/$Recycle.Bin/S-1-5-21-2425377081-3129163575-2985601102-1000/$RJEMT64.exe
/$Recycle.Bin/S-1-5-21-2425377081-3129163575-2985601102-1000/$RJEMT64.exe:Zone.Identifier
/Users/informant/Desktop/Download/ccsetup504.exe
/Users/informant/Desktop/Download/ccsetup504.exe:Zone.Identifier
```

```powershell
# System32 配下の削除済みエントリ
> ntfsfind.exe .\example.E01 --path /Windows/System32 --deleted-only
/Windows/System32/LogFiles/WMI/RtBackup/EtwRTEventlog-Security.etl
/Windows/System32/LogFiles/WMI/RtBackup/EtwRTUBPM.etl
/Windows/System32/LogFiles/WMI/RtBackup/EtwRTMsMpPsSession7.etl
```

```powershell
# 5MB以上の実行ファイルをテーブル表示
> ntfsfind.exe .\example.E01 -e exe --size ">=5MB" --output-format table
72096  ALLOC-FILE    68.3 MiB  2015-03-23 19:55:47  2015-03-23 19:56:53  -     /Users/informant/Downloads/icloudsetup.exe
72096  ALLOC-FILE    68.3 MiB  2015-03-23 19:55:47  2015-03-23 19:56:53  -     /Users/informant/Downloads/icloudsetup.exe:Zone.Identifier
75186  DELETED-FILE  5.1 MiB   2015-03-25 14:48:28  2015-03-25 14:48:28  -     /Users/informant/Desktop/Download/ccsetup504.exe
75186  DELETED-FILE  5.1 MiB   2015-03-25 14:48:28  2015-03-25 14:48:28  -     /Users/informant/Desktop/Download/ccsetup504.exe:Zone.Identifier
```

代表的なフィルタ条件は下記の通り。複数指定時はAND条件で結合されます。

| オプション | 説明 |
| --- | --- |
| `--extension, -e` | 拡張子で絞り込み(カンマ区切り、例: `-e evtx,exe`) |
| `--path` | パス前方一致 |
| `--size` | サイズ(例: `>=10MB`, `<1KB`, `4KB..10MB`) |
| `--created / --modified / --accessed` | タイムスタンプ(例: `2024-01-01..2024-12-31`) |
| `--timestamp-source` | 比較に使うタイムスタンプ(`$STANDARD_INFORMATION` か `$FILE_NAME`) |
| `--deleted-only / --allocated-only` | 削除済み/アロケート済みエントリ |
| `--files-only / --dirs-only` | ファイル/ディレクトリ |
| `--ads-only / --no-ads` | 代替データストリームの有無 |
| `--attributes` | ファイル属性(例: `hidden,system,readonly`) |

### 出力フォーマット

`--output-format` で出力形式を選べます。

| フォーマット | 説明 |
| --- | --- |
| `text` | 1行1パス。デフォルト。パイプ向け |
| `json` | JSON Lines |
| `csv` | CSV |
| `table` | 人間向けのテーブル表示 |


### MFTのエクスポート

同じイメージに対して何度も検索する場合、毎回巨大なイメージにアクセスするよりも、先に$MFTだけ取り出してしまうと効率的です。

```powershell
# イメージからMFTをエクスポート
> ntfsfind.exe --out-mft C:\tmp\my_mft.bin .\example.E01

# 以降はダンプしたMFTファイルを直接検索
> ntfsfind.exe C:\tmp\my_mft.bin ".evtx"
```


## ファイルのダンプ

`ntfsdump` は、パスを指定してイメージファイルからファイルを直接抽出するツールです。

```powershell
# 単一ファイル
> ntfsdump.exe .\example.E01 "/hoge.txt"

# ディレクトリ配下を再帰的にダンプ
> ntfsdump.exe -o .\dump .\example.E01 /Windows/System32/winevt/Logs

# 代替データストリーム(ADS)
> ntfsdump.exe -o .\dump .\example.E01 '/$Extend/$UsnJrnl:$J'
```

出力先には、元のディレクトリ構造が再現されます(`./dump/Windows/System32/winevt/Logs/System.evtx` など)。  
フラットにしたいときは `--flat` オプション指定で同一フォルダに吐き出せます。


### ntfsfindとの連携

ntfsfindで検索した結果のファイルを直接 ntfsdump で出力することもできます。  
一度テキストファイルへリダイレクトして、必要なパスだけ残してから ntfsdump に食わせてもよいです。

```powershell
> ntfsfind.exe .\example.E01 ".*\.evtx" | ntfsdump.exe -o .\dump .\example.E01
```


## VMスナップショットからの検索・ダンプ

VMware仮想マシンについては、VMディレクトリを直接指定できます。  
スナップショットを指定して、その時点のNTFSを検索・抽出することもできます。

```powershell
# スナップショットの一覧を表示
> ntfsfind.exe .\WindowsVM --list-snapshots
ID  NAME          CREATED              PARENT
1   Initialized   2026-09-01 12:33:43  -
5   NetConnect    2026-09-02 02:25:08  1
6   PrepareTools  2026-09-10 17:46:35  5
7   SetConfigs    2026-09-25 00:51:39  5

# スナップショット5時点のNTFSを検索
> ntfsfind.exe .\WindowsVM --snapshot 5 ".*\.evtx"
```

```powershell
# スナップショット5時点のSYSTEMハイブを抽出
> ntfsdump.exe .\WindowsVM --snapshot 5 /Windows/System32/config/SYSTEM
```

`--snapshot` で指定するIDは、VMwareで管理されているスナップショットUIDです。  
表示名での指定はできないので、`--list-snapshots` の表示IDから指定してあげてください。

`--snapshot` を省略した場合は、VMの **現在の状態** を読みます。


### 仮想ディスクの選択

VMに複数の仮想ディスクが接続されている場合は、`--list-disks` で確認して `--disk` で選択します。

```powershell
> ntfsdump.exe .\WindowsVM --list-disks
ID  NODE     SIZE     VMDK
0   nvme0:0  100 GiB  Windows10_22H2(x64).vmdk
1   scsi0:1  500 GiB  Data.vmdk

# 2台目の仮想ディスクを読む
> ntfsdump.exe .\WindowsVM --disk 1 /Evidence
```

ディスクが1つだけなら自動選択されます。  
複数ある場合には誤って別ディスクを読むことを避けるために明示的に選択することを求めます。

差分VMDKを直接指定することもできます。

```powershell
> ntfsdump.exe .\WindowsVM\Windows-000003.vmdk /$MFT
```


### Hyper-Vのチェックポイント

Hyper-Vも同様に、`AVHDX`(差分ディスク)を直接指定することができます。

```powershell
> ntfsfind.exe .\HyperVM\Disk_0.avhdx ".*\.evtx"
```

差分VHDXファイル単体を渡した場合でも、ディスクリプタから親チェーンを自動で辿るので、ベースディスク + 差分すべてを1つの論理ディスクとして扱えます。

なお、現時点で VMwareのようなスナップショット一覧表示は実装していないので、AVHDXファイルを直接指定してあげてください。


## おわりに

このツールに限ったことではないですが、「普段使うことはないけど、いざというときにバイナリ1つ持っておけば便利！」的なツールを目指しています。

もし気に入っていただければ、ツールボックスの片隅に置いといていただければ嬉しいです。

おわり
