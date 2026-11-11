---
slug: feasibility
name: Feasibility & Constraints
phase: ideation
execution: CONDITIONAL
condition: Execute when there are integration constraints, regulatory requirements, or significant technical uncertainty. Skip for trivial changes with no technical risk.
lead_agent: aidlc-architect-agent
support_agents:
  - aidlc-aws-platform-agent
  - aidlc-compliance-agent
mode: inline
summary_confirmation: required
produces:
  - feasibility-assessment
  - constraint-register
  - raid-log
  - feasibility-questions
consumes:
  - artifact: intent-statement
    required: true
  - artifact: competitive-analysis
    required: false
  - artifact: market-trends
    required: false
  - artifact: build-vs-buy
    required: false
requires_stage:
  - intent-capture
  - market-research
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
  - mvp
inputs: Intent statement from intent-capture stage, market research from market-research stage (if executed)
outputs: feasibility-assessment.md, constraint-register.md, raid-log.md, feasibility-questions.md (under this stage's record dir, engine-resolved)
---

# Feasibility & Constraint Analysis

## Steps

### Step 1: Load Prior Context

- `<record>/ideation/intent-capture/` からインテントステートメントを読み込む
- `<record>/ideation/market-research/` から市場調査を読み込む（存在する場合）
- ガードレールを
  `aidlc/spaces/<active-space>/memory/{org,team,project}.md` から読み込む

### Step 2: Generate Clarifying Questions

次の質問を含む `<record>/ideation/feasibility/feasibility-questions.md` を作成する:
- どの既存システムと連携する必要があるか？
- 規制・コンプライアンス要件（PCI、HIPAA、SOC2、データレジデンシー）はあるか？
- チームの現在の技術スタックとスキル構成はどうなっているか？
- 予算とスケジュールの制約は何か？
- 組織上の阻害要因（変更凍結期間、競合する優先事項）はあるか？
- 現在どの AWS サービスとアカウントを使用しているか？

stage-protocol.md の質問フローに従う。

### Step 3: Collect and Analyze Answers

曖昧さの検出と矛盾の分析を行う。

### Step 4: Generate Artifacts

フィージビリティ評価（技術的な実現可能性、リスク分析）、制約レジスタ（技術・組織・規制）、RAID ログ（Risks、Assumptions、Issues、Dependencies）を作成する。

オーケストレーターは、これらの成果物を AWS 環境評価のために aidlc-aws-platform-agent へ、規制スキャンのために aidlc-compliance-agent へ渡し、すべての入力を統合する。

### Step 5: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage feasibility --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 6: Present Completion & Request Approval

完了時の絵文字: :test_tube:
レビュー対象のパス: `<record>/ideation/feasibility/`
標準の承認ゲート（Approve / Request Changes）。

## Sensors

このステージの出力は `<record>/ideation/feasibility/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `intent-statement`, `competitive-analysis`, `market-trends`, `build-vs-buy`.

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
