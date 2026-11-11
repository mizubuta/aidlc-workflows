---
slug: rough-mockups
phase: ideation
execution: CONDITIONAL
condition: Execute when user-facing UI is part of the initiative; for API/backend, produce system interaction diagrams. Skip for non-UI, API-only, or infrastructure-only initiatives.
lead_agent: aidlc-design-agent
support_agents:
  - aidlc-product-agent
mode: inline
summary_confirmation: required
reviewer: aidlc-product-lead-agent
review_artifact: wireframes
reviewer_max_iterations: 2
review_class: advisory
produces:
  - wireframes
  - user-flow
  - rough-mockups-questions
consumes:
  - artifact: intent-statement
    required: true
  - artifact: scope-document
    required: true
  - artifact: intent-backlog
    required: true
requires_stage:
  - scope-definition
  - team-formation
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
  - mvp
inputs: Intent statement, scope definition, intent backlog
outputs: wireframes.md, user-flow.md, rough-mockups-questions.md (under this stage's record dir, engine-resolved)
---

# Rough Mockups & Concept Visualization

## Steps

### Step 1: Load Prior Context

- `<record>/ideation/intent-capture/` からインテントステートメントを読み込む
- `<record>/ideation/scope-definition/` からスコープ定義とインテントバックログを読み込む

### Step 2: Generate Clarifying Questions

次の質問を含む `<record>/ideation/rough-mockups/rough-mockups-questions.md` を作成する:
- 利用者の主な入口と、主要な画面/ビューは何か？
- 中核となるユーザーフロー（ハッピーパス）は何か？
- 情報の階層はどのようになっているか？
- 従うべき既存のブランドガイドライン、デザインシステム、UI パターンはあるか？
- サポートしなければならないデバイス/フォームファクターは何か？
- 既知のアクセシビリティ要件（WCAG レベル、スクリーンリーダー対応、キーボードのみでの操作）はあるか？
- UI を持たない取り組みの場合: 主要なシステム間のやり取りとデータフローは何か？

stage-protocol.md の質問フローに従う。

### Step 3: Collect and Analyze Answers

UX への期待とスコープの制約との間で矛盾の分析を行う。

### Step 4: Generate Artifacts

UI を持つ取り組みの場合: ローフィデリティのワイヤーフレーム（ASCII アートまたは構造化された説明）、中核となるユーザーフロー図、情報アーキテクチャの概要を作成する。各画面に1行のアクセシビリティに関する注記を含める: 見出しレベル（h1–h3）、主要なランドマーク領域（header/main/nav/footer）、キーボード操作の起点。

UI を持たない取り組みの場合: システムコンテキスト図と、主要なやり取りのフロー図のスケッチを作成する。

すべての図は stage-protocol.md の ASCII 図の標準に従う。

### Step 5: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage rough-mockups --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 6: Present Completion & Request Approval

完了時の絵文字: :pencil2:
レビュー対象のパス: `<record>/ideation/rough-mockups/`
標準の承認ゲート（Approve / Request Changes）。

## Sensors

このステージの出力は `<record>/ideation/rough-mockups/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `intent-statement`, `scope-document`, `intent-backlog`.

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学習ステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
