---
id: upstream-coverage
kind: deterministic
command: {{INVOKE}} engine sensor-upstream-coverage
default_severity: advisory
fire_on: gate
description: Checks the stage's deliverables reference the upstream artifacts the stage frontmatter declares it requires
category: document-shape
matches: "**/{aidlc-docs,intents,codekb}/**"
input_schema:
  output_path: string
  stage_slug: string
  consumes: string[]
  deliverables: string[]
output_schema:
  pass: boolean
  unreferenced_artifacts: string[]
timeout_seconds: 300
---

# upstream-coverage sensor

ステージ frontmatter の `consumes:` リストを読み込み、存在する必須の上流成果物それぞれを
ステージの成果物が参照しているかを検査する。任意のもの（`required: false`）は、
出力の根拠になっていない可能性があるため検査しない。ただし、ステージがそれに `cited: true` を付けている場合は検査する。参照は、フレームワークの
成果物が実際に使う次のいずれの形式でも数える。

- 独立したトークンとしての成果物 slug（より長いケバブ形式のトークンに埋め込まれたものは
  数えない: `nfr-requirements` の中の `requirements` は数えない）
- wikilink（`[[<slug>]]`）またはバッククォートで囲んだファイル名（`` `<slug>.md` ``）
- パスセグメントとしての生成元ステージのディレクトリ（例: `nfr-requirements/` を引用する
  出自ヘッダーは、そのステージが生成するすべての成果物をカバーする）

カバレッジはファイルごとではなく、ステージの出力全体の性質である。
検査は出力ディレクトリ内のステージが宣言した成果物の和集合に対して実行される（足場ファイル - `*-questions.md`、`*-timestamp.md`、
`memory.md` - は双方から除外される）ため、兄弟の成果物での引用も数える。

frontmatter からの純粋な導出であり、ステージごとの設定は不要。

## Failure mode

`SENSOR_FAILED` を発行し、`consumes:` で宣言された必須または引用対象の成果物のうち、
ステージのどの成果物にも、受け入れられるどの形式でも現れないものを列挙した詳細を書き込む。
