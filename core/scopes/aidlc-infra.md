---
name: infra
depth: Standard
keywords:
  - infrastructure
  - deploy
  - infra
description: Infrastructure changes
skeleton: on
guard_policy: off
sensors: on
learnings: on
summary_confirmation: on
plan_approval: on
collaborators: off
---

# infra scope

インフラストラクチャ変更向けの Standard 深度。グラフの後半に寄った経路をとる唯一のスコープであり、
ideation と、プロダクト設計に関する inception および construction のステージをスキップする。
代わりに practices-discovery、NFR 設計パス、infrastructure-design、
IaC と CI パイプラインのための code-generation と build-and-test、そして operation における
デプロイ + オブザーバビリティ一式を実行する。

Guard Policy の既定値は off である。承認後に変化したプロビジョニングやデプロイの入力は、承認をやり直すのではなく記録して通知する。指示のない作業については、プラン承認、レビューフリーズ、状態遷移、レビュアーの読み取りスコープの強度を下げる。人の関与（Human presence）は高いまま維持する。

## Why these stages, why skip those

インフラストラクチャの作業はプロダクト機能に関するものではないため、ideation、
user-stories、domain-design、units-generation、functional-design
はスキップする。対象はシステムをどうプロビジョニングし運用するかであるため、
operation のステージ（deployment-pipeline、environment-provisioning、
deployment-execution、observability-setup）と、それらに入力を与える NFR 設計および
インフラストラクチャ設計を EXECUTE とする。

IaC そのもの（CDK、Terraform、CloudFormation）はコードであるため、code-generation が
インフラストラクチャ設計をもとに、利用者の Plan Approval のもとでそれを書く。
units-generation がスキップされるため、これはユニット数ゼロの 1 パスとして実行される。
続いて build-and-test が IaC の静的チェックとテストを実行する。Infrastructure
Design はインフラストラクチャを記述するだけであり、Environment Provisioning は
code-generation が書いた IaC を適用するだけである。どちらも IaC を作成しない。

`reverse-engineering` が SKIP となるのはこのスコープだけである。インフラストラクチャの変更は
アプリケーションのソースではなくデプロイトポロジーから出発するためである。また、
enterprise/feature 以外で operation フェーズを実行する唯一のスコープでもある。

## Membership

キーワードトリガー: `infrastructure`、`deploy`、`infra`。Initialization、
practices-discovery、requirements-analysis、NFR + インフラストラクチャ設計の
各ステージ、code-generation、build-and-test、ci-pipeline、そして
デプロイ/プロビジョニング/オブザーバビリティの operation ステージを実行する。
