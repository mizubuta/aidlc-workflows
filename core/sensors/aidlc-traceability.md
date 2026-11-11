---
id: traceability
kind: deterministic
command: {{INVOKE}} engine sensor-traceability
default_severity: advisory
fire_on: gate
description: Verifies element-level upstream coverage, downstream targets, and derived orphans in traceability.json
category: document-traceability
matches: "**/traceability.json"
input_schema:
  output_path: string
  stage_slug: string
output_schema:
  pass: boolean
  gaps: string[]
  orphans: string[]
  missing_from_table: string[]
  missing_from_upstream_ids: string[]
  invalid_entries: string[]
  invalid_targets: string[]
  findings_count: integer
  reason: string
timeout_seconds: 300
---

# traceability sensor

各ステージの JSON カバレッジ表を要素レベルで決定論的に検証する。ステージの
ドキュメントが引き渡されるとき（そのレビュー依頼時、またはレビューが先に
受け取らなかった場合はゲートを開くレポート時）に実行され、ゲートが開くときにも、
その時点の表に対して再度実行される:

1. 実行時の JSON の形と、閉じたステータス集合を検証する。
2. `GAP`、`ORPHAN`、欠落したカバレッジ行、または宣言されていない上流 ID があれば失敗とする。
   サブ要件も上流にある要件（`FR1` と並ぶ
   `FR1.1`）はサブ要件を通じてトレースされ、それ自体の行は不要である。
3. `OK`、`Deferred`、`N/A` には空でない target を必須とする。`Deferred`
   と `N/A` では target が理由となる。
4. 作成済みの成果物から上流 ID を解決し、期待されるソースが欠落しているか
   ID が得られない場合はフェイルクローズする。FR と NFR の ID は、Requirements Analysis が
   自身の traceability.json に記録する `ids` から取得され、requirements.md での
   表記どおりに綴られる（それぞれ `kind` を持ち、サブ要件は自身の要件の `of` となる）。
   その記録がない場合、表の要件は、その表に列挙されたものと、FR および NFR の
   ID パターンが requirements.md で見つけたものとなる。
5. 可能な場合は決定論的な target を検証する: ストーリー、Unit マッピング、
   ビジネスルール、ワークスペース相対のコードパス。種別上 `rules.md` を書かない
   Functional Design の Unit（`ui` Unit）は、ビジネスルールの代わりに、各行で
   自身の設計ファイル（`functional-spec.md`、`frontend-components.md`）の 1 つを指定する。
6. 自己申告の `reverse` 配列だけを信頼するのではなく、
   `rules.md` から functional-design の孤立要素（orphan）を導出する。

後続のステージは、同じツールを `--stage <slug> --output-path
<its traceability.json> --write-upstream` で実行して表を開始する。これにより、このチェックが
要求するすべての上流 ID がそれぞれ `GAP` 行付きで `upstream_ids` に追加される。表が
既に保持している内容は維持され、欠落がなければ何も書き込まれない。

Build and Test のカバレッジゲートは、同じツールを `--stage build-and-test`
付き、出力パスなしで実行する。すべての Code Generation の表を読み込み、各
要件と受け入れ基準について構築済みかどうかを出力する。サブ要件を持つ要件は、
各 Unit がそれぞれ自分の分を構築したときに構築済みとなり、親だけの行では
サブ要件はカバーされない。それ以外の要件は、自身の行によって、または User Stories が
実行された場合は User Stories がトレースしたストーリーの受け入れ基準を通じて構築済みとなる。
何によっても構築されない要件は、それをトレースした最新のステージ（User Stories、
NFR Requirements、次いで Code Generation）が、そこで当該要件を列挙するすべての表で
理由付きの `N/A` とした場合に決着済みとなる。理由のない `N/A` は決着済みではない。

## Expected JSON schema

```json
{
  "stage": "functional-design",
  "unit": "u1-auth",
  "upstream_ids": ["AC1.1.1", "AC1.2.1"],
  "coverage": [
    { "id": "AC1.1.1", "status": "OK", "target": "BR1.1" },
    { "id": "AC1.2.1", "status": "GAP" }
  ],
  "reverse": [
    { "id": "BR1.3", "status": "ORPHAN" },
    { "id": "BR1.4", "status": "N/A", "target": "technical validation rule" }
  ]
}
```


有効なステータスは `OK`、`GAP`、`ORPHAN`、`Deferred`、`N/A` である。すべての
上流 ID は `upstream_ids` と `coverage` に含め、該当しない場合はそこで `N/A` とする。
`reverse` には、ステージが生成したもののうち上流の何にもトレースされないものだけを列挙する。
