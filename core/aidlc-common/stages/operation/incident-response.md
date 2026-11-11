---
slug: incident-response
phase: operation
execution: CONDITIONAL
condition: Execute when operational runbooks and incident response procedures are needed
lead_agent: aidlc-operations-agent
support_agents: []
mode: inline
summary_confirmation: required
produces:
  - runbooks
  - incident-plan
  - escalation-matrix
  - incident-response-questions
consumes:
  - artifact: dashboards
    required: true
  - artifact: alarms
    required: true
  - artifact: reliability-design
    required: true
  - artifact: security-design
    required: true
  - artifact: infrastructure-specification
    required: true
requires_stage:
  - observability-setup
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
  - workshop
inputs: Observability setup from observability-setup stage, NFR design from nfr-design stage, infrastructure design from infrastructure-design stage
outputs: runbooks.md, incident-plan.md, escalation-matrix.md, incident-response-questions.md (under this stage's record dir, engine-resolved)
---

# Incident Response & Runbook Generation

## Steps

### Step 1: Load Prior Context

- `<record>/operation/observability-setup/` からオブザーバビリティのセットアップを読み込む
- `<record>/construction/nfr-design/` から NFR 設計を読み込む
- `<record>/construction/infrastructure-design/` からインフラ設計を読み込む

### Step 2: Generate Clarifying Questions

次の内容を扱う質問ファイルを作成する:
- 最も起こりやすい障害モードは何か？
- エスカレーションパスとオンコールのローテーションはどうなっているか？
- どのような自動修復が可能か？
- インシデント発生中のコミュニケーション手順はどうなっているか？
- RTO/RPO の目標値は何か？

stage-protocol.md の質問フローに従う。

### Step 3: Generate Artifacts

SSM Automation ランブックライブラリ、インシデント対応計画（AWS Incident Manager と連携）、エスカレーションマトリクス、自動修復ドキュメント、災害復旧手順、AWS Backup の構成を作成する。

### Step 4: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage incident-response --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 5: Present Completion & Request Approval

完了時の絵文字: :fire_engine:
レビュー対象のパス: `<record>/operation/incident-response/`
標準の 2 択の承認（Approve / Request Changes）。

## Sensors

このステージの出力は `<record>/operation/incident-response/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `dashboards`, `alarms`, `reliability-design`, `security-design`, `infrastructure-specification`.

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
