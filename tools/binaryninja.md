# BinaryNinja
に関するめも。適宜追記。

## はじめに

BinaryNinjaが10周年らしく、[35%オフになっていた](https://binary.ninja/2026/07/22/10-years-of-binary-ninja.html)。めでたいねぇ。  
やっす～い、と思ったらドル円が163円だった。日本円で36,000円くらい。たっか～い。

でも周りでの評判はかなり良いし、IDAに比べればタダみたいなもんなので買ってみた。

## 形から入る

公式の [GitHub](https://github.com/Vector35/community-themes) にThemeファイルもあった。  
ぼくの好きな `catppuccin` もある。おめめにやさしそうな `frappe` が好み。`C:\Users\ユーザ名\AppData\Roaming\Binary Ninja\themes` フォルダを作っておいておけばよい。

![catppuccin](https://github.com/user-attachments/assets/c628a695-747d-47e7-9819-bcd26fc8eda6)

## IDAとの差異

Ghidraはキーバインドがかなり違うので警戒していたのだが、こっちはそうでもないらしい。  
あと公式がマイグレーティングガイドを公開していた。ドキュメントが豊富ですごくいいね。

[Migrating from IDA](https://docs.binary.ninja/guide/migration/migrationguideida.html)

クロスリファレンスだけdialogにしておいた。  
デフォではコンソール部分にペインが出現するが、ダイアログだと全面に小さいウィンドウが出る。慣れているからかこっちのが見やすい。

> ui.defaultXrefInterface

あとはデフォでいいかな。

## リバエンの練習

[crackme](https://crackmes.one/) で人気の検体を探してみる。  
`Windows-x64-C/C++` で Downloads の降順に並べたら下記が引っ掛かった。

https://crackmes.one/crackme/698d2206e2ba6023bfacaa4f

zip展開パスワードはドメイン名。

### レイアウト

下記赤枠で囲ったあたりをポチポチした。

<img width="2480" height="1258" alt="image" src="https://github.com/user-attachments/assets/ccac4e97-a500-45e7-8fba-353df400413b" />

真ん中のペインを左右にsplitして、左側はグラフビュー（ディスアセンブル）に、
右側はデコンパイル表現（ハイレベル中間言語）にしている。疑似Cコードにしてもいいと思う（というか違いがまだ分かっていない）。　

後々自動化したりもするかなと思ってコンソールも開いておいた。APIがいい感じらしい。

`Window > Layout > Save Current Layout as Default` から現在のレイアウトをデフォにできる。

### 色付け

call文を色付けしたい。のでチャッピーに作らせた。  
https://github.com/sumeshi/binaryninja-scripts/blob/main/scripts/colorize_calls.py

<img width="914" height="590" alt="image" src="https://github.com/user-attachments/assets/9c71612a-5bf9-41ae-a731-bc6b9cebaee2" />

こんな感じで色付けされる。目立つね。

|命令|色|
|-|-|
|直接CALL|シアン|
|間接CALL|黄色|
|TAILCALL|マゼンタ|
|SYSCALL|赤|

`File > Run Script...` から都度実行する。  
デフォにもできるんだろうが、あんまりカスタムするのは [自分のポリシー](https://en.wikipedia.org/wiki/KISS_principle) に反する。

### 見てみる

う～ん、ぱっと見る感じコンソールで文字列を入力すると、その文字長をチェックするのと、それが `EasyPassword` かどうかを2重チェックしているらしい。
両方OKならおめ～と言ってくれる。のだろう。

## ローカルAIで解析してみる

[fosdickio/binary_ninja_mcp](https://github.com/fosdickio/binary_ninja_mcp) が人気あるらしい。開発も比較的アクティブ。

### ゲスト側設定

[Releases](https://github.com/fosdickio/binary_ninja_mcp/releases) から落としたzipを下記フォルダ直下に展開した。

`C:\Users\ユーザ名\AppData\Roaming\Binary Ninja\plugins`

展開したフォルダは `binary_ninja_mcp` の末尾に `-1.2.1` がついているが、ハイフンがあると認識しないっぽい。  
Pythonのパッケージ仕様に引っかかってるのかな。

BinaryNinjaを再起動する。
うまいこと設定できていれば左下にMCPの表示があるはず。バイナリを開いた状態でクリックするとオンオフ切り替えられる。

<img width="2480" height="1258" alt="image" src="https://github.com/user-attachments/assets/b6490c2b-5cec-41aa-8ec7-23017f49d09c" />

コマンドプロンプトから下記コマンドを打つとちゃんと開いているバイナリの概要が返ってくる。

```
> curl.exe http://127.0.0.1:9009/binaries
```

ただ、このままではまだ外部からアクセスができない。Settingsから下記項目にチェックを入れて外部からアクセス可能にしよう。

<img width="1542" height="167" alt="image" src="https://github.com/user-attachments/assets/4c6c517f-d531-48a8-a8fe-cf4f4cb9ad8b" />

設定したらMCPをポチポチして再起動する。Windows Firewallの設定を変えていいか聞かれるので許可。

### ホスト側設定

まずはホスト側からアクセス可能かチェックする。IPアドレスは適宜読み替えること。

```
> curl.exe http://192.168.239.128:9009/binaries
```

ぼくは [LMStudio](https://lmstudio.ai/) を使う。簡単なので。

`mcp.json` の設定をいい感じにする。これもIPは読み替えてね。

<img width="441" height="417" alt="image" src="https://github.com/user-attachments/assets/e7de77fd-5273-407d-a528-9d403bbcba45" />

mcpサーバの実行にnodeが必要なので、ホスト側にインストールしておきましょう。  
うまくいけばこんな感じでToolsが表示される。

<img width="417" height="583" alt="image" src="https://github.com/user-attachments/assets/dfe0cc8d-4034-4f95-9065-a868a47f7700" />


### ためしてみる

適当な指示をだす。

```
binary-ninja-mcpの /binaries にアクセスしてプログラムの概要を把握してください。そのプログラムについてコードの解説してちょ。
```

一般的なゲーミングPCでも動くモデル `google/gemma-4-12b-qat` で動かした。  
対象を指定しないと自分から探してくれない。気が利かない。

<img width="1152" height="1117" alt="image" src="https://github.com/user-attachments/assets/08ba45e9-4b27-4cb2-b626-ad60865c229c" />

結果は下記の通り。

```
このプログラムは、ユーザーに入力を促し、それが特定のパスワードと一致するかどうかを判定する典型的な「CrackMe（解析用サンプルプログラム）」です。

以下に、プログラムの動作を日本語で分かりやすく解説します。

1. プログラムの全体像
このプログラムは、標準的なC++の入出力機能（std::cin, std::cout）を使用して、ユーザーから文字列を受け取り、それを内部に隠された正解の文字列と比較します。

2. 主要な動作のステップ
デコンパイルされたコードから、以下の流れを読み取ることができます。

入力の取得:

プログラムはまず、ユーザーからの入力を待ち受けます。
入力された文字列は、メモリ上の特定のバッファ（変数）に格納されます。
長さのチェック:

プログラムは入力された文字列の長さが12文字であるかどうかを確認します。
もし長さが正しくなければ、即座に「Wrong Password, Please Try Again（間違ったパスワードです。もう一度やり直してください）」というメッセージを準備します。
内容の比較:

次に、入力された文字列と、プログラム内にハードコード（直接書き込み）されている文字列を比較します。
ここで比較される正解の文字列は "EasyPassword" です。
比較の結果、一致しなければ「Wrong Password...」というメッセージが選ばれます。
結果の表示:

最後に、判定されたメッセージを画面に表示します。
正解だった場合のみ、「Congrats!!! You Cracked The Code（おめでとう！コードを解読しました）」と表示されます。
3. 結論（攻略法）
このプログラムをクリア（パスワードを正解させる）するための条件は以下の通りです。

正解のパスワード: EasyPassword
条件: 文字数が正確に12文字であること。
解析のポイントとしては、memcmp（メモリ比較）のような関数を使って、ユーザーの入力バッファとプログラム内のデータ領域を照合している箇所を見つけるのが鍵となります。
```

適当な指示でもなんとかなるもんだな。

おわり
