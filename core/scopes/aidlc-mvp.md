---
name: mvp
depth: Standard
keywords:
  - mvp
  - minimum viable
description: Skip operations, ship the core
skeleton: on
runner: true
guard_policy: off
sensors: on
learnings: on
summary_confirmation: on
plan_approval: on
collaborators: off
---

# mvp scope

Standard の深度だが、ワークフローの前後を削ってコアを素早く
リリースする。Ideation は縮小した手順で実行し（market-research、
team-formation、approval-handoff なし）、operation フェーズ全体を
スキップする。MVP はプロダクトを実証するものであり、まだ本番運用の
重みを担うものではない。

Guard Policy のデフォルトは off である: 変更された入力は記録されて通知され、実行は継続し、指示のない作業についてはプラン承認、レビューフリーズ、状態遷移、レビュアーの読み取り範囲が緩められる。人間の在席確認は維持される。

## Why these stages, why skip those

inception と construction の各工程はすべて EXECUTE のままとする: MVP も
設計、コード、テストを必要とする本物のソフトウェアである。スキップするのは、
規模が大きくなって初めて元が取れる探索のオーバーヘッド（market-research、
team-formation）と、最初の実証を終えたプロダクトに属する operation のステージ（deployment-pipeline、
environment-provisioning、deployment-execution、observability-setup、
incident-response、performance-validation、feedback-optimization）である。
MVP を卒業したら `feature` または
`enterprise` に昇格させる。

## Membership

キーワードトリガー: `mvp`、`minimum viable`。Initialization、縮小した
ideation のセット、inception のすべて、construction のビルド経路を実行し、
operation はまるごとスキップする。
