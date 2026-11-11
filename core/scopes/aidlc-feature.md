---
name: feature
depth: Standard
keywords: []
description: Full lifecycle for new features, practical depth
skeleton: on
runner: true
guard_policy: off
sensors: on
learnings: on
summary_confirmation: on
plan_approval: on
collaborators: off
---

# feature scope

新機能の開発作業を実用的な深さで進める、ライフサイクル全体のスコープ。
`enterprise` と同様にグラフ内のすべてのステージを実行するが、ステージ本体は
Comprehensive ではなく Standard の深さを適用する。手続きは軽くなるが、
ideation から operation までのエンドツーエンドの網羅性は同じである。

Guard Policy の既定値は off である。変更された入力は記録・通知され、実行は継続し、指示のない作業についてはプラン承認、レビューフリーズ、状態遷移、レビュアーの読み取り範囲が緩和される。人間の関与（human presence）は維持される。

## Why every stage

新機能であっても、一連の流れ全体が必要である。問題を理解し（ideation）、
設計し（inception）、構築・テストし（construction）、運用する（operation）。
`feature` はこれらをすべて保持するため、何かが黙って抜け落ちることはない。
`enterprise` との違いは深さであり、それはステージ本体と org/team のルール層で
表現される。どのステージを実行するかの違いではない。

## Membership

`feature` は 33 個のステージすべてを EXECUTE とする。暗黙の自由形式フォールバックであり続け、
`--scope feature` や `/aidlc-feature` ランナーからも利用できる。
独自のキーワードトリガーは持たない。
