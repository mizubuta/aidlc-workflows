---
slug: deployment-pipeline
phase: operation
execution: CONDITIONAL
condition: Execute when CD pipeline needs creation or significant modification
lead_agent: aidlc-pipeline-deploy-agent
support_agents: []
mode: inline
summary_confirmation: required
produces:
  - cd-config
  - deployment-strategy
  - rollback-runbook
  - deployment-pipeline-questions
consumes:
  - artifact: ci-config
    required: true
  - artifact: quality-gates
    required: true
  - artifact: infrastructure-specification
    required: true
  - artifact: cicd-pipeline
    required: true
requires_stage:
  - ci-pipeline
  - infrastructure-design
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
  - infra
  - bugfix
  - refactor
  - security-patch
  - workshop
  - express
inputs: CI pipeline config from ci-pipeline stage, infrastructure design from infrastructure-design stage
outputs: cd-config.md, deployment-strategy.md, rollback-runbook.md, deployment-pipeline-questions.md (under this stage's record dir, engine-resolved)
---

# Deployment Pipeline Configuration

## Steps

### Step 1: Load Prior Context

- `<record>/construction/ci-pipeline/` から CI パイプラインの設定を読み込む（存在する場合）
- `<record>/construction/infrastructure-design/` からインフラストラクチャ設計を読み込む（存在する場合）
- `<record>/construction/nfr-design/` から NFR 設計（デプロイ関連の NFR）を読み込む（存在する場合）

増分型のスコープ（`bugfix`、`refactor`、`security-patch`）と `express` は、
設計上 CI Pipeline と Infrastructure Design をスキップする。brownfield では、
ワークスペースの既存のパイプライン設定とインフラストラクチャ設定、および
コードナレッジベースを調査する。Express の greenfield では、承認済みの要件、
Build and Test の結果、ワークスペースで生成されたデプロイ用成果物
（たとえば Dockerfile、サービスマニフェスト、IaC）を使う。デプロイ可能な対象が
存在しない場合、この CONDITIONAL ステージは skipped を報告する。実在する根拠に対してのみ
設計する — 欠けている CI やインフラストラクチャの成果物を決して捏造しない。

### Step 2: Generate Clarifying Questions

次の内容を扱う質問ファイルを作成する:
- どのデプロイ戦略にするか（blue/green、canary、rolling）？
- どの環境昇格ゲートを設けるか（dev → staging → prod）？
- 本番環境ではどの承認ワークフローにするか？
- どのロールバック手順にするか？
- どのフィーチャーフラグ戦略にするか（CloudWatch Evidently、AppConfig）？

stage-protocol.md の質問フローに従う。

### Step 3: Generate Artifacts

CD パイプラインの設定、デプロイ戦略ドキュメント、ロールバックランブック、フィーチャーフラグの設定、環境昇格マトリクスを作成する。

### Step 4: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage deployment-pipeline --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 5: Present Completion & Request Approval

完了時の絵文字: :rocket:
レビュー対象のパス: `<record>/operation/deployment-pipeline/`
標準の 2 択承認（Approve / Request Changes）。

## Sensors

このステージの出力は、`<record>/operation/deployment-pipeline/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `ci-config`, `quality-gates`, `infrastructure-specification`, `cicd-pipeline`.

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
