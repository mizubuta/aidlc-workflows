---
slug: environment-provisioning
phase: operation
execution: CONDITIONAL
condition: Execute when AWS environments need provisioning or validation
lead_agent: aidlc-aws-platform-agent
support_agents:
  - aidlc-devsecops-agent
  - aidlc-compliance-agent
mode: inline
summary_confirmation: required
produces:
  - environment-inventory
  - validation-report
  - environment-provisioning-questions
consumes:
  - artifact: infrastructure-specification
    required: true
  - artifact: cd-config
    required: true
requires_stage:
  - infrastructure-design
  - deployment-pipeline
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
  - infra
  - workshop
inputs: Infrastructure design from infrastructure-design stage, CD pipeline config from deployment-pipeline stage
outputs: environment-inventory.md, validation-report.md, environment-provisioning-questions.md (under this stage's record dir, engine-resolved)
---

# Environment Provisioning

## Steps

### Step 1: Load Prior Context

- `<record>/construction/infrastructure-design/` からインフラストラクチャ設計を読み込む
- `<record>/construction/nfr-requirements/` からセキュリティ要件を読み込む

### Step 2: Generate Clarifying Questions

次の内容を扱う質問ファイルを作成する:
- すべての環境が Infra Design のとおりにプロビジョニングされているか？
- VPC、サブネット、セキュリティグループ、NACL は正しいか？
- Secrets Manager / Parameter Store のシークレットは正しく注入されているか？
- クロスアカウント / クロス VPC の接続性は検証されているか？

stage-protocol.md の質問フローに従う。

### Step 3: Provision and Validate

Code Generation が Plan Approval のもとで作成し、Build and Test が確認した IaC を使って、対象の AWS 環境をプロビジョニングする。このステージはその IaC を適用するだけであり、IaC を作成はしない。プロビジョニングに必要な変更がその IaC に欠けている場合は、ここで書くのではなく、停止して不足箇所を明示する。インフラストラクチャの構成を検証する。オーケストレーターがセキュリティ態勢の検証のために aidlc-devsecops-agent を呼び出す。

### Step 4: Generate Artifacts

プロビジョニング済み環境のインベントリ、インフラストラクチャ検証レポート、シークレットおよび Parameter Store の監査、スタックのデプロイログ、環境ヘルスチェックの結果を作成する。

### Step 5: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage environment-provisioning --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 6: Present Completion & Request Approval

完了時の絵文字: :cloud:
レビュー対象のパス: `<record>/operation/environment-provisioning/`
標準の2択の承認（Approve / Request Changes）。

## Sensors

このステージの出力は `<record>/operation/environment-provisioning/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `infrastructure-specification`, `cd-config`.

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
