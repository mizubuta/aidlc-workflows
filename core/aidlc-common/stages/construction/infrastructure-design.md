---
slug: infrastructure-design
phase: construction
execution: CONDITIONAL
condition: Infrastructure services need mapping, deployment architecture required, or cloud resources needed. Skip if no infrastructure changes and infrastructure already defined.
lead_agent: aidlc-aws-platform-agent
support_agents:
  - aidlc-devsecops-agent
  - aidlc-compliance-agent
mode: inline
summary_confirmation: required
reviewer: aidlc-architecture-reviewer-agent
review_artifact: cicd-pipeline
reviewer_max_iterations: 2
for_each: unit-of-work
produces:
  - infrastructure-specification
  - monitoring-design
  - cicd-pipeline
  - traceability
produces_kinds:
  infrastructure-specification: [service, ui, packaging]
  monitoring-design: [service, ui, packaging]
  cicd-pipeline: [service, ui, packaging, library]
  traceability: [service, ui, packaging, library]
consumes:
  - artifact: performance-design
    required: true
  - artifact: security-design
    required: true
  - artifact: scalability-design
    required: true
  - artifact: reliability-design
    required: true
  - artifact: observability-design
    required: true
  - artifact: logical-components
    required: true
  - artifact: components
    required: true
  - artifact: functional-spec
    required: true
  - artifact: contract-summary
    required: false
requires_stage:
  - units-generation
  - nfr-design
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
  - classic
  - workshop
inputs: NFR design artifacts, domain design components.md, functional design
outputs: "infrastructure-specification.md (deployment + services + shared, tabular), monitoring-design.md (tabular), cicd-pipeline.md, traceability.json (under this stage's per-unit record dir, engine-resolved); per-kind applicability via produces_kinds (a spec unit owes none)"
---

# Infrastructure Design

## Constraints

これは設計ステージである。成果物は、どのようなインフラストラクチャがなぜ必要かを記述するものであり、そのまま実装できるコードではない。完全な IaC（CDK コンストラクト、Terraform モジュール、CloudFormation）、完全な Lambda ハンドラー、IAM ポリシードキュメントは code-generation に属する。code-generation は、このステージを実行するすべてのスコープ（`infra` を含む）でこのステージの後に実行され、Plan Approval のもとでそれらを書き込む。コードは、設計上の判断を明確にする短い例示用スニペット（擬似コードまたはインターフェースレベル、15 行以下）に限る。

## Steps

### Execution Modes

このステージは、オーケストレーターが制御する 2 つの実行モードをサポートする:

**QUESTION-ONLY モード**（Bolt の質問フェーズ中にオーケストレーターが呼び出す）:
ステップ 1〜3 のみを実行する（成果物の読み込み、質問の生成、回答の収集）。
設計や成果物の生成に進んではならない。制御をオーケストレーターに返す。

**ARTIFACT-ONLY モード**（Bolt の設計フェーズ中にオーケストレーターが呼び出す）:
ステップ 1〜3 をスキップする（質問は収集・承認済み）。
ユニットごとのディレクトリから回答済みの質問ファイルを読み込む。
ステップ 4〜7 のみを実行する（インフラストラクチャの設計、成果物の生成、状態の更新、完了）。

**Full モード**（既定 — 単一ユニットのプロジェクト、またはステージの直接呼び出し）:
すべてのステップを記載どおり順に実行する。

### Step 1: Read Prior Artifacts

コンテキストとして、これまでの設計成果物をすべて読み込む:
- `<record>/construction/{unit-name}/nfr-design/` から NFR 設計（存在する場合）
- `<record>/construction/{unit-name}/functional-design/` から機能設計（存在する場合）
- `<record>/inception/domain-design/components.md` からドメイン設計（コンポーネントカタログ）（存在する場合）
- `<record>/inception/contract-design/contract-summary.md` からユニット間の契約（作成されている場合）— 境界の連携方式（同期/非同期/共有ストア）は、ネットワーク、メッセージング、共有リソースのプロビジョニングの判断材料になる
- `<record>/construction/{unit-name}/nfr-requirements/` から NFR 要件（存在する場合）

インクリメンタルなスコープ（infra）は、設計上、domain-design と functional-design の連鎖をスキップする。それらの入力がない場合は、NFR 要件と、ブラウンフィールドであれば `aidlc/spaces/<active-space>/codekb/<repo>/` にあるリバースエンジニアリング済みのコードナレッジベースから、コンポーネントのトポロジーを導き出す。欠けている成果物の内容を決して捏造しない。

### Step 2: Generate Infrastructure Questions

`<record>/construction/{unit-name}/infrastructure-design/infrastructure-design-questions.md` に、[Answer]: タグを使ったコンテキストに即した質問を含む質問ファイルを作成する。

重点領域:
- デプロイ戦略（コンテナ、サーバーレス、ハイブリッド、マルチリージョン）
- コンピュート/ストレージ/ネットワーク（サイジング、トポロジー、レイテンシー要件）
- 監視のアプローチ（メトリクス、ログ、トレーシング、アラートのしきい値）
- CI/CD パイプライン（ビルドステージ、デプロイ戦略、ロールバック手順）
- シークレット管理（Vault、環境変数、ローテーションポリシー）
- スケーリングポリシー（オートスケーリングのトリガー、キャパシティの上限、コストの制約）

### Step 3: Collect and Analyze Answers

stage-protocol.md §3 の質問フローに従って回答を収集する（対話モードの選択肢を提示し、回答を収集し、ファイルに書き戻す）。回答の収集後、**必須**の曖昧さ分析を行う:
- 曖昧な回答を特定する（"cloud-based"、"auto-scale"、"standard monitoring"）
- 回答間の矛盾を確認する
- 成果物の生成に必要だが欠けている詳細を指摘する

