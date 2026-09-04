# AIにフォレンジックさせるのをやめよう
AI-assisted Forensic Scribing という考え方と、その実装について。

## はじめに

以前、「[ローカルLLMはフォレンジック調査官の夢を見るか？](https://sumeshi.github.io/posts/works/do-localllms-dream-of-forensic-investigator)」という記事を書きました。これは、**AIが自律的にフォレンジック調査を進めるために作ったプロジェクト「[FORENSIA](https://github.com/sumeshi/forensia)」** に関するものでした。

これは、さまざまなアーティファクトを統一フォーマットとして取り込み、ルールで調査起点を作り、仮説を立て、実際の証拠と突き合わせ、その結果を次の仮説生成やレポート生成に回す...という**ループを構成するハーネス**でした。LLMにすべてを任せず、小さな単位に分解した仕事だけをLLMに任せる。LLMがやらなくてもいい仕事は機械的にやる。そういうアーキテクチャです。

なぜこんなことをしたかというと、当時（2026年4月ごろ）、ローカルで動くLLMというとあまり強いものはなく、せいぜい Gemma 4（2026年3月31日リリース）がギリギリ使い物になるかな、といった状況でした。そのため、潤沢なリソースが使えない **貧者のLLM戦略** （実際、約2万円の中古GPUで動かしています）をコンセプトに、できることがないかを模索した結果でした。

このプロジェクトの発想自体は今でも面白いと思っているのですが、どちらかというと実用よりは夢を追いかけたPoC的なイメージで作っていました。
ところが、ここ最近になって明確にローカルLLMのステージが変わったと感じています。

個人的に衝撃を受けたのは、[Ornith-1.5](https://ornith.ai/ornith_1_5.html)です。9B版でも軽快に動き、長時間走らせても思考が循環したり、考えが無限に発散したりすることなく、ひたすら働き続けてくれます。そういった意味で、エージェント性能が非常に高いと感じています。

Ornith-1.5 は Qwen3.5 をベースに追加学習されたモデルなので、言語性能や知識についてはピュアな Gemma 4 ほど優秀ではないのですが、そういった点についてはそもそもローカルLLMに期待していません。というか期待してはいけません。

必要なのは、必要な情報と指示を与えればちゃんとそのとおり動いてくれること。これだけ。


## AIにフォレンジックをさせることの課題

私と同じようなことを考える人は山ほどいるわけで、AIにアーティファクトを食わせてレポート生成まで一気通貫！というのをちょこちょこ見かけます。どちらかといえば、マネージドの 卍最強モデル卍 をガンガン使って、精度の高い結果を得ようとするほうが多いかな。

もちろんそれもすごく面白いのですが、やっぱり機密性やデータの取り扱いを考えるとローカルに魅力を感じてしまいます。

![localllm](https://github.com/user-attachments/assets/fe3a24b7-caf2-46af-8240-7225507748f4)
> [FORENSIA: ローカルLLMフォレンジックハーネス](https://speakerdeck.com/sumeshi/forensia-rokarullmhuorenzitukuhanesu?slide=3)

かといって、固定資産扱いにならない価格帯の機器では、ローカルLLMに調査そのものを任せられない。
どうしようかなぁと考えていてふと思いました。**別にフォレンジックやらせなくてもいいな**と。

フォレンジックで本当に大変なのは、調査そのものではないんですよね。**全体像を把握できる形**で**一貫性を保ちながら整理し続ける**ことです。

フォレンジックでは、とにかくいろんなアーティファクトやログを見ます。もちろんパースするためのツールも無数にあり、ものによって見る部分も粒度も全く違う。

もちろん、EDRとセット販売されている分析ツールや、統合フォレンジックツールのようなものもありますよ。それでも、**後から出てきたネットワーク図のような謎の壁画** や **現場SEによる証言**、**調査本部が行った対応** など諸々をタイムラインに整理し続けるのはつらい。

まとめるツールが Excel とか PowerPoint だとより辛い。マジに気が狂います。


## AIにフォレンジックをさせないという選択

というかフォレンジックで一番楽しいのって **ヤバそうな痕跡を見つけること** なのに、それをAIに任せてどうするんだ。さらに、任せた結果がいまいちだと悲しくなってしまいます。**「これぼくがやったほうが早いのでは？」**と思ってしまうことも。

なので、ぼくがやることにします。

> 「イベントログによると、1月12日の4時56分にこのIPからログオンしている」  
> 「その直後にPowerShellが動いている」  
> 「ファイアウォールログを見るとその付近で外部通信が発生している」  

こんなふうに、人間様は見つけた事実をAIにポンポンなげるだけ。  
AIは関連する証拠を探し、証拠への参照付きでまとめる形の分業がいいんじゃないかなって。

完全にHuman-in-the-Loopを前提としたローカルLLMの運用をするということですね。  
これを、**AI-assisted Forensic Scribing** (AI支援型フォレンジック書記) と呼ぼうと思います。


## Pi: Coding Agent で始めるフォレンジック生活

前述の [FORENSIA](https://github.com/sumeshi/forensia)では、LLMを調査ループの中でまともに走らせるために、かなり過剰な制御機構を組みました。~~取り込んだ証拠の正規化、検知ルールによる調査起点列挙、仮説の生成、調査カバレッジの管理、コンテキスト管理、出力を鵜呑みにしないための検証...~~ 自律的な調査をそれなりの精度でやろうとすると、最低でもこのくらいは必要になります。

でも、今回のアイデアならもっと雑に始められます。  
既存のハーネスに組み込まれているループの仕組みに乗っかり、ウワモノを乗せてあげれば十分です。

ベースは [Pi Coding Agent](https://pi.dev/) にしました。  
これはミニマルな Harness で、デフォルトでは `read` `write` `edit` `bash` の機能しかありません。でも、そのミニマルさがイイ！となってオレオレハーネスを作っている層が一定いるとか。カレーをスパイスから作るタイプかもね。ぼくはそう。


## 実際に作ってみる

せっかくなのでちょいと作ってみて威力を確かめようかなと。

このプロジェクトは仮に **[Lepisma](https://github.com/sumeshi/lepisma)** と名付けました。  
由来は紙魚の学名から。


### Pi の準備

[Pi](https://pi.dev/) のインストールは公式Webサイトを見てください。npmやらなんやらで入れればOK。

あとは使うモデルも設定してあげましょう。
**baseUrl** にAPIサーバのURLを指定して、あとはVRAM容量を見ながら **contextWindow**, **maxTokens** をいい感じに調整する。

下記の設定は私の使っているものとほぼ同じですが、**寄せ集めパーツのデスクトップPC** + 2万円くらいで買った **RTX 2070 SUPER** + **[llama-server](https://github.com/ggml-org/llama.cpp/tree/master/tools/server)** で十分に動きます。

`~/.pi/agent/models.json`
```
{
    "providers": {
        "ornith": {
            "baseUrl": "http://192.168.1.123:8080/v1",
            "api": "openai-completions",
            "apiKey": "none",
            "compat": {
                "supportsDeveloperRole": false,
                "supportsReasoningEffort": false
            },
            "models": [
                {
                    "id": "Ornith-1.5-9B",
                    "name": "Ornith-1.5-9B",
                    "reasoning": true,
                    "contextWindow": 131072,
                    "maxTokens": 16384
                }
            ]
        }
    }
}
```


### 構成

Pi Coding Agent の動きは以下の2つの要素で制御します。

- `AGENTS.md`
- `.pi/skills/<skill_name>/SKILL.md`

フォルダ構成にするとこんなかんじ。

```
lepisma/
 ├── AGENTS.md
 ├── timeline.csv
 ├── sources/
 └── .pi/
    └── skills/
        ├── lepisma-search/
        │   └── SKILL.md
        ├── lepisma-summarize/
        │   └── SKILL.md
        ├── lepisma-tag/
        │   └── SKILL.md
        └── lepisma-timeline/
            └── SKILL.md
```

ひとつずつ見ていきましょう。

### AGENTS.md

セッション中は常時ロードされる。  
プロジェクトの全体方針などを書く。

```
# Lepisma

あなたはデジタルフォレンジック調査を支援するアシスタントです。

調査の主体は分析者であり、自律的に調査方針を決定しない。
分析者から与えられた情報を調査の起点として扱い、必要に応じて以下のSkillを利用する。

* `lepisma-search`: 関連する証拠レコードを検索する
* `lepisma-summarize`: 証拠レコードを簡潔かつ客観的に要約する
* `lepisma-tag`: イベントに一貫した `event_type` と `tags` を付与する
* `lepisma-timeline`: 証拠を確認し、`timeline.csv` を更新する

## 基本方針

分析者から時刻、IPアドレス、ホスト名、ユーザー名、プロセス名、ファイル名、その他の調査上の情報を与えられた場合は、その情報を起点として作業する。

事実として記録する内容は、可能な限り `sources/` 配下の元データから確認する。

分析者の発言だけを根拠として、確認できていない内容を事実として記録しない。

証拠に存在しない情報を推測して補完しない。

明示的に依頼されていない方向へ調査を広げない。

## timeline.csv

調査結果の時系列記録は `timeline.csv` に保存する。

新しい情報を追加する前に、既存の `timeline.csv` を確認する。

同じイベントを重複して追加しない。

既存の `event_type` と `tags` を確認し、同じ意味の値が存在する場合は再利用する。

表記違いだけの分類やタグを新しく作らない。

元の証拠を確認できるよう、`source_record` と `source_file` を保持する。

## sources/

`sources/` 配下のファイルは証拠または解析結果として扱う。

原則として内容を変更しない。

検索、参照、読み取りの対象として利用する。

## 分析者との役割分担

分析者は、何を見るか、何が重要か、次にどこを調査するかを決める。

Lepismaは、分析者から与えられた起点をもとに証拠を探し、確認し、要約し、分類し、時系列として整理する。

調査官として振る舞うのではなく、分析者の調査を記録・整理する補助者として振る舞う。
```

### SKILL.md

必要に応じてロードされる。  
手順が明確で定型化しやすい処理を簡潔に書く。関数的な使い方。

#### lepisma-search/SKILL.md

```
---
name: lepisma-search
description: Search source files for evidence related to analyst-provided investigation anchors.
---

Use analyst-provided timestamps, IP addresses, hostnames, usernames, process names, filenames, and other identifiers as search anchors.

Search relevant records under `sources/`.

Also search for keywords directly derived from the analyst-provided information to reduce missed relevant records.

For each result, preserve the complete original record and its source file.

Do not infer or invent information that cannot be verified from the evidence.
```

#### lepisma-summarize/SKILL.md

```
---
name: lepisma-summarize
description: Summarize an evidence record into a short and objective timeline entry.
---

Summarize the provided evidence record in approximately 50 characters or a similarly concise length.

The summary must allow a human reader to quickly understand what happened.

Do not add information that does not exist in the original record.

Do not include speculation, interpretation, or evaluation.
```

#### lepisma-tag/SKILL.md

```
---
name: lepisma-tag
description: Assign consistent event types and tags to timeline events.
---

Assign `event_type` and `tags` to an event.

Before assigning values, inspect the existing `timeline.csv`.

Reuse an existing value when the same or a sufficiently similar value already exists.

Do not create new values that differ only in spelling, capitalization, wording, or formatting.

Create a new value only when the existing values cannot appropriately describe the event.

Use `event_type` to describe the type of event itself.

Use `tags` to provide keywords that help identify, search, and correlate related events later.
```

#### lepisma-timeline/SKILL.md

```
---
name: lepisma-timeline
description: Search evidence and update the forensic timeline based on analyst-provided information.
---

Use the `lepisma-search` skill to search for evidence related to the information provided by the analyst.

Use the search results to update `timeline.csv`.

Do not add an event if the same event already exists in the timeline.

Use the following columns:

- `timestamp`: Original timestamp recorded in the source data.
- `timestamp_utc`: Timestamp normalized to UTC (`UTC+00:00`).
- `event_type`: Event classification such as `Logon`, `Logoff`, or `UserAdd`. Use the `lepisma-tag` skill.
- `summary`: Short and objective description of the event. Use the `lepisma-summarize` skill.
- `source_record`: Complete original record used as evidence.
- `source_file`: Source file containing the original record.
- `note`: Additional notes or analyst-provided context.
- `tags`: Keywords used to search and correlate related events. Use the `lepisma-tag` skill.

When the timezone can be determined, normalize the timestamp to UTC and store it in `timestamp_utc`.

Always preserve `source_record` and `source_file`.

Use `-` when a value cannot be determined.
```

#### sources

調査元データをおいておきます。テキストがいいかな。  
検証にあたっては、[CFReDS - Data Leakage Case](https://cfreds-archive.nist.gov/data_leakage_case/data-leakage-case.html) のイベントログを [EvtxECmd](https://github.com/EricZimmerman/evtx) でパースして置いておきました。


#### timeline.csv

まとめ先のスーパータイムライン。カラムは適当。

| カラム名            | 概要                                                  |
| --------------- | --------------------------------------------------- |
| `timestamp`     | 元データに記録されているタイムスタンプ                                 |
| `timestamp_utc` | UTC（UTC+00:00）に正規化したタイムスタンプ                         |
| `event_type`    | イベントの分類。`Logon`、`Logoff`、`UserAdd` など、内容に応じて任意に設定する |
| `summary`       | イベントの内容を簡潔にまとめた説明                                   |
| `source_record` | 判断の根拠となった元ログの内容                                     |
| `source_file`   | 元データが格納されているソースファイル                                 |
| `note`          | 補足事項や分析者のメモ                                         |
| `tags`          | 後から関連イベントを検索・関連付けするためのタグ                            |


#### 結果

![lepisma](https://github.com/user-attachments/assets/5b8eb8ac-d8b0-44f7-8d22-c6e38249356c)

これだけでもまぁまぁ形になっています。Skillの呼び出しも適宜LLM側で判断して使えているみたい。  

課題としては、**CSVファイルを編集するためにいちいちPythonスクリプトを作って** ガチャガチャいじったりしているみたいなのでそこで若干時間とトークンを食う感じはしました。**あらかじめスクリプトやSkillを用意しておく**か、**[SQLite](https://www.sqlite.org/)や[DuckDB](https://duckdb.org/)にまとめる** 方式でもいいかもしれませんね。

あとは、見つけたものをワッと入れて休憩している間に整理してくれたら嬉しいので、**キューイングや並列化** などを考えてもいいと思います。


## おわりに

フォレンジック中いつも思うのですが、重い解析ソフトを使うことはあまりないので、マシンリソースには割と余裕はあるんですよね。  
機械は使い倒してナンボなので、**調査しながらめんどい整理仕事はAIにぶん投げる**、くらいのゆるい使い方でも悪くないんじゃないかなと思いました。

ね？ハム太郎。

おしまい
