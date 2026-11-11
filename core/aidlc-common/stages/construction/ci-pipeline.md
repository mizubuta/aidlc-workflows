---
slug: ci-pipeline
name: CI Pipeline
phase: construction
execution: CONDITIONAL
condition: Execute when CI pipeline needs creation or significant modification. Skip if CI already exists and is adequate.
lead_agent: aidlc-pipeline-deploy-agent
support_agents: []
mode: inline
summary_confirmation: required
produces:
  - ci-config
  - quality-gates
  - ci-pipeline-questions
consumes:
  - artifact: code-summary
    required: true
  - artifact: build-and-test-summary
    required: true
  - artifact: build-test-results
    required: true
requires_stage:
  - build-and-test
sensors:
  - required-sections
  - upstream-coverage
  - linter
  - type-check
scopes:
  - enterprise
  - feature
  - mvp
  - infra
  - workshop
inputs: Code generation output from code-generation stage, build/test results from build-and-test stage
outputs: ci-config.md, quality-gates.md, ci-pipeline-questions.md (under this stage's record dir, engine-resolved)
---

# CI Pipeline

## Steps

### Step 1: Load Prior Context

- `<record>/construction/build-and-test/` からビルド/テスト結果を読み込む（存在する場合）
- `<record>/construction/{unit-name}/code-generation/` からコードサマリーを読み込む（存在する場合）
- `<record>/construction/infrastructure-design/` からインフラストラクチャ設計を読み込む（存在する場合）
- 既存の CI 設定を把握するためにワークスペースプロファイルを読み込む

このインテントで code-generation または build-and-test が実行されなかった場合（利用者がスキップしたステージ）、それらの入力は存在しない。その場合は、パイプラインのステージをワークスペースの既存のビルド/テスト構成（リポジトリ自体から検出したもの）に基づかせる。欠けている成果物の内容を決して捏造しない。

### Step 2: Generate Clarifying Questions

`<record>/construction/ci-pipeline/ci-pipeline-questions.md` を作成し、次の質問を記載する:
- どの CI ツールを使用しているか（CodePipeline、CodeBuild、GitHub Actions、Jenkins）?
- ブランチ戦略は何か?
- マージ前に必要な品質ゲートは何か?
- どの成果物リポジトリを使用しているか（ECR、CodeArtifact、S3）?

stage-protocol.md の質問フローに従う。

### Step 3: Collect and Analyze Answers

CI の選択を、既存のインフラストラクチャおよびチームの能力と照らして検証する。

### Step 4: Generate Artifacts

CI パイプラインの設定（buildspec.yml、ワークフロー YAML、または同等のもの）、品質ゲートの定義、成果物リポジトリの設定を作成する。

### Step 5: Phase Boundary Verification

Construction → Operation の検証チェックを実行する:
- 次を読み込む:
  `<record>/construction/build-and-test/cross-unit-traceability.md`。
- 次のすべてを読み込む:
  `<record>/construction/*/code-generation/traceability.json`。
- すべてのユニットがビルドおよびテストされていること、すべての code-generation の表に
  未解決の指摘がないこと、ユニット横断の FR/NFR/AC ゲートが合格したことを確認する。
- CI の品質ゲートが、Build and Test が記録したビルドおよびテストのコマンドを
  強制していることを確認する。
- 境界の判定結果を次に書き込む:
  `<record>/verification/phase-check-construction.md`。

トレーサビリティファイルが欠けている、または未解決の指摘が残っている場合は、
Construction → Operation への遷移を中止し、担当ステージに戻る。

### Step 6: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage ci-pipeline --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 7: Present Completion & Request Approval

完了絵文字: :gear:
レビューパス: `<record>/construction/ci-pipeline/`
標準の 2 択承認（Approve / Request Changes）。

## Sensors

このステージの出力は、`<record>/construction/ci-pipeline/` 配下の Markdown の設計成果物である。一部のセクションにはコードサンプルが含まれ、コード形状のセンサーがそれを指摘することもある。

インポート: `required-sections`、`upstream-coverage`、`linter`、`type-check`。

上流のターゲット: `code-summary`、`build-and-test-summary`、`build-test-results`。

`linter` と `type-check` は、設計出力内の該当する TypeScript/JavaScript の
スニペットを検査する。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実行する。そのモジュールが、いつ利用者に何かを尋ねるかを
規定している。モジュールがない場合は、日誌とステップの両方をスキップする。
