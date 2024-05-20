# セキュリティのおツール達
思い出したら追記。

## はじめに
ツール選定はGPT-4。  
100個考えてねって言ったら出してくれたけど重複あるしあんましな感じ。

|Category|Name|Overview|License|Note|
|-|-|-|-|-|
|OS|[CAINE](https://sumeshi.github.io/posts/tools/caine)|フォレンジックに特化したLinux Distro|[LGPL](https://www.caine-live.net)||
|OS|Kali Linux|ペネトレーションテストに特化したLinux Distro|||
|OS|Parrot Security|ペネトレーションテストに特化したLinux Distro|||
|OS|Qubes OS|匿名性に特化したLinux Ditro|||
|OS|REMnux|マルウェア解析に特化したLinux Distro|||
|ToolKit|Sysinternals Suite|Windowsシステムユーティリティ|||
|Disassembler|[IDA](https://hex-rays.com/ida-pro/)|静的解析用ディスアセンブラ&デバッガ|||
|Disassembler|[Ghidra](https://ghidra-sre.org/)|静的解析用ディスアセンブラ&デバッガ|||
|Disassembler|[Binary Ninja](https://binary.ninja/)|静的解析用ディスアセンブラ&デバッガ|||
|Disassembler|[Malcat](https://malcat.fr/index.html)|静的解析用ディスアセンブラ&デバッガ|||
|Debugger|[WinDbg](https://learn.microsoft.com/ja-jp/windows-hardware/drivers/debugger/)|(x86-x64)カーネルモードに対応したデバッガ|||
|Debugger|[x64dbg](https://x64dbg.com/)|(x86-x64)デバッガ|||
|Debugger|[Immunity Debugger](https://www.immunityinc.com/products/debugger/)|(x86)デバッガ|||
|Debugger|[OllyDbg](https://www.ollydbg.de/)|(x86)デバッガ|||
|Debugger|[Radare2](https://rada.re/n/radare2.html)|(x86-x64)TUIデバッガ|||
|BinaryEditor|[Detect-It-Easy](https://github.com/horsicq/Detect-It-Easy)|バイナリファイルのパッカーや暗号化ツールの識別に。|||
|BinaryEditor|[HxD](https://mh-nexus.de/en/hxd/)|高機能なバイナリエディタ。バイナリをC言語表現とかでコピーできるので、シェルコードの編集とかに有用。|||
|BinaryEditor|FileInsight|高機能なバイナリエディタ。プラグインが豊富。||[DLリンク](https://downloadcenter.trellix.com/products/mcafee-avert/fileinsight.msi)は生存|
|BinaryEditor|[Dependency Walker](https://www.dependencywalker.com/)|実行ファイルの依存関係を一覧表示してくれる。|||
|BinaryEditor|[PeStudio](https://www.winitor.com/)|PEファイルの解析に非常に有用。大体なんでもできる。|||
|BinaryEditor|[PE-bear](https://github.com/hasherezade/pe-bear)|PEファイルの解析に非常に有用。|||
|BinaryEditor|[XPEViewer](https://github.com/horsicq/XPEViewer)|PEファイルの解析に非常に有用。色々項目の編集もできる。￥|||
|BinaryEditor|[XELFViewer](https://github.com/horsicq/XELFViewer)|実行ファイルの解析に特化したツール。|||
|BinaryEditor|[ResourceHacker](https://www.angusj.com/resourcehacker/)|Exeのアイコン変えるのに|||
||Wireshark|ネットワークトラフィックのキャプチャと分析。|||
||VirusTotal|ファイルやURLを複数のアンチウイルスでスキャン。|||
||Maltego|データマイニングと情報収集ツール。|||
||YARA|マルウェアのパターンと識別特徴を定義。|||
||Process Hacker|システムプロセスやサービスを監視。|||
||Volatility|メモリフォレンジックフレームワーク。|||
||Cuckoo Sandbox|マルウェアを安全な環境で実行して分析。|||
||Apktool|Android APKファイルのデコンパイルと再構築。|||
||Fiddler|HTTPトラフィックのキャプチャと分析。|||
||Tcpdump|コマンドラインベースのパケットアナライザ。|||
||Regshot|レジストリの変更を監視。|||
||Burp Suite|ウェブアプリケーションセキュリティテスティング。|||
||Snort|ネットワーク侵入検出システム。|||
||Suricata|高性能なネットワークIDS、IPS、NSM。|||
||Binwalk|ファームウェア解析ツール。|||
||Autopsy|デジタルフォレンジックプラットフォーム。|||
||Metasploit Framework|ペネトレーションテスト用ツールキット。|||
||FireEye FLARE VM|マルウェア解析用のWindowsベースのVM。|||
||Sandboxie|アプリケーションを隔離して実行。|||
||Packer Identifier (PID)|実行ファイルのパッカー識別。|||
||Plaso|ログファイルの解析とタイムライン作成。|||
||Redline|メモリとファイル解析ツール。|||
||ThreatGrid|Ciscoのクラウドベースのマルウェア解析。|||
||Zeek (以前のBro)|ネットワークモニタリングフレームワーク。|||
||NetworkMiner|パケットキャプチャとネットワークフォレンジック。|||
||CapTipper|マルウェアのHTTPトラフィック解析ツール。|||
||Angry IP Scanner|ネットワークスキャンツール。|||
||Nmap|ネットワークマッピングとセキュリティ監査。|||
||OSSIM|オープンソースセキュリティ情報管理システム。|||
||Nexpose|ネットワークの脆弱性スキャナ。|||
||Splunk|データ分析とセキュリティ情報管理。|||
||Argus|ネットワークトラフィック監視ツール。|||
||GRR Rapid Response|リモートフォレンジックフレームワーク。|||
||PowerShell Empire|ポストエクスプロイトフレームワーク。|||
||Docker|アプリケーションコンテナ化。|||
||Mimikatz|Windows認証データの抽出。|||
||Wireshark|ネットワークトラフィック分析。|||
||FTK Imager|デジタルフォレンジックイメージングツール。|||
||Xplico|ネットワークトラフィックデコーダー。|||
||Hopper Disassembler|バイナリ解析およびディスアセンブリング。|||
||Joxean Koret's Pyew|Pythonによるコマンドラインヘキサエディタ兼解析ツール。|||
||X-Ways Forensics|高度なフォレンジック分析ツール。|||
||CyberChef|Webベースの暗号、デコード、データ分析ツール。|||
||de4dot|.NETのアセンブリのオブファスケーションを解除するためのツール。|||
||PDF Stream Dumper|PDFファイル解析および修復ツール。|||
||SysAnalyzer|自動的に実行可能ファイルの動作を分析。|||
||bin2h|バイナリファイルをC/C++のヘッダファイルに変換。|||
||Bokken|GUIベースのバイナリ解析ツール。|||
||Paros Proxy|Webアプリケーションセキュリティテスト用のプロキシ。|||
||Ether|アクティブなWindowsシステムに対するマルウェア解析。|||
||Faraday|協調的なペネトレーションテストおよび脆弱性管理プラットフォーム。|||
||Flare VM|Windowsベースのセキュリティリサーチおよびマルウェア解析用のVM。|||
||FRIDA|ダイナミック・インストルメンテーション・ツールキット。|||
||KeeFarce|KeePassのパスワードを抽出。|||
||Malzilla|マルウェアのWebページ解析ツール。|||
||Netcat|ネットワーク読み書き用の汎用ユーティリティ。|||
||NoVirusThanks OSArmor|マルウェア攻撃からWindowsを保護。|||
||Phantom Evasion|Pythonによるアンチウイルス回避ツール。|||
||ProcDOT|マルウェアの動的行動を視覚化。|||
||Shellcode2Exe|シェルコードを実行可能ファイルに変換。|||
||SpyStudio|WindowsのAPI呼び出しとCOMオブジェクトのトラッキング。|||
||VeraCrypt|ディスク暗号化ソフトウェア。|||
||Wazuh|マルチプラットフォームセキュリティモニタリング。|||
||Xenotix XSS Exploit Framework|XSS脆弱性テスト用フレームワーク。|||
||YASAT|単純なセキュリティ監査ツール。|||
||ZAP (Zed Attack Proxy)|Webアプリケーションセキュリティスキャナ。|||
||BMC (Binary Malware Classifier)|バイナリベースのマルウェア分類。|||
||Bochs|エミュレータとデバッガ。|||
||Cobalt Strike|レッドチーム作戦と脆弱性テストツール。|||
||CrowdStrike Falcon Sandbox|クラウドベースのマルウェア解析環境。|||
||DRAKVUF|ハイパーバイザーベースのマルウェア分析システム。|||
||Fakenet-NG|マルウェアのネットワークアクティビティを分析。|||
||GFI Languard|ネットワークセキュリティスキャナと脆弱性管理。|||
||hashcat|高速なパスワードクラッキングツール。|||
||Honeyd|ネットワークホストシミュレータ。|||
||Hybrid Analysis|クラウドベースのマルウェア解析サービス。|||
||Immunity Canvas|ペネトレーションテストツール。|||
||John the Ripper|パスワードクラックツール。|||
||JTAGulator|ハードウェアハッキングデバイス。|||
||Kismet|ワイヤレスネットワークおよびデバイスの検出。|||
||MISP (Malware Information Sharing Platform & Threat Sharing)|脅威情報共有プラットフォーム。|||
||NESSUS|脆弱性スキャナ。|||
||NinjaRMM|リモート監視および管理ソフトウェア。|||
||Omnipeek|ネットワーク分析ツール。|||
