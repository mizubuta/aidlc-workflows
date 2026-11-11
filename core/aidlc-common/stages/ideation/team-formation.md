---
slug: team-formation
phase: ideation
execution: CONDITIONAL
condition: Execute when team composition, capacity, or mob planning is relevant. Skip for solo developer or small team projects.
lead_agent: aidlc-delivery-agent
support_agents: []
mode: inline
summary_confirmation: required
produces:
  - team-assessment
  - skill-matrix
  - mob-composition
  - team-formation-questions
consumes:
  - artifact: scope-document
    required: true
  - artifact: intent-backlog
    required: true
  - artifact: feasibility-assessment
    required: false
requires_stage:
  - scope-definition
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
inputs: Scope definition, intent backlog, feasibility assessment
outputs: team-assessment.md, skill-matrix.md, mob-composition.md, team-formation-questions.md (under this stage's record dir, engine-resolved)
---

# Team Formation & Mob Planning

## Steps

### Step 1: Load Prior Context

- `<record>/ideation/scope-definition/` からスコープ定義を読み込む
- フィージビリティ評価と制約レジスター（存在する場合）を読み込む
- 作業量の見積もりのため、インテントバックログを読み込む

### Step 2: Generate Clarifying Questions

次の質問を含む `<record>/ideation/team-formation/team-formation-questions.md` を作成する:
- どのチームと個人が参加可能か？
- 現在のキャパシティと稼働率はどの程度か？
- 必要なスキルと保有しているスキルはそれぞれ何か？
- 同じ人材プールを取り合う競合イニシアチブはあるか？
- 望ましいチームトポロジーは何か？
- チームメンバーはどのタイムゾーン・拠点にいるか？
- 外部パートナー、契約者、AWS Professional Services は必要か？
- 各フェーズの意思決定者は誰か？

stage-protocol.md の質問フローに従う。

### Step 3: Collect and Analyze Answers

必要なスキルと保有スキルのギャップ分析を行う。

### Step 4: Generate Artifacts

チームの稼働可能性評価、スキルマトリクス（ギャップ分析付き）、モブ編成計画、RACI マトリクス、キャパシティ配分の合意書、スキルギャップ解消計画、オンボーディングチェックリストを作成する。

### Step 5: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage team-formation --result <outcome>` を介して、
完了処理を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 6: Present Completion & Request Approval

完了時の絵文字: :people_holding_hands:
レビューパス: `<record>/ideation/team-formation/`
標準の承認ゲート（Approve / Request Changes）。

## Sensors

このステージの出力は `<record>/ideation/team-formation/` 配下の Markdown 成果物である。

取り込み: `required-sections`, `upstream-coverage`.

上流ターゲット: `scope-document`, `intent-backlog`, `feasibility-assessment`.

## Learn

`directive.protocol_modules` に `learnings` が含まれる場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を記録し、
承認ゲートの前に学びのステップを実行する。そのモジュールに、いつ利用者に質問するかが
定められている。モジュールがない場合は、日誌とステップの両方をスキップする。
