---
slug: approval-handoff
name: Approval & Handoff
phase: ideation
execution: ALWAYS
condition: Always executes — compiles all Ideation artifacts into initiative brief for approval
lead_agent: aidlc-delivery-agent
support_agents:
  - aidlc-product-agent
mode: inline
summary_confirmation: required
produces:
  - initiative-brief
  - decision-log
  - approval-handoff-questions
consumes:
  - artifact: intent-statement
    required: true
  - artifact: stakeholder-map
    required: true
  - artifact: scope-document
    required: true
  - artifact: intent-backlog
    required: true
  - artifact: competitive-analysis
    required: false
  - artifact: feasibility-assessment
    required: false
  - artifact: constraint-register
    required: false
  - artifact: team-assessment
    required: false
  - artifact: wireframes
    required: false
requires_stage:
  - intent-capture
  - feasibility
  - scope-definition
  - team-formation
  - rough-mockups
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
inputs: All Ideation phase artifacts (intent, market research, feasibility, scope, team, mockups)
outputs: initiative-brief.md, decision-log.md, approval-handoff-questions.md (under this stage's record dir, engine-resolved)
---

# Initiative Approval & Handoff

## Steps

### Step 1: Load Prior Context

Ideation フェーズのすべての成果物を読み込む:
- `<record>/ideation/intent-capture/` からインテントステートメントとステークホルダーマップ
- `<record>/ideation/market-research/` から市場調査（存在する場合）
- `<record>/ideation/feasibility/` から実現可能性評価、制約レジスター、RAID ログ（存在する場合）
- `<record>/ideation/scope-definition/` からスコープ定義とインテントバックログ
- `<record>/ideation/team-formation/` からチーム編成の成果物（存在する場合）
- `<record>/ideation/rough-mockups/` からモックアップ/ワイヤーフレーム（存在する場合）

### Step 2: Generate Approval Questions

次の質問を含む `<record>/ideation/approval-handoff/approval-handoff-questions.md` を作成する:
- すべてのステークホルダーがインテントとスコープに合意しているか？
- 重大なリスクはすべて、緩和策とともに認識されているか？
- 予算/リソースの確約はあるか？
- ラフモックアップは共有されたビジョンを反映しているか？
- 市場調査は投資を裏付けているか？
- モブの要員配置とスケジュールは決まっているか？

stage-protocol.md の質問フローに従う。

### Step 3: Compile Initiative Brief

次の内容をまとめた1ページの `<record>/ideation/approval-handoff/initiative-brief.md` を作成する:
- インテントと課題ステートメント
- 市場検証の要約
- 実現可能性とリスクの要点
- スコープの境界
- コンセプトのビジュアル
- チーム計画
- 実施可否（Go/no-go）の推奨

`<record>/ideation/approval-handoff/decision-log.md` を作成する — Ideation 中に下されたすべての判断の記録。

### Step 4: Phase Boundary Verification

Ideation → Inception の検証チェックを実行する:
- インテント → スコープ → インテントバックログの一貫性
- すべてのスコープ項目に実現可能性の裏付けがあること
- 結果を `<record>/verification/phase-check-ideation.md` に書き込む

### Step 5: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage approval-handoff --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 6: Present Completion & Request Approval

完了時の絵文字: :white_check_mark:
レビュー対象のパス: `<record>/ideation/approval-handoff/`
承認ゲート: Approve（Continue to `<directive.next_stage>`。エンジンが指定するステージへ進む）/ Request Changes。

## Sensors

このステージの出力は `<record>/ideation/approval-handoff/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `intent-statement`, `stakeholder-map`, `scope-document`, `intent-backlog`, `competitive-analysis`, `feasibility-assessment`, `constraint-register`, `team-assessment`, `wireframes`.

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学習ステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
