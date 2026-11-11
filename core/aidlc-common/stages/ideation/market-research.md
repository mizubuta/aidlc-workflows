---
slug: market-research
phase: ideation
execution: CONDITIONAL
condition: Execute when initiative has external market positioning or build-vs-buy considerations. Skip for internal tools, bug fixes, or refactors.
lead_agent: aidlc-product-agent
support_agents: []
mode: inline
summary_confirmation: required
produces:
  - competitive-analysis
  - market-trends
  - build-vs-buy
  - market-research-questions
consumes:
  - artifact: intent-statement
    required: true
requires_stage:
  - intent-capture
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
inputs: Intent statement from intent-capture stage
outputs: competitive-analysis.md, market-trends.md, build-vs-buy.md, market-research-questions.md (under this stage's record dir, engine-resolved)
---

# Market Research & Competitive Analysis

## Steps

### Step 1: Load Prior Context

- `<record>/ideation/intent-capture/` からインテントステートメントを読み込む
- 取り組みのうち、市場に関係する側面を特定する

### Step 2: Generate Clarifying Questions

次の質問を含む `<record>/ideation/market-research/market-research-questions.md` を作成する:
- 市場にはどのような競合製品やソリューションが存在するか？
- それぞれの強み・弱み・価格モデルは何か？
- 関連する業界トレンドや規制の変化は何か？
- 顧客が「あって当然（table-stakes）」と考えるものと、差別化要因と考えるものは何か？
- 社内向けの取り組みの場合: 既存のツール、SaaS 製品、オープンソースの代替手段はあるか？
- 自社開発・購入・提携（build-vs-buy-vs-partner）の判断はどうなるか？
- 対象とする市場規模や想定利用者層はどれくらいか？

stage-protocol.md の質問フロー（Guide Me / Edit File / Chat）に従う。

### Step 3: Collect and Analyze Answers

すべての回答に対して、曖昧さの検出と矛盾の分析を行う。

### Step 4: Generate Artifacts

回答と調査に基づき、競合分析、市場トレンドレポート、build-vs-buy 評価、差別化戦略の概要を作成する。

### Step 5: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage market-research --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 6: Present Completion & Request Approval

完了時の絵文字: :bar_chart:
レビュー対象のパス: `<record>/ideation/market-research/`
標準の承認ゲート（Approve / Request Changes）。

## Sensors

このステージの出力は `<record>/ideation/market-research/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `intent-statement`.

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
