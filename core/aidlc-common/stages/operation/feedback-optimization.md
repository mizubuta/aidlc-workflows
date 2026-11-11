---
slug: feedback-optimization
name: Feedback & Optimization
phase: operation
execution: CONDITIONAL
condition: Execute when ongoing operational monitoring and optimization are needed
lead_agent: aidlc-operations-agent
support_agents:
  - aidlc-aws-platform-agent
mode: inline
summary_confirmation: required
produces:
  - slo-report
  - cost-analysis
  - drift-report
  - feedback-loop
  - feedback-optimization-questions
consumes:
  - artifact: dashboards
    required: true
  - artifact: alarms
    required: true
  - artifact: slo-config
    required: true
  - artifact: deployment-log
    required: true
  - artifact: load-test-results
    required: false
  - artifact: incident-plan
    required: false
requires_stage:
  - observability-setup
  - deployment-execution
  - incident-response
  - performance-validation
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
  - workshop
inputs: All Operation phase artifacts, production monitoring data
outputs: slo-report.md, cost-analysis.md, drift-report.md, feedback-loop.md, feedback-optimization-questions.md (under this stage's record dir, engine-resolved)
---

# Continuous Feedback & Optimization

## Steps

### Step 1: Load Prior Context

- `<record>/operation/observability-setup/` からオブザーバビリティの設定を読み込む
- `<record>/operation/performance-validation/` からパフォーマンス検証の結果を読み込む
- SLO/SLI の設定を読み込む
- ドリフト比較のためにインフラストラクチャ設計を読み込む

### Step 2: Generate Questions

次の内容を網羅する質問ファイルを作成する:
- SLO は達成されているか。エラーバジェットの消費率（バーンレート）はどうか。
- コスト最適化の機会はあるか。
- 構成やインフラストラクチャのドリフトはあるか。
- 新機能や問題を示唆するユーザー行動パターンはあるか。
- 自動化できる運用上の負荷（トイル）は何か。

stage-protocol.md の質問フローに従う。

### Step 3: Generate Artifacts

SLO 準拠レポート、AWS Cost Explorer による分析と最適化の推奨事項、AWS Config によるドリフト検出レポート、Trusted Advisor の推奨事項のレビュー、運用上の知見と改善提案、およびフィードバックループ文書（次の Ideation サイクルへの入力）を作成する。

### Step 4: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage feedback-optimization --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 5: Present Completion & Request Approval

完了時の絵文字: :recycle:
レビュー対象のパス: `<record>/operation/feedback-optimization/`
承認ゲート: Approve（ワークフロー完了）/ Request Changes。

これは最終ステージである。承認されると、AI-DLC ワークフロー全体が完了する。ユーザーが反復を続けることを選んだ場合、フィードバックループ文書が知見を次の Ideation サイクルへ還流させる。

## Sensors

このステージの出力は `<record>/operation/feedback-optimization/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `dashboards`, `alarms`, `slo-config`, `deployment-log`, `load-test-results`, `incident-plan`.

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
