# 君たちはどうWindowsイベントログを調査するか
Windowsイベントログ調査のぼんやりメモ。  

---

ソースが載っていない部分はAI-Poweredなので注意。随時追記。

## 前提

最低限これは頭に入れておけ。

- イベントログはあくまで残存している記録
  - そもそも記録されないイベントも多い。監査ポリシー依存。
  - 古いイベントログは上書きされてしまう。ローテート設定次第。
  - 侵害者によって削除される場合もある。ランサムウェア事案だと削除が自動化されているケースも多い。
  - ログの改ざん可能性は捨てきれない。EVTXには整合性確認のためのチェックサム機構があるが、暗号学的な改ざん防止機構ではない。他のアーティファクトと突合して、侵害者の動きについて仮説を立てながら調査を進めることがアナリストには必要
- どうノイズを減らすか？が重要
  - 平時にどう記録されるかを知ることがノイズを減らす第一歩。
  - いいから全部見ろって？ムリムリ。

---

## 目的

ログを調査して何を確かめたいのか、わからないまま解析をすることはできない。  
大きな目的からブレイクダウンして、調査の目的を明確にすることが重要。

- 侵害は発生したのか？（検知）
- どこから侵入したのか？（初期侵入）
- どのように拡大したのか？（横展開・持続化）
- 何をされたのか？（活動内容）
- 何が影響を受けたのか？（影響範囲）
- なぜそれが可能だったのか？（原因）

ま、大概の場合は「侵害があったかどうか」あったなら「情報漏洩があったかどうか」そんなんだろう。  

じゃあそれを調べるためには何を見る必要があるのか？ ログに記録されているのはあくまで具象的なイベントなので、それらの連なりから何が起きたか、確度の高い推測をする必要がある。

たとえば「情報漏洩があったか」を知りたいとして、イベントログに「情報漏洩しました」とは書かれるはずもない。  
見るべきは、不審なログオン、大量のファイルアクセス、圧縮、転送、外部通信、リムーバブルメディア接続、管理共有へのアクセスなど、漏洩につながる可能性のある行動の痕跡である。

---

## スコープを切る

ログ調査なんて際限がない（マジに）。  
リソースは突っ込んだら突っ込んだだけ消えるので、最初にスコープを切ることが重要。  
調査結果を見て範囲が足りなかったら、その時考えれば良いのだ。

- どのホストのログを調べるのか？（サーバー？クライアント？）
- どの期間のログを調べるのか？（侵害があった期間？）
- どのログを調べるのか？（Security？System？Application？PowerShell？TaskScheduler？）
- どのユーザーに関するログを調べるのか？（管理者？一般ユーザー？サービスアカウント？）
- どのイベントIDを調べるのか？（ログオンイベント？プロセス作成イベント？ネットワーク接続イベント？）

---

## 調査の勘所

とりあえず主要なイベントIDは一通り見ておく。  
あとはこれらをもとに仮説を立ててからかな。。

Sysmonがあれば細かいイベントも見れるのだが、ほとんどの環境では入っていない。期待するな。

基本的には [MS Learn](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4624) などの公式ドキュメントを参考にするのがよい。

[Security Log Encyclopedia](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/) も非常に参考になる。Undocumentedなイベントも多いので、気になるイベントIDがあったらここで調べるとよい。

