# AI使い方おメモ
備忘。

## サービスの使い分け

### ChatGPT / Codex

ChatGPT とマイクで話しながら作りたいソフトウェアの方針をある程度固めていく。  
いい感じになったら、 Fable 5 や Sakana Fugu などの強めのモデルに投げるためのプロンプトを最後に吐き出させる。

ChatGPT と Codex の Quota は共通じゃないというのが非常に良い。

### Claude

コーディング性能は高くないので設計とチェックを主にさせる。  
Fable 5 / Opus しか使わない。設計や修正点は直接実装させずに、TASKS.md とかに具体化してまとめさせる。

### OpenCode

Public なプロジェクトの実装担当をさせる。 deepseek-v4-flash を使う。
冗談みたいにやすいので、10 並列以上で動かす。opencode で使っていたが、最近また hermes-agent 使い始めた。どっちがよいか体感上は正直わからんが、hermes のが処理時間が長いのでいろいろ考えているのだろう。

### Sakana

Sakana Fugu しか使わない。
プロジェクトの設計だけさせる。 Quota が厳しいが。

### OpenRouter

1,000req/day 使えるので適当な用途に使っていた。

が、 Codex から使ったときに free モデルでも課金されてしまった。
サポートに問い合わせたが週に1回くらいボットからの定型文が返ってくるのみで埒が明かないのでもう使っていない。たぶんもう使うこともない。

### Gemini / Cursor

前は契約してたがもう使ってない。
Cursor はバリバリ開発するならまた使ってもよいのだが、趣味プログラミング程度なら上記で間に合っている。


## ハーネス

基本的には Codex CL IとかClaude Codeをノンカスで使っている。
私の思想として、プラグインとかでゴテゴテにするのは美しくないと考えているのもあるし、単純にソフトウェアの進化が早いのでそれに追従できなくなるのは嫌だからというのもある。

ラッパーの headroom だけは使っている。

```
$ headroom wrap codex
$ headroom wrap claude
```

これで起動して裏で serena とか回してくれるのでよい。トークン消費量がかなり落ちる。


hermes-agent の場合は `config.yaml` を下記のように設定する。

```
# ~/.hermes/config.yaml
model:
  default: deepseek-v4-flash
  provider: opencode-go
  base_url: http://127.0.0.1:8787/v1
  api_mode: chat_completions

mcp_servers:
  headroom:
    command: /home/linuxbrew/.linuxbrew/bin/headroom
    args:
      - mcp
      - serve
```

headroom の起動は下記の通り。

```
$ OPENAI_TARGET_API_URL=https://opencode.ai/zen/go headroom proxy --port 8787
```

この状態でこう。

```
$ hermes --tui
```

思い出したら追記。