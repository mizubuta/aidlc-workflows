---
slug: units-generation
phase: inception
execution: ALWAYS
condition: Always executes when in scope. Produces the dependency DAG that Stage 2.9 Delivery Planning consumes for Bolt sequencing. In the compiled scope grid, 2.7 (Units Generation) and 2.9 (Delivery Planning) travel together — both EXECUTE or both SKIP per scope.
lead_agent: aidlc-architect-agent
support_agents:
  - aidlc-delivery-agent
mode: inline
summary_confirmation: required
reviewer: aidlc-architecture-reviewer-agent
review_artifact: unit-of-work
reviewer_max_iterations: 2
review_class: advisory
produces:
  - unit-of-work
  - unit-of-work-dependency
  - unit-of-work-story-map
  - traceability
consumes:
  - artifact: components
    required: true
  - artifact: decisions
    required: false
  - artifact: requirements
    required: true
  - artifact: stories
    required: false
requires_stage:
  - domain-design
sensors:
  - required-sections
  - upstream-coverage
  - traceability
scopes:
  - enterprise
  - feature
  - mvp
  - classic
  - workshop
inputs: <record>/inception/domain-design/components.md, <record>/inception/requirements-analysis/requirements.md, <record>/inception/user-stories/stories.md (if produced)
outputs: unit-of-work.md, unit-of-work-dependency.md, unit-of-work-story-map.md, traceability.json (under this stage's record dir, engine-resolved)
---

# Units Generation

NOTE: **Stage 2.7 は依存関係 DAG（トポロジー）を作成する。Stage 2.9 Delivery Planning は、その DAG を通る経済的な経路（Bolt の順序）を選ぶ。** 2.7 は実装順序を推奨したりクリティカルパスを特定したりしてはならない。それらは 2.9 の経済的な順序付けの判断である。このステージは何が何に依存し得るかを記述し、2.9 は何を最初に出荷するか、そしてその理由を決める。

---

## Steps

### PART 1: Planning

### Step 1: Load Prior Context

- `<record>/inception/domain-design/components.md` からコンポーネントカタログ（フェンス付きの `yaml` ブロックに加え、図、概要、根拠）を読み込む
- `<record>/inception/domain-design/decisions.md` からアーキテクチャ決定記録を読み込む（作成されている場合）。境界／所有権に関する ADR は、コンポーネントをユニットにまとめる方法を制約する（たとえば、2 つのコンポーネントを別々にデプロイ可能に保つという決定があれば、それらを 1 つのユニットにまとめることはできない）
- `<record>/inception/requirements-analysis/requirements.md` を読み込む
- `<record>/inception/user-stories/stories.md` を読み込む（作成されている場合）

### Step 2: Create Decomposition Plan with Questions

[Answer]: タグ形式を使った質問を含む `<record>/inception/units-generation/units-generation-questions.md` を作成する:
- ユニット境界の戦略（サービス別、機能別、ドメイン別、デプロイ先別）
- ユニットの粒度の希望（粗粒度か細粒度か）
- 依存関係の順序付けの希望（厳密なトポロジカル順のみか、独立したユニット間の並列化を許すか）
- ユニット間の統合ポイントと契約（API、共有データ、イベント）
- デプロイモデル（モノリシックなデプロイ、独立したデプロイ、ハイブリッド）

skeleton-on が適用される場合は、解決済みの DAG 順で最初のユニットを、
統合経路をエンドツーエンドで動かせるだけの実際の実装を備えた、最小限の動作する
統合スライスとして形作る。後続の機能は後のユニットとして分解する。最初のユニットを、
後のユニットなしには動作しない孤立した設計文書やレイヤーだけにしてはならない。本当の
依存関係とアーキテクチャ上の境界は尊重する。両立しない分解は、承認する前に人間と
相談して解決する。想定されるデモと実際の統合チェックは Delivery Planning が確認する。
経済的な優先順位（残りの作業を価値優先にするかリスク優先にするか）は引き続き
Delivery Planning の担当であり、ここでその聞き取りを重複して行わない。

### Step 3: Collect and Analyze Answers

stage-protocol.md §3 の質問フローに従って回答を収集する（対話モードの選択肢を提示し、回答を収集し、ファイルに書き戻す）。
- 必須の曖昧さ分析: 曖昧な表現、矛盾、欠落している詳細を洗い出す
- 曖昧さが 1 つでも見つかった場合は、追加の質問を作成する
- 先に進む前にすべての曖昧さを解消する

### Step 4: Get Plan Approval

分解計画を構造化された質問としてユーザーに提示する:
- アプローチを要約する: ユニット境界の戦略、想定ユニット数、依存構造、ユニットごとに提案する種類（service/spec/ui/packaging/library）。これにより、各ユニットが Construction に持ち込む設計成果物のスコープを人間が確認できる
- 選択肢: Approve Plan / Revise Plan

---

### PART 2: Generation

### Step 5: Execute Plan — Generate Unit Artifacts

承認された計画に基づき、`<record>/inception/units-generation/` に 4 つの成果物を生成する（以下の 3 つのユニット成果物と、内容をこのステップの最後で定める `traceability.json`）:

**unit-of-work.md:**
- ユニットの定義（名前、説明、境界）
- すべてのユニットとそのディレクトリに対する安定した短い ID `U{n}`。ディレクトリは、下記のエッジブロックにおけるユニット名である。エンジンはユニットの Construction 成果物を `construction/<unit-name>/` 配下に書き込むため、各ユニットを `u{n}-{description}` と命名し、ディレクトリから ID が読み取れるようにする。下流のツールがストーリーマップの ID とファイルシステムのパスを結び付けられるよう、両方を表（`Unit ID` と `Directory`）に含める。
- ユニットの責務（各ユニットが所有し提供するもの）
- ユニットごとのデプロイモデル（standalone、shared、embedded）
- ユニットごとの相対的な複雑さの見積もり（S/M/L/XL）
- ユニットごとのユニット種別: `service` | `spec` | `ui` | `packaging` | `library`（そのユニットが何であるかを表し、どの Construction 設計成果物が適用されるかを決める。spec にはスケーラビリティ文書は不要であり、packaging ユニットにはビジネスロジックモデルは不要である）。`service` = デプロイされる実行可能物、`spec` = その場で利用される契約／スキーマ、`ui` = フロントエンドの画面、`packaging` = ビルド／配布用の成果物、`library` = 単独のランタイムを持たない再利用可能なコード。本当にどれにも当てはまらない場合に限り省略する。種別のないユニットには、設計成果物のマトリクス全体が適用される。
- ユニットごとの実装上の注意点と制約

**unit-of-work-dependency.md:**
- ユニット間の依存関係 DAG（有向エッジ: "A depends on B"（A は B に依存する））。循環があってはならない。
  ユニットは、自身のコードやテストのビルドまたは成功に必要なコードを持つすべてのユニットに依存する。互いにエッジのないユニットは同時にビルドされることがあり、その場合は他のユニットの作業を含まないプロジェクトのコピー上でそれぞれビルドされる。そのため、各ユニットのコードとテストは、自身のコード、依存先のユニット、現時点のプロジェクトだけで成功しなければならない。
- ユニット間の統合ポイント（API、共有データ、イベント）
- 並列開発の機会（互いに依存関係のないユニットの集合 — 有効なトポロジカル順序は複数存在する）
- skeleton-on が適用される場合は、解決済み DAG の最初のユニットを統合スライスとして特定し、後のユニットより前にどう動作できるかを説明する。`bolt-plan.md` にだけ記したマーカーでは、実行時の DAG 順序は変わらない。
- 必須のフェンス付き `yaml` エッジブロック（下記）。散文の DAG を機械可読な形で写したものである。下流のバッチ展開は散文ではなくこのブロックから計算されるため、存在し、正しい形式で、循環がないことが必須である。エンジンはこのステージのレビュー前とゲート前にこれを読み込み、読み込めない場合は、先に直すべき欠陥を正確に返す。`required-sections` センサーもゲートでこれをチェックする。

フェンス付きブロックには、すべてのユニットを、その直接の依存先（依存するユニット名）と、任意で各ユニットの `kind` とともに列挙する。独立したユニットには `depends_on: []` を付ける。新しいユニット名は、小文字のパスセグメント識別子として作成する。すなわち、小文字の英字で始まり、小文字の英字、数字、ハイフンが続く、最大 64 文字の名前である。ランタイムは、数字で始まる名前や、大文字、アンダースコア、ドットを含む、安全なレガシーの単一セグメント名も保持する。自律スウォームはそれらの名前を決定的な内部 Bolt slug に対応付けつつ、ディレクティブと監査記録では元のユニット識別子を保持する。表記を正規化するためだけに進行中のレガシーユニットの名前を変更しない。各ユニットはちょうど 1 回だけ記述する。`depends_on` リスト内のすべての名前は宣言済みのユニットでなければならない。ユニットは自分自身に依存してはならない。エッジは非循環でなければならない。各 `kind:` は、存在する場合、`service | spec | ui | packaging | library` のいずれかでなければならない（無効な値はこのゲートでエッジブロックのセンサーが失敗とする）。省略すると、そのユニットには Construction の設計成果物マトリクス全体が適用される:

```yaml
units:
  - name: <unit-name>
    kind: service
    depends_on: []
  - name: <another-unit>
    kind: spec
    depends_on: [<unit-name>]
```

NOTE: この成果物はトポロジーだけを記述する。単一の "recommended build order"（推奨ビルド順序）を選んだりクリティカルパスを特定したりはしない。それらは、この DAG を入力として 2.9（Delivery Planning）で行う経済的な判断である。

**unit-of-work-story-map.md:**
- 上流の項目ごとに 1 行とし、下記のトレーサビリティの列挙と同じキーを付ける。`stories.md` が作成されている場合は、すべての行のキーを `USx.y` にする。読み込まれるのは `USx.y` の行だけである。それ以外の場合は `FR` をキーにし、このスコープで NFR も追跡するなら、任意で `NFR` の行を追加する。サブ要件を持つ要件はその `FR{n}.{m}` ID で追跡されるため、その行のキーにはそれらを使う。各行には、実装するユニットの `U{n}` ID とディレクトリ名を記す（例: `| US1.1 | U1 | u1-ledger |`）。1 つのセルに複数のユニットを書く場合は `;` で区切る。行に記されたすべてのユニットは、その Functional Design でその行の受け入れ基準をすべて追跡する。したがって、ユニットを行に記すのは、そのユニットが当該項目の一部を実装する場合に限る
- 複数のユニットにまたがる行（横断的関心事）
- 各ユニット内の実装順序
- カバレッジの検証: 列挙されたすべての ID が割り当てられ、すべてのユニットが行を持つ

`{{INVOKE}} engine sensor-traceability --stage units-generation --output-path <that path> --write-upstream` を実行して
`<record>/inception/units-generation/traceability.json` を作成する。
`stories.md` が存在する場合はすべての `USx.y` が、存在しない場合は Requirements Analysis が
記録したすべての `FR`（サブ要件を持つ要件はその `FR{n}.{m}` ID）が、
それぞれ `GAP` 行として列挙される。各行の `status` と `target` を設定する。各 `OK` の target は、1 つのユニット ID または Construction ディレクトリであり、
`unit-of-work-story-map.md` のその ID の行にも記されているものでなければならない:

```json
{
  "stage": "units-generation",
  "upstream_ids": ["US1.1", "US1.2"],
  "coverage": [
    { "id": "US1.1", "status": "OK", "target": "U1" },
    { "id": "US1.2", "status": "GAP" }
  ]
}
```

### Step 6: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage units-generation --result <outcome>` を介して、
完了処理を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 7: Present Completion & Request Approval

stage-protocol.md の完了テンプレートを使い、完了時の絵文字は :wrench: とする
- 定義したユニット（各ユニットの種別付き）、対応付けた依存関係、割り当てたストーリーの概要
- レビューパス: `<record>/inception/units-generation/`
- 選択肢付きの構造化された承認質問: Approve（`<directive.next_stage>`、すなわちエンジンが指定するステージへ進む）/ Request Changes

## Sensors

このステージの出力は `<record>/inception/units-generation/` 配下の Markdown 成果物である。

取り込み: `required-sections`, `upstream-coverage`, `traceability`.

上流ターゲット: `components`, `decisions`, `requirements`, `stories`.

`unit-of-work-dependency.md` については、`required-sections` は正しい形式で
循環のないフェンス付き `yaml` エッジブロックも要求する。`traceability` は
`traceability.json` を担い、ユニットの集合を導出し、すべてのストーリー
（`stories.md` が作成されていない場合はフォールバックとしてすべての `FR`）が、
宣言されたターゲットユニットに対応付けられていることを検証する。

## Learn

`directive.protocol_modules` に `learnings` が含まれる場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を記録し、
承認ゲートの前に学びのステップを実行する。そのモジュールに、いつ利用者に質問するかが
定められている。モジュールがない場合は、日誌とステップの両方をスキップする。
