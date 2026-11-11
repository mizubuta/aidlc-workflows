---
name: bugfix
depth: Minimal
keywords:
  - fix
  - bug
  - broken
  - bugfix
description: Fix a specific bug
skeleton: off
existing_code: true
runner: true
review_cap: advisory
guard_policy: off
sensors: on
learnings: off
summary_confirmation: off
plan_approval: on
collaborators: off
---

# bugfix scope

既存コードベースの特定のバグを 1 件修正するための Minimal 深度のスコープ。
新たに探索すべきプロダクトはないため ideation は丸ごとスキップし、
reverse-engineering で現在のコードを把握し、修正の要件を整理したうえで、
修正を生成・テスト・デプロイする。

Guard Policy の既定値は off である。入力が変更された場合、承認をやり直すのではなく記録して通知する。計画承認（plan approval）、レビュー凍結（review freeze）、状態遷移（state transition）、レビュアーの読み取りスコープ（reviewer read scope）は、指示のない作業向けに緩和される。人間の関与（Human presence）は高いまま維持される。

学びと要約確認は off である。ステージの日誌はなく、修正後の "Keep that
for next time?" という質問もなく、成果物を書き出す前に確認する要約もない。
センサー、すべてのステージ承認、計画承認は on のままである。
いずれもインテントごとに `/aidlc --learnings on` または
`/aidlc --summary-confirmation on` で再び有効にできる。Collaborators は off のため、
各ステージはリードエージェントのみで実行される。`/aidlc --collaborators on` で
サポートエージェントが加わる。

## Why these stages, why skip those

バグ修正は、既知のシステムに対する段階的な作業である。
既存のものを理解し（reverse-engineering）、「修正済み」が何を意味するかを明示し
（requirements-analysis）、変更と検証を行う（code-generation、
build-and-test）必要がある。market-research、user-stories、
domain-design、環境のプロビジョニング、より広範な運用準備は不要だが、
検証済みの修正を出荷できるよう deployment-pipeline と deployment-execution は残している。
このスコープは、walking-skeleton の手順をスキップする 3 つの段階的スコープの 1 つである
（`refactor` と `security-patch` と同様）。立ち上げるべきものが何もないためである。

## Membership

キーワードトリガー: `fix`, `bug`, `broken`, `bugfix`（単語境界で照合するため、
"debug" や "fixture" ではトリガーされない）。Initialization、
reverse-engineering、requirements-analysis、code-generation、build-and-test、
deployment-pipeline、deployment-execution を実行し、それ以外は SKIP となる。
