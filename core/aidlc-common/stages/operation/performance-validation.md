---
slug: performance-validation
phase: operation
execution: CONDITIONAL
condition: Execute when NFR performance targets need validation under load
lead_agent: aidlc-quality-agent
support_agents: []
mode: inline
summary_confirmation: required
produces:
  - load-test-plan
  - load-test-results
  - nfr-validation-matrix
  - performance-validation-questions
consumes:
  - artifact: performance-requirements
    required: true
  - artifact: scalability-requirements
    required: true
  - artifact: performance-design
    required: true
  - artifact: scalability-design
    required: true
  - artifact: dashboards
    required: true
requires_stage:
  - nfr-requirements
  - nfr-design
  - observability-setup
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
  - workshop
inputs: NFR requirements from nfr-requirements stage, NFR design from nfr-design stage, deployed application, observability data from observability-setup stage
outputs: load-test-plan.md, test-results.md, nfr-validation-matrix.md, performance-validation-questions.md (under this stage's record dir, engine-resolved)
---


# Performance Validation & Load Testing

## Steps

### Step 1: Load Prior Context

- `<record>/construction/nfr-requirements/` から NFR 要件を読み込む
- `<record>/construction/nfr-design/` から NFR 設計を読み込む
- `<record>/operation/observability-setup/` からオブザーバビリティ設定を読み込む

### Step 2: Generate Clarifying Questions

次の内容を扱う質問ファイルを作成する:
- 想定されるトラフィックパターン（定常状態、ピーク、バースト）は何か？
- 目標とするレイテンシのパーセンタイル（p50、p95、p99）は何か？
- システムが維持しなければならないスループットはどれくらいか？
- ボトルネックになりそうな箇所はどこか？

stage-protocol.md の質問フローに従う。

### Step 3: Design and Execute Tests

負荷テスト計画を設計し、本番相当の環境に対してパフォーマンステストを実行し、CloudWatch/X-Ray のエビデンスを用いて結果を分析する。

### Step 4: Generate Artifacts

負荷テスト計画、パフォーマンステスト結果（レイテンシ、スループット、エラー率）、ボトルネック分析、オートスケーリングの検証レポート、キャパシティプランニングの推奨事項、NFR 検証マトリクス（目標値と実測値の比較）を作成する。

### Step 5: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage performance-validation --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 6: Present Completion & Request Approval

完了時の絵文字: :zap:
レビューパス: `<record>/operation/performance-validation/`
標準の 2 択承認（Approve / Request Changes）。

## Sensors

このステージの出力は `<record>/operation/performance-validation/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `performance-requirements`, `scalability-requirements`, `performance-design`, `scalability-design`, `dashboards`.

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学習ステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
