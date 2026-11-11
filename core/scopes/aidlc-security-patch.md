---
name: security-patch
depth: Minimal
keywords:
  - security
  - CVE
  - vulnerability
  - patch
description: CVE response
skeleton: off
existing_code: true
runner: true
guard_policy: off
sensors: on
learnings: on
summary_confirmation: on
plan_approval: on
collaborators: off
---

# security-patch scope

CVE や脆弱性に迅速に対応するための Minimal 深度のスコープ。
次の狭い経路をたどる: コードを理解し（reverse-engineering）、パッチが
満たすべきことを述べ（requirements-analysis）、セキュリティ上の制約を
記録し（nfr-requirements）、修正してテストし（code-generation、build-and-test）、
その後デプロイステージを通じて出荷し、パッチを確実に本番環境へ
届ける。

Guard Policy のデフォルトは off である。承認後に入力が変化したパッチは、再承認されるのではなく記録・通知される。指示のない作業については、プラン承認、レビューの凍結、状態遷移、レビュアーの読み取りスコープが緩和される。人間の関与（Human presence）は維持される。

## Why these stages, why skip those

セキュリティパッチは緊急かつ漸進的であり、必ずデプロイしなければならない。
変更が的を絞ったものであるため、設計のセレモニー全体（ideation、domain-design、units-generation、
nfr-design、infrastructure-design）をスキップする。他の
漸進的なスコープと同様に、deployment-pipeline と
deployment-execution は EXECUTE のまま残す。デプロイされないパッチでは
脆弱性は塞がらないからである。このスコープに特有のステージは nfr-requirements であり、
セキュリティ上の制約を記録する。requirements-analysis も実行し、脆弱性と
その是正基準について監査可能な記述を残す（nfr-requirements と code-generation が
消費する `requirements` 成果物）。walking-skeleton のセレモニーをスキップする
3 つの漸進的スコープの 1 つである。

## Membership

キーワードトリガー: `security`、`CVE`、`vulnerability`、`patch`。
Initialization、reverse-engineering、requirements-analysis、
nfr-requirements、code-generation、build-and-test、deployment-pipeline、
deployment-execution を実行し、それ以外は SKIP となる。
