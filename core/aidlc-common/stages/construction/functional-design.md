---
slug: functional-design
phase: construction
execution: CONDITIONAL
condition: New data models, complex business logic, or business rules need design. Skip if simple logic changes with no new business logic.
lead_agent: aidlc-architect-agent
support_agents:
  - aidlc-developer-agent
mode: inline
summary_confirmation: required
reviewer: aidlc-architecture-reviewer-agent
review_artifact: functional-spec
reviewer_max_iterations: 2
for_each: unit-of-work
produces:
  - entities
  - rules
  - functional-spec
  - traceability
optional_produces:
  - frontend-components
produces_kinds:
  entities: [service, spec, library]
  rules: [service, spec, library]
  functional-spec: [service, spec, ui, library]
  traceability: [service, spec, ui, library]
  frontend-components: [ui]
consumes:
  - artifact: unit-of-work
    required: true
  - artifact: unit-of-work-story-map
    required: false
  - artifact: stories
    required: false
  - artifact: requirements
    required: true
  - artifact: components
    required: true
  - artifact: contract-summary
    required: false
requires_stage:
  - units-generation
  - contract-design
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
  - refactor
  - classic
  - workshop
inputs: unit-of-work.md, unit-of-work-story-map.md, user-stories stories.md (if produced), requirements.md, domain-design components.md, contract-design contract-summary.md (if produced)
outputs: "entities.md, rules.md, functional-spec.md, traceability.json, CONDITIONAL: frontend-components.md (under this stage's per-unit record dir, engine-resolved); per-kind applicability via produces_kinds (untagged unit: all). entities.md and rules.md each carry a fenced ```yaml source-of-truth block; functional-spec.md is the source of truth for workflows and state machines and carries derived ER-diagram and rules-summary views."
---

# Functional Design
## Constraints

これは設計ステージである。成果物はビジネスロジック、ドメインモデル、ルールをアーキテクチャのレベルで記述するものであり、そのまま実装できるコードではない。完全な関数本体、クラスの実装、フレームワーク固有のコードは code-generation で扱う。コードは、設計上の決定を明確にする短い例示用のスニペット（擬似コードまたはインターフェースレベル、15 行以内）に限定する。

## Steps

### Execution Modes

このステージは、オーケストレーターが制御する 2 つの実行モードをサポートする:

**QUESTION-ONLY モード**（Bolt の質問フェーズでオーケストレーターが呼び出す）:
ステップ 1〜3 のみを実行する（コンテキストの読み込み、質問の生成、回答の収集）。
成果物の生成には進んではならない。制御をオーケストレーターに返す。

**ARTIFACT-ONLY モード**（Bolt の設計フェーズでオーケストレーターが呼び出す）:
ステップ 1〜3 をスキップする（質問はすでに収集・承認済み）。
ユニットごとのディレクトリから、回答済みの質問ファイルを読み込む。
ステップ 4〜6 のみを実行する（成果物の生成、状態の更新、完了）。

**Full モード**（デフォルト — 単一ユニットのプロジェクト、またはステージの直接呼び出し）:
記載どおりにすべてのステップを順番に実行する。

### Step 1: Read Unit Context

`<record>/inception/units-generation/unit-of-work.md` からユニットの定義を、`<record>/inception/units-generation/unit-of-work-story-map.md` から割り当てられたストーリーを（存在する場合）、`<record>/inception/user-stories/stories.md` からそれらのストーリーの受け入れ基準（`ACx.y.z`）を（存在する場合）読み込む。`<record>/inception/requirements-analysis/requirements.md`（存在する場合）、`<record>/inception/domain-design/components.md` のコンポーネントカタログ（存在する場合）、`<record>/inception/contract-design/contract-summary.md` にあるこのユニットの境界に関するコントラクト（存在する場合）を読み込む。

