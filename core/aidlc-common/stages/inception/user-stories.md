---
slug: user-stories
phase: inception
execution: CONDITIONAL
condition: Execute when user-facing features, multiple personas, complex business logic, or cross-team work is involved. Skip for pure refactoring, isolated bug fixes, infrastructure-only changes, or developer tooling.
lead_agent: aidlc-product-agent
support_agents:
  - aidlc-design-agent
  - aidlc-developer-agent
  - aidlc-quality-agent
mode: mob
summary_confirmation: required
reviewer: aidlc-product-lead-agent
review_artifact: stories
reviewer_max_iterations: 2
review_class: advisory
produces:
  - stories
  - personas
  - user-stories-assessment
  - traceability
consumes:
  - artifact: requirements
    required: true
  - artifact: business-overview
    required: false
    conditional_on: brownfield
  - artifact: component-inventory
    required: false
    conditional_on: brownfield
  - artifact: team-practices
    required: false
requires_stage:
  - requirements-analysis
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
inputs: <record>/inception/requirements-analysis/requirements.md, RE artifacts (if brownfield)
outputs: stories.md, personas.md, user-stories-assessment.md, traceability.json (under this stage's record dir, engine-resolved)
---

# User Stories

## Steps

### Step 1: Load the Lead Persona (mob stage)

このステップは aidlc-product-agent のペルソナ（`directive.lead_persona`）を伴う。このステージは
そのエージェントとして作業する。ステージプロトコルに従い、`directive.inline_context_paths` のすべての
パスを読み込む。このモブでは、そこにはプロダクトエージェントの共有ナレッジとロール別ナレッジだけが含まれる。
インラインのドラフトと統合作業はプロダクトマネージャーが担う。

このステージは `mode: mob`（stage-protocol-ensemble.md §5 "Multi-agent stages"）で実行される。サポートエージェント（ユーザー体験を担う aidlc-design-agent、実装可能性を担う aidlc-developer-agent、テスト容易性を担う aidlc-quality-agent）は、自分が演じる声では決してない。それらは PART 2 の間に独立した参加者としてディスパッチされる。それらのペルソナを自分のコンテキストに読み込んではならない。

### Step 2: Validate User Stories Are Needed

このプロジェクトにとってユーザーストーリーが価値を加えるかどうかを評価し、理由を示す:
- **実行する場合**: ユーザー向けの機能、複数のユーザーペルソナ、複雑なビジネスロジック、チーム間の調整が必要な場合
- **スキップする場合**: 純粋なリファクタリング、独立したバグ修正、インフラのみの変更、開発者向けツール

評価を記録する `<record>/inception/user-stories/user-stories-assessment.md` を作成する:
- 決定: 実行またはスキップ
- 根拠: このプロジェクトにユーザーストーリーが必要な理由、または不要な理由
- 考慮した要素: プロジェクトの種類、ユーザー向けのスコープ、複雑さの兆候
- 実行する場合: ストーリーが最も価値を加える主な領域
- スキップする場合: 代わりにどのような網羅手段があるか（例: 要件だけで十分）

スキップする場合は、
`{{INVOKE}} engine orchestrate report --stage user-stories --result skipped --reason '<reason>'` を実行する。
エンジンがスキップを記録し、スコープ内の次のステージへ進む。

### Step 3: Load Prior Context

- `<record>/inception/requirements-analysis/requirements.md` を読み込む
- brownfield の場合: `aidlc/spaces/<active-space>/codekb/<repo>/`（`codekb-path --repo <repo>` が出力するディレクトリ）から関連する RE 成果物を読み込む

---

## PART 1: Planning

### Step 4: Create Story Plan with Questions

`<record>/inception/user-stories/user-stories-questions.md` に、次を含むストーリー計画を作成する:
- **ペルソナ策定のアプローチ** — ユーザーは誰か。その目標は何か。
- **ストーリーの形式** — INVEST 基準（Independent、Negotiable、Valuable、Estimable、Small、Testable）を用いる
- **ストーリーの優先順位付け** — 要件分析に基づき、各ストーリーに MoSCoW の優先度（Must Have / Should Have / Could Have / Won't Have）を割り当てる。MVP の境界は Delivery Planning で正式に決定される。ストーリーの優先度はその決定の材料になる。
- **分割アプローチの選択肢** — 機能別、ペルソナ別、ワークフロー別、ドメイン領域別、エピック別
- **埋め込み質問** — ペルソナやストーリーの粒度について利用者の入力を得るため、[Answer]: タグ形式を用いる

### Step 5: Collect Answers

stage-protocol.md §3 の質問フロー（対話モードの選択肢を提示し、回答を集め、ファイルに書き戻す）に従って回答を集める。

### Step 6: Analyze Answers

必須の曖昧さ分析:
- すべての回答に曖昧な表現（"mix of"、"not sure"、"depends"、"probably"）がないか確認する
- 回答間の矛盾を確認する
- 欠けている詳細を特定する
- 曖昧さが 1 つでも見つかった場合はフォローアップの質問を作成する

### Step 7: Present plan and generate

ストーリー計画の概要（ペルソナ数、ストーリー数、分割アプローチ）をインラインで提示する。その後すぐに PART 2: Generation へ進む。利用者は、完了ゲートで計画と生成されたストーリーを合わせた出力をレビューし、承認する。

生成が完了する前に利用者がフィードバックを差し挟んだ場合は、修正依頼として扱い、生成を続ける前に計画をそれに合わせて更新する。

---

## PART 2: Generation (mob elaboration)

### Step 8: Execute Plan — Generate Stories and Personas via the Mob

これはモブ・エラボレーションの儀式である。プロダクトマネージャー（リード）が
ドラフトを担い、開発者と QA（およびデザイン）が独立した参加者として協働し、
その後プロダクトリーダーがレビューする（`stage-protocol-reviewer.md` §12a）。

**ラウンド 0 — リードがドラフトを作成する。** リードとして、承認された計画に基づき次をドラフトする:

**`<record>/inception/user-stories/personas.md`:**
- ユーザーペルソナの定義（名前、役割、目標、ペインポイント、コンテキスト）
- ペルソナ間の関係と優先順位

**`<record>/inception/user-stories/stories.md`:**
- 標準形式のユーザーストーリー: "As a [persona], I want [goal], so that [benefit]"。各ストーリーには安定した `US{group}.{seq}` ID（例: `US1.1`）を付ける。
- 各ストーリーの受け入れ基準。各基準には 3 つのセグメントからなる `AC{story-group}.{story-seq}.{criterion-seq}` ID（例: `AC1.1.1`）を付ける。
- ストーリーの優先度（Must Have / Should Have / Could Have / Won't Have）
- ストーリーの依存関係と相互関係
- INVEST への準拠に関するメモ

**ラウンド 1 — モブをディスパッチする。** stage-protocol-collaborators.md の `mode: mob` に従い、
ディレクティブが `support_agents` に挙げるコラボレーターだけを正確にディスパッチする。
ドラフトに対して並列に、互いの内容を見せずに実行する（成果物はパスで渡す: 2 つのドラフト成果物、Q&A
ファイル、requirements.md。ルールは `stage-protocol.md` § "For subagent stages" のステップ 2 に従う）。
各コラボレーターは、自分のコントリビューションファイルを
`<record>/inception/user-stories/contributions/<agent-slug>.md` に書き込む（§11 の形式:
1 行目に識別マーカー、Contribution、Positions）。デザインは UX とペルソナの忠実性、
開発者は実装可能性とストーリーのサイズ、品質は受け入れ基準のテスト容易性を担当する。

**`directive.support_agents` が空の場合は、ラウンド 1 と以下のトリアージをスキップする** —
このスコープではコラボレーターのスイッチがオフなので、リードが単独で作業する
（§5）。ドラフトがそのままユーザーストーリーとなり、ディスパッチ・統合・トリアージする
モブのラウンドも、コントリビューションファイルもない。

**統合とトリアージ。** リードとして、コントリビューションを 2 つの成果物に取り込み、
stage-protocol-collaborators.md に従って未解決の異議をトリアージする。判断の問題（どちらの
立場も正当）は、構造化された質問として今すぐ利用者に提示する（先に質問ファイルに
追加し、`[Answer]:` タグは空にしておく）。知識に関する論争は
**ラウンド 2** に回す — 異議を唱えたエージェントだけに、修正したドラフトと
他の参加者の立場を渡して再ディスパッチする（各エージェントは自分の
コントリビューションファイルを更新する）。維持された反対意見は、Step 10 の
完了サマリーにそのまま引用する。ディスパッチしたコラボレーターのコントリビューションファイルは、
このステージのアンサンブルの証拠である — ディレクティブの実効的な `support_agents` のいずれかの
ファイルが欠けている間、エンジンは承認を拒否する。単独で作業するリードにはこのファイルはない。

**要素レベルのトレーサビリティを書き込む。**
`{{INVOKE}} engine sensor-traceability --stage user-stories --output-path <that path> --write-upstream` を実行して
`<record>/inception/user-stories/traceability.json` を作成する。
このファイルには、Requirements Analysis が `upstream_ids` に記録したすべての `FR` と `NFR` の ID が、
それぞれ `GAP` 行とともに列挙される（サブ要件を持つ要件は
その `FR{n}.{m}` ID を通じて追跡されるため、要件自体の行は不要である）。各
行の `status` と `target` を設定する。`OK` のターゲットには、既存の
`USx.y` ID を 1 つ以上指定しなければならない。`Deferred` は
下流のステージ名を指定する場合に限り、`N/A` は正当な理由がある場合に限り使う:

```json
{
  "stage": "user-stories",
  "upstream_ids": ["FR1", "FR2", "NFR1"],
  "coverage": [
    { "id": "FR1", "status": "OK", "target": "US1.1, US1.2" },
    { "id": "NFR1", "status": "Deferred", "target": "nfr-requirements" },
    { "id": "FR2", "status": "GAP" }
  ]
}
```

### Step 9: Open the Approval Gate

リードの 3 つの成果物と、ディスパッチしたすべてのコラボレーターの
コントリビューションファイル（リードが単独で作業する場合はなし）を確認した後、次を実行する:

```bash
{{INVOKE}} engine orchestrate report \
  --stage user-stories --result awaiting-approval
```

エンジンがアンサンブルの証拠の欠落や不正な形式を理由に拒否した場合は、
人間のゲートを提示する前にその証拠を復元する。

### Step 10: Present Completion & Request Approval

完了の絵文字 :books: を使い、stage-protocol.md の完了テンプレートを用いる:
- 作成したペルソナとストーリーの概要
- レビューパス: `<record>/inception/user-stories/`
- 選択肢 Approve / Request Changes を持つ構造化された承認の質問。Approve の選択肢の説明には `Continue to <next stage name>` と書く。その名前は、ゲートを開いた応答の `next_stage` フィールドから取り、なければ run-stage ディレクティブのものから取る（null の場合は `Complete workflow`）。利用者には実際のステージ名を見せ、フィールド名は決して見せない。

人間の応答を待って必ず停止し、その応答を読む。**Approve** は
`--result approved --user-input "Approve"` で報告する。
**Request Changes** は `--result rejected --user-input "Request Changes"` で報告し
（利用者の言葉は記録とともに保存される。補足したい場合にだけ `--reason` を追加する）、
修正ループを実行し、再提示する前に `--result revised` を報告する。ライフサイクルの
遷移と前進はすべてエンジンが担う。

## Sensors

このステージの出力は `<record>/inception/user-stories/` 配下の markdown 成果物である。

インポート: `required-sections`、`upstream-coverage`、`traceability`。

上流のターゲット: `requirements`、`business-overview`、`component-inventory`、`team-practices`。

`traceability` は `traceability.json` を担い、すべての要件が宣言され網羅されていることを検証し、
各 `OK` ターゲットが `stories.md` に存在することを確認する。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを行う。利用者に何かを尋ねるタイミングは
そのモジュールが定める。モジュールがない場合は、日誌と学びのステップの両方を
スキップする。
