# プロンプトエンジニアリング
GPT-4 契約してるけど全然つかってない。

## 明確な指示
明確な指示を出す。  
偉い人がいいがちないい感じでよしなにエイヤ、ではゴミみたいなWikipediaの概要みたいな応答しか帰ってこない。これは人間に対してもそう。

また、対象やシチュエーションを明確にするとよい。  

> 例: 会社の幹部向けに、経営に関する興味を引き出すように◯◯を説明したいです。～

## 情報量のレベル
個数や行数、想定する媒体を説明するとよい。  
これがないと虚無の文章を提示されて生成中に白目をむくことになる。

> 例:
> - 〇〇についてn行程度で説明してください。
> - 〇〇をA4サイズ1枚、n文字程度で説明したいです。

## 例示
例をいくつか提示して、その続きを書かせるようにするとうまくいく。

> 例: 
> 〇〇について5段落程度で説明したいです。下記を参考に、続きを書いてください。
> 1. 〇〇の概要
> 2. ～

## シンプルな条件
アンチパターンは示さない。  
言語処理の分野で否定文や打消しとかやるのってマジムズいしたぶん生成AIも同じ。  
コードの条件式が `if not item.isNotEmpty()` みたいなのだと頭がこんがらがるからやめてほしいよね。

> 例:
> 　◯: 明るい雰囲気の文章で書いてください。, 暗い雰囲気の文章で書いてください。
> 　✕: 明るい雰囲気の文章では書かないでください。暗い雰囲気の文章で書かないでください。

## 参考文献
[Google - プロンプト設計戦略](https://ai.google.dev/docs/prompt_best_practices?hl=ja)