インクリメンタルなスコープ（refactor）は意図的に user-stories、units-generation、domain-design をスキップするため、そこではこれらの入力は設計上存在しない。入力が存在しない場合は、スコープが提供するもの — 要件と、ブラウンフィールドのワークスペースでは `aidlc/spaces/<active-space>/codekb/<repo>/`（`codekb-path --repo <repo>` が出力するディレクトリ）にあるリバースエンジニアリングされたコードナレッジベース — をもとに作業し、既存のコード構造を事実上のドメイン設計として扱う。存在しない成果物の内容を決してでっち上げてはならない。

### Step 2: Create Functional Design Plan

ユニットのスコープを分析し、[Answer]: タグを使った文脈に即した質問を含む機能設計の質問ファイルを `<record>/construction/{unit-name}/functional-design/functional-design-questions.md` に作成する。

重点領域:
- ビジネスロジックのワークフローとアルゴリズム
- ドメインモデルとエンティティの関係
- ビジネスルール、制約、バリデーションのロジック
- データフローと変換
- 他のユニットや外部システムとの連携ポイント
- エラー処理とエッジケース（Standard と Comprehensive の深さでは、各コンポーネントを `{{HARNESS_DIR}}/knowledge/aidlc-product-agent/corner-checklist.md` と照らし合わせて確認する。要件ですでに決着しているコーナーケースはそのまま引き継ぎ、利用者の判断が必要な未解決のコーナーケースだけを質問にする）
- フロントエンドのコンポーネント（コンポーネント階層、props/state、インタラクションフロー、フォームのバリデーション）
- ビジネスシナリオ（エンドツーエンドのユーザージャーニー、正常系／異常系、並行処理のエッジケース）

### Step 3: Collect and Analyze Answers

stage-protocol.md §3 の質問フローに従って回答を収集する（インタラクションモードの選択肢を提示し、回答を収集し、ファイルに書き戻す）。回答を収集した後、**必須**の曖昧さの分析を行う:
- 曖昧な回答を特定する（"mix of"、"not sure"、"depends"、"probably" など）
- 回答間の矛盾を確認する
- 成果物の生成に必要な詳細の欠落を指摘する

曖昧さが 1 つでも見つかった場合: フォローアップの質問を作成し、解決してから先に進む。

### Step 4: Generate Artifacts

`<record>/construction/{unit-name}/functional-design/` に次のものを生成する。技術に依存しない内容とし、どの言語でも実装できるようにする。コード、SQL、フレームワークへの言及は含めない。

- **entities.md**: エンティティモデル。フェンス付きの ```yaml による信頼できる唯一の情報源のブロックを持ち、各エンティティについて、説明、属性（名前、論理型、必須／一意、参照、許容値、デフォルト、最小／最大、制約）、エンティティレベルの制約、関係（カーディナリティと方向）を列挙する。ブロックの後に、エンティティ群の人間が読める短い要約を続ける。
- **rules.md**: ビジネスルール。フェンス付きの ```yaml による信頼できる唯一の情報源のブロックを持ち、番号付きの各ルール（`id: BRx.y`、例: `BR1.1` — トレーサビリティセンサーが認識する `BR{group}.{seq}` 形式）について、その記述、カテゴリ（validation/authorization/constraint/calculation/policy）、適用対象、トリガー、ロジック（平易な言葉による IF…THEN）、違反時の振る舞い、出典（FR{n}/NFR{n}）を列挙する。ブロックの後に、人間が読める短いルールの要約表を続ける。
- **functional-spec.md**: 振る舞いの仕様。これは**ワークフローとステートマシンに関する信頼できる唯一の情報源**であり、ユースケースがたどる番号付きのステップの順序と、ライフサイクルを持つエンティティの状態遷移を記述する。`entities.md`（データの形）と `rules.md`（判断のロジック）では、順序付けられた振る舞いや遷移を表現できないためである。また、読みやすさのために 2 つの**派生**ビューも含める: エンティティ関係の `mermaid` 図（`entities.md` から派生。そこにある YAML が信頼できる唯一の情報源）と、ルールの要約（`rules.md` から派生）である。`entities.md`/`rules.md` なしで `functional-spec.md` を生成する UI のみのユニットでは、このファイルは自己完結している。ユニットの定義と要件から、インタラクションのワークフローと画面／状態の遷移を正式に規定し、エンティティやルールへの依存は持たない。`stories.md` が存在する場合は、このユニットが実装するストーリー（`USx.y`）と並べてその名前を記載する。
- **frontend-components.md**（条件付き — ユニットにフロントエンド／UI が含まれる場合のみ）: コンポーネント階層、props/state の設計、インタラクションフロー、フォームのバリデーションルール、API との連携ポイント