曖昧さが 1 つでも見つかった場合: フォローアップの質問を作成し、解消してから先に進む。

### Step 4: Design Infrastructure

インフラストラクチャを 4 つの領域にわたって設計する:

- **デプロイアーキテクチャ**: コンピュートモデル（コンテナ、サーバーレス、VM）、ネットワークトポロジー、ストレージ戦略、環境構成（dev/staging/prod）
- **インフラストラクチャサービス**: データベース（種類、サイジング、レプリケーション）、キャッシュ（戦略、エビクション）、メッセージキュー、検索サービス、CDN、DNS、ロードバランサー
- **監視とオブザーバビリティ**: メトリクス収集、ログ集約、分散トレーシング、アラートルール、ダッシュボード、SLI/SLO の追跡
- **CI/CD パイプライン**: ビルドステージ、テストステージ、デプロイステージ、環境の昇格、ロールバック戦略、フィーチャーフラグ、成果物の管理

### Step 5: Generate Artifacts

`<record>/construction/{unit-name}/infrastructure-design/` に以下を生成する。内容は**表形式**に保つ。デプロイ、サービス、共有のセクションは表とし、監視も可能な限り表形式にする。散文は根拠の説明にのみ使い、表で保持できるデータには使わない。

**1. `infrastructure-specification.md`** — 中核となるインフラストラクチャ設計。デプロイ、インフラストラクチャサービス、共有リソースを 1 つの文書にまとめる。次の構成にする:

- **Deployment** — デプロイの観点の表:
  `| Facet | Choice | Rationale |`
  コンピュートモデル（コンテナ/サーバーレス/VM/ハイブリッド）、ネットワークトポロジー（イングレス/エグレス、VPC/サブネット）、ストレージ戦略、環境（dev/staging/prod）、IaC のアプローチ、リソースのサイジングの各行を含める。
- **Infrastructure Services** — サービスをキーとする表:
  `| Service | Role | Configuration | Notes |`
  （role = database / cache / queue / search / cdn / dns / load-balancer、configuration = サイジング、レプリケーション、エビクションなど）。
- **Shared Infrastructure**（CONDITIONAL — 複数のユニットがリソースを共有する場合のみ）— 次の表:
  `| Shared Resource | Owner Unit | Consumer Units | Access Boundary |`

**2. `monitoring-design.md`** — NFR Design の `observability-design` 戦略を実装する、プラットフォーム固有の監視。可能な限り表形式にする:

- **Metrics & KPIs** — `| Metric | Source | Threshold | Why it matters |`
- **Alerts** — `| Alert | Condition | Severity | Routes to |`
- **SLIs / SLOs** — `| SLI | SLO target | Measurement window |`
- **Logs & Tracing** — ログ集約の戦略とトレーシングの設定（短い散文または小さな表）、ダッシュボードの仕様。

**3. `cicd-pipeline.md`** — デリバリーパイプライン。ビルドステージ、テスト自動化との統合、デプロイ戦略（blue-green / canary / rolling）、ロールバック手順、環境の昇格、CI/CD におけるシークレット管理。ステップは本質的に順序を持つので、ここでは散文や順序付きリストでよい。役立つ場合は、ステージ→ゲートの対応に表を使う。

次のコマンドを実行して
`<record>/construction/{unit-name}/infrastructure-design/traceability.json` を作成する:
`{{INVOKE}} engine sensor-traceability --stage infrastructure-design --output-path <that path> --write-upstream`
このコマンドは、このユニットの NFR 設計にあるすべての `NFRx.y`（NFR Design が実行されなかった場合は
NFR Requirements のもの、NFR Requirements も実行されなかった場合は Requirements Analysis が記録したすべての `NFRn`）を、
それぞれ `GAP` 行として列挙する。インフラストラクチャを必要とする各設計判断を
具体的なリソースまたは設定に対応付け、何も必要としないものは
理由を添えて `N/A` とする:

```json
{
  "stage": "infrastructure-design",
  "unit": "u1-auth",
  "upstream_ids": ["NFR1.1", "NFR3.1"],
  "coverage": [
    { "id": "NFR1.1", "status": "OK", "target": "ElastiCache cluster" },
    { "id": "NFR3.1", "status": "GAP" }
  ]
}
```

### Step 6: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage infrastructure-design --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 7: Completion

完了メッセージと承認ゲートを提示する:

```
# :cloud: Infrastructure Design Complete — {unit-name}
```

インフラストラクチャに関する判断とサービス選定のサマリーを示し、続けて:

```
**Review:** `<record>/construction/{unit-name}/infrastructure-design/`
```

承認ゲート: 厳密に 2 択（Approve / Request Changes）。

## Sensors

このステージの出力は `<record>/construction/{unit-name}/infrastructure-design/` 配下の Markdown の設計成果物である。一部のセクションにはコードサンプルが含まれ、コード形状のセンサーがそれを指摘することもある。

インポート: `required-sections`, `upstream-coverage`, `linter`, `type-check`, `traceability`.

上流の対象: `performance-design`, `security-design`, `scalability-design`, `reliability-design`, `observability-design`, `logical-components`, `components`, `functional-spec`, `contract-summary`.

`linter` と `type-check` は、該当する TypeScript/JavaScript のスニペットを検査する。
`traceability` は、インフラストラクチャに関係するすべての `NFRx.y` の設計判断が
宣言され、カバーされていることを検証する。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
