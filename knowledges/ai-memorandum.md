# 生成AI使い方おメモ (2026.08.25 更新)
備忘。

## サービスの使い分け

使っているうえでの勝手なイメージ

- 思考力: Sakana Fugu > GPT-5.6 Sol medium > それ以外
- 実装力: GPT-5.6 Sol > Grok > GPT-5.6 Luna > OpenCode の良さげモデル

### 1. OpenAI

ChatGPT とマイクで話しながら作りたいソフトウェアの方針をある程度固めていく。  
いい感じになったら、 **Sakana Fugu** など強めのモデルに投げるための方針を示すプロンプトを最後に吐き出させる。

その後の細かい実装は、 Codex の **GPT-5.6 Sol (medium)** を使う。 **Luna** サブエージェントをふんだんに使って実装させる。 Plus プランでもそうそう Quota を使い果たすことはない。

ChatGPT と Codex の Quota は共通じゃないというのが非常に良い。

[ChatGPT](https://chatgpt.com/)


### 2. Sakana AI

**Sakana Fugu** しか使わない。プロジェクトの一番大きい方針の決定だけさせる。  
要は、 `/goal` だけ決めさせるイメージ。その後のタスク化と実装は `GPT` 。

Quota がかなり厳しいが。

[sakana.ai](https://sakana.ai/)


### 3. SpaceXAI

Xアカウントを Premium にしたので、 **Grok 4.6** が使えるようになった。  
性能はそんなに悪くないし、 Quota もたっぷりある。

**GPT** と **Sakana** の Quota を使い果たして暇になったときは、これつかってリポジトリのメンテとかスライド資料を作ったりしている。

[SpaceXAI](https://x.ai/)


### 4. OpenCode Go

Public なプロジェクトの実装担当をさせる。 **Hy3** や **Deepseek V4 Flash (New)** を使う。  
冗談みたいにやすいので、10 並列以上で動かす。

[OpenCode Go](https://opencode.ai/ja/go)


### 5. Anthropic

思考力においても実装力においてもゴミなので解約した。

[Anthropic](https://www.anthropic.com/)


### 6. Cursor

前は契約してたがもう使ってない。  
性能はかなりいいので、 **Claude** の代わりに契約するとしたらこれかな。

現在は SpaceXAI 傘下なので **Grok** も使える。

[Cursor](https://cursor.com/ja)


### 7. Gemini

前は契約してたがもう使ってない。  
安いしいろんなサービスに使えるので、コーディングしないなら悪くないかもしれない。

[Gemini](https://gemini.google.com/app)

### 8. Manus

独立に際して無料なので一瞬使っていた。  
分析力は悪くない。 Web ブラウザからスライド作ったりファイルをこねこねしたり使うもので、コーディングエージェントから使うものではなさそう。

自分の Public なプロジェクトを分析させてスライドにまとめ、改善点を探すのに使っていた。

[Manus Free Access：キャンペーンルール](https://help.manus.im/ja/articles/16312548-manus-free-access-%E3%82%AD%E3%83%A3%E3%83%B3%E3%83%9A%E3%83%BC%E3%83%B3%E3%83%AB%E3%83%BC%E3%83%AB)


### 7. OpenRouter

1,000req/day 使えるので適当な用途に使っていた。

が、 Codex から使ったときに free モデルでも課金されてしまった。  
サポートに問い合わせたが週に1回くらいボットからの定型文が返ってくるのみで埒が明かないのでもう使っていない。たぶんもう使うこともない。

[OpenRouter](https://openrouter.ai/)

## ハーネス

基本的には **Codex CLI** とか **Grok Build** をノンカスで使っている。

私の思想として、プラグインとかでゴテゴテにするのは美しくないと考えているのもあるし、単純にソフトウェアの進化が早いのでそれに追従できなくなるのは嫌だからというのもある。

**Skill も作らない。** 大抵足を引っ張るだけになる。  
**`AGENTS.md` だけは維持管理** していて、過去俺がキレたポイント（指示の取り違え、余計なテストコードの追加、設計の見落とし、ドキュメントの更新忘れなど）をまとめている。

ラッパーの headroom だけは使っている。

```bash
$ headroom wrap codex
```

これで起動して裏で serena とか回してくれるのでよい。トークン消費量がかなり落ちる。


hermes-agent の場合は `config.yaml` を下記のように設定する。

```yaml
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

```bash
$ OPENAI_TARGET_API_URL=https://opencode.ai/zen/go headroom proxy --port 8787
```

この状態でこう。

```bash
$ hermes --tui
```

## ローカルLLM

ローカルは正直キッツイなぁとおもっていたが、最近出た **[Ornith-1.5-9B](https://huggingface.co/ornith-ai/Ornith-1.5-9B)** がマジですごい。 **Qwen3.5** をベースに頑張って学習させたものらしい。

文章力はあまり高くない(**Gemma 4** より低い) のだが、Tool Callingがかなり安定している。あと軽くて早い。中古で2万で買った RTX2070 SUPER でも **30t/s** でる。

たまに思考がループに陥るのは **Qwen** から引き継いだ悪いところかな。

[Pi Coding Agent](https://pi.dev/) で十分使える。カスタムがめんどいなと思ったので「Web検索使えるようにしといて」と指示したらあっさりやってくれた。

もっとスペックが厳しいノートPCとかなら、[Gemma 4 E2B](https://huggingface.co/google/gemma-4-E2B) と [Aider](https://aider.chat/) ならなんとかなる。

詳しくは [会社支給のパソコンでも(ローカルLLMで)Vibe Codingがしたい！](https://zenn.dev/sum3sh1/articles/126b28e0b3f8c5) を参照。

---

思い出したら追記。