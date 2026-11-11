---
slug: nfr-design
name: NFR Design
phase: construction
execution: CONDITIONAL
condition: NFR Requirements was executed and NFR patterns need design. Skip if NFR Requirements was skipped.
lead_agent: aidlc-architect-agent
support_agents:
  - aidlc-aws-platform-agent
mode: inline
summary_confirmation: required
reviewer: aidlc-architecture-reviewer-agent
review_artifact: security-design
reviewer_max_iterations: 2
for_each: unit-of-work
produces:
  - performance-design
  - security-design
  - scalability-design
  - reliability-design
  - observability-design
  - logical-components
  - traceability
produces_kinds:
  performance-design: [service, ui]
  scalability-design: [service]
  reliability-design: [service]
  observability-design: [service]
  logical-components: [service, ui, library]
consumes:
  - artifact: performance-requirements
    required: true
  - artifact: security-requirements
    required: true
  - artifact: scalability-requirements
    required: true
  - artifact: reliability-requirements
    required: true
  - artifact: observability-requirements
    required: true
  - artifact: tech-stack-decisions
    required: true
  - artifact: functional-spec
    required: true
  - artifact: contract-summary
    required: false
requires_stage:
  - units-generation
  - nfr-requirements
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
inputs: NFR requirements artifacts, functional design artifacts
outputs: "performance-design.md, security-design.md, scalability-design.md, reliability-design.md, observability-design.md, logical-components.md, traceability.json (under this stage's per-unit record dir, engine-resolved); per-kind applicability via produces_kinds (untagged unit: all)"
---

# NFR Design

## Constraints

これは設計ステージである — 成果物はアーキテクチャのパターン、戦略、意思決定を記述するものであり、そのまま実装に使えるコードではない。完全な実装（ミドルウェア、インターセプター、リトライライブラリ、暗号化ルーチン）は code-generation に属する。コードは、設計上の意思決定を明確にするための短い説明用スニペット（擬似コードまたはインターフェースレベル、15 行以下）に限定する。

## Steps

### Execution Modes

このステージは、オーケストレーターが制御する 2 つの実行モードをサポートする:

**QUESTION-ONLY モード**（Bolt の質問フェーズ中にオーケストレーターから呼び出される）:
Step 1〜3 のみを実行する（成果物の読み込み、質問の生成、回答の収集）。
設計や成果物の生成には進んではならない。制御をオーケストレーターに返す。

**ARTIFACT-ONLY モード**（Bolt の設計フェーズ中にオーケストレーターから呼び出される）:
Step 1〜3 をスキップする（質問はすでに収集・承認済み）。
ユニットごとのディレクトリから回答済みの質問ファイルを読み込む。
Step 4〜7 のみを実行する（ソリューションの設計、成果物の生成、状態の更新、完了）。

**Full モード**（デフォルト — 単一ユニットのプロジェクト、またはステージを直接呼び出した場合）:
記載どおりにすべてのステップを順番に実行する。

### Step 1: Read Prior Artifacts

`<record>/construction/{unit-name}/nfr-requirements/` から NFR 要件を読み込む。`<record>/construction/{unit-name}/functional-design/` から機能設計の成果物を読み込む（存在する場合）。`<record>/inception/contract-design/contract-summary.md` からユニット間の契約を読み込む（作成されている場合） — 各境界における統合メカニズムと障害時の振る舞いが、ここで設計するレジリエンスとスケーラビリティのパターンを左右する。アーキテクチャ上のコンテキストとして、`<record>/inception/domain-design/components.md` から domain-design のコンポーネントカタログを読み込む（存在する場合）。スコープがそれらの設計ステージをスキップした場合は、NFR 要件と、ブラウンフィールドであればコードナレッジベースからアーキテクチャ上のコンテキストを導き出す — 存在しない成果物の内容を決して捏造しない。

### Step 2: Generate Design Questions

`<record>/construction/{unit-name}/nfr-design/nfr-design-questions.md` に質問ファイルを作成し、[Answer]: タグを使ってコンテキストに応じた質問を記載する。

重点領域:
- レジリエンスパターン（サーキットブレーカー、バルクヘッド、フォールバック戦略）
- スケーラビリティパターン（水平 対 垂直、データパーティショニング、キャッシュ階層）
- パフォーマンス最適化（レイテンシバジェット、スループット目標、リソースプーリング）
- セキュリティのアプローチ（多層防御、ゼロトラスト、暗号化標準）
- オブザーバビリティのアプローチ（メトリクスと SLI/SLO 目標、構造化ログ、トレーシングの深さ、アラートの方針、ダッシュボードのニーズ）
- 論理コンポーネントの境界（サービスの分離、障害ドメイン、影響範囲）

### Step 3: Collect and Analyze Answers

stage-protocol.md §3 の質問フロー（インタラクションモードの選択肢を提示し、回答を収集し、ファイルに書き戻す）に従って回答を収集する。回答の収集後、**必須**の曖昧さ分析を行う:
- 曖昧な回答（"mix of"、"not sure"、"depends"、"probably" など、「混在」「わからない」「場合による」「おそらく」の類）を特定する
- 回答間の矛盾を確認する
- 成果物の生成に必要な詳細の欠落を指摘する

曖昧さが 1 つでも見つかった場合は、フォローアップの質問を作成し、解消してから先に進む。

### Step 4: Design NFR Solutions

NFR のカテゴリごとに具体的なソリューションを設計する:

- **Performance**: キャッシュ戦略、クエリ最適化、コネクションプーリング、非同期処理、CDN の活用、遅延読み込み、ページネーション
- **Security**: 認証フロー、認可モデル、暗号化（保存時と転送時）、入力のバリデーション、CSRF/XSS 対策、シークレット管理、監査ログ
- **Scalability**: 水平／垂直スケーリングのアプローチ、ロードバランシング、データのパーティショニング／シャーディング、キューによる疎結合化、ステートレス設計
- **Reliability**: サーキットブレーカー、バックオフ付きのリトライポリシー、ヘルスチェック、グレースフルデグラデーション、フェイルオーバー戦略、データレプリケーション
- **Observability**: メトリクス収集戦略、構造化ログの設計、分散トレーシングのアーキテクチャ、アラートルール、ダッシュボードの仕様、SLI/SLO の追跡、相関 ID の伝播

### Step 5: Generate Artifacts

`<record>/construction/{unit-name}/nfr-design/` に次を生成する:

- **performance-design.md**: キャッシュアーキテクチャ、最適化戦略、リソースプーリング、非同期パターン、パフォーマンスバジェット
- **security-design.md**: 認証／認可のアーキテクチャ、暗号化の設計、入力バリデーション戦略、セキュリティヘッダー、コンプライアンス統制
- **scalability-design.md**: スケーリングのアーキテクチャ、負荷分散、データパーティショニング戦略、キャパシティのしきい値、オートスケーリングのルール
- **reliability-design.md**: レジリエンスパターン、サーキットブレーカーの構成、リトライポリシー、ヘルスチェックの設計、フェイルオーバー手順、バックアップ戦略
- **observability-design.md**: メトリクス収集のアーキテクチャ、構造化ログの設計、分散トレーシング戦略、アラートルールとエスカレーション、ダッシュボードの仕様、SLI/SLO の定義、相関 ID の伝播
- **logical-components.md**: 論理的なインフラコンポーネントの一覧 — サービス境界、障害ドメイン、影響範囲のマッピング、コンポーネントの分離戦略、共有リソースの特定。NFR パターンがどこに適用されるかをコンポーネントレベルで示すことで、NFR 設計の意思決定と Infrastructure Design を橋渡しする。

次を実行して `<record>/construction/{unit-name}/nfr-design/traceability.json` を作成する:
`{{INVOKE}} engine sensor-traceability --stage nfr-design --output-path <that path> --write-upstream`
このファイルには、このユニットの NFR 要件にあるすべての `NFRx.y`（NFR Requirements が実行されなかった場合は、
Requirements Analysis が記録したすべての `NFRn`）が、それぞれ `GAP` 行として列挙される。それぞれを
具体的な設計ソリューションに対応付ける:

```json
{
  "stage": "nfr-design",
  "unit": "u1-auth",
  "upstream_ids": ["NFR1.1", "NFR1.2"],
  "coverage": [
    { "id": "NFR1.1", "status": "OK", "target": "Redis cache with connection pooling" },
    { "id": "NFR1.2", "status": "GAP" }
  ]
}
```

### Step 6: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage nfr-design --result <outcome>` を通じて、
完了処理を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 7: Completion

完了メッセージと承認ゲートを提示する:

```
# :shield: NFR Design Complete — {unit-name}
```

NFR のカテゴリごとの設計上の意思決定のサマリーを示し、続いて次を示す:

```
**Review:** `<record>/construction/{unit-name}/nfr-design/`
```

承認ゲート: 厳密に 2 択（Approve / Request Changes）。

## Sensors

このステージの出力は、`<record>/construction/{unit-name}/nfr-design/` 配下の Markdown の設計成果物である。一部のセクションには、コード形状のセンサーも検出対象とするコードサンプルが含まれる。

インポート: `required-sections`、`upstream-coverage`、`linter`、`type-check`、`traceability`。

上流のターゲット: `performance-requirements`、`security-requirements`、`scalability-requirements`、`reliability-requirements`、`observability-requirements`、`tech-stack-decisions`、`functional-spec`、`contract-summary`。

`linter` と `type-check` は、該当する TypeScript/JavaScript のスニペットを検査する。
`traceability` は、詳細なすべての NFR 要件が宣言され、
設計ソリューションによってカバーされていることを検証する。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
