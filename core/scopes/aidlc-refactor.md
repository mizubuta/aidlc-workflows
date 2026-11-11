---
name: refactor
depth: Minimal
keywords:
  - refactor
  - clean up
  - simplify
description: Clean up existing code
skeleton: off
existing_code: true
guard_policy: off
sensors: on
learnings: on
summary_confirmation: on
plan_approval: on
collaborators: off
---

# refactor scope

既存コードを、振る舞いを変えずにクリーンアップするための Minimal 深度。
`bugfix` と同様に ideation と大半の operations をスキップするが、
functional-design を加え直す（リファクタリングは構造を作り変えるため、
維持すべき振る舞いの設計が重要になる）。また、検証済みの変更を出荷するために
必要なデプロイ系ステージも残す。

Guard Policy の既定値は off である。変更された入力は承認をやり直すのではなく、記録して告知する。計画承認、レビュー凍結、状態遷移、レビュアーの読み取りスコープは、指示のない作業向けに緩和される。人間の関与（Human presence）は高いまま維持される。

## Why these stages, why skip those

リファクタリングは、既知のコードベースに対する構造保存型の変更である。
reverse-engineering（既存のものを理解する）、requirements-analysis
（維持すべき振る舞いを確定する）、functional-design（目標とする
形）を実行し、続いて code-generation と build-and-test（変更を適用し、
既存のテストスイートがグリーンのままであることを検証する）を実行する。
新しいプロダクトやインフラの面がないため、discovery、環境のプロビジョニング、
より広範な運用準備はスキップする。一方、deployment-pipeline と deployment-execution
は、検証済みのリファクタリングを既存のデリバリー経路で届ける。
walking-skeleton の手順をスキップする 3 つのインクリメンタルなスコープの 1 つである。

## Membership

キーワードトリガー: `refactor`, `clean up`, `simplify`。Initialization、
reverse-engineering、requirements-analysis、functional-design、
code-generation、build-and-test、deployment-pipeline、
deployment-execution を実行し、それ以外は SKIP となる。
