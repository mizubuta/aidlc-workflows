---
slug: contract-design
phase: inception
execution: CONDITIONAL
condition: Execute when the system has any formal contract to pin down — an inter-unit boundary (more than one unit that must integrate) OR a unit that exposes a public/external API consumed outside the system. Skip only for a single self-contained unit with no inter-unit boundaries and no externally consumed API.
lead_agent: aidlc-architect-agent
support_agents:
  - aidlc-aws-platform-agent
mode: inline
summary_confirmation: required
reviewer: aidlc-architecture-reviewer-agent
review_artifact: contract-summary
reviewer_max_iterations: 2
review_class: advisory
produces:
  - contract-summary
consumes:
  - artifact: unit-of-work
    required: true
  - artifact: unit-of-work-dependency
    required: true
  - artifact: components
    required: false
  - artifact: requirements
    required: false
requires_stage:
  - units-generation
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
  - mvp
  - classic
  - workshop
inputs: <record>/inception/units-generation/unit-of-work.md, <record>/inception/units-generation/unit-of-work-dependency.md, <record>/inception/domain-design/components.md (if produced), <record>/inception/requirements-analysis/requirements.md
outputs: contract-summary.md (under this stage's record dir, engine-resolved) — a human-readable overview of every contract (inter-unit boundaries and public/external APIs), each with a fenced spec block (OpenAPI / AsyncAPI / shared schema) inline
---

# Contract Design

システムが守らなければならない正式なコントラクトを定義し、チームが確信をもって並行して構築できるようにする。コントラクトとは、境界をまたぐ正式な合意である。どのデータがその境界を越えるか、どのような形で、どのプロトコルを介してか、そして問題が起きたときに何が起こるかを定める。対象となる境界は 2 種類ある:

- **ユニット間の境界** — システム内部の提供側ユニットと利用側ユニットの間の合意。それぞれを、2 つの会社の 2 つのチーム間の B2B 契約のように扱う。誤ったコントラクトは統合を手戻りの大惨事に変えるため、最初から正しくなければならない。
- **公開/外部 API の境界** — ユニットとシステムの*外部*の利用者（別のチーム、パートナー、パブリックインターネット）との間の合意。ユニット間のエッジを持たない単一ユニットのシステムであっても、そのような API を公開する場合は、Code Generation の前にこのコントラクトを確定させる必要がある。外部 API の仕様を担う他のステージは存在しない。

このステージはワークフローごとに 1 回（ユニットごとではなく）実行され、境界の全体を一度にマッピングする。どのユニットどうしが通信するかを知るために Units Generation の依存 DAG を使い、公開 API のコントラクトについては各ユニットの外部から利用される面を使う。

## Steps

### Step 1: Load Prior Context

- `<record>/inception/units-generation/unit-of-work.md` を読み込む（ユニットの定義と種別）
- `<record>/inception/units-generation/unit-of-work-dependency.md` を読み込む（依存 DAG — すべてのエッジがコントラクトの候補）
- `<record>/inception/domain-design/components.md` を読み込む（生成されている場合） — エンティティの形がペイロード設計の参考になる
- `<record>/inception/requirements-analysis/requirements.md` を読み込む（生成されている場合） — NFR が SLA とエラーバジェットを形作る

### Step 2: Create Contract Plan with Questions

[Answer]: タグ形式を使い、状況に応じた質問を含む `<record>/inception/contract-design/contract-design-questions.md` を作成する:
- 公開/外部 API の面（どのユニットがシステム外部から利用される API を公開するか、その形） — 単一ユニットでこのステージが実行される契機となる
- 境界ごとの統合メカニズム（同期 REST/HTTP、非同期イベント/メッセージ、共有スキーマ、gRPC など）
- コントラクトの所有権（どのユニットが各仕様を所有するか）
- バージョニングと破壊的変更のポリシー
- 各境界でのエラー、タイムアウト、リトライの挙動

### Step 3: Collect and Analyze Answers

stage-protocol.md §3 の質問フローに従って回答を収集する（インタラクションモードの選択肢を提示し、回答を収集し、ファイルに書き戻す）。
- **必須**の曖昧さ分析: 曖昧な表現、矛盾、欠落している詳細を洗い出す
- 曖昧さが 1 つでも見つかれば、追加の質問を作成する
- 先に進む前にすべての曖昧さを解消する

### Step 4: Generate the Contract Summary

`<record>/inception/contract-design/contract-summary.md` を作成する。この単一の成果物に、人間が読める概要とコントラクト仕様そのものの両方を含める。

**コントラクト表** — 境界（ユニット間および公開/外部）ごとに 1 行:

`| # | Provider Unit | Consumer | Mechanism | Owner |`

外部の境界では、Consumer 列に外部の利用者を記載する（例: `External: partner API`、`External: public web`）。

**コントラクトごとの仕様** — 各境界について、適切な形式で実際の仕様を記述したフェンス付きコードブロック:

- 同期 REST/HTTP コントラクト用: ```yaml OpenAPI block for synchronous REST/HTTP contracts
- a fenced ```yaml AsyncAPI ブロック（イベント駆動/メッセージベースのコントラクト用）
- 共有データベースまたは共有モデルのコントラクトには、フェンス付きの ```yaml shared-schema ブロック
- 統合メカニズムに適したその他のコントラクト形式

**コントラクト所有権のルール** — 各仕様を誰が所有するか、破壊的変更をどのように合意するか、追加的な変更をどのように安全に保つか（利用側は未知のフィールドを無視する）を述べる短いリスト。

**未解決の質問** — 未解決のコントラクト上の論点と、それぞれがどのユニットをブロックするかの表:
`| Contract | Question | Blocks |`

### Step 5: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage contract-design --result <outcome>`
を介して、完了処理を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 6: Present Completion & Request Approval

stage-protocol.md の完了テンプレートを、完了絵文字 :handshake: とともに使用する:
- 定義したコントラクトの要約（件数、メカニズム、所有権）
- レビューパス: `<record>/inception/contract-design/`
- 選択肢付きの構造化された承認質問: Approve（次のステージへ進む）/ Request Changes

## Sensors

このステージの出力は、`<record>/inception/contract-design/` 配下の Markdown 成果物である。

インポート: `required-sections`、`upstream-coverage`。

上流のターゲット: `unit-of-work`、`unit-of-work-dependency`、`components`、`requirements`。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。利用者に何かを尋ねるタイミングは
そのモジュールが定める。モジュールがない場合は、日誌とステップの両方を省略する。
