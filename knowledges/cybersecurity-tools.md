# セキュリティのおツール達
思い出したら追記。

## はじめに
ツール選定はGPT-4。  
100個考えてねって言ったら出してくれたけど重複あるしあんましな感じ。

|Category|Name|Overview|License|Note|
|-|-|-|-|-|
|OS|[CAINE](https://sumeshi.github.io/posts/tools/caine)|フォレンジック用Linux|[LGPL](https://www.caine-live.net)||
||Wireshark|ネットワークトラフィックのキャプチャと分析。|||
||IDA Pro|強力なディスアセンブラとデバッガ。|||
||OllyDbg|対話型デバッギングツール。|||
||Ghidra|NSAが開発したオープンソースの逆コンパイラ。|||
||VirusTotal|ファイルやURLを複数のアンチウイルスでスキャン。|||
||Maltego|データマイニングと情報収集ツール。|||
||YARA|マルウェアのパターンと識別特徴を定義。|||
||PEiD|バイナリファイルのパッカーや暗号化ツールを識別。|||
||Process Hacker|システムプロセスやサービスを監視。|||
||Volatility|メモリフォレンジックフレームワーク。|||
||Cuckoo Sandbox|マルウェアを安全な環境で実行して分析。|||
||Apktool|Android APKファイルのデコンパイルと再構築。|||
||Fiddler|HTTPトラフィックのキャプチャと分析。|||
||Sysinternals Suite|Windowsシステムユーティリティのコレクション。|||
||Tcpdump|コマンドラインベースのパケットアナライザ。|||
||Regshot|レジストリの変更を監視。|||
||x64dbg|64ビットプラットフォーム向けデバッガ。|||
||Burp Suite|ウェブアプリケーションセキュリティテスティング。|||
||Binary Ninja|低レベルのバイナリ解析ツール。|||
||Radare2|ポータブルリバースエンジニアリングフレームワーク。|||
||Snort|ネットワーク侵入検出システム。|||
||Suricata|高性能なネットワークIDS、IPS、NSM。|||
||Binwalk|ファームウェア解析ツール。|||
||REMnux|マルウェア解析のためのLinuxディストリビューション。|||
||Autopsy|デジタルフォレンジックプラットフォーム。|||
||Metasploit Framework|ペネトレーションテスト用ツールキット。|||
||Immunity Debugger|プログラム解析とデバッグ。|||
||FireEye FLARE VM|マルウェア解析用のWindowsベースのVM。|||
||Sandboxie|アプリケーションを隔離して実行。|||
||Packer Identifier (PID)|実行ファイルのパッカー識別。|||
||Kali Linux|セキュリティテスト用のLinuxディストリビューション。|||
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
||Radare2|Unixライクなリバースエンジニアリングフレームワーク。|||
||IDA Free|フリー版の逆アセンブラおよびデバッガ。|||
||X-Ways Forensics|高度なフォレンジック分析ツール。|||
||CyberChef|Webベースの暗号、デコード、データ分析ツール。|||
||de4dot|.NETのアセンブリのオブファスケーションを解除するためのツール。|||
||PeStudio|実行ファイルの解析に特化したツール。|||
||PDF Stream Dumper|PDFファイル解析および修復ツール。|||
||SysAnalyzer|自動的に実行可能ファイルの動作を分析。|||
||bin2h|バイナリファイルをC/C++のヘッダファイルに変換。|||
||Bokken|GUIベースのバイナリ解析ツール。|||
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
||IDA Home|ホームユーザー向けの逆アセンブラとデバッガ。|||
||Immunity Canvas|ペネトレーションテストツール。|||
||John the Ripper|パスワードクラックツール。|||
||JTAGulator|ハードウェアハッキングデバイス。|||
||Kismet|ワイヤレスネットワークおよびデバイスの検出。|||
||MISP (Malware Information Sharing Platform & Threat Sharing)|脅威情報共有プラットフォーム。|||
||NESSUS|脆弱性スキャナ。|||
||NinjaRMM|リモート監視および管理ソフトウェア。|||
||Omnipeek|ネットワーク分析ツール。|||
||Paros Proxy|Webアプリケーションセキュリティテスト用のプロキシ。|||
||Qubes OS|セキュリティに焦点を当てたデスクトップOS。|||