`{{INVOKE}} engine sensor-traceability --stage functional-design --output-path <that path> --write-upstream`
を実行して、
`<record>/construction/{unit-name}/functional-design/traceability.json` を作成する。
これは、このユニットが責任を持つすべての ID を `upstream_ids` に列挙し、それぞれに `GAP` の行を付ける:

- `stories.md` と `unit-of-work-story-map.md` の両方がある場合: ストーリーマップが
  このユニットに割り当てたすべてのストーリーのすべての `ACx.y.z`。別のユニットが提供する
  共有ストーリーの基準も含む。
- そうでなく、`unit-of-work-story-map.md` がある場合: このユニットの行にあるすべての `FR`。
- それ以外の場合: Requirements Analysis が記録したすべての `FR`。

サブ要件を持つ要件は、その `FR{n}.{m}` の ID を通じて追跡され、
それ自体の行は不要である。

各 ID の `coverage` の行を設定する: `OK` の場合は、`rules.md` に存在する 1 つ以上の
`BRx.y` の ID を `target` に記載する（`ui` ユニットのように、その種類が `rules.md` を
書かないユニットは、代わりにその ID をカバーするこのステージのファイル、
`functional-spec.md` または `frontend-components.md` を単独で記載する。そのファイルは
存在していなければならない）。`N/A` の場合は理由を `target` に記載する
（例: `delivered by u2-profile`）。`Deferred` の場合は、それをカバーする後のステージを
`target` に記載する。この設計でカバーしていない間は `GAP` とする。上流の ID を
意図的に持たないルールを説明するには、任意の `reverse` 配列を使用する。説明のないルールは
機械的に孤立（orphan）として導出される:

```json
{
  "stage": "functional-design",
  "unit": "u1-auth",
  "upstream_ids": ["AC1.1.1", "AC1.1.2", "AC1.2.1"],
  "coverage": [
    { "id": "AC1.1.1", "status": "OK", "target": "BR1.1" },
    { "id": "AC1.1.2", "status": "GAP" },
    { "id": "AC1.2.1", "status": "N/A", "target": "delivered by u2-profile" }
  ],
  "reverse": [
    { "id": "BR1.3", "status": "N/A", "target": "technical validation rule" }
  ]
}
```

### Step 5: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage functional-design --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 6: Completion

完了メッセージと承認ゲートを提示する:

```
# :clipboard: Functional Design Complete — {unit-name}
```

生成した成果物の要約を示し、続けて次を示す:

```
**Review:** `<record>/construction/{unit-name}/functional-design/`
```

承認ゲート: 厳密に 2 択（Approve / Request Changes）。

## Sensors

このステージの出力は、`<record>/construction/{unit-name}/functional-design/` 配下の Markdown の設計成果物である。一部のセクションにはコードサンプルが含まれ、コードの形を扱うセンサーがそれを指摘することもある。

インポート: `required-sections`, `upstream-coverage`, `linter`, `type-check`, `traceability`.

上流の対象: `unit-of-work`, `unit-of-work-story-map`, `stories`, `requirements`, `components`, `contract-summary`.

`linter` と `type-check` は、一致する TypeScript/JavaScript のスニペットを検査する。
`traceability` は、このユニットに対応付けられたストーリーのすべての受け入れ基準、
またはフォールバックのすべての `FR` が宣言・カバーされていることを検証し、`BRx.y` の
ターゲットを `rules.md` と照合し、説明のないビジネスルールの孤立を検出する。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
