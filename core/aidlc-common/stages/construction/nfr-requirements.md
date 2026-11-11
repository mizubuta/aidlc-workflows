---
slug: nfr-requirements
name: NFR Requirements
phase: construction
execution: CONDITIONAL
condition: Performance, security, scalability, reliability, or observability requirements needed, or tech stack selection needed. Skip if no NFR requirements and tech stack already determined.
lead_agent: aidlc-architect-agent
support_agents:
  - aidlc-devsecops-agent
  - aidlc-compliance-agent
  - aidlc-quality-agent
mode: inline
summary_confirmation: required
reviewer: aidlc-architecture-reviewer-agent
review_artifact: security-requirements
reviewer_max_iterations: 2
for_each: unit-of-work
produces:
  - performance-requirements
  - security-requirements
  - scalability-requirements
  - reliability-requirements
  - observability-requirements
  - tech-stack-decisions
  - traceability
produces_kinds:
  performance-requirements: [service, ui]
  scalability-requirements: [service]
  reliability-requirements: [service]
  observability-requirements: [service]
consumes:
  - artifact: functional-spec
    required: true
  - artifact: rules
    required: true
  - artifact: requirements
    required: true
  - artifact: contract-summary
    required: false
  - artifact: technology-stack
    required: false
    conditional_on: brownfield
requires_stage:
  - units-generation
  - functional-design
sensors:
  - required-sections
  - upstream-coverage
  - linter
  - type-check
  - traceability
scopes:
  - enterprise
  - feature
  - mvp
  - infra
  - security-patch
  - classic
  - workshop
inputs: functional design artifacts, requirements.md, RE artifacts
outputs: "performance-requirements.md, security-requirements.md, scalability-requirements.md, reliability-requirements.md, observability-requirements.md, tech-stack-decisions.md, traceability.json (under this stage's per-unit record dir, engine-resolved); per-kind applicability via produces_kinds (untagged unit: all)"
---

# NFR Requirements

## Steps

### Execution Modes

このステージは、オーケストレーターが制御する 2 つの実行モードをサポートする:

**QUESTION-ONLY モード**（Bolt の質問フェーズ中にオーケストレーターが呼び出す）:
Step 1〜4 のみを実行する（成果物の読み込み、カテゴリの評価、質問の生成、回答の収集）。
成果物の生成には決して進まない。制御をオーケストレーターに返す。

**ARTIFACT-ONLY モード**（Bolt の設計フェーズ中にオーケストレーターが呼び出す）:
Step 1〜4 をスキップする（質問は収集・承認済み）。
ユニット単位のディレクトリから回答済みの質問ファイルを読み込む。
Step 5〜7 のみを実行する（成果物の生成、状態の更新、完了）。

**Full モード**（既定 — 単一ユニットのプロジェクト、またはステージの直接呼び出し）:
書かれたとおりにすべてのステップを順に実行する。

### Step 1: Read Prior Artifacts

`<record>/construction/{unit-name}/functional-design/` から機能設計の成果物を読み込む（存在する場合）。`<record>/inception/requirements-analysis/requirements.md`（存在する場合）、`<record>/inception/contract-design/contract-summary.md` のユニット間契約（生成されている場合。その SLA、リトライ/タイムアウト、統合メカニズムの決定が、このユニットの NFR 目標を制約する）、そして `aidlc/spaces/<active-space>/codekb/<repo>/`（`codekb-path --repo <repo>` が出力するディレクトリ）にあるリバースエンジニアリングの成果物を読み込む。インクリメンタルなスコープ（infra）は設計上 functional-design をスキップする。その成果物がない場合は、代わりに要件とコードナレッジベースから NFR のコンテキストを導出する。欠けている成果物の内容を決して捏造しない。

### Step 2: Assess NFR Categories

ユニットを NFR カテゴリ全体にわたって分析する:
- **Performance**: 応答時間、スループット、レイテンシ目標、リソース使用率
- **Security**: 認証、認可、データ保護、コンプライアンス要件
- **Scalability**: 負荷への対応、成長予測、スケーリング戦略
- **Reliability**: 可用性目標、フォールトトレランス、災害復旧、データの耐久性
- **Observability**: モニタリング、ロギング、アラート、トレーシングの要件

### Step 3: Generate Questions

