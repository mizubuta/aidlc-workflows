---
slug: refined-mockups
phase: inception
execution: CONDITIONAL
condition: Execute when user-facing UI exists and rough mockups were produced in Ideation; for APIs, refine interaction diagrams
lead_agent: aidlc-design-agent
support_agents:
  - aidlc-product-agent
mode: inline
summary_confirmation: required
reviewer: aidlc-product-lead-agent
review_artifact: mockups
reviewer_max_iterations: 2
review_class: advisory
produces:
  - mockups
  - interaction-spec
  - design-system-mapping
  - accessibility-checklist
  - refined-mockups-questions
consumes:
  - artifact: wireframes
    required: true
  - artifact: user-flow
    required: true
  - artifact: stories
    required: false
  - artifact: requirements
    required: true
  - artifact: team-practices
    required: false
requires_stage:
  - user-stories
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
  - mvp
  - classic
  - workshop
inputs: Rough mockups from rough-mockups stage, user stories from user-stories stage, requirements from requirements-analysis stage
outputs: mockups.md, interaction-spec.md, design-system-mapping.md, accessibility-checklist.md, refined-mockups-questions.md (under this stage's record dir, engine-resolved)
---

# Refined Mockups & UX Design

## Steps

### Step 1: Load Prior Context

- `<record>/ideation/rough-mockups/` からラフモックアップを読み込む（存在する場合）
- `<record>/inception/user-stories/` からユーザーストーリーを読み込む
- `<record>/inception/requirements-analysis/` から要件を読み込む

classic スコープは設計上 rough-mockups をスキップする（Ideation フェーズがない）。ワイヤーフレームとユーザーフローの入力が存在しない場合は、ユーザーストーリーと要件から直接リファインドモックアップを設計する。存在しない成果物の内容を決してでっち上げてはならない。

### Step 2: Generate Clarifying Questions

次の質問を含む `<record>/inception/refined-mockups/refined-mockups-questions.md` を作成する:
- 各ユーザーストーリーを UI でどのように表現すべきか？
- どのようなインタラクションパターンが必要か（モーダル、インライン編集、ウィザード、段階的な開示）？
- 各画面が扱うべき状態は何か（読み込み中、空、エラー、成功、部分的）？
- デザインは既存のデザインシステム／コンポーネントライブラリと整合しているか？
- どのアクセシビリティ要件が適用されるか（WCAG のレベル）？
- どのレスポンシブのブレークポイントが必要か？
- API の場合: 開発者体験はどのようなものになるか？

stage-protocol.md の質問フローに従う。

### Step 3: Collect and Analyze Answers

設計上の決定をユーザーストーリーと要件に照らして検証し、一貫性を確認する。

### Step 4: Generate Artifacts

中〜高忠実度のモックアップ（ユーザーストーリー／画面ごと）、インタラクション仕様書（コンポーネントレベルの仕様の形式として `{{HARNESS_DIR}}/knowledge/aidlc-design-agent/component-spec-template.md` を使用する）、デザインシステムへのマッピング、レスポンシブ時の振る舞いの仕様、アクセシビリティ準拠のチェックリストを作成する。

UI 以外の場合: API の開発者体験の仕様を作成する。

### Step 5: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage refined-mockups --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 6: Present Completion & Request Approval

完了時の絵文字: :art:
レビュー対象のパス: `<record>/inception/refined-mockups/`
標準の承認ゲート（Approve / Request Changes）。

## Sensors

このステージの出力は `<record>/inception/refined-mockups/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `wireframes`, `user-flow`, `stories`, `requirements`, `team-practices`.

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
