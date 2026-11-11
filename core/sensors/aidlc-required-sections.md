---
id: required-sections
kind: deterministic
command: {{INVOKE}} engine sensor-required-sections
default_severity: advisory
fire_on: gate
description: Checks at the gate that stage output contains the required H2 headings
category: document-shape
matches: "**/{aidlc-docs,intents,codekb}/**"
input_schema:
  output_path: string
  stage_slug: string
output_schema:
  pass: boolean
  h2_count: integer
  headings: string[]
  findings_count: integer
  edge_block: string
  template: string
  template_expected: string[]
  template_missing: string[]
  config_warning: string
  message: string
timeout_seconds: 300
---

# required-sections sensor

出力に対するテンプレートがない場合、必須の見出しはない。ファイルの構成はステージの散文が決めるため、
チェックは合格とし、出力に含まれる `##` 見出しを報告するだけにとどめる。
テンプレートが適用される場合（後述）は、その `##` 見出しが必須となる。
不合格の結果には `message` が含まれる。これは何が欠けているかを述べる平易な 1 文であり、
詳細ファイルに保存され、エージェントはそれをもとに修正する。

`unit-of-work-dependency.md`（units-generation 2.7）については、さらに
フェンス付きの `yaml` `units:` エッジブロックが存在し、整形式で、
循環がないことを要求する。これはランタイムコンパイラがバッチのファンアウトへと解析する、
機械可読の DAG である。チェックは `edge_block` を `ok`、`absent`、
`malformed`、`cyclic` のいずれかとして報告する。`ok` 以外はゲートでセンサーを不合格にするため、
不正なブロックがコンパイラに届くことはない。その他の成果物には
後述のテンプレートチェックだけが適用される。

## Heading-set overrides — two paths, with precedence

見出しセットを必須にする方法は 2 つある。**優先順位: 解決できた
`templates/<artifact>.md` は、`## Sensors` に記述された見出しセットより優先される。**

1. **テンプレート上書きレイヤー（ファイル駆動、推奨）。** チームが
   `aidlc/spaces/<space>/memory/templates/<artifact>.md` を配置する（出力ファイル名の語幹をキーとする。
   成果物 `X` は `X.md` に書き込まれる）。出力パスに対してこれが解決されると、その
   `##` 見出しが期待セットとなり、センサーは
   `expected ⊆ output` の場合に限り合格する。欠けている見出しはそのまま正確な指摘事項となる。見出しは
   先頭の絵文字の有無にかかわらず一致する（`## 📌 Overview` は `## Overview` とみなす）。
   これは claim-sources センサーが見出しを読むときと同じルールである。文書全体が対象で、
   助言レベル（advisory）であり、判断はゲートで人が行う。同じファイルがエージェントの埋める
   スケルトンでもある（ステージプロトコルの § Template overrides を参照）ため、生成される
   構成とチェックされる構成がずれることはない。テンプレートが適用されるのは
   テンプレート対象の成果物（ステージの散文にある `produces` エントリで、
   ディスパッチャーが受け渡すもの）だけである。questions/timestamp マーカーに対して解決された
   テンプレートは、設定警告を出したうえで無視する。

2. **`## Sensors` 散文による上書き（ステージ内、レガシーの先行策）。** ステージの
   `## Sensors` 本文には、見出しセットの上書きを記述できる（マイルストーン 12 以降の
   仕組み）。これは出力に対してテンプレートファイルが解決されない場合にのみ適用される。

フレームワークは既定でステージごとの `## Sensors` 上書きを同梱していない。チームは
テンプレートを作成する（経路 1）か、正当な理由があるときに §13 の学びのループを通じて、
特定の見出し構成を導入する。どちらの上書きもない場合、必須の見出しはない。

## Failure mode

必須の見出しが欠けている場合、`SENSOR_FAILED` を発行し、詳細を
`aidlc/spaces/<active-space>/intents/<active-intent>/.aidlc-engine/sensors/<stage-slug>/required-sections-<fire-id>.md` に書き込む。
スペースとインテントはアクティブなカーソルから取得する。fire id は、アクティブな記録の
`audit/<host>-<clone-id>.md` シャードにある `SENSOR_FIRED` 行の
8 桁 16 進の相関 ID である。詳細には欠けている見出しを列挙する。