不明確な NFR 領域について、[Answer]: タグを使った質問ファイルを `<record>/construction/{unit-name}/nfr-requirements/nfr-requirements-questions.md` に作成する。定量化可能な目標と具体的な制約に焦点を当てる。

### Step 4: Collect and Analyze Answers

stage-protocol.md §3 の質問フローに従って回答を収集する（対話モードの選択肢を提示し、回答を収集し、ファイルに書き戻す）。必須のあいまいさ分析を行う:
- あいまいな回答（「十分に速い」「高可用」「セキュア」など）を特定する
- NFR 目標間の矛盾を確認する
- 欠けている定量的な目標を指摘する

あいまいさが 1 つでも見つかった場合: フォローアップ質問を作成し、先に進む前に解決する。

### Step 5: Generate Artifacts

`<record>/construction/{unit-name}/nfr-requirements/` に次のものを生成する:

- **performance-requirements.md**: 応答時間の目標、スループット要件、レイテンシの予算、リソースの制約、ベンチマーク
- **security-requirements.md**: 認証要件、認可モデル、データ保護、コンプライアンス、脅威に関する考慮事項
- **scalability-requirements.md**: 負荷予測、スケーリングのトリガー、キャパシティプランニング、データの増加、同時実行性の目標
- **reliability-requirements.md**: 可用性目標（SLA/SLO）、フォールトトレランス要件、バックアップ/リカバリー、グレースフルデグラデーション
- **observability-requirements.md**: モニタリング要件、ロギング標準、分散トレーシングの必要性、アラートのしきい値、ダッシュボード要件、SLI/SLO の定義
- **tech-stack-decisions.md**: 技術の選定と根拠 — 言語、フレームワーク、データベース、インフラストラクチャツール、および各選択の正当性

詳細な要件はそれぞれ、Inception の NFR ID を継承し、サブ番号を付加する
（`NFR4.1`、`NFR4.2` など）。これらの ID をすべての要件の
行に記載する。

次を実行して
`<record>/construction/{unit-name}/nfr-requirements/traceability.json` を作成する:
`{{INVOKE}} engine sensor-traceability --stage nfr-requirements --output-path <that path> --write-upstream`
これは、Requirements Analysis が記録したすべての Inception の `NFR{n}` を列挙する。User Stories が
実行されず、`unit-of-work-story-map.md` がこのユニットに NFR の行を与えている場合は、それらの行の NFR を
列挙する。それぞれに `GAP` の行が付く。導出した `NFRx.y` の ID をターゲットにする。このユニットに適用されないものは
`coverage` で `N/A` とし、その理由をターゲットとして記載する:

```json
{
  "stage": "nfr-requirements",
  "unit": "u1-auth",
  "upstream_ids": ["NFR1", "NFR4"],
  "coverage": [
    { "id": "NFR1", "status": "OK", "target": "NFR1.1, NFR1.2" },
    { "id": "NFR4", "status": "N/A", "target": "no persistent data in this Unit" }
  ]
}
```

### Step 6: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage nfr-requirements --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 7: Completion

完了メッセージと承認ゲートを提示する:

```
# :bar_chart: NFR Requirements Complete — {unit-name}
```

対応した NFR カテゴリと主要な目標のサマリーを示し、続けて:

```
**Review:** `<record>/construction/{unit-name}/nfr-requirements/`
```

承認ゲート: 厳密に 2 択（Approve / Request Changes）。

## Sensors

このステージの出力は、`<record>/construction/{unit-name}/nfr-requirements/` 配下の Markdown の設計成果物である。一部のセクションにはコードサンプルが含まれ、コード形状のセンサーがそれを指摘することもある。

インポート: `required-sections`、`upstream-coverage`、`linter`、`type-check`、`traceability`。

上流のターゲット: `functional-spec`、`rules`、`requirements`、`contract-summary`、`technology-stack`。

`linter` と `type-check` は、該当する TypeScript/JavaScript のスニペットを検査する。
`traceability` は、Inception の NFR ID が宣言され、ユニットごとの
`NFRx.y` 要件によってカバーされていることを検証する。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実行する。そのモジュールが、いつ利用者に何かを尋ねるかを
規定している。モジュールがない場合は、日誌とステップの両方をスキップする。
