---
slug: deployment-execution
phase: operation
execution: CONDITIONAL
condition: Execute after deployment pipeline and environment are ready
lead_agent: aidlc-pipeline-deploy-agent
support_agents:
  - aidlc-developer-agent
mode: inline
summary_confirmation: required
produces:
  - deployment-log
  - smoke-test-results
  - health-check-report
  - deployment-execution-questions
consumes:
  - artifact: cd-config
    required: true
  - artifact: deployment-strategy
    required: true
  - artifact: environment-inventory
    required: true
  - artifact: build-test-results
    required: true
requires_stage:
  - deployment-pipeline
  - environment-provisioning
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
inputs: CD pipeline config from deployment-pipeline stage, provisioned environments from environment-provisioning stage, built artifacts from Construction
outputs: deployment-log.md, smoke-test-results.md, health-check-report.md, deployment-execution-questions.md (under this stage's record dir, engine-resolved)
---

# Deployment Execution

## Steps

### Step 1: Load Prior Context

- `<record>/operation/deployment-pipeline/` から CD パイプラインの設定とデプロイ戦略を読み込む（存在する場合）
- `<record>/operation/environment-provisioning/` から環境インベントリを読み込む（存在する場合）
- `<record>/construction/build-and-test/` からビルド／テストの結果を読み込む（存在する場合）
- ロールバックのランブックを読み込む（存在する場合）

インクリメンタルなスコープ（`bugfix`、`refactor`、`security-patch`）と `express`
は、設計上 Build and Test を維持しつつ Environment Provisioning をスキップする。`infra`
は両方を維持する。ワークスペースの既存のパイプラインがすでに十分な場合、Deployment Pipeline も
スキップを報告することがある。その場合、`cd-config` と `deployment-strategy` の成果物が
存在しないのは想定どおりであり、このステージは欠落した成果物の復旧を呼び出すのではなく、
ワークスペース内の実際のパイプライン設定を調べて使用しなければならない。実際のデプロイ先の環境は、
そのワークスペースの設定と、承認済みの Deployment Pipeline の成果物から
洗い出す。Express のグリーンフィールドでは、それらのファイルが実際のデプロイ先を
特定している場合にのみデプロイを進める。そうでなければ、この CONDITIONAL ステージはスキップを報告する。
環境インベントリやデプロイ経路を決してでっち上げてはならない。

### Step 2: Pre-Deployment Checks

デプロイ先が AWS の場合（パイプラインまたはデプロイコマンドが AWS CLI、CDK、SAM、
CloudFormation を使用する場合）、質問を書く前に、まず自分の
シェルから AWS のアイデンティティを確認する: デプロイで使用するプロファイルとリージョン
（同じ `AWS_PROFILE` と `AWS_REGION`、またはパイプラインが指定する `--profile` と `--region`）で
`aws sts get-caller-identity` を実行する。
利用者のターミナルではサインインしていても、ハーネスがコマンドを実行するシェルからは
その認証情報が見えないことがある。それがデプロイの開始時に初めて判明すると、
それまでのすべての回答にかけた時間が無駄になる。

- `aws` コマンド自体がインストールされていない場合は、このチェックをスキップして先に進む。CDK
  と SAM は AWS CLI なしでもデプロイできる。
- 成功した場合は先に進む。アカウント ID や完全な ARN を、成果物、質問ファイル、
  監査に決して書き込んではならない。アイデンティティを記録する場合は、
  アカウントの下 4 桁（`********1234`）とロール名またはユーザー名のみを残す。
- 失敗した場合（`NoCredentials`、`ExpiredToken`、`AccessDenied`、プロファイルの
  欠落）は、まだデプロイに関する質問をしてはならない。stage-protocol.md の質問フローを通じて、
  何が失敗したかをエラーの文言そのままで伝え、修正方法を示す質問を 1 つ行う: 認証情報は、
  このエージェントがコマンドを実行するシェルから見える必要がある。そのため利用者は、
  そのプロファイルで再度サインインし（たとえば
  `aws sso login --profile <profile>`）、ハーネスがシェルを起動する場所でプロファイルとリージョンを
  利用できるようにし（または、それらが設定されたターミナルからハーネスを再起動し）、
  完了したら返答する。その後、再度チェックを実行し、成功してからデプロイする。

このステージの後半で同じ失敗（デプロイ途中でのトークンの期限切れ）が起きた場合も、
同じように対処する: 停止し、修正方法を示して質問し、再度チェックしてから続行する。

次の内容を扱う質問ファイルを作成する:
- デプロイ前のチェックはすべて合格しているか？
- データベースのマイグレーションは必要か、またテスト済みか？
- 依存するサービスは利用可能で、正常に稼働しているか？
- デプロイのウィンドウ（実施時間帯）はいつか？

stage-protocol.md の質問フローに従う。

### Step 3: Execute Deployment

成果物をパイプラインに流す。スモークテストを実行する。ヘルスチェックを検証する。必要に応じてデータベースのマイグレーションを実行する: マイグレーションの実行は、subagent_type="aidlc-developer-agent" を指定して Task ツールに委任する。

### Step 4: Generate Artifacts

デプロイの実行ログ、スモークテストの結果、ヘルスチェックの検証レポート、データベースのマイグレーションログ（該当する場合）を作成する。

### Step 5: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage deployment-execution --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 6: Present Completion & Request Approval

完了時の絵文字: :package:
レビュー対象のパス: `<record>/operation/deployment-execution/`
標準の 2 択の承認（Approve / Request Changes）。

## Sensors

このステージの出力は `<record>/operation/deployment-execution/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `cd-config`, `deployment-strategy`, `environment-inventory`, `build-test-results`.

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
