---
slug: observability-setup
phase: operation
execution: CONDITIONAL
condition: Execute when monitoring, dashboards, alarms, or tracing need configuration
lead_agent: aidlc-operations-agent
support_agents: []
mode: inline
summary_confirmation: required
produces:
  - dashboards
  - alarms
  - slo-config
  - log-queries
  - tracing-config
  - anomaly-config
  - observability-setup-questions
consumes:
  - artifact: performance-design
    required: true
  - artifact: security-design
    required: true
  - artifact: reliability-design
    required: true
  - artifact: monitoring-design
    required: true
  - artifact: infrastructure-specification
    required: true
requires_stage:
  - nfr-design
  - infrastructure-design
  - deployment-execution
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
  - infra
  - workshop
  - express
inputs: NFR design from nfr-design stage, infrastructure design from infrastructure-design stage, deployed application
outputs: dashboards.md, alarms.md, slo-config.md, log-queries.md, tracing-config.md, anomaly-config.md, observability-setup-questions.md (under this stage's record dir, engine-resolved)
---

# Observability Setup

## Steps

### Step 1: Load Prior Context

- `<record>/construction/nfr-design/` から NFR 設計（オブザーバビリティ戦略）を読み込む
- `<record>/construction/infrastructure-design/` からインフラ設計を読み込む
- `<record>/operation/deployment-execution/` からデプロイ実行ログを読み込む

`express` は設計上、NFR Design と Infrastructure Design をスキップする。それらの
成果物が存在しない場合は、承認済みの要件、デプロイ済みアプリケーションの
ワークスペース設定、Build and Test の結果、Deployment Execution の証跡から、
最小限の観測対象を導き出す。これらの情報源から観測できない SLO、シグナル、
保持期間、エスカレーションの判断については質問する。
欠けている設計成果物を決してでっち上げない。デプロイ済みのターゲットが存在しない場合、
この CONDITIONAL ステージは skipped を報告する。

### Step 2: Generate Clarifying Questions

次の内容を扱う質問ファイルを作成する:
- 追跡すべきゴールデンシグナル（レイテンシ、トラフィック、エラー、サチュレーション）は何か？
- どのような SLO/SLI が定義されているか？
- チームにはどのようなダッシュボードのレイアウトが必要か？
- どのようなログの保持ルールと集約ルールが適用されるか？
- どのような分散トレーシングの計装が必要か？

stage-protocol.md の質問フローに従う。

### Step 3: Generate Artifacts

CloudWatch ダッシュボードの設定、アラームの定義（重大度、SNS ルーティング、エスカレーションを含む）、SLO/SLI の追跡設定、CloudWatch Logs Insights の保存済みクエリ、X-Ray トレーシングの設定、異常検知の設定を作成する。

### Step 4: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage observability-setup --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 5: Present Completion & Request Approval

完了時の絵文字: :eyes:
レビュー対象のパス: `<record>/operation/observability-setup/`
標準の 2 択の承認（Approve / Request Changes）。

## Sensors

このステージの出力は `<record>/operation/observability-setup/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `performance-design`, `security-design`, `reliability-design`, `monitoring-design`, `infrastructure-specification`.

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
