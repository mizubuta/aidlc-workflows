---
name: poc
depth: Minimal
keywords:
  - proof of concept
  - prototype
  - poc
  - spike
description: Prove feasibility fast
skeleton: on
review_cap: advisory
guard_policy: off
sensors: on
learnings: on
summary_confirmation: on
plan_approval: off
collaborators: off
---

# poc scope

実現可能性を素早く証明することを目的とした Minimal 深度。動くコードに至る
最短経路以外はほぼすべてスキップする: インテントを捉え、既存コードがあれば
リバースエンジニアリングし、要件を引き出し、その後コードを生成してテストする。
設計の儀式も、運用も、デリバリー計画も行わない。

Guard Policy の既定値は off である: 変更された入力は記録・通知されるがスパイクは止まらずに進み、指示のない作業に対しては計画承認、レビューフリーズ、状態遷移、レビュアーの読み取りスコープが引き下げられる。人の関与（human presence）は維持される。

計画承認は off である: コード計画が書かれると、それを示す 1 行が表示されて
コード生成が始まり、その行にはいつでも計画の確認を依頼できる旨が示される。
構築前に計画を見せるよう依頼することも、すべての計画について確認を求めるよう
依頼することもできる。

## Why these stages, why skip those

PoC（概念実証）が答えるのは「これは実現できるか？」という 1 つの問いだけである。
そのため、答えにたどり着くためのステージだけを残す: intent-capture、reverse-engineering、
requirements-analysis、code-generation、build-and-test。スパイクは使い捨てなので、
それ以外（domain-design、units-generation、NFR 関連の作業、Operation フェーズ）を
捨てることこそが要点である。答えが yes なら、`feature`/`mvp` にスコープを変更し、
本番の構築で全工程を実行する。

## Membership

キーワードトリガー: `proof of concept`, `prototype`, `poc`, `spike`.
Initialization と、実現可能性を確かめる最小限の経路だけを実行する。それ以外はすべて
SKIP となる。
