---
slug: scope-definition
phase: ideation
execution: ALWAYS
condition: Always executes — defines the scope boundary and prioritized backlog
lead_agent: aidlc-product-agent
support_agents:
  - aidlc-delivery-agent
mode: inline
summary_confirmation: required
produces:
  - scope-document
  - intent-backlog
  - scope-definition-questions
consumes:
  - artifact: intent-statement
    required: true
  - artifact: feasibility-assessment
    required: false
  - artifact: constraint-register
    required: false
requires_stage:
  - intent-capture
  - feasibility
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
  - mvp
inputs: Intent statement, feasibility assessment, constraint register
outputs: scope-document.md, intent-backlog.md, scope-definition-questions.md (under this stage's record dir, engine-resolved)
---

# Scope Definition & Prioritization

## Steps

### Step 1: Load Prior Context

- `<record>/ideation/intent-capture/` からインテントステートメントを読み込む
- `<record>/ideation/feasibility/` から実現可能性評価を読み込む（存在する場合）
- 制約登録簿と RAID ログを読み込む（存在する場合）

### Step 2: Generate Clarifying Questions

次の質問を含む `<record>/ideation/scope-definition/scope-definition-questions.md` を作成する:
- 価値を提供できる最小限のスコープは何か？
- どの機能が必須で、どの機能があれば望ましい程度か？
- 機能間の依存関係は何か？
- 順序付けの方針はどれを優先するか（リスク優先、価値優先、依存関係優先）？
- 特定の機能に結び付いた厳格な期限はあるか？

stage-protocol.md の質問フローに従う。

### Step 3: Collect and Analyze Answers

曖昧さの検出、矛盾の分析、スコープとタイムラインの整合性の検証を行う。

### Step 4: Generate Artifacts

スコープ定義ドキュメント（対象範囲内と対象範囲外の境界）、優先順位付けされたインテントバックログ（MoSCoW/WSJF/RICE を用いたユニットの原型）、およびバリューストリームマップを作成する。

### Step 5: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage scope-definition --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 6: Present Completion & Request Approval

完了時の絵文字: :dart:
レビュー対象のパス: `<record>/ideation/scope-definition/`
標準の承認ゲート（Approve / Request Changes）。

## Sensors

このステージの出力は `<record>/ideation/scope-definition/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `intent-statement`, `feasibility-assessment`, `constraint-register`.

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
