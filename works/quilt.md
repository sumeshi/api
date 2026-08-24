# ログ解析パッチワーク
ログ解析向けデータ処理ツール「Quilt」の設計思想について

## はじめに

![quilt-logo](https://github.com/user-attachments/assets/06953756-430c-49d3-98bd-11c4b16c8bea)

機器の障害や侵害の調査を行う際など、単発でログを読みたいなというケースは割とあります。  
grepやcut, awkなどの標準コマンドで一生懸命加工してもかまわないのですが、実際やってみるとフォーマット崩れや整形ミスが起きることもしばしば。

もちろん、CSVやJSONを加工するツールは無数にありますが、これ！というのがなく、結局Pythonでスクリプトを書いたりすることが多かったのです。

## 悩み

例えば、次のようなWindowsイベントログがあったとします。

```csv
RecordNumber,EventRecordId,TimeCreated,EventId,Level,Provider,Channel,ProcessId,ThreadId,Computer,ChunkNumber,UserId,MapDescription,UserName,RemoteHost,PayloadData1,PayloadData2,PayloadData3,PayloadData4,PayloadData5,PayloadData6,ExecutableInfo,HiddenRecord,SourceFile,Keywords,ExtraDataOffset,Payload
227443,227443,2016-10-06 01:47:21.9551972,4624,LogAlways,Microsoft-Windows-Security-Auditing,Security,572,1244,WIN-WFBHIBE5GXZ.example.co.jp,2,,Successful logon,-\-," (fe80::4888:72d0:4f06:d1a1)",Target: EXAMPLE\WIN-WFBHIBE5GXZ$,LogonType 3,LogonId: 0x1F9CD52,,,,-,False,C:\Users\sumeshi\Downloads\EvtxECmd\Security.evtx,Audit success,0,"{""EventData"":{""Data"":[{""@Name"":""SubjectUserSid"",""#text"":""S-1-0-0""},{""@Name"":""SubjectUserName"",""#text"":""-""},{""@Name"":""SubjectDomainName"",""#text"":""-""},{""@Name"":""SubjectLogonId"",""#text"":""0x0""},{""@Name"":""TargetUserSid"",""#text"":""S-1-5-18""},{""@Name"":""TargetUserName"",""#text"":""WIN-WFBHIBE5GXZ$""},{""@Name"":""TargetDomainName"",""#text"":""EXAMPLE""},{""@Name"":""TargetLogonId"",""#text"":""0x1F9CD52""},{""@Name"":""LogonType"",""#text"":""3""},{""@Name"":""LogonProcessName"",""#text"":""Kerberos""},{""@Name"":""AuthenticationPackageName"",""#text"":""Kerberos""},{""@Name"":""WorkstationName""},{""@Name"":""LogonGuid"",""#text"":""5c3c5e3e-f799-7dba-5237-223cc0b4dc93""},{""@Name"":""TransmittedServices"",""#text"":""-""},{""@Name"":""LmPackageName"",""#text"":""-""},{""@Name"":""KeyLength"",""#text"":""0""},{""@Name"":""ProcessId"",""#text"":""0x0""},{""@Name"":""ProcessName"",""#text"":""-""},{""@Name"":""IpAddress"",""#text"":""fe80::4888:72d0:4f06:d1a1""},{""@Name"":""IpPort"",""#text"":""49547""}]}}"
227445,227445,2016-10-06 01:47:21.9551972,4624,LogAlways,Microsoft-Windows-Security-Auditing,Security,572,668,WIN-WFBHIBE5GXZ.example.co.jp,2,,Successful logon,-\-," (::1)",Target: EXAMPLE\WIN-WFBHIBE5GXZ$,LogonType 3,LogonId: 0x1F9CE07,,,,-,False,C:\Users\sumeshi\Downloads\EvtxECmd\Security.evtx,Audit success,0,"{""EventData"":{""Data"":[{""@Name"":""SubjectUserSid"",""#text"":""S-1-0-0""},{""@Name"":""SubjectUserName"",""#text"":""-""},{""@Name"":""SubjectDomainName"",""#text"":""-""},{""@Name"":""SubjectLogonId"",""#text"":""0x0""},{""@Name"":""TargetUserSid"",""#text"":""S-1-5-18""},{""@Name"":""TargetUserName"",""#text"":""WIN-WFBHIBE5GXZ$""},{""@Name"":""TargetDomainName"",""#text"":""EXAMPLE""},{""@Name"":""TargetLogonId"",""#text"":""0x1F9CE07""},{""@Name"":""LogonType"",""#text"":""3""},{""@Name"":""LogonProcessName"",""#text"":""Kerberos""},{""@Name"":""AuthenticationPackageName"",""#text"":""Kerberos""},{""@Name"":""WorkstationName""},{""@Name"":""LogonGuid"",""#text"":""5c3c5e3e-f799-7dba-5237-223cc0b4dc93""},{""@Name"":""TransmittedServices"",""#text"":""-""},{""@Name"":""LmPackageName"",""#text"":""-""},{""@Name"":""KeyLength"",""#text"":""0""},{""@Name"":""ProcessId"",""#text"":""0x0""},{""@Name"":""ProcessName"",""#text"":""-""},{""@Name"":""IpAddress"",""#text"":""::1""},{""@Name"":""IpPort"",""#text"":""0""}]}}"
227449,227449,2016-10-06 01:47:21.9551972,4624,LogAlways,Microsoft-Windows-Security-Auditing,Security,572,1244,WIN-WFBHIBE5GXZ.example.co.jp,2,,Successful logon,-\-," (192.168.16.1)",Target: EXAMPLE\WIN-WFBHIBE5GXZ$,LogonType 3,LogonId: 0x1F9CE52,,,,-,False,C:\Users\sumeshi\Downloads\EvtxECmd\Security.evtx,Audit success,0,"{""EventData"":{""Data"":[{""@Name"":""SubjectUserSid"",""#text"":""S-1-0-0""},{""@Name"":""SubjectUserName"",""#text"":""-""},{""@Name"":""SubjectDomainName"",""#text"":""-""},{""@Name"":""SubjectLogonId"",""#text"":""0x0""},{""@Name"":""TargetUserSid"",""#text"":""S-1-5-18""},{""@Name"":""TargetUserName"",""#text"":""WIN-WFBHIBE5GXZ$""},{""@Name"":""TargetDomainName"",""#text"":""EXAMPLE""},{""@Name"":""TargetLogonId"",""#text"":""0x1F9CE52""},{""@Name"":""LogonType"",""#text"":""3""},{""@Name"":""LogonProcessName"",""#text"":""Kerberos""},{""@Name"":""AuthenticationPackageName"",""#text"":""Kerberos""},{""@Name"":""WorkstationName""},{""@Name"":""LogonGuid"",""#text"":""5c3c5e3e-f799-7dba-5237-223cc0b4dc93""},{""@Name"":""TransmittedServices"",""#text"":""-""},{""@Name"":""LmPackageName"",""#text"":""-""},{""@Name"":""KeyLength"",""#text"":""0""},{""@Name"":""ProcessId"",""#text"":""0x0""},{""@Name"":""ProcessName"",""#text"":""-""},{""@Name"":""IpAddress"",""#text"":""192.168.16.1""},{""@Name"":""IpPort"",""#text"":""49548""}]}}"
227453,227453,2016-10-06 01:47:21.9551972,4624,LogAlways,Microsoft-Windows-Security-Auditing,Security,572,692,WIN-WFBHIBE5GXZ.example.co.jp,2,,Successful logon,-\-," (fe80::4888:72d0:4f06:d1a1)",Target: EXAMPLE\WIN-WFBHIBE5GXZ$,LogonType 3,LogonId: 0x1F9CE72,,,,-,False,C:\Users\sumeshi\Downloads\EvtxECmd\Security.evtx,Audit success,0,"{""EventData"":{""Data"":[{""@Name"":""SubjectUserSid"",""#text"":""S-1-0-0""},{""@Name"":""SubjectUserName"",""#text"":""-""},{""@Name"":""SubjectDomainName"",""#text"":""-""},{""@Name"":""SubjectLogonId"",""#text"":""0x0""},{""@Name"":""TargetUserSid"",""#text"":""S-1-5-18""},{""@Name"":""TargetUserName"",""#text"":""WIN-WFBHIBE5GXZ$""},{""@Name"":""TargetDomainName"",""#text"":""EXAMPLE""},{""@Name"":""TargetLogonId"",""#text"":""0x1F9CE72""},{""@Name"":""LogonType"",""#text"":""3""},{""@Name"":""LogonProcessName"",""#text"":""Kerberos""},{""@Name"":""AuthenticationPackageName"",""#text"":""Kerberos""},{""@Name"":""WorkstationName""},{""@Name"":""LogonGuid"",""#text"":""5c3c5e3e-f799-7dba-5237-223cc0b4dc93""},{""@Name"":""TransmittedServices"",""#text"":""-""},{""@Name"":""LmPackageName"",""#text"":""-""},{""@Name"":""KeyLength"",""#text"":""0""},{""@Name"":""ProcessId"",""#text"":""0x0""},{""@Name"":""ProcessName"",""#text"":""-""},{""@Name"":""IpAddress"",""#text"":""fe80::4888:72d0:4f06:d1a1""},{""@Name"":""IpPort"",""#text"":""49549""}]}}"
227511,227511,2016-10-06 01:49:00.9378317,4624,LogAlways,Microsoft-Windows-Security-Auditing,Security,572,1244,WIN-WFBHIBE5GXZ.example.co.jp,2,,Successful logon,-\-," (fe80::4888:72d0:4f06:d1a1)",Target: EXAMPLE\WIN-WFBHIBE5GXZ$,LogonType 3,LogonId: 0x1F9E26B,,,,-,False,C:\Users\sumeshi\Downloads\EvtxECmd\Security.evtx,Audit success,0,"{""EventData"":{""Data"":[{""@Name"":""SubjectUserSid"",""#text"":""S-1-0-0""},{""@Name"":""SubjectUserName"",""#text"":""-""},{""@Name"":""SubjectDomainName"",""#text"":""-""},{""@Name"":""SubjectLogonId"",""#text"":""0x0""},{""@Name"":""TargetUserSid"",""#text"":""S-1-5-18""},{""@Name"":""TargetUserName"",""#text"":""WIN-WFBHIBE5GXZ$""},{""@Name"":""TargetDomainName"",""#text"":""EXAMPLE""},{""@Name"":""TargetLogonId"",""#text"":""0x1F9E26B""},{""@Name"":""LogonType"",""#text"":""3""},{""@Name"":""LogonProcessName"",""#text"":""Kerberos""},{""@Name"":""AuthenticationPackageName"",""#text"":""Kerberos""},{""@Name"":""WorkstationName""},{""@Name"":""LogonGuid"",""#text"":""66bd2719-a7cd-c37d-5aab-12b65ae0735f""},{""@Name"":""TransmittedServices"",""#text"":""-""},{""@Name"":""LmPackageName"",""#text"":""-""},{""@Name"":""KeyLength"",""#text"":""0""},{""@Name"":""ProcessId"",""#text"":""0x0""},{""@Name"":""ProcessName"",""#text"":""-""},{""@Name"":""IpAddress"",""#text"":""fe80::4888:72d0:4f06:d1a1""},{""@Name"":""IpPort"",""#text"":""49551""}]}}"```
```

欲しい列だけ抜き出して、

| TimeCreated                 | Computer                      | UserName | RemoteHost                  | PayloadData2 |
| --------------------------- | ----------------------------- | -------- | --------------------------- | ------------ |
| 2016-10-06 01:47:21.9551972 | WIN-WFBHIBE5GXZ.example.co.jp | -\-      | (fe80::4888:72d0:4f06:d1a1) | LogonType 3  |
| 2016-10-06 01:47:21.9551972 | WIN-WFBHIBE5GXZ.example.co.jp | -\-      | (::1)                       | LogonType 3  |
| 2016-10-06 01:47:21.9551972 | WIN-WFBHIBE5GXZ.example.co.jp | -\-      | (192.168.16.1)              | LogonType 3  |
| 2016-10-06 01:47:21.9551972 | WIN-WFBHIBE5GXZ.example.co.jp | -\-      | (fe80::4888:72d0:4f06:d1a1) | LogonType 3  |
| 2016-10-06 01:49:00.9378317 | WIN-WFBHIBE5GXZ.example.co.jp | -\-      | (fe80::4888:72d0:4f06:d1a1) | LogonType 3  |

くらいにするだけなら、まぁ標準コマンドでもなんとかできますよ。

```bash
$ cut -d, -f 3,10,14,15,17 eventlog.csv
```

じゃあ、こんな表が欲しいときは？

| 時刻(JST)                       | ログオン先                         | ユーザ名 | ログオン元                     | ログオンタイプ |
| ----------------------------- | ----------------------------- | ---- | ------------------------- | ------- |
| 2016-10-06 10:47:21.955197200 | WIN-WFBHIBE5GXZ.example.co.jp | -\-  | fe80::4888:72d0:4f06:d1a1 | 3       |
| 2016-10-06 10:47:21.955197200 | WIN-WFBHIBE5GXZ.example.co.jp | -\-  | ::1                       | 3       |
| 2016-10-06 10:47:21.955197200 | WIN-WFBHIBE5GXZ.example.co.jp | -\-  | 192.168.16.1              | 3       |
| 2016-10-06 10:47:21.955197200 | WIN-WFBHIBE5GXZ.example.co.jp | -\-  | fe80::4888:72d0:4f06:d1a1 | 3       |
| 2016-10-06 10:49:00.937831700 | WIN-WFBHIBE5GXZ.example.co.jp | -\-  | fe80::4888:72d0:4f06:d1a1 | 3       |

やることが一気にふえます。

- 列名を分かりやすい日本語へ揃える
- 元の時刻をUTCからJSTへ変換する
- `LogonType` や `()` など余計な文字列を取り除く

sedやらawkやらなにやらで一生懸命加工してもいいですが、結果が本当にあっているのか不安です（大体はイレギュラーな値を含む行から先が壊れている）。  
じゃあExcelなりなんなりのGUIツールでやればいいかもしれませんが、それを100個とか1,000個とかやるのはつらい。非常に。

それで気づいたのですが、私がやりたいのって抽出・フィルタ・整形だけじゃなく、加工もなんですよね。
調査の途中で思いついた処理をガチャガチャつなげて、最後は人間が読みやすい形で出してほしい。

でもそこまでできるツールってあんまりないなぁと思いました。


## Quilt

そこで、ログ解析に使ういろんなデータ処理をつなげて実行できるようにしたのがQuiltです。
基本構文は次のとおりです。

```bash
$ qlt {INITIALIZER} - {CHAINABLE} - {FINALIZER}
```

| 種類            | 役割          | 例                               |
| ------------- | ----------- | ------------------------------- |
| `INITIALIZER` | データを読み込む処理  | `load`                          |
| `CHAINABLE`   | データを加工する処理  | `select`, `grep`, `sed`, `sort` |
| `FINALIZER`   | 処理結果を出力する処理 | `showtable`, `dump`             |

各処理は `-` で区切って左から右へ処理をつなげていきます。

### CLI

CLIから使うのであれば、下記のように使います。  
ロード、列の選択、整形表示を行っています。

```bash
$ qlt load eventlog.csv - select TimeCreated,Computer,UserName - showtable
```

さっきの表を作りたいなら、次のように書けます。

```bash
$ qlt load eventlog.csv \
  - changetz TimeCreated --from-tz UTC --to-tz Asia/Tokyo --output-format '%Y-%m-%d %H:%M:%S.%f' \
  - sed '^LogonType ' '' --column PayloadData2 \
  - sed '^[[:space:]]*\(' '' --column RemoteHost \
  - sed '\)[[:space:]]*$' '' --column RemoteHost \
  - select TimeCreated,Computer,UserName,RemoteHost,PayloadData2 \
  - renamecol TimeCreated "時刻(JST)" \
  - renamecol Computer "ログオン先" \
  - renamecol UserName "ユーザ名" \
  - renamecol RemoteHost "ログオン元" \
  - renamecol PayloadData2 "ログオンタイプ" \
  - showtable
```

ちょっと長いですが、見ての通り

- `eventlog.csv` をロードする
- `TimeCreated` 列の日時を `UTC` から `Asia/Tokyo` に変換する
- `PayloadData2` 列から `LogonType ` という文字列を取り除く
- `RemoteHost` 列の先頭と末尾に付いている括弧を取り除く
- 調査に必要な列だけを選択する
- 各列を分かりやすい日本語の名前に変更する
- 最後に表形式で表示する

という処理を、上から順番に実行しています。

UNIXのパイプと似ていますが、それぞれの処理結果を逐次受け渡しているわけではありません。  
Quiltでは処理を遅延評価として組み立て、最後にまとめて実行します。

これは、数十GBを超えるようなログでも、不要な列の読み込みや中間データの生成をなるべく減らすためです。  
ただし、sortのようにデータ全体を見る必要がある処理は例外です（なので処理の後ろに書くほうがよい）。


### RUN

また、Quiltにはルールによる一括実行もあります。

```yaml
version: 1

stages:
  - name: formatted
    steps:
      - load: {}

      - changetz:
          column: TimeCreated
          from-tz: UTC
          to-tz: Asia/Tokyo
          output-format: "%Y-%m-%d %H:%M:%S.%f"

      - sed:
          pattern: '^LogonType '
          replacement: ''
          column: PayloadData2

      - sed:
          pattern: '^[[:space:]]*\('
          replacement: ''
          column: RemoteHost

      - sed:
          pattern: '\)[[:space:]]*$'
          replacement: ''
          column: RemoteHost

      - select:
          columns:
            - TimeCreated
            - Computer
            - UserName
            - RemoteHost
            - PayloadData2

      - renamecol:
          old: TimeCreated
          new: "時刻(JST)"

      - renamecol:
          old: Computer
          new: "ログオン先"

      - renamecol:
          old: UserName
          new: "ユーザ名"

      - renamecol:
          old: RemoteHost
          new: "ログオン元"

      - renamecol:
          old: PayloadData2
          new: "ログオンタイプ"

      - showtable: {}
```

Quiltでは、CLIで使っている処理をほぼそのままYAMLのワークフローへ移せます。  
そのため、調査中はワンライナーで試行錯誤、手順が固まったら再利用可能な処理として残す、という使い方ができます。もちろんgit管理したっていい。

```bash
$ qlt run extract-logons.yaml eventlogs.csv
```

実行するときはこう。

また、Quiltには単純なgrepやselectだけでなく、timesliceによる時間範囲の抽出、changetzによるタイムゾーン変換、正規表現から値を列として取り出すextractなど、ログ解析で頻繁に使う加工も用意しています。  
これによって、他プログラムとの行ったり来たりを可能な限り減らしています。

- changetz — タイムゾーン変換
- timeslice — 時間範囲で切る
- bucket — 5分単位などへ丸める
- delta — 前レコードとの差分
- extract — regexのnamed captureを列化
- flatten — JSONLのネストを展開
- parse-size — 10MB 等をbyteへ変換


## ログ解析で使ってみる

せっかくなので、ちょびっと実際のログ解析に近い使い方をしてみます。  

今回はJPCERT/CCが公開している [ログ分析トレーニング バージョン2](https://jpcertcc.github.io/log-analysis-training_v2/) を使います。
この教材はActive Directory環境への攻撃を題材として、Windowsイベントログを使った初動調査を学べるようになっています。

### EVTXをCSVにする

QuiltはWindowsイベントログそのものを解析するツールではありませんので、Eric Zimmerman氏の [EvtxECmd](https://ericzimmerman.github.io/) を使ってEVTXをCSVへ変換します。

```powershell
> EvtxECmd.exe -f Security.evtx --csv ./output --csvf Security.csv
```

### まず構造を見る

最初にヘッダーを確認します。

```bash
$ qlt load Security.csv - headers
┌────┬─────────────────┐
│ #  ┆ Column Name     │
╞════╪═════════════════╡
│ 00 ┆ RecordNumber    │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 01 ┆ EventRecordId   │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 02 ┆ TimeCreated     │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 03 ┆ EventId         │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 04 ┆ Level           │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 05 ┆ Provider        │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 06 ┆ Channel         │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 07 ┆ ProcessId       │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 08 ┆ ThreadId        │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 09 ┆ Computer        │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 10 ┆ ChunkNumber     │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 11 ┆ UserId          │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 12 ┆ MapDescription  │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 13 ┆ UserName        │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 14 ┆ RemoteHost      │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 15 ┆ PayloadData1    │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 16 ┆ PayloadData2    │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 17 ┆ PayloadData3    │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 18 ┆ PayloadData4    │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 19 ┆ PayloadData5    │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 20 ┆ PayloadData6    │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 21 ┆ ExecutableInfo  │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 22 ┆ HiddenRecord    │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 23 ┆ SourceFile      │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 24 ┆ Keywords        │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 25 ┆ ExtraDataOffset │
├╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 26 ┆ Payload         │
└────┴─────────────────┘
```

あとは数件だけ見てみたり。

```bash
$ qlt load Security.csv - head 5 - showtable
shape: (5, 27)
┌────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┬────────┐
│ Record ┆ EventR ┆ TimeCr ┆ EventI ┆ Level  ┆ Provid ┆ Channe ┆ Proces ┆ Thread ┆ Comput ┆ ChunkN ┆ UserId ┆ MapDes ┆ UserNa ┆ Remote ┆ Payloa ┆ Payloa ┆ Payloa ┆ Payloa ┆ Payloa ┆ Payloa ┆ Execut ┆ Hidden ┆ Source ┆ Keywor ┆ ExtraD ┆ Payloa │
│ Number ┆ ecordI ┆ eated  ┆ d      ┆        ┆ er     ┆ l      ┆ sId    ┆ Id     ┆ er     ┆ umber  ┆        ┆ cripti ┆ me     ┆ Host   ┆ dData1 ┆ dData2 ┆ dData3 ┆ dData4 ┆ dData5 ┆ dData6 ┆ ableIn ┆ Record ┆ File   ┆ ds     ┆ ataOff ┆ d      │
│        ┆ d      ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ on     ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ fo     ┆        ┆        ┆        ┆ set    ┆        │
╞════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╪════════╡
│ 2730   ┆ 2730   ┆ 2023-0 ┆ 1102   ┆ Info   ┆ Micros ┆ Securi ┆ 1100   ┆ 1236   ┆ WIN-M4 ┆ 0      ┆ null   ┆ Event  ┆ WIN-M4 ┆ null   ┆ SID:   ┆ null   ┆ null   ┆ null   ┆ null   ┆ null   ┆ null   ┆ false  ┆ C:\Use ┆ 0x4020 ┆ 0      ┆ {"User │
│        ┆        ┆ 9-06   ┆        ┆        ┆ oft-Wi ┆ ty     ┆        ┆        ┆ 9DEV3A ┆        ┆        ┆ log    ┆ 9DEV3A ┆        ┆ (S-1-5 ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ rs\sum ┆ 000000 ┆        ┆ Data": │
│        ┆        ┆ 01:05: ┆        ┆        ┆ ndows- ┆        ┆        ┆        ┆ F9N    ┆        ┆        ┆ cleare ┆ F9N\Ad ┆        ┆ -21-95 ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ eshi\D ┆ 000000 ┆        ┆ {"LogF │
│        ┆        ┆ 50.212 ┆        ┆        ┆ Eventl ┆        ┆        ┆        ┆        ┆        ┆        ┆ d      ┆ minist ┆        ┆ 778768 ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ ownloa ┆        ┆        ┆ ileCle │
│        ┆        ┆ 3480   ┆        ┆        ┆ og     ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ rator  ┆        ┆ 4-1770 ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ ds\Evt ┆        ┆        ┆ ared": │
│        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ 014200 ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ xECmd\ ┆        ┆        ┆ {"Subj │
│        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ -900…  ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ Sec…   ┆        ┆        ┆ ect…   │
├╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┤
│ 2731   ┆ 2731   ┆ 2023-0 ┆ 4688   ┆ LogAlw ┆ Micros ┆ Securi ┆ 4      ┆ 92     ┆ WIN-M4 ┆ 0      ┆ null   ┆ A new  ┆ WIN-M4 ┆ null   ┆ Parent ┆ PID:   ┆ Parent ┆ Mandat ┆ Target ┆ null   ┆ C:\Win ┆ false  ┆ C:\Use ┆ Audit  ┆ 0      ┆ {"Even │
│        ┆        ┆ 9-06   ┆        ┆ ays    ┆ oft-Wi ┆ ty     ┆        ┆        ┆ 9DEV3A ┆        ┆        ┆ proces ┆ 9DEV3A ┆        ┆ proces ┆ 0x12D8 ┆ PID:   ┆ ory    ┆ User:  ┆        ┆ dows\S ┆        ┆ rs\sum ┆ succes ┆        ┆ tData" │
│        ┆        ┆ 01:05: ┆        ┆        ┆ ndows- ┆        ┆        ┆        ┆ F9N    ┆        ┆        ┆ s has  ┆ F9N\Ad ┆        ┆ s: C:\ ┆        ┆ 0xD38  ┆ label: ┆ -\-    ┆        ┆ ystem3 ┆        ┆ eshi\D ┆ s      ┆        ┆ :{"Dat │
│        ┆        ┆ 50.257 ┆        ┆        ┆ Securi ┆        ┆        ┆        ┆        ┆        ┆        ┆ been   ┆ minist ┆        ┆ Window ┆        ┆        ┆ SECURI ┆        ┆        ┆ 2\lpre ┆        ┆ ownloa ┆        ┆        ┆ a":[{" │
│        ┆        ┆ 8399   ┆        ┆        ┆ ty-Aud ┆        ┆        ┆        ┆        ┆        ┆        ┆ create ┆ rator  ┆        ┆ s\Syst ┆        ┆        ┆ TY_MAN ┆        ┆        ┆ move.e ┆        ┆ ds\Evt ┆        ┆        ┆ @Name" │
│        ┆        ┆        ┆        ┆        ┆ iting  ┆        ┆        ┆        ┆        ┆        ┆        ┆ d      ┆        ┆        ┆ em32\S ┆        ┆        ┆ DATORY ┆        ┆        ┆ xe     ┆        ┆ xECmd\ ┆        ┆        ┆ :"Subj │
│        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ ys…    ┆        ┆        ┆ _HIG…  ┆        ┆        ┆        ┆        ┆ Sec…   ┆        ┆        ┆ ect…   │
├╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┤
│ 2732   ┆ 2732   ┆ 2023-0 ┆ 4688   ┆ LogAlw ┆ Micros ┆ Securi ┆ 4      ┆ 92     ┆ WIN-M4 ┆ 0      ┆ null   ┆ A new  ┆ WIN-M4 ┆ null   ┆ Parent ┆ PID:   ┆ Parent ┆ Mandat ┆ Target ┆ null   ┆ C:\Win ┆ false  ┆ C:\Use ┆ Audit  ┆ 0      ┆ {"Even │
│        ┆        ┆ 9-06   ┆        ┆ ays    ┆ oft-Wi ┆ ty     ┆        ┆        ┆ 9DEV3A ┆        ┆        ┆ proces ┆ 9DEV3A ┆        ┆ proces ┆ 0xD38  ┆ PID:   ┆ ory    ┆ User:  ┆        ┆ dows\S ┆        ┆ rs\sum ┆ succes ┆        ┆ tData" │
│        ┆        ┆ 01:05: ┆        ┆        ┆ ndows- ┆        ┆        ┆        ┆ F9N    ┆        ┆        ┆ s has  ┆ F9N\Ad ┆        ┆ s: C:\ ┆        ┆ 0xDB4  ┆ label: ┆ -\-    ┆        ┆ ystem3 ┆        ┆ eshi\D ┆ s      ┆        ┆ :{"Dat │
│        ┆        ┆ 50.277 ┆        ┆        ┆ Securi ┆        ┆        ┆        ┆        ┆        ┆        ┆ been   ┆ minist ┆        ┆ Window ┆        ┆        ┆ SECURI ┆        ┆        ┆ 2\conh ┆        ┆ ownloa ┆        ┆        ┆ a":[{" │
│        ┆        ┆ 8581   ┆        ┆        ┆ ty-Aud ┆        ┆        ┆        ┆        ┆        ┆        ┆ create ┆ rator  ┆        ┆ s\Syst ┆        ┆        ┆ TY_MAN ┆        ┆        ┆ ost.ex ┆        ┆ ds\Evt ┆        ┆        ┆ @Name" │
│        ┆        ┆        ┆        ┆        ┆ iting  ┆        ┆        ┆        ┆        ┆        ┆        ┆ d      ┆        ┆        ┆ em32\l ┆        ┆        ┆ DATORY ┆        ┆        ┆ e      ┆        ┆ xECmd\ ┆        ┆        ┆ :"Subj │
│        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ pr…    ┆        ┆        ┆ _HIG…  ┆        ┆        ┆        ┆        ┆ Sec…   ┆        ┆        ┆ ect…   │
├╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┤
│ 2733   ┆ 2733   ┆ 2023-0 ┆ 4798   ┆ LogAlw ┆ Micros ┆ Securi ┆ 652    ┆ 3016   ┆ WIN-M4 ┆ 0      ┆ null   ┆ A      ┆ WIN-M4 ┆ null   ┆ Target ┆ Subjec ┆ Caller ┆ Caller ┆ null   ┆ null   ┆ null   ┆ false  ┆ C:\Use ┆ Audit  ┆ 0      ┆ {"Even │
│        ┆        ┆ 9-06   ┆        ┆ ays    ┆ oft-Wi ┆ ty     ┆        ┆        ┆ 9DEV3A ┆        ┆        ┆ user's ┆ 9DEV3A ┆        ┆ : WIN- ┆ tLogon ┆ Proces ┆ Proces ┆        ┆        ┆        ┆        ┆ rs\sum ┆ succes ┆        ┆ tData" │
│        ┆        ┆ 01:05: ┆        ┆        ┆ ndows- ┆        ┆        ┆        ┆ F9N    ┆        ┆        ┆ local  ┆ F9N\Ad ┆        ┆ M49DEV ┆ Id:    ┆ sName: ┆ sId:   ┆        ┆        ┆        ┆        ┆ eshi\D ┆ s      ┆        ┆ :{"Dat │
│        ┆        ┆ 50.738 ┆        ┆        ┆ Securi ┆        ┆        ┆        ┆        ┆        ┆        ┆ group  ┆ minist ┆        ┆ 3AF9N\ ┆ 0x2282 ┆ C:\Win ┆ 0x12D8 ┆        ┆        ┆        ┆        ┆ ownloa ┆        ┆        ┆ a":[{" │
│        ┆        ┆ 8823   ┆        ┆        ┆ ty-Aud ┆        ┆        ┆        ┆        ┆        ┆        ┆ member ┆ rator  ┆        ┆ Admini ┆ F      ┆ dows\S ┆        ┆        ┆        ┆        ┆        ┆ ds\Evt ┆        ┆        ┆ @Name" │
│        ┆        ┆        ┆        ┆        ┆ iting  ┆        ┆        ┆        ┆        ┆        ┆        ┆ ship   ┆ (S-1-5 ┆        ┆ strato ┆        ┆ ystem3 ┆        ┆        ┆        ┆        ┆        ┆ xECmd\ ┆        ┆        ┆ :"Targ │
│        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ was    ┆ -21…   ┆        ┆ r (…   ┆        ┆ 2\…    ┆        ┆        ┆        ┆        ┆        ┆ Sec…   ┆        ┆        ┆ etU…   │
│        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ enu…   ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        │
├╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌┤
│ 2734   ┆ 2734   ┆ 2023-0 ┆ 4739   ┆ LogAlw ┆ Micros ┆ Securi ┆ 652    ┆ 3016   ┆ WIN-M4 ┆ 0      ┆ null   ┆ null   ┆ null   ┆ null   ┆ null   ┆ null   ┆ null   ┆ null   ┆ null   ┆ null   ┆ null   ┆ false  ┆ C:\Use ┆ Audit  ┆ 0      ┆ {"Even │
│        ┆        ┆ 9-06   ┆        ┆ ays    ┆ oft-Wi ┆ ty     ┆        ┆        ┆ 9DEV3A ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ rs\sum ┆ succes ┆        ┆ tData" │
│        ┆        ┆ 01:05: ┆        ┆        ┆ ndows- ┆        ┆        ┆        ┆ F9N    ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ eshi\D ┆ s      ┆        ┆ :{"Dat │
│        ┆        ┆ 50.743 ┆        ┆        ┆ Securi ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ ownloa ┆        ┆        ┆ a":[{" │
│        ┆        ┆ 3207   ┆        ┆        ┆ ty-Aud ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ ds\Evt ┆        ┆        ┆ @Name" │
│        ┆        ┆        ┆        ┆        ┆ iting  ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ xECmd\ ┆        ┆        ┆ :"Doma │
│        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆        ┆ Sec…   ┆        ┆        ┆ inP…   │
└────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┴────────┘
```

大量の列があって読みづらいですね。必要な列だけに絞りましょう。

```bash
$ qlt load Security.csv \
  - select TimeCreated,EventId,Computer,UserName,RemoteHost,MapDescription \
  - head 10 \
  - showtable
shape: (8+, 6) [showing first 8 rows]
┌─────────────────────────────┬─────────┬─────────────────┬──────────────────────────────────────────┬────────────┬──────────────────────────────────────────┐
│ TimeCreated                 ┆ EventId ┆ Computer        ┆ UserName                                 ┆ RemoteHost ┆ MapDescription                           │
╞═════════════════════════════╪═════════╪═════════════════╪══════════════════════════════════════════╪════════════╪══════════════════════════════════════════╡
│ 2023-09-06 01:05:50.2123480 ┆ 1102    ┆ WIN-M49DEV3AF9N ┆ WIN-M49DEV3AF9N\Administrator            ┆ null       ┆ Event log cleared                        │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 2023-09-06 01:05:50.2578399 ┆ 4688    ┆ WIN-M49DEV3AF9N ┆ WIN-M49DEV3AF9N\Administrator            ┆ null       ┆ A new process has been created           │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 2023-09-06 01:05:50.2778581 ┆ 4688    ┆ WIN-M49DEV3AF9N ┆ WIN-M49DEV3AF9N\Administrator            ┆ null       ┆ A new process has been created           │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 2023-09-06 01:05:50.7388823 ┆ 4798    ┆ WIN-M49DEV3AF9N ┆ WIN-M49DEV3AF9N\Administrator (S-1-5-21… ┆ null       ┆ A user's local group membership was enu… │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 2023-09-06 01:05:50.7433207 ┆ 4739    ┆ WIN-M49DEV3AF9N ┆ null                                     ┆ null       ┆ null                                     │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 2023-09-06 01:05:50.7864262 ┆ 4738    ┆ WIN-M49DEV3AF9N ┆ WIN-M49DEV3AF9N\Administrator            ┆ null       ┆ A user account was changed               │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 2023-09-06 01:05:50.7864678 ┆ 4724    ┆ WIN-M49DEV3AF9N ┆ WIN-M49DEV3AF9N\Administrator (S-1-5-21… ┆ null       ┆ An attempt was made to reset an account… │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 2023-09-06 01:05:50.8005315 ┆ 4738    ┆ WIN-M49DEV3AF9N ┆ WIN-M49DEV3AF9N\Administrator            ┆ null       ┆ A user account was changed               │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ ⋮                           ┆ ⋮       ┆ ⋮               ┆ ⋮                                        ┆ ⋮          ┆ ⋮                                        │
└─────────────────────────────┴─────────┴─────────────────┴──────────────────────────────────────────┴────────────┴──────────────────────────────────────────┘
```

ま、多少は見やすいかな。


### VPNからのログオンを探す

実践編Hands-on 1では、VPNのIPアドレスとして `10.10.100.254` が与えられ、そこから行われたログオンを特定していきます。
設問は時刻、アカウント、攻撃手法などをログから埋めていく形式です。

ログオン成功を見るなら、まずEvent ID 4624です。このあたりは [君たちはどうWindowsイベントログを調査するか](https://sumeshi.github.io/posts/knowledges/windows-eventlog-analysis-101) を参照。

```bash
$ qlt load Security.csv \
  - isin EventId 4624 \
  - grep 10.10.100.254 \
  - select TimeCreated,EventId,Computer,UserName,RemoteHost,PayloadData1,PayloadData2 \
  - sort TimeCreated \
  - showtable
shape: (4, 7)
┌─────────────────────────────┬─────────┬───────────────────────────┬──────────────────────┬─────────────────────────────────┬───────────────────────────┬──────────────┐
│ TimeCreated                 ┆ EventId ┆ Computer                  ┆ UserName             ┆ RemoteHost                      ┆ PayloadData1              ┆ PayloadData2 │
╞═════════════════════════════╪═════════╪═══════════════════════════╪══════════════════════╪═════════════════════════════════╪═══════════════════════════╪══════════════╡
│ 2023-10-11 23:44:25.0293453 ┆ 4624    ┆ ad-win-C.handsonlab.local ┆ -\-                  ┆ external-win1-C (10.10.100.254) ┆ Target: handsonlab\domadm ┆ LogonType 3  │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 2023-10-11 23:44:27.4776285 ┆ 4624    ┆ ad-win-C.handsonlab.local ┆ -\-                  ┆ external-win1-C (10.10.100.254) ┆ Target: handsonlab\domadm ┆ LogonType 3  │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 2023-10-11 23:44:32.1753309 ┆ 4624    ┆ ad-win-C.handsonlab.local ┆ handsonlab\ad-win-C$ ┆ ad-win-C (10.10.100.254)        ┆ Target: handsonlab\domadm ┆ LogonType 10 │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 2023-10-11 23:44:32.1753759 ┆ 4624    ┆ ad-win-C.handsonlab.local ┆ handsonlab\ad-win-C$ ┆ ad-win-C (10.10.100.254)        ┆ Target: handsonlab\domadm ┆ LogonType 10 │
└─────────────────────────────┴─────────┴───────────────────────────┴──────────────────────┴─────────────────────────────────┴───────────────────────────┴──────────────┘
```

`10.10.100.254` から `handsonlab\domadm` へのRDPログオンですね。  
さっくりみつかりました。

### 見つけた情報から深掘りする

実際のログ解析では、一発の検索で答えが出ることはあまりありません。

一つのイベントから、

- アカウント
- IPアドレス
- ホスト名
- 発生時刻
- Event ID

といった情報を拾い、それを次の検索条件に使います。

例えば怪しいアカウントが分かったなら、

```bash
$ qlt load Security.csv \
  - grep TARGET_USER \
  - sort TimeCreated \
  - showtable
```

さらに発生時間帯が分かったなら、

```bash
$ qlt load Security.csv \
  - timeslice TimeCreated --start START_TIME --end END_TIME \
  - grep TARGET_USER \
  - sort TimeCreated \
  - showtable
```

と絞っていきます。

実際に、この教材ではイベントID 4769を手掛かりとして不審なサービスチケット要求を追えるログが含まれています。  
こうして調査中に見つけた観点をYAMLへ落とし、別の端末やデータにも繰り返し適用できるようにする。こうした「調査で得た知見を再利用可能な処理へ落とす」という流れは、Detection Engineeringにもつながる話ですね。


## おわりに

フォレンジックやマルウェア解析では、用途ごとにさまざまなツールを使います。EVTXを読むならEvtxECmd、別の証跡ならまた別のパーサ、といった具合です。
Quiltはそれらすべてを置き換えたいのではなく、さまざまなツールが出力するCSVやJSONLに対して小さな処理を組み合わせて、調査者が見たい形へ持っていくことが目的です。

Quiltという名前も、そういう小さな処理を継ぎ接ぎしていくイメージから付けています。
ログ解析はパッチワークや！

おわり