[大和セキュリティによる、DFIRと脅威ハンティングのためのWindowsイベントログ設定のガイド](https://github.com/Yamato-Security/EnableWindowsLogSettings/blob/main/README-Japanese.md) などを見ても良い。デフォルトのイベントログ設定がどうなっているかなどに言及がある。

### 主要なイベントID

#### ログオン・認証
- Security.evtx
    - [4624](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): ユーザーがコンピュータに正常にログオン
    - [4625](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): ログオンに失敗
    - [4634](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): ユーザーのログオフ処理が完了
    - [4647](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): ユーザーのログオフ処理を開始
    - [4648](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): ユーザーが既に別のユーザーとしてログオンしている状態で、明示的な資格情報を使用してコンピュータにログオンすることに成功
    - [4672](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4672): 新しいログオンに特権が割り当てられた
    - [4673](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4673): 特権サービスが呼び出された
    - [4674](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4674): 特権オブジェクトに対する操作が試行された
    - [4778](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4778): ターミナルサーバーセッションに再接続
    - [4779](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4779): ユーザーがログオフせずにターミナルサーバーセッションを切断

- Microsoft-Windows-TerminalServices-LocalSessionManager\Operational.evtx
    - [21](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: セッションログオン成功
    - [22](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: シェル開始通知を受信
    - [23](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: セッションログオフ成功
    - [24](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: セッション切断
    - [25](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: セッション再接続成功
    - [39](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: 別セッションによるセッション切断
    - [40](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: セッション切断（理由付き）

- Microsoft-Windows-TerminalServices-RDPClient\Operational.evtx
    - [1024](https://jpcertcc.github.io/ToolAnalysisResultSheet_jp/): RDP ClientActiveX がサーバーに接続しようとしています
    - [1026](https://jpcertcc.github.io/ToolAnalysisResultSheet_jp/): RDP ClientActiveX が切断しました 

- Microsoft-Windows-TerminalServices-RemoteConnectionManager\Operational.evtx
    - [1146](https://gist.github.com/MHaggis/138c6bf563bacbda4a2524f089773706): Remote Desktop Services: セッション初期化に成功しました
    - [1147](https://gist.github.com/MHaggis/138c6bf563bacbda4a2524f089773706): Remote Desktop Services: セッション接続に成功しました
    - [1148](https://gist.github.com/MHaggis/138c6bf563bacbda4a2524f089773706): Remote Desktop Services: セッション接続に失敗しました
    - [1149](https://jpcertcc.github.io/ToolAnalysisResultSheet_jp/): Remote Desktop Services: ユーザー認証に成功しました

- System.evtx
    - [9009](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): デスクトップウィンドウマネージャが終了(RDPセッションの切断など)

イベントID4624の [LogonType](https://learn.microsoft.com/ja-jp/windows-server/identity/securing-privileged-access/reference-tools-logon-types) は非常に重要。基本的には3(Network), 10(RemoteInteractive), 11(CachedInteractive)あたりを中心に見ることになる。

| Logon Type | 説明 |
| --- | --- |
| [0](https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-logonsession) | SYSTEMアカウントでのみ使用されるログオンタイプ。 |
| 1 | 情報なし。[Reddit](https://www.reddit.com/r/windows/comments/18wqzs8/what_is_or_was_logon_type_1/)では、NT 3.x時代の名残では？というウワサ。 |
| [2](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | Interactive（対話型）。ユーザーが端末を対話的に使用するログオン。コンソールログオン、RUNAS、リモートシェル、KVM、Lights-Outカード経由の操作など。 |
| [3](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | Network（ネットワーク）。ネットワーク経由で対象にアクセスするログオン。`net use` による共有アクセス、リモートコンピュータへのMMCスナップイン、PowerShell WinRM、PsExec、Remote Registry、Remote Desktop Gateway、脆弱性スキャナ、IIS統合Windows認証、SQL Windows認証など。`LogonUser` はこのログオンタイプの資格情報をキャッシュしない。再利用可能な資格情報は原則として宛先LSAセッションに残らないが、Kerberos委任が有効な場合などは例外に注意。PsExecは明示的な認証情報を指定した場合、Network + Interactive の複数ログオンセッションを作ることがある。 |
| [4](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | Batch（バッチ）。ユーザーの直接操作なしに、ユーザーの代理でプロセスを実行するためのログオン。スケジュールタスクなど。メール/Webサーバーなど、一度に多くの平文認証試行を処理する高性能サーバー用途でも使われる。`LogonUser` はこのログオンタイプの資格情報をキャッシュしない。ただしスケジュールタスクでは、パスワードがLSA Secretとしてディスク上に保存される場合がある。 |
| [5](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | Service（サービス）。サービスによるログオン。対象アカウントには「サービスとしてログオン」権限が必要。サービス実行用の資格情報は再利用可能な資格情報としてLSAセッションに残り得る。また、パスワードがLSA Secretとしてディスク上に保存される場合がある。 |
| [6](https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-logonsession) | Proxy（プロキシログオン）。公式にはプロキシタイプのログオンと説明される。[Reddit](https://www.reddit.com/r/windows/comments/18wqzs8/what_is_or_was_logon_type_1)では、内部/開発ビルド専用では？というウワサ。 |
| [7](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | Unlock（ロック解除）。ワークステーションのロック解除。GINA DLLなど、対話的に端末を使用しているユーザーのロック解除を記録するためのログオンタイプ。 |
| [8](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | NetworkCleartext。認証パッケージ内に名前とパスワードを保持し、サーバーがクライアントを偽装しながら他のネットワークサーバーへ接続できる。IIS Basic認証、CredSSPを用いたPowerShell WinRMなど。再利用可能な資格情報が宛先側に残るため、資格情報窃取リスクが高い。 |
| [9](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | NewCredentials。現在のトークンを複製し、外向きのネットワーク接続に別の資格情報を指定する。ローカルでは元のIDのまま、ネットワーク接続時だけ指定した資格情報を使う。`RUNAS /NETWORK`など。再利用可能な資格情報はLSAセッションに残り得るため注意。 |
| [10](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | RemoteInteractive。リモートかつ対話型のターミナルサービスセッション。Remote Desktopなど。RDPログオン成功だけでなく、ログオン失敗でもRemoteInteractiveとして記録されることがある。再利用可能な資格情報が宛先LSAセッションに残るため、侵害端末への特権アカウントRDPは危険。 |
| [11](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | CachedInteractive。ネットワークにアクセスせず、キャッシュされた資格情報を使用した対話型ログオン。ドメインコントローラへ問い合わせて認証したログオンとは限らない。 |
| [12](https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-logonsession) | CachedRemoteInteractive。RemoteInteractiveと同様。内部監査用途。 |
| [13](https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-logonsession) | CachedUnlock。キャッシュされた資格情報を使用したワークステーションのロック解除。 |

---

### プロセス実行
- 4688: 新しいプロセスの作成

ほとんどの場合、デフォでは記録されない。

---

### 横展開・リモート操作
- SMBClient.evtx
    - 5140: ネットワーク共有アクセス
    - 5145: 共有オブジェクト詳細アクセス
- SMBServer.evtx
    - 5140: ネットワーク共有アクセス
    - 5145: 共有オブジェクト詳細アクセス

- 7045: サービス作成
- 4697: サービスインストール
- 4698: スケジュールタスク作成

👉 見るポイント:
- 管理共有（ADMIN$, C$ など）
- サービス名 / 実行バイナリ
- タスク実行内容

---

### ネットワーク
- 5156: 接続許可（WFP）

👉 注意:
- 監査設定依存
- 有効化環境ではログ量が非常に多い

👉 見るポイント:
- 実行プロセス
- 宛先IP / ポート

---

### PowerShell / スクリプト
- 4103: Module Logging
- 4104: Script Block Logging

---

### ログ改ざん・痕跡削除
- 1102: Securityログクリア
- 104: その他ログクリア（System）

---

## ユースケースごとの見方

### 侵害の有無を調べたい

「異常な振る舞いがあるか」を見る。

- 不審なログオン（4624/4625）
- 普段いない時間帯・端末からのアクセス
- 管理者権限の取得（4672）
- 不審なプロセス実行（4688）
- サービス・タスク作成（7045 / 4698）
- ログ消去（1102）

---

### 侵害の範囲・経路を調べたい

「どこからどこへ広がったか」を見る。

- 4624 の Source → Destination を追う
- Logon ID でセッションを紐付ける
- 5140 / 5145 で共有アクセスを確認
- 7045 / 4697 / 4698 でリモート実行を確認
- 4688 で実行されたプロセスを追う

---

### 侵害の手口を調べたい

「どの技術が使われたか」を特定する。

例:

- PsExec系  
  → 7045 + 5140 + 4688
- RDP  
  → 4624 (Logon Type 10) + TerminalServicesログ
- PowerShell Remoting  
  → 4624 + 4104 + WinRMログ
- WMI / タスク  
  → 4698 + 4688

### 情報漏洩の有無を調べたい

「漏洩につながる行動」を見る。

- 不審なログオン
- ファイルアクセス（共有・ローカル）
- 圧縮・アーカイブ
- 外部通信（5156 など）
- リムーバブルメディア
- 転送ツール・クラウド利用

JPCERT/CCが公開する、[個別ツール別分析結果](https://jpcertcc.github.io/ToolAnalysisResultSheet_jp/) なども参考にするとよい。

---

## ツールの使い分け

### 非アナリスト

あなたが現場のSEとか機器管理者で、最初にログを見て侵害の可能性があるか確かめろって言われたら、まずはこれ。

Windows標準のイベントビューアを使え。  
GUIで検索やフィルタができるので、最初の確認には十分使える。

ただし、CSVでエクスポートしたものをCSIRTやアナリストに渡すな。  
マジでやめろ。  
調査する側からすると、まず整形から始める必要があり、余計な手間になる。

調査担当に渡すなら、CSVではなく `.evtx` 形式で保全すること。

イベントビューアの「すべてのイベントを名前を付けて保存」でもよいし、`wevtutil epl` でエクスポートしてもよい。  
対象ファイルは `C:\Windows\System32\winevt\Logs\` 以下にあるが、単純コピーだけに頼るより、可能なら正規のエクスポート手順で取得したほうが安全。

例:

```powershell
wevtutil epl Security C:\Temp\Security.evtx
wevtutil epl System C:\Temp\System.evtx
wevtutil epl Application C:\Temp\Application.evtx
````

とにかく、CSVで渡すな。
最低でも `.evtx` で渡せ。

---

### アナリスト

あなたがCSIRTとかフォレンジック調査員なら、やり方は好みのものを選択すればよい。
ただし、組織内であらかじめ合意しておくとよい。

たとえあなたが逆張りオタクで人と違うツールを使うのが好きだとしても、そのツールによって証拠性が担保できるということは、あなたが説明しなければならないことを忘れるな。

「自分が慣れているから」だけでは弱い。
そのツールが何をどうパースして、どんな形式で出力し、元ログとどう対応できるのかを説明できる必要がある。

---

### 取得・保全

| ツール        | 概要                                             | 備考                            |
| ---------- | ---------------------------------------------- | ----------------------------- |
| イベントビューア   | Windows標準のGUIツール。イベントログの閲覧・保存に利用できる。           | 非アナリストの初動確認には十分使える。           |
| wevtutil   | Windows標準のコマンドラインツール。イベントログのエクスポートや情報取得ができる。   | `.evtx` 形式での取得に使える。           |
| PowerShell | Windows標準のスクリプト環境。Get-WinEventなどでイベントログを取得できる。 | 柔軟性は高いが、使い方によっては取得漏れや整形ミスに注意。 |

---

### パース・変換

| パーサ        | 概要                                              | 備考                                     |
| ---------- | ----------------------------------------------- | -------------------------------------- |
| PowerShell | Windows標準のスクリプト言語。イベントログの抽出・整形に利用できる。           | 柔軟性が高く、カスタマイズが容易。                      |
| Log Parser | Microsoftが提供していたコマンドラインツール。SQLライクなクエリでログを解析できる。 | 公式ツールだが、開発は終了している。                     |
| EvtxECmd   | EVTXファイルを解析してCSVなどに出力するコマンドラインツール。              | Eric Zimmerman氏のツール群の一つ。DFIR用途で広く使われる。 |
| evtx2es    | EVTXファイルをElasticsearchにインポートするためのコマンドラインツール。    | 大量のイベントログを横断検索したい場合に便利。                |
| evtx2json  | EVTXファイルをJSON形式に変換するツール。                        | JSONで後続処理したい場合に便利。                     |

---

### 閲覧・クエリ

| ツール                | 概要                       | 備考                          |
| ------------------ | ------------------------ | --------------------------- |
| Event Log Explorer | イベントログの閲覧と分析に特化したGUIツール。 | フィルタリングや検索がしやすい。大量ログ確認にも便利。 |
| Log Parser Studio  | Log ParserのGUIフロントエンド。   | クエリの作成や結果確認がしやすい。           |
| Log Parser Lizard  | Log Parser系のGUIツール。      | SQLライクにイベントログなどを確認できる。      |

---

### 分析・整形

| 分析・整形ツール                | 概要                                    | 備考                                   |
| ----------------------- | ------------------------------------- | ------------------------------------ |
| Excel                   | Microsoftが提供する表計算ソフト。ログの分析や可視化に利用できる。 | 多くの人が使い慣れているため、手軽に利用できる。ただし大量ログには弱い。 |
| LibreOffice Calc        | オープンソースの表計算ソフト。ログの分析や可視化に利用できる。       | Excelがない環境でも使える。ただし巨大CSVは厳しい。        |
| Timeline Explorer       | CSVなどをタイムライン形式で閲覧するツール。イベントの時系列把握に強い。 | Eric Zimmerman氏のツール群の一つ。大量CSVの確認に便利。 |
| Quilter CSV / quilt系ツール | CSVの確認・絞り込み・フラグ付けなどに使うツール。            | 大量CSVのトリアージ用途。                       |
| Splunk                  | 商用のログ管理・分析プラットフォーム。大量ログの検索・可視化に強い。    | ライセンスや利用条件は導入時点で要確認。                 |
| Elasticsearch + Kibana  | 検索エンジンと可視化ツールの組み合わせ。大量ログを効率的に分析できる。   | ログのインデックス作成と横断検索に強い。                 |
| Grafana                 | 可視化ツール。各種データソースのダッシュボード化に利用できる。       | 単体でイベントログを調査するツールではない。検索基盤と組み合わせる。   |

---

## 実際の流れ

### 1. 目的の明確化

何を最優先に調査したいのか考えろ。

偉い人に「すべてを調査しろ」と言われたとしても、それは絶対に無理だ。
無限に時間とリソースを溶かしていいなら別だが、ログ調査が必要なケースでは、大抵の場合そうではないはずだ。

まず決めるべきことは以下。

* 侵害の有無を見たいのか
* 侵害範囲を見たいのか
* 横展開経路を見たいのか
* 情報漏洩の可能性を見たいのか
* 原因を見たいのか
* 復旧に必要な情報を見たいのか

目的が違えば、見るログもイベントIDも変わる。

---

### 2. スコープの設定

ヒアリングや過去の作業記録から、大丈夫だった可能性の高いポイントと、そうでないことを確認したいポイントを洗い出して、どこから調査するか決めろ。

実際にログ調査を他社に依頼するのであれば、一週間から一ヶ月単位で絞り込むと、あまり時間がかからなくて良いぞ。

スコープとして決めるべきものは以下。

* 対象ホスト
* 対象期間
* 対象ログ
* 対象ユーザー
* 対象イベントID
* 優先して確認する仮説

最初から広げすぎるな。
ただし、狭くしすぎても見逃す。
最初は合理的に狭く切り、必要に応じて広げる。

---

### 3. ログの収集と保全

これはマジで真っ先にやれ。

この記事を読んでいる間にも、新しいイベントは記録され、古いイベントは上書きされている。
迷っている暇があったら、まず保全しろ。

Securityログなんか数日残ってればいいほうだし、環境によっては数時間分しか残っていないこともある。
監査ポリシーが強めに設定されているサーバや、多数のログオン・アクセスが発生する環境では、想像以上の速度で古いログが上書きされる。

最低限、以下は保全しておきたい。

* Security
* System
* Application
* Windows PowerShell
* Microsoft-Windows-PowerShell/Operational
* Microsoft-Windows-TaskScheduler/Operational
* Microsoft-Windows-TerminalServices-LocalSessionManager/Operational
* Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational
* Microsoft-Windows-WinRM/Operational
* Microsoft-Windows-SMBClient/Security
* Microsoft-Windows-SMBServer/Security

もちろん全部取れるなら取ればいい。
ただし、まず重要なものを押さえる。

---

### 4. ログの解析と仮説の立案

ログを見て、何が起きた可能性があるのか、どんな攻撃手法が考えられるのか、どんな被害が考えられるのか、仮説を立てろ。

その仮説を検証するために、どんなイベントを探せばいいのか、どんなアーティファクトを突合すればいいのか、調べろ。

どう調べればいいかわからなければ、AIに聞いてもいい。
ただ、ちょっと詳細な質問をするなら精度はそんなに高くないことは頭の片隅に入れておけ。

AIに聞くなら、こう聞いたほうがよい。

```text
Windowsイベントログで、PsExecによる横展開を調べたい。
Security、System、PowerShellログがある。
見るべきイベントIDと、確認すべきフィールドを教えて。
```

こういう聞き方ならまだ使える。

逆に、

```text
このイベントログから侵害の有無を調べて
```

みたいな雑な聞き方をすると、雑な答えが返ってくる。

---

### 5. 仮説の検証と追加調査

ログオン一覧とか見て、ある程度何が起きたかわかったら、侵害者は何を目的に行動していたのか仮説を立てろ。
そして仮説を検証するために、ログをさらに掘り下げて、関連するイベントを探せ。

必要なら他のアーティファクトも突合して、仮説の確度を高めろ。

* イベントログ
* MFT
* USN Journal
* Prefetch
* Amcache
* Shimcache
* SRUM
* ブラウザ履歴
* EDRログ
* プロキシログ
* VPNログ
* ファイルサーバー監査ログ

仮説が間違っていることがわかったら、別の仮説を立てて、また検証しろ。
これの繰り返しだ。

ログ調査は「正解のイベントIDを探す作業」ではない。
仮説を立て、検証し、間違っていたら修正する作業である。

---

### 6. 結果のまとめと報告

調査の結果、何が起きた可能性があるのか、これからどうすべきかを報告しろ。
あるいは、それがアナリストの仕事ではないなら、どこから先は別の担当領域なのかを明確にしろ。

報告時には、以下を説明できるようにしておく。

* 調査の目的
* 調査スコープ
* 調査対象ログ
* 立てた仮説
* 仮説を検証するために確認したイベント
* 確認できたこと
* 確認できなかったこと
* 追加調査が必要なこと
* 判断の根拠
* 判断の限界

質問されて即答できないことがあってもよい。
ただし、なぜ確認していないのか、追加確認するなら何を見るのかは説明できなければならない。

ついでに、報告する内容は絞ってもよい。
というのも、調べて立てた仮説すべてを報告する必要はまったくない。

相手を不安にさせたり、混乱させないためにも、最も確度の高い仮説と、それを検証するために行った調査の内容を中心に報告すれば十分だ。
あとは質問されたら答えられるよう資料を揃えておけば良い。


