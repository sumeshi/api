# 君たちはどうWindowsイベントログを調査するか
Windowsイベントログ調査のぼんやりメモ。思い出したら追記。

<img width="960" height="540" alt="win" src="https://github.com/user-attachments/assets/519ecf99-7a14-40a1-8e17-cc8690da1e86" />

## はじめに

Windowsイベントログ調査をするときの日本語ドキュメント的なものが欲しかったので。  
主に **インシデント対応目線** でのお話。

調査の前提として、以下のことは最低限頭に入れておくこと。

- イベントログは残存しているものしかわからない。
    - デフォで記録されないイベントも結構多い。[監査ポリシー](https://learn.microsoft.com/ja-jp/windows-server/identity/ad-ds/plan/security-best-practices/audit-policy-recommendations?tabs=winclient) 依存。
    - 古いイベントログは上書きされてしまう。[ローテート設定](https://learn.microsoft.com/ja-jp/security-updates/planningandimplementationguide/19869709) 次第。
    - 侵害者によってログが削除されてしまう場合もある。ランサムウェア事案などだと削除も自動化されていて、ログだけではなにもわからんケースも多い。
    - ログの改ざん可能性も捨てきれない。 EVTX には [整合性確認のためのチェックサム機構](https://github.com/libyal/libevtx/blob/main/documentation/Windows%20XML%20Event%20Log%20(EVTX).asciidoc) があるが、暗号学的な改ざん防止機構ではない。単純な削除よりコストが高いので見かけるケースはあまりないが。
    - いずれにせよ、他のアーティファクトと突合して、侵害者の動きを仮説を立てながら調査する視点がアナリストには必要
- 大量のログからどうノイズを減らすか？が重要
    - 1台あたりの容量で、クライアントなら数GB, サーバなら数十～数百GBを見る覚悟がいる。
    - 平時にどう記録されるかを知ることがノイズを減らす第一歩。
    - いいからとりあえず全部見ろって？殴りますよ。

あとログの保全。これはマジで真っ先にやれ。

この記事を読んでいる間にも、新しいイベントは記録され、古いイベントは上書きされている。
Securityログなんか数日残ってればいいほうだし、環境によっては数時間分しか残っていないこともある。
迷っている暇があったらまず保全。


## 目的

**何を確かめたいのかわからないままログ調査をすることはできない。**  
大きな目的からブレイクダウンして、調査の目的を明確にすることが重要。

5W1Hでもいいし、[Cyber Kill Chain](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html) でもなんでもいいから、なんらかのフレームワークをベースに整理するとヌケモレが発生しづらい。

- 侵害は発生したのか？（検知）
- どこから侵入したのか？（初期侵入）
- どのように拡大したのか？（横展開・持続化）
- 何をされたのか？（活動内容）
- 何が影響を受けたのか？（影響範囲）
- なぜそれが可能だったのか？（原因）

とかね。

ま、調査したい理由なんて大概の場合は「侵害があったかどうか」、あったなら「情報漏洩があったかどうか」とかそんなんだろう。  
じゃあそれを調べるためには何を見る必要があるのか？ログに記録されているのはあくまで具象的なイベントなので、それらの連なりから何が起きたか、状況証拠から確度の高い推測をする必要がある。  

たとえば **「情報漏洩があったか」を知りたいとして、イベントログに「情報漏洩しました」とは書かれるはずもない。** 見るべきは、不審なログオン、ファイルアクセス、外部通信、リムーバブルメディア接続、管理共有へのアクセス、不審ツールの実行など、漏洩につながる可能性のある行動の痕跡である。


## スコープを切る

ログ調査なんて際限がない（マジに）。  
**リソースは突っ込んだら突っ込んだだけ消える** ので、最初にスコープを切ることが重要。  

調査結果を見て範囲が足りなかったら、その時考えれば良いのだ。

- どのホストのログを調べるのか？（サーバー？クライアント？）
- どの期間のログを調べるのか？（侵害があった期間？）
- どのログを調べるのか？（Security？System？Application？PowerShell？TaskScheduler？）
- どのユーザーに関するログを調べるのか？（管理者？一般ユーザー？サービスアカウント？）
- どのイベントIDを調べるのか？（ログオンイベント？プロセス作成イベント？ネットワーク接続イベント？）


## 調査の勘所

とりあえず主要なイベントIDは一通り見ておく。  
あとはこれらをもとに仮説を立ててから、何を調べるべきか考えればよい。

ログの解釈は経験だよりなところもあるが。。基本的には [MS Learn](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4624) などの信頼できる公式ドキュメントを参考にするのがよい。  

あとは [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) があれば細かいイベントも見れるのだが、、、ほとんどの環境では入っていない。期待してはいけない。

[Security Log Encyclopedia](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/) も非常に参考になる。Undocumentedなイベントも多いので、気になるイベントIDがあったらここで調べるとよい。

それでも見つからないときには、[MyEventlog](https://www.myeventlog.com/search/show/980) で探すと見つかることもある。EventIDはびっくりすることに一意の識別番号ではない狂気の仕様なのだが、このサイトだとちゃんとProvider/Channelごとに検索結果を表示してくれる。

さらにニッチなやつは [exabeam Documentation](https://docs.logrhythm.com/?l=en) とか見ると書いてあったりする。それでもないならググりまくるしかない。

使われたツールや技術がわかっているなら、JPCERT/CCが公開する、[個別ツール別分析結果](https://jpcertcc.github.io/ToolAnalysisResultSheet_jp/) などを参照しながら痕跡を探すとよい。

大和セキュリティによる、[DFIRと脅威ハンティングのためのWindowsイベントログ設定のガイド](https://github.com/Yamato-Security/EnableWindowsLogSettings/blob/main/README-Japanese.md) などを見ても良い。[Sigma](https://github.com/sigmahq/sigma) をもとに、デフォルトのイベントログ設定がどうなっているかなどに言及がある。可能なら元の [Sigma](https://github.com/sigmahq/sigma) ルール定義を見ればよいが、大量にありすぎてとても覚えられるものではない。


## ログの解析と仮説の立案

ある程度ログを見たら、何が起きた可能性があるのか、どんな攻撃手法が考えられるのか、どんな被害が考えられるのか仮説を立てる。  
その仮説を検証するために、さらにどんなイベントを探せばいいのか、どんなアーティファクトを突合すればいいのか深堀りして調べていく。

どう調べればいいかわからなければ、AIに聞いてもいい。
ただ、ちょっと詳細な質問をするなら精度はそんなに高くないことは頭の片隅に入れておこう。

## 報告
調査の結果、何が起きた可能性があるのか、これからどうすべきかを報告する。  
あるいは、それがアナリストの仕事ではないなら、どこから先が別の担当領域なのかを明確にしておくとよい。**インシデント対応ではヤバい仕事の押し付け合いになる**こともままあるので、あらかじめネゴっておくとスムーズ。

また、報告先へ渡す資料については、何を聞かれてもすべて説明できるくらい脳に叩き込んでおかなければならない。
それ以外については、聞かれたら答えられるように手元に資料を揃えておけばよい。

セカンドオピニオンがついている場合や、お客様がセキュリティに知見がある場合、◯◯は確認しましたか？など聞かれる場合もある。リソースには限りがあるので全てに備える必要はないが、**なぜその項目を確認していないのかは説明できるようにしておくこと。** これこれこういう理由で調査対象からは外しました。契約範囲外です。とかね。

また、報告する内容は絞ってよい。というか、**調査結果から考えられる仮説すべてを報告する必要はまったくない。**
相手を不必要に不安にさせたり、混乱させないためにも、最も確度の高い仮説と、それを検証するために行った調査の内容を中心に報告すれば十分。


## Appendix. 主要なイベントID
網羅的に見るのではなく、**とっかかりを見つける** という観点で私は下記を見ることが多い。  
怪しいイベントが分かれば、その前後を一生懸命見ていけば未知のイベントでも役立つものがあるはず。

### ログオン・認証
誰が、いつ、どこから、どの端末へログオンしたのか(横展開)を見る。  
記録されるイベントがログオン元、ログオン先どっちに記録されるのかも意識するように。

- Security.evtx
    - [4624](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): ユーザーがコンピュータに正常にログオン
    - [4625](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): ログオンに失敗
    - [4634](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): ユーザーのログオフ処理が完了
    - [4647](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): ユーザーのログオフ処理を開始
    - [4648](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events): ユーザーが既に別のユーザーとしてログオンしている状態で、明示的な資格情報を使用してコンピュータへのログオンに成功
    - [4672](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4672): 新しいログオンに特権が割り当てられた
    - [4673](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4673): 特権サービスが呼び出された
    - [4674](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4674): 特権オブジェクトに対する操作が試行された

イベントID4624の [LogonType](https://learn.microsoft.com/ja-jp/windows-server/identity/securing-privileged-access/reference-tools-logon-types) は非常に重要。基本的には3(Network), 10(RemoteInteractive) を中心に、9(NewCredentials), 12(CachedRemoteInteractive)あたりを補助的に見ることになる。

イベントID4625のログオン失敗は、[ログオンエラーコード](https://www.softwareisac.jp/wp/active-directory-domain-controller-security-log/) を見ることでなぜ失敗したのかがわかる。ユーザ名がそもそもないとか、ユーザ名はあっているけどパスワードが間違っているとか。。これによって攻撃者がその行為を行った時点でどの程度の情報を把握していたのかわかる。例えば、一発ログオン成功しているならどっかでパスワードダンプしているか、同じものを使いまわしているとか。

| Logon Type | 説明 |
| --- | --- |
| [0](https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-logonsession) | **SYSTEMアカウントでのみ使用されるログオンタイプ。** |
| 1 | **情報なし。** [Reddit](https://www.reddit.com/r/windows/comments/18wqzs8/what_is_or_was_logon_type_1/)では、NT 3.x時代の名残では？というウワサ。 |
| [2](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **Interactive（対話型）。ユーザーが端末を対話的に使用するログオン。** コンソールログオン、RUNAS、リモートシェル、KVM、Lights-Outカード経由の操作, IIS Basic Auth(6.0未満) など。 |
| [3](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **Network（ネットワーク）。ネットワーク経由で対象にアクセスするログオン。** `net use` による共有アクセス、リモートコンピュータへのMMCスナップイン、PowerShell WinRM、PsExec、Remote Registry、Remote Desktop Gateway、脆弱性スキャナ、IIS統合Windows認証、SQL Windows認証など。`LogonUser` はこのログオンタイプの資格情報をキャッシュしない。再利用可能な資格情報は原則として宛先LSAセッションに残らないが、Kerberos委任が有効な場合などは例外に注意。PsExecは明示的な認証情報を指定した場合、Network + Interactive の複数ログオンセッションを作ることがある。 |
| [4](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **Batch（バッチ）。ユーザーの直接操作なしに、ユーザーの代理でプロセスを実行するためのログオン。** スケジュールタスクなど。メール/Webサーバーなど、一度に多くの平文認証試行を処理する高性能サーバー用途でも使われる。`LogonUser` はこのログオンタイプの資格情報をキャッシュしない。ただしスケジュールタスクでは、パスワードがLSA Secretとしてディスク上に保存される場合がある。 |
| [5](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **Service（サービス）。サービスによるログオン。** 対象アカウントには「サービスとしてログオン」権限が必要。サービス実行用の資格情報は再利用可能な資格情報としてLSAセッションに残り得る。また、パスワードがLSA Secretとしてディスク上に保存される場合がある。 |
| [6](https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-logonsession) | **Proxy（プロキシログオン）。公式にはプロキシタイプのログオンと説明される。** [Reddit](https://www.reddit.com/r/windows/comments/18wqzs8/what_is_or_was_logon_type_1)では、内部/開発ビルド専用では？というウワサ。 |
| [7](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **Unlock（ロック解除）。ワークステーションのロック解除。** GINA DLLなど、対話的に端末を使用しているユーザーのロック解除を記録するためのログオンタイプ。 |
| [8](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **NetworkCleartext（ネットワーク平文認証）。認証パッケージ内に名前とパスワードを保持し、サーバーがクライアントを偽装しながら他のネットワークサーバーへ接続できる。** IIS Basic認証(6.0以降)、CredSSPを用いたPowerShell WinRMなど。再利用可能な資格情報が宛先側に残るため、資格情報窃取リスクが高い。 |
| [9](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **NewCredentials（新しい資格情報）。現在のトークンを複製し、外向きのネットワーク接続に別の資格情報を指定する。** ローカルでは元のIDのまま、ネットワーク接続時だけ指定した資格情報を使う。`RUNAS /NETWORK`など。再利用可能な資格情報はLSAセッションに残り得るため注意。 |
| [10](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **RemoteInteractive（リモート対話型）。リモートかつ対話型のターミナルサービスセッション。Remote Desktopなど。** RDPログオン成功だけでなく、4625のログオン失敗でもRemoteInteractiveとして記録されることがある。再利用可能な資格情報が宛先LSAセッションに残るため、侵害端末への特権アカウントRDPは危険。 |
| [11](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/basic-audit-logon-events) | **CachedInteractive（キャッシュされた対話型）。ネットワークにアクセスせず、キャッシュされた資格情報を使用した対話型ログオン。** ドメインコントローラへ問い合わせて認証したログオンとは限らない。 |
| [12](https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-logonsession) | **CachedRemoteInteractive（キャッシュされたリモート対話型）。RemoteInteractiveと同様。内部監査用途。** |
| [13](https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-logonsession) | **CachedUnlock（キャッシュされたロック解除）。キャッシュされた資格情報を使用したワークステーションのロック解除。** |


### Kerberos / NTLM
DC上で、どのアカウントがどのサービスに認証しようとしたかを見る。  
不正常事案など端末単体の場合は基本見ないな。

- Security.evtx
    - [4768](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4768): Kerberos 認証チケット（TGT）が要求された
    - [4769](https://learn.microsoft.com/ja-jp/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration): Kerberos サービスチケットが要求された
    - [4770](https://learn.microsoft.com/ja-jp/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration): Kerberos サービスチケットが更新された
    - [4771](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4771): Kerberos 事前認証に失敗した
    - [4776](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4776): NTLM認証が試行された


### RDP / TerminalServices
RDP接続の試行、認証、セッション開始、切断、再接続など。  
Securityイベントログが消されていても、RDP関連が残っていることは割とあるのでこっちから流れを追える場合もある。

- Security.evtx
    - [4778](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4778): ターミナルサーバーセッションに再接続
    - [4779](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4779): ユーザーがログオフせずにターミナルサーバーセッションを切断
- Microsoft-Windows-TerminalServices-LocalSessionManager%4Operational.evtx
    - [21](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: セッションログオン成功
    - [22](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: シェル開始通知を受信
    - [23](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: セッションログオフ成功
    - [24](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: セッション切断
    - [25](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: セッション再接続成功
    - [39](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: 別セッションによるセッション切断
    - [40](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): Remote Desktop Services: セッション切断（理由付き）
- Microsoft-Windows-TerminalServices-RDPClient%4Operational.evtx
    - [1024](https://jpcertcc.github.io/ToolAnalysisResultSheet_jp/): RDP ClientActiveX がサーバーに接続しようとしています
    - [1026](https://jpcertcc.github.io/ToolAnalysisResultSheet_jp/): RDP ClientActiveX が切断しました 
- Microsoft-Windows-TerminalServices-RemoteConnectionManager%4Operational.evtx
    - [261](https://jpcertcc.github.io/ToolAnalysisResultSheet_jp/): リスナー RDP-Tcp で接続を受信しました
    - [1146](https://gist.github.com/MHaggis/138c6bf563bacbda4a2524f089773706): Remote Desktop Services: セッション初期化に成功しました
    - [1147](https://gist.github.com/MHaggis/138c6bf563bacbda4a2524f089773706): Remote Desktop Services: セッション接続に成功しました
    - [1148](https://gist.github.com/MHaggis/138c6bf563bacbda4a2524f089773706): Remote Desktop Services: セッション接続に失敗しました
    - [1149](https://jpcertcc.github.io/ToolAnalysisResultSheet_jp/): Remote Desktop Services: ユーザー認証に成功しました (ただし、これ単体でRDPログオン成功と断定できるものではない)
- System.evtx
    - [9009](https://ponderthebits.com/2018/02/windows-rdp-related-event-logs-identification-tracking-and-investigation/): デスクトップウィンドウマネージャが終了(RDPセッションの切断など)


### プロセス実行
何が、誰の権限で、どこから起動されたのかなど。  
無効になっていることが多いのであれば儲けたと思えば良い。

- Security.evtx
    - [4688](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4688): 新しいプロセスが作成された
    - [4689](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4689): プロセスが終了した

4688は「Audit Process Creation」が有効な場合に記録される。  
さらにコマンドラインは別途「Include command line in process creation events」を有効化しないと記録されない。


### PowerShell / Scripting
PowerShellの起動、実行内容、スクリプトブロック、ログなど。

- Windows PowerShell.evtx
    - [400](https://www.myeventlog.com/search/show/970): PowerShellエンジン開始
    - [403](https://www.myeventlog.com/search/show/971): PowerShellエンジン終了
- Microsoft-Windows-PowerShell%4Operational.evtx
    - [4103](https://www.myeventlog.com/search/show/977): モジュールログ
    - [4104](https://www.myeventlog.com/search/show/980): PowerShell スクリプトブロックログ
    - [4105](https://docs.logrhythm.com/devices/docs/evid-4105-4106): スクリプトブロック実行開始
    - [4106](https://docs.logrhythm.com/devices/docs/evid-4105-4106): スクリプトブロック実行完了

これも設定依存かな。あればなるべく見る。  
定期実行されているものとかもあるので、普段の運用把握と合わせてノイズは頑張って減らす。


### WMI
WMI経由の操作や永続化に使われるイベントフィルター、コンシューマーなど。  
ノイズ多めなので一生懸命見なくてもよい。

- Microsoft-Windows-WMI-Activity%4Operational.evtx
    - [5857](https://docs.logrhythm.com/devices/docs/evid-5857-operation-started): WMI操作が開始された
    - [5858](https://learn.microsoft.com/ja-jp/troubleshoot/windows-client/system-management-components/wmi-activity-event-5858-logged-with-resultcode-0x80041032): WMIクエリ/操作の失敗・結果コード


### 横展開・リモート操作
共有アクセス、管理共有、SMB経由のファイル操作やリモート操作の痕跡を見る。  
結構悪用される事例を見るので要チェック。

- Security.evtx
    - [5140](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5140): ネットワーク共有オブジェクトにアクセス
    - [5142](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5142): ネットワーク共有オブジェクトが追加
    - [5143](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5143): ネットワーク共有オブジェクトが変更
    - [5144](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5144): ネットワーク共有オブジェクトが削除
    - [5145](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5145): 共有オブジェクト詳細アクセス
    - [5168](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5168): SMB/SMB2 の SPN チェックに失敗

### サービス変更
サービスの作成、起動、停止、起動種別変更を見る。  
PsExec系、永続化、EDR/AV停止、バックアップ製品停止などの痕跡が残っていることがある。

- Security.evtx
    - [4697](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4697): サービスインストール
- System.evtx
    - [7036](https://learn.microsoft.com/en-us/answers/questions/262317/whats-the-audit-event-id-for-windows-service-start): サービスが開始/停止状態に遷移した
    - [7040](https://learn.microsoft.com/en-us/answers/questions/262317/whats-the-audit-event-id-for-windows-service-start): サービスの開始の種類が変更された
    - [7045](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-7045): サービスがシステムにインストールされた
  

### スケジュールタスク
タスクの作成、削除、有効化、無効化、更新を見る。  
マルウェアの永続化や遅延実行でよく使われるので、タスク名、実行コマンド、作成者、作成時刻を見て怪しいものがないかチェックしておく。

- Security.evtx
    - [4698](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4698): スケジュールタスク作成
    - [4699](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4699): スケジュールタスクが削除された
    - [4700](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4700): スケジュールタスクが有効化された
    - [4701](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4701): スケジュールタスクが無効化された
    - [4702](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4702): スケジュールタスクが更新された


### アカウント・グループ・ポリシー変更
アカウント作成、削除、有効化、パスワード変更、グループ追加、ポリシー変更を見る。  
割と長期で入られてた事例とかではよく変なアカウントが追加されてたりする。

- Security.evtx
    - [4719](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4719): システム監査ポリシーが変更された
    - [4720](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4720): ユーザーアカウントが作成された
    - [4722](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4722): ユーザーアカウントが有効化された
    - [4723](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4723): アカウントのパスワード変更が試行された
    - [4724](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4724): アカウントのパスワードリセットが試行された
    - [4726](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4726): ユーザーアカウントが削除された
    - [4728](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4728): セキュリティ有効グローバルグループにメンバーが追加された
    - [4729](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4729): セキュリティ有効グローバルグループからメンバーが削除された
    - [4730](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4730): セキュリティ有効グローバルグループが削除された
    - [4731](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4731): セキュリティ有効ローカルグループが作成された
    - [4732](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4732): セキュリティ有効ローカルグループにメンバーが追加された
    - [4733](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4733): セキュリティ有効ローカルグループからメンバーが削除された
    - [4734](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4734): セキュリティ有効ローカルグループが削除された
    - [4735](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4735): セキュリティ有効ローカルグループが変更された
    - [4737](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4737): セキュリティ有効グローバルグループが変更された
    - [4738](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4738): ユーザーアカウントが変更された
    - [4739](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4739): ドメインポリシーが変更された
    - [4756](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4756): セキュリティ有効ユニバーサルグループにメンバーが追加された


### ネットワーク
Windows Filtering Platformで許可された通信を見る。  
ノイズ多めで役に立つことは少ないが、見たっていい。

- Security.evtx
    - [5156](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-5156): 接続許可（WFP）


### Microsoft Defender (旧 Windows Defender)
マルウェア検知、駆除・隔離、防御機能の無効化を見る。  
検知名、対象パス、実行された処理、防御機能がいつ止められたかを確認する。

検知はしてるけど駆除・隔離してないために実行されてしまっている事例とかもそこそこ。

evtxの名前は環境によって違うことがあるので注意。

- Microsoft-Windows-Windows Defender%4Operational.evtx
    - [1116](https://learn.microsoft.com/ja-jp/defender-endpoint/troubleshoot-microsoft-defender-antivirus): マルウェアまたは望ましくない可能性のあるソフトウェアを検出した
    - [1117](https://learn.microsoft.com/ja-jp/defender-endpoint/troubleshoot-microsoft-defender-antivirus): 検出した脅威に対して処理を実行した
    - [5001](https://learn.microsoft.com/ja-jp/defender-endpoint/troubleshoot-microsoft-defender-antivirus): リアルタイム保護が無効化された


### ログ改ざん・痕跡削除
ログクリア、イベントログサービス停止、監査イベント破棄など。  
これに限らないが、ログ調査をするときにはどのログがいつからいつまで残っているのかを最初に一覧にしたほうがよい。「不審なログオンはありませんでした！」「Securityログはいつから残っていたんですか？」「...」なんて事にならないように。

- Security.evtx
    - [1100](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-1100): Event Logサービスがシャットダウンされた
    - [1101](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=1101): 監査イベントがトランスポートによって破棄された
    - [1102](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-1102): Securityログクリア
- System.evtx
    - [104](https://docs.logrhythm.com/devices/docs/evid-104-log-cleared): その他ログクリア（System）


### ログローテート
これは調査で使うことはあまりないかもしれないが、知っておくとログの欠落を見たときになぜそれが起きたのか判断しやすい。

イベントログは、Channel ごとに「最大容量の指定」と「満杯になったときにどうするか」を指定できる。後者には以下3つがある。

1. Overwrite events as needed（必要に応じて古いイベントを上書きする）
2. Archive the log when full, do not overwrite events（満杯になったファイルをアーカイブして新しいログファイルに記録する）
3. Do not overwrite events / Clear log manually（満杯になったら新規イベントを記録しない）

1は、FIFO的に古いログ領域を再利用して新しいイベントを書き込んでいく。そのため、RecordID/RecordNumberの古い番号が欠落しているように見える。ログが欠落しているからといって安易に「侵害者による削除」と断定するのではなく、ログローテート設定を確かめてなぜ消えたかを考えること。

2は、満杯になったログをアーカイブ（自動バックアップ）しながら新しいファイルを作って記録し続ける。調査する人間としてはこれが一番うれしいのだが、当然ながら容量をガンガン消費する。ちゃんとログ管理を設計したい場合、ローカルディスクだけに抱え込むのではなく、イベントログ収集サーバやSIEMなどに転送して保管する設計にした方がよい。またこの場合、Securityログに `1105` が記録される。

3は、満杯になった時点から新規イベントを記録しなくなる。この設定はマジで困るのでやめてほしい。どういう嬉しさがあるのだろう。またこの場合、Securityログに `1104` が記録される。

- Security.evtx
    - [1104](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-1104): セキュリティログが満杯になった
    - [1105](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-1105): イベントログが自動でバックアップされた


### 時刻・電源・再起動・シャットダウン
ログの時刻設定など地味だが重要。  
あとは電源状態と他の痕跡に齟齬がないかなどは見ている。電源落ちているはずの時刻に何かが記録されていれば、どっちかがおかしい。(もしくは自分がおかしくなっている場合もままある)

- Security.evtx
    - [4616](https://learn.microsoft.com/ja-jp/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4616): システム時刻が変更された
- System.evtx
    - [12](https://learn.microsoft.com/ja-jp/troubleshoot/windows-server/performance/troubleshoot-unexpected-reboots-system-event-logs): Kernel-General: OSが起動した
    - [13](https://learn.microsoft.com/ja-jp/troubleshoot/windows-server/performance/troubleshoot-unexpected-reboots-system-event-logs): Kernel-General: OSがシャットダウンを開始した
    - [41](https://learn.microsoft.com/ja-jp/troubleshoot/windows-client/performance/event-id-41-restart): Kernel-Power: システムが正常にシャットダウンされずに再起動した
    - [1074](https://learn.microsoft.com/ja-jp/troubleshoot/windows-server/performance/troubleshoot-unexpected-reboots-system-event-logs): User32: プロセスまたはユーザーによってシャットダウン/再起動が開始された
    - [6005](https://learn.microsoft.com/ja-jp/troubleshoot/windows-server/performance/troubleshoot-unexpected-reboots-system-event-logs): EventLog: Event Logサービスが開始された
    - [6006](https://learn.microsoft.com/ja-jp/troubleshoot/windows-server/performance/troubleshoot-unexpected-reboots-system-event-logs): EventLog: Event Logサービスが停止された
    - [6008](https://learn.microsoft.com/ja-jp/troubleshoot/windows-server/performance/troubleshoot-unexpected-reboots-system-event-logs): EventLog: 直前のシャットダウンが予期しないものだった
    - [6009](https://learn.microsoft.com/ja-jp/troubleshoot/windows-server/performance/troubleshoot-unexpected-reboots-system-event-logs): EventLog: OS起動時のバージョン情報
    - [6013](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=6013): EventLog: システム稼働時間


### アプリケーション異常
アプリケーションのクラッシュなど。  
ユーザによってインストールされたアプリケーションの情報があったりなかったり。

- Application.evtx
    - [1000](https://www.myeventlog.com/search/show/43): アプリケーションエラー
    - [1001](https://www.myeventlog.com/search/show/446): アプリケーションハング
    - [1002](https://www.myeventlog.com/search/show/726): アプリケーションハング


### Sysmon
[Sysmon](https://learn.microsoft.com/ja-jp/sysinternals/downloads/sysmon) が入っているなら、ぶっちゃけこれとログオンだけ見ときゃそんなに困らない。  

これから設定しますとか、解析環境でマルウェア動かして挙動観測したいとかなら入れておけば良い。  
とはいえインストールしておわりではなく、ある程度自前でコンフィグをいじってあげないとノイズが多くなりがち。定番の設定は [SwiftOnSecurity/sysmon-config](https://github.com/swiftonsecurity/sysmon-config) 。


[Win11からは標準搭載](https://learn.microsoft.com/ja-jp/windows-server/administration/windows-commands/sysmon) （有効化されているとは言ってない）

- Microsoft-Windows-Sysmon%4Operational.evtx
    - [1](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90001):	Process creation
    - [2](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90002):	A process changed a file creation time
    - [3](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90003):	Network connection detected
    - [4](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90004):	Sysmon service state changed
    - [5](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90005):	Process terminated
    - [6](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90006):	Driver loaded
    - [7](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90007):	Image loaded
    - [8](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90008):	CreateRemoteThread
    - [9](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90009):	RawAccessRead
    - [10](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90010):	ProcessAccess
    - [11](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90011):	FileCreate
    - [12](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90012):	RegistryEvent (Object create and delete)
    - [13](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90013):	RegistryEvent (Value Set)
    - [14](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90014):	RegistryEvent (Key and Value Rename)
    - [15](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90015):	FileCreateStreamHash
    - [16](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90016):	Sysmon config state changed
    - [17](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90017):	Pipe created
    - [18](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90018):	Pipe connected
    - [19](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90019):	WmiEventFilter activity detected
    - [20](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90020):	WmiEventConsumer activity detected
    - [21](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90021):	WmiEventConsumerToFilter activity detected
    - [22](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90022):	DNSEvent
    - [23](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90023):	FileDelete
    - [24](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90024):	ClipboardChange
    - [25](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90025):	Process Tampering
    - [26](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90026):	File Delete Logged
    - [27](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90027):	File Block Executable
    - [28](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90028):	File Block Shredding
    - [29](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90029):	File Executable Detected
    - [225](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=90225):	Error


## 最低限見るべきログ
保全自体はとりあえず何でもかんでもやっておけばよいのだが、いざ見るぞ！というときになんにもわからんなら気合の目grepだ。  
とりま Provider/Channel で件数絞ってパラ読みしたいときには以下を中心にみればよい。

- ActiveDirectoryWebService.evtx
- Application.evtx
- DFS Replication.evtx
- DNS Server.evtx
- Directory Service.evtx
- Microsoft-Windows-AppLocker%4EXE and DLL.evtx
- Microsoft-Windows-AppLocker%4MSI and Script.evtx
- Microsoft-Windows-AppLocker%4Packaged app-Deployment.evtx
- Microsoft-Windows-AppLocker%4Packaged app-Execution.evtx
- Microsoft-Windows-Bits-Client%4Operational.evtx
- Microsoft-Windows-CodeIntegrity%4Operational.evtx
- Microsoft-Windows-DNSServer%4Audit.evtx
- Microsoft-Windows-GroupPolicy%4Operational.evtx
- Microsoft-Windows-Kernel-Boot%4Operational.evtx
- Microsoft-Windows-Microsoft Defender%4Operational.evtx
- Microsoft-Windows-NTLM%4Operational.evtx
- Microsoft-Windows-PowerShell%4Admin.evtx
- Microsoft-Windows-PowerShell%4Operational.evtx
- Microsoft-Windows-RemoteDesktopServicesRdpCoreTS%4Operational.evtx
- Microsoft-Windows-SENSE%4Operational.evtx
- Microsoft-Windows-SMBServer%4Operational.evtx
- Microsoft-Windows-SMBServer%4Security.evtx
- Microsoft-Windows-SmbClient%4Connectivity.evtx
- Microsoft-Windows-Sysmon%4Operational.evtx
- Microsoft-Windows-TaskScheduler%4Operational.evtx
- Microsoft-Windows-TerminalServices-LocalSessionManager%4Operational.evtx
- Microsoft-Windows-TerminalServices-RDPClient%4Operational.evtx
- Microsoft-Windows-TerminalServices-RemoteConnectionManager%4Operational.evtx
- Microsoft-Windows-Time-Service%4Operational.evtx
- Microsoft-Windows-WMI-Activity%4Operational.evtx
- Microsoft-Windows-WinRM%4Operational.evtx
- Microsoft-Windows-Windows Defender%4Operational.evtx
- Microsoft-Windows-Windows Firewall With Advanced Security%4Firewall.evtx
- Microsoft-Windows-Windows Firewall With Advanced Security%4FirewallDiagnostics.evtx
- Microsoft-Windows-WindowsUpdateClient%4Operational.evtx
- OpenSSH%4Admin.evtx
- OpenSSH%4Operational.evtx
- Security.evtx
- Setup.evtx
- System.evtx
- Windows PowerShell.evtx

参考: [徳島県つるぎ町立半田病院 コンピュータウイルス感染事案 有識者会議調査報告書 ― 技術編 ―](https://www.handa-hospital.jp/topics/2022/0616/index.html)


## ツールの使い分け

### 非アナリスト向け

<img width="1040" height="664" alt="image" src="https://github.com/user-attachments/assets/78ccffb2-ad2d-4c9b-938f-80677004d9e8" />

[イベントビューアー](https://learn.microsoft.com/ja-jp/shows/inside/event-viewer)

あなたが現場のSEとか機器管理者で、最初にログを見て侵害の可能性があるか確かめろって言われたら、まずはこれ。

Windows標準のイベントビューアーを使え。  
GUIで検索やフィルタができるので、最初の確認には十分使える。

ただし、**CSVでエクスポートしたものをCSIRTやアナリストに渡すな。マジでやめろ。**  
調査する側からすると、まずゴミみたいなCSVを使いやすいように整形するところから始める必要があり、余計な手間になる。あとかなり情報量が落ちる。調査担当に渡すなら、`.csv` ではなく `.evtx` 形式で保全したものにすること。

<img width="999" height="333" alt="image" src="https://github.com/user-attachments/assets/1575ba11-560f-4f4f-b90e-f3a578c68d19" />

イベントビューアーからエクスポートしたわけのわからないCSVファイルの例

ファストフォレンジックを見越してデータ取得するなら、[CDIR-Collector](https://github.com/CyberDefenseInstitute/CDIR) あたりをつかっておけばよい。`exe` ポチポチで主要なデータはゴソッと取れる。  
ただし、海外のセキュリティベンダ相手に渡すとなにそれ？ってなることもあるので、どのツールで取得したらよいか聞いておこう。[CyLR](https://github.com/orlikoski/CyLR), [KAPE](https://www.kroll.com/en/services/cyber/incident-response-recovery/kroll-artifact-parser-and-extractor-kape), [Velociraptor](https://github.com/Velocidex/velociraptor) あたりかな。。

手作業で1ファイルとか2ファイルちょちょっとやるなら、イベントビューアの「すべてのイベントを名前を付けて保存」でもよいし、`wevtutil epl` でエクスポートしてもよい。  

対象ファイルは `C:\Windows\System32\winevt\Logs\` 以下にあるが、ファイルコピーするとロックの兼ね合いで失敗することもあるので、可能なら正規のエクスポート手順で取得したほうが安全。

例:

```powershell
> wevtutil epl Security C:\Temp\Security.evtx
> wevtutil epl System C:\Temp\System.evtx
> wevtutil epl Application C:\Temp\Application.evtx
```


### アナリスト

あなたが CSIRT とかフォレンジック調査員なら、やり方は好みのものを選択すればよい。
ただし、チーム内で何を使うのかはあらかじめ合意しておくとよい。

**たとえあなたが逆張りオタクで人と違うツールを使うのが好きだとしても、そのツールによって証拠性が担保できるということは、あなたが説明しなければならないことを忘れてはいけない。**

使ったツールが何をどうパースして、どんな形式で出力し、元ログとどう対応しているのかを説明できる必要がある。

以下は個人的おすすめ順。


| パーサ        | 概要                                              | 備考                                     |
| ---------- | ----------------------------------------------- | -------------------------------------- |
| [EvtxECmd](https://github.com/EricZimmerman/evtx) | EVTXファイルをCSVやJSONに変換できるコマンドラインツール。              | SANS講師の [Eric Zimmerman](https://www.sans.org/profiles/eric-zimmerman) 氏のツール群の一つ。DFIR用途で広く使われている。 |
| [evtx2es/evtx2json](https://github.com/sumeshi/evtx2es)    | EVTXファイルをElasticsearchにインポートする/JSON変換するコマンドラインツール。    | 当時のイベントログパーサがバカ遅かったので作った。大量のイベントログを横断検索したい場合に便利。                |
|[plaso](https://github.com/log2timeline/plaso)|いろんなアーティファクトからスーパータイムラインを作れるツール|単一の変換ツールというよりはフレームワークに近い。かなり複雑。|
|[log2timeline](https://code.google.com/archive/p/log2timeline/downloads)|plasoの前身。こっちは比較的シンプル。|はるか古代のツールなのでよほどの理由がなければ今から採用しなくてもよい。|
| PowerShell | Windows標準のスクリプト言語。イベントログの抽出・整形に利用できる。           | 一度作っておけば使い回せる。バグがあると二次被害が起きたりするので注意。                     |
| [Log Parser](https://www.microsoft.com/ja-jp/download/details.aspx?id=24659) | Microsoftが提供していたコマンドラインツール。SQLライクなクエリでログを解析できる。 | 公式ツールだが、開発は終了している。ドキュメントも全然ない。誰か書いてほしい。                     |


| 分析・整形ツール                | 概要                                    | 備考                                   |
| ----------------------- | ------------------------------------- | ------------------------------------ |
| [Timeline Explorer](https://www.sans.org/tools/timeline-explorer)       | CSVなどをタイムライン形式で閲覧するツール。フィルタやグルーピングに強い。 | [Eric Zimmerman](https://www.sans.org/profiles/eric-zimmerman) 氏のツール群の一つ。パッと見るなら基本これでいいが、プロジェクトファイルの保存・読み込みが弱い。 |
| [Quilter-CSV](https://github.com/sumeshi/qsv-rs) | CSVのフィルタ、変換ツール。            | もとは [xsv](https://github.com/burntsushi/xsv) を使っていたが、かゆいところに手が届かないので作った。                       |
| [LibreOffice Calc](https://ja.libreoffice.org/)        | オープンソースの表計算ソフト。| Excelがない環境でも使える。巨大CSVはちょっと厳しいね。        |
| Excel                   | Microsoftが提供する表計算ソフト。みんな知ってるね。| 有償。利用者は多いが、大量ログは読み込めなかったり、勝手に時刻が数値変換されたりする。張り倒すぞ。 |
| Elasticsearch + Kibana  | 検索エンジン/可視化ツール。超大量のログを効率的に分析できる。   | ログのインデックス作成と横断検索に強い。クエリは初見殺し。                 |
| [Event Log Explorer](https://eventlogxp.com/ja/) | イベントログの閲覧と分析に特化したGUIツール。 | 商用利用は有償。SANSトレーニングでも使われている信頼のあるツールだが、あまり使いやすいとは思えない。 |
| [Log Parser Studio](https://learn.microsoft.com/ja-jp/exchange/iis-logs-and-log-parser-studio-reports-exchange-2013-help)  | Log ParserのGUIフロントエンド。   | たぶん配布停止。探せばまだあるが。           |
| [Log Parser Lizard](https://log-parser.com/)  | Log Parser系のGUIツール。      | Log Parser Studioより高機能。でも使い方がイマイチわからない。      |
| Splunk                  | 商用のログ管理・分析プラットフォーム。大量ログの検索・可視化に強い。    | 嫌い。                 |

あと上記とはちょっと毛色が違うけど **ハンティングツール？** の紹介。事前に作っておいた検知ルールでログをスキャンして怪しいイベントを引っ掛けたりすることができる。

単純な抽出だけではなく加工や正規化もしているので、この結果を直接報告に使うのは難しいが、ざっとスキャンして全体像を掴み、その後で詳細分析をすると素早く怪しいポイントを見つけやすい。

とはいえ FalsePositive は防げないので、結果を鵜呑みにせず、バイアスには注意すること。

| ハンティングツール| 概要| 備考|
| - | - | - |
| [Zircolite](https://github.com/wagga40/Zircolite)  | [Sigma](https://github.com/SigmaHQ/sigma) ルールでイベントログにスキャンかけて検知を行うツール。 | この手のツールの走り。のイメージ。（違ったらごめんなさい） |
| [Chainsaw](https://github.com/WithSecureLabs/chainsaw)  | これも同上。爆速。 | 海外では結構使われているイメージ。 |
| [Hayabusa](https://github.com/Yamato-Security/hayabusa) | これも同上。超高機能。 | 言わずとしれたYamato Securityのプロダクト。ドキュメントがかなり豊富で使いやすい。コミット頻度もエグい。 |

#### EvtxEcmd によるパース

フォーマットはcsvが一番取り回しやすい。grepもできるし。
ファイル指定 `-f` もしくはフォルダ指定 `-d` で変換する。フォルダ指定すると配下の `.evtx` が1つの `.csv` に統合される。  

```.ps1
> EvtxECmd.exe -f Security.evtx --csv . --csvf Security.csv
```

<img width="1051" height="628" alt="image" src="https://github.com/user-attachments/assets/7b9bb6c4-e731-4c82-bb08-ad737ebee891" />


#### Timeline Explorer によるフィルタリング

<img width="1257" height="761" alt="image" src="https://github.com/user-attachments/assets/03952f18-8d18-4cde-bb75-cf68ca3b0239" />

`Event Id = 4624` のように条件式でフィルタをかけたり、カラムをクリックしてソートしたり、グルーピングしたり、思いつくほとんどのことはできる。  
自動化はできないが、とりあえずいろいろ見てみるべ～のときはこれだけでいい。


#### Quilter-CSV によるフィルタリング

[Quilter-CSV](https://github.com/sumeshi/qsv-rs) は自作ツールだが結構気に入っている。  

次のように複数の処理をチェインさせて処理できる。簡単な処理ならこれだけでもいい。

```.ps1
> qsv load Security.csv - isin EventId 4624 - select TimeCreated,EventId,PayloadData1 - sort TimeCreated - head 10 - showtable
```

<img width="1051" height="628" alt="image" src="https://github.com/user-attachments/assets/b276b742-bc38-41ca-ba07-58809baeb043" />


あるいは、あらかじめ定義したyamlファイルに沿って処理したりもできる。  
複数ファイルに適用したいときなどはこっちのがいいね。

```
> qsv quilt test.yaml --var input=Security.csv
```

```.yaml
title: test

stages:
  load_stage:
    type: process
    steps:
      load:
        path: ${input}

  filtering:
    type: process
    source: load_stage
    steps:
      isin:
        colname: EventId
        values:
          - "4624"
      select:
        colnames:
          - TimeCreated
          - EventId
          - PayloadData1
      sort:
        colnames:
          - TimeCreated
      head:
        number: 10

  output_stage:
    type: process
    source: filtering
    steps:
      showtable:
```

<img width="1051" height="628" alt="image" src="https://github.com/user-attachments/assets/2727736a-dc4c-4f9c-8ebd-775fd5681d84" />

Securityログは [JPCERTCC/LogonTracer] より


## おわりに

もうイベントログ調査なんてこりごりだ～！

おしまい
