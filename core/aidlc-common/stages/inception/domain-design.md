---
slug: domain-design
phase: inception
execution: CONDITIONAL
condition: Execute when new components or logical building blocks are needed. Skip when changes are modifications to existing components only.
lead_agent: aidlc-architect-agent
support_agents:
  - aidlc-aws-platform-agent
  - aidlc-design-agent
mode: inline
summary_confirmation: required
reviewer: aidlc-architecture-reviewer-agent
review_artifact: components
reviewer_max_iterations: 2
review_class: advisory
produces:
  - components
  - decisions
  - traceability
consumes:
  - artifact: requirements
    required: true
  - artifact: stories
    required: false
  - artifact: architecture
    required: false
    conditional_on: brownfield
  - artifact: component-inventory
    required: false
    conditional_on: brownfield
  - artifact: team-practices
    required: false
requires_stage:
  - requirements-analysis
  - refined-mockups
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
inputs: <record>/inception/requirements-analysis/requirements.md, <record>/inception/user-stories/stories.md (if produced), RE artifacts (if brownfield)
outputs: components.md (fenced ```yaml component catalogue plus a human-readable mermaid diagram and summary table), decisions.md (Architecture Decision Records), and traceability.json — all under this stage's record dir, engine-resolved
---

# Domain Design

システムの**論理的な構成要素**、つまりコードを書く対象となるコンポーネントを特定し、詳細化する。コンポーネントとは、独自のビジネスロジック、エンティティ、ライフサイクルを持つ、境界のあるソフトウェアの一部である。**デプロイするインフラではなく、書くコードである。** データベース、キャッシュ、キュー、サードパーティのサービスはコンポーネントの依存先であり、コンポーネントそのものではない。

このステージでは、デプロイのトポロジー（モノリス、マイクロサービス、サーバーレスなど）は決定**しない** — それは Units Generation の役割である。Domain Design は構成要素を作り出し、チームがそれをデプロイ可能なユニットにどうまとめるかを後で決められるようにする。また、技術スタックや NFR のパターンも選択しない — それらは NFR とインフラのステージに属する。

## Steps

### Step 1: Load Prior Context

- `<record>/inception/requirements-analysis/requirements.md` を読み込む
- `<record>/inception/user-stories/stories.md` を読み込む（作成されている場合）
- ブラウンフィールドの場合: 関連する RE 成果物（特に architecture.md、component-inventory.md、dependencies.md）を読み込む

### Step 2: Create Design Plan with Questions

[Answer]: タグ形式を使い、文脈に適した質問を含む `<record>/inception/domain-design/domain-design-questions.md` を作成する:
- コンポーネント境界の決定（何を独立した構成要素とし、それはなぜか）
- エンティティの所有（各エンティティを所有するコンポーネントはちょうど 1 つ — 曖昧さは設計上の悪い兆候である）
- コンポーネントの責務（各構成要素がどのビジネスロジックを所有するか）
- コンポーネント間のやり取り（どのコンポーネントがどれを呼び出すか、それはなぜか）
- 既存コンポーネントとの統合方法（ブラウンフィールド）
- UI コンポーネントの構造（ユーザー向けの場合。UX デザイナーの視点を踏まえる）

### Step 3: Collect and Analyze Answers

stage-protocol.md §3 の質問フロー（インタラクションモードの選択肢を提示し、回答を収集し、ファイルに書き戻す）に従って回答を収集する。
- **必須**の曖昧さ分析: 曖昧な表現、矛盾、欠けている詳細がないかを精査する
- 曖昧さが 1 つでも見つかったら、追加の質問を作成する
- 先に進む前に、すべての曖昧さを解消する

### Step 4: Generate the Component Catalogue

`<record>/inception/domain-design/components.md` を作成する。この 1 つの成果物に、機械可読なカタログと人間が読むためのビューの両方を含める。

**エンティティを記述する深さ。** エンティティは**所有 + 形**のレベルでのみ記述する — 各エンティティをどのコンポーネントが所有するか、その識別子、属性名、コンポーネントをまたぐ参照。ここではデータ型、検証制約、許容値、関係の多重度は指定**しない** — その完全なスキーマは Functional Design（`entities.md`）に属する。すべてのエンティティを所有するコンポーネントは**ちょうど 1 つ**である。所有が曖昧なのは設計上の悪い兆候であり、ゲートの前に解消する。

**パート A — 機械可読なカタログ（フェンス付きの `yaml` ブロック）。** ファイルの冒頭付近に、すべてのコンポーネントを列挙するフェンス付きの ```yaml ブロックを書く。このブロックが信頼できる情報源であり、以下の人間向けのビューはここから導出する:

```yaml
components:
  - name: <ComponentName>              # PascalCase, unique
    summary: <one-line purpose>
    behaviour: >
      <business rules, validation logic, security constraints, key behaviours — be specific>
    responsibilities:
      - <what this component owns>
    depends_on:                        # components it CALLS ([] if none)
      - component: <OtherComponentName>
        interaction: <why / what for>
        style: <sync | async | event>
    dependents:                        # components that CALL this one ([] if none)
      - component: <OtherComponentName>
        interaction: <why / what for>
    external_dependencies:             # infra / third-party this component USES (optional)
      - name: <e.g. PostgreSQL | Redis | Stripe API>
        kind: <database | cache | queue | object-store | third-party-api | other>
        purpose: <what it's used for>
    entities:                          # entities owned by THIS component ([] if none)
      - name: <EntityName>
        identifier: <attribute that uniquely identifies it>
        attributes: [<attributeName>, <attributeName>]
        references:                    # entities in OTHER components this points to (optional)
          - entity: <OtherEntityName>
            owned_by: <OwningComponentName>
            relationship: <plain-language, e.g. "each Order belongs to one Customer">
```

整形式のルール（すべてを満たさなければならない）: 各コンポーネント名は一意である。どこかで名前が挙がっている `component:`/`owned_by` はすべて宣言済みのコンポーネントである。自分自身に依存するコンポーネントはない。`depends_on`/`dependents` は対称である（A が B に depends_on なら、B は dependents に A を挙げる）。すべてのエンティティはちょうど 1 つのコンポーネントに所有され、識別子を持つ。すべての `references.entity` は、その `owned_by` のコンポーネントの下で宣言されている。依存関係グラフは非循環である（意図的な循環があれば Rationale で明記する）。インフラ、データベース、キャッシュ、キュー、サードパーティのサービスは `external_dependencies` であり、決してコンポーネントではない。

**パート B — 人間が読むためのビュー（ブロックの下）。** 次のセクションをカタログから導出する — 同じデータを人間向けに提示する:

- **Component Diagram** — `mermaid` のグラフ。コンポーネントごとに 1 ノード、`depends_on` ごとにラベル付きのエッジを 1 本。
- **Component Summary** — 表: `| Component | Purpose | Depends On | Dependents | Entities Owned |`。
- **Entity Ownership** — 表: `| Entity | Owning Component | Identifier | Attributes | References |`。
- **External Dependencies** — 表: `| Component | Dependency | Kind | Purpose |`。
- **Rationale** — 各コンポーネントが独立した構成要素である理由を説明する表（異なるライフサイクル、異なる関心事、異なるデータ所有、異なる変更頻度 — 当てはまるものを選ぶ）。

#### Component-boundary options (when >1 viable decomposition)

分割の選択に実行可能なアプローチが複数ある場合は、
決定を記録する前にトレードオフを提示する:

- Option A — <name>: 長所 / 短所 / 可逆性
- Option B — <name>: 長所 / 短所 / 可逆性
- Recommendation: <option>。理由は <trade-off tied to responsibilities/change rate>

チームがゲートで選択し（所有権はチームにある）、その後、選ばれた分割と
**Alternatives Rejected** の注記を components.md の Rationale セクションに
記録する。

実行可能な分割が 1 つしかない場合は、その理由を述べてこのブロックを省略する。

### Step 5: Record Architecture Decisions (ADRs)

`<record>/inception/domain-design/decisions.md` を作成する。`components.md` の Rationale 表はコンポーネントごとの簡単な根拠づけであり、`decisions.md` は Inception フェーズのルールが要求する永続的な Architecture Decision Record のログである。ここで行った**重要な**設計上の選択ごとに ADR を 1 つ記録する — コンポーネント境界の分割、エンティティ所有の判断、コンポーネント間のやり取りのスタイル、意図的な依存関係の循環など。

各 ADR は（Inception フェーズのガードレールに従い）**必ず**次の構造に従う:

- **ADR-NNN: <short title>**
  - **Context** — 決定を必要とした要因と制約
  - **Decision** — 何を選んだか
  - **Consequences** — 結果として生じるトレードオフ（良い面と悪い面の両方）
  - **Alternatives Rejected** — 検討した他の実行可能な選択肢と、それらを選ばなかった理由

ADR には連番を振る（`ADR-001`, `ADR-002`, …）。決定が Step 4 のコンポーネント境界の選択肢ブロックから来た場合、却下された選択肢をその ADR の **Alternatives Rejected** に記入する。重要な決定がなかった場合（トレードオフのない自明な分割が 1 つだけの場合）は、ファイルを空のままにせず、そのことを 1 つの ADR に明記する。

### Step 6: Record Traceability

`{{INVOKE}} engine sensor-traceability --stage domain-design --output-path <that path> --write-upstream`
を実行して `<record>/inception/domain-design/traceability.json` を作成する。
`stories.md` が存在する場合はすべての `USx.y` が、存在しない場合は
Requirements Analysis が記録したすべての `FR` が（サブ要件を持つ要件はその `FR{n}.{m}` ID を通じて追跡され、それ自体の行は不要）、それぞれ `GAP` の行として列挙される。各上流 ID を、それを実現する `components.md` 内の**コンポーネントまたはエンティティ**に対応づける —
それらが、このステージの信頼できる情報源が定義する唯一の識別子である（サービスや公開メソッドはここでは名指ししない。
メソッドや API レベルの対象は、後の Contract Design と
Functional Design で確定する）:

```json
{
  "stage": "domain-design",
  "upstream_ids": ["US1.1", "US1.2"],
  "coverage": [
    { "id": "US1.1", "status": "OK", "target": "AuthComponent" },
    { "id": "US1.2", "status": "GAP" }
  ]
}
```

### Step 7: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage domain-design --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 8: Present Completion & Request Approval

stage-protocol.md の完了テンプレートを、完了時の絵文字 :building_construction: とともに使う:
- 特定したコンポーネントのサマリー（数、主要な境界、エンティティの所有）
- 主要な境界の決定を強調する（`decisions.md` の ADR ログへの参照を添える）
- レビューパス: `<record>/inception/domain-design/`
- 次の選択肢を持つ構造化された承認の質問:
  - Approve（次のステージへ進む）
  - Request Changes（修正のフィードバックを提供する）
  - Approve, and add Units Generation next（実行計画でスキップされていた場合）

利用者が "Approve, and add Units Generation next" を選んだ場合は、
`{{INVOKE}} engine orchestrate next --add units-generation` を実行し、その
ステップに従う。そのステップが承認を記録する report を指定する。その後 Units Generation
が次に実行される。利用者が Units Generation の追加だけを求めた場合は、同じ
`next --add units-generation` を実行し、計画に追加したことを 1 行で伝え、
設計をこのまま承認するか、何か変更を望むかを一度だけ尋ねる。

## Sensors

このステージの出力は、`<record>/inception/domain-design/` 配下の Markdown 成果物（`components.md` と `decisions.md`）と `traceability.json` である。

インポート: `required-sections`, `upstream-coverage`, `traceability`.

上流の対象: `requirements`, `stories`, `architecture`, `component-inventory`, `team-practices`.

`traceability` は `traceability.json` を所有し、すべてのストーリー、または
フォールバックとしてのすべての機能要件が宣言され、カバーされていることを確認する。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
