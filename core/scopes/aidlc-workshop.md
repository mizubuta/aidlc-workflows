---
name: workshop
depth: Standard
testStrategy: Minimal
keywords:
  - workshop
  - lab
  - training
description: Facilitated group session with mandatory gates
skeleton: on
review_cap: advisory
guard_policy: off
sensors: on
learnings: on
summary_confirmation: on
plan_approval: on
collaborators: off
---

# workshop scope

必須ゲートを備えたファシリテーター主導のグループセッション向けの Standard 深度だが、
テスト戦略は Minimal とする。これは、教育の場でテストの最低基準を軽く保つための
`testStrategy` の上書きである。inception、construction、operation の流れを端から端まで
実行し（参加者がライフサイクル全体を見られるようにするため）、ファシリテーターが事前に
手作業で済ませておく ideation の探索ステージはスキップする。

Guard Policy の既定値は off である。移動された入力は一度だけ報告され、指示のない作業に対しては plan approval、review freeze、state transition、reviewer read scope が緩和される。human presence は有効のまま維持され、承認を再び必須にしてすべてのフェンスを有効にしたいファシリテーターは strict を設定する。

## Why these stages, why skip those

ワークショップはグループを方法論に沿って導くものなので、実質的な構築と運用のステージは
見える状態に保つ。reverse-engineering、practices-discovery、inception の設計パス全体、
construction、そして operation フェーズはすべて実行される。初期の ideation の手続き
（market-research、feasibility、scope-definition、team-formation、
rough-mockups、approval-handoff）はスキップする。ファシリテーターがこれらのステージを
その場で実行するのではなく、演習のスコープを事前に決めておくためである。
`testStrategy: Minimal` の上書きにより、テストは実演として扱われ、本番レベルの
最低基準は求められない。

## Membership

キーワードトリガー: `workshop`、`lab`、`training`。Initialization、inception の
ステージ群、construction のすべて、operation のすべてが実行され、ideation の探索
ステージは SKIP となる。
