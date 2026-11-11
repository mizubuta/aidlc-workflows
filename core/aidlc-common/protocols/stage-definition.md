# Stage Definition Format

このファイルは、`{{HARNESS_DIR}}/aidlc-common/stages/` 配下のすべてのステージファイルの形を定める
正式なコントラクトである。スキーマ（`stage-schema.ts`）、
YAML パーサー（`lib.ts` 内の `parseStageFrontmatter`）、および YAML のステージ
ファイルは、すべてこのドキュメントに基づいて実装されている。

すべてのステージ `.md` の先頭にある YAML frontmatter が正式なものである。
ビルドステップ `{{INVOKE}} engine graph compile` は、YAML ソースから
`{{HARNESS_DIR}}/tools/data/stage-graph.json` を再生成する。ランタイムは
`lib.ts:282-289` にある変更されていない `loadStageGraph()` API を通じて、コンパイル済みの JSON を読み込む。
CI のドリフトチェック `aidlc-graph compile --check` は、JSON が YAML と食い違う場合に
ビルドを失敗させる。

---

## File layout

```yaml
---
# YAML frontmatter — authored fields
---

# [Stage Title]

## Steps
# prose body — required, always populated

## Sensors
# compact Imports/Upstream-targets summary; stage-specific exceptions stay local

## Learn
# conditional pointer to stage-protocol-learnings.md; absent module means no diary or ritual
```

---

## Authored fields

トップレベルの記述フィールド（および 4 つの `consumes[]` サブフィールド）。
optional と記されたもの以外はすべて必須である。`stage-schema.ts` のスキーマは
この表をそのまま写している。

| Field | Type | Required | Enum / Constraint |
|-------|------|----------|--------------------|
| `slug` | string | yes | kebab-case。ファイル名の語幹と必ず一致させる |
| `name` | string | optional | ユーザー向けの表示名の上書き。slug をタイトルケースにしたもので正しい場合は省略する。頭字語、接続詞、確立された句読点の表記に使用する |
| `phase` | string | yes | `initialization` \| `ideation` \| `inception` \| `construction` \| `operation`（小文字） |
| `execution` | string | yes | `ALWAYS` \| `CONDITIONAL` |
| `condition` | string | yes | 自由形式。`ALWAYS` の場合は常に実行される根拠を、`CONDITIONAL` の場合は分岐条件を記述する |
| `lead_agent` | string | yes | エージェント slug。`loadAgents()` を通じて `{{HARNESS_DIR}}/agents/*.md` に照らして動的に検証される — ハードコードされた enum はない |
| `support_agents` | string[] | yes | 空リストも可。各要素は有効なエージェント slug。`mode: pipeline` の場合、`lead_agent` と `support_agents` を合わせたすべての要素は一意でなければならず、順序付けられた各受領証の位置が 1 つの識別子を持つようにする。散文の `Supporting Agents:` から名前を変えたもの（形式上の名前変更のみ） |
| `mode` | string | yes | `inline` \| `subagent` \| `pipeline` \| `mob` \| `agent-team`。ステージの**コミュニケーショントポロジー**、つまり本文の実行中に誰が誰と話すかを表す。`inline`（コンダクターがすべての声を担い、ディスパッチはゼロ）、`subagent`（ハブアンドスポーク: コンダクターはドラフトのためにリードをディスパッチし、次に各 `support_agents[]` の要素を、互いに見えない実際のスポークとしてディスパッチし、最後に統合のためにリードをディスパッチする）、`pipeline`（チェーン: 各リンクが共同で成果物を作成し、各リンクは上流のすべての作業を見て作業成果物を直接前進させる。各リンクが戻るたびにコンダクターが `PIPELINE_LINK_COMPLETED` を発行し、最後のリンクが成果物を完成させる）、`mob`（メッシュ: 1 つの部屋で相互に議論し、異論は記録される。判断を要する異議はステージの途中で人間に提示される）が有効である。`pipeline`/`mob` には空でない `support_agents` が必要である。書き込みモデル: ディスパッチされた各サポートエージェントは、自身の貢献ファイル（`contributions/<agent-slug>.md`、stage-protocol-collaborators.md）を書き込む。ステージの `produces[]` 成果物を編集するのはリードだけであるが、直列化されたパイプラインのリンクは例外で、それらを直接前進させる。`mob` のステージ、およびサポートを伴う `subagent` のステージでは、貢献ファイルが完了のエビデンスとなる。`pipeline` では、現在の試行における順序付けられた完全な受領証チェーンが完了のエビデンスとなる。**`agent-team` は予約済みである**: メッシュ型コラボレーションのための将来のネイティブバスによるトランスポートである（`mob` がポータブルなモードである）。利用側が出荷されるまで、どのステージもこれを宣言しない。`mode` を読み込むオーケストレーターのコードは、`agent-team` を**必ず**明示的に処理する（最低限 "not yet implemented" をスローする）。デフォルトの経路にフォールスルーしてはならない。レビューループはモード「ではない」: `reviewer` + `reviewer_max_iterations` が、すべてのモードで二者間の批評トポロジーを提供する（stage-protocol-reviewer.md §12a） |
| `reviewer` | string | optional | エージェント slug。成果物の生成後、承認ゲートの前に呼び出される |
| `review_artifact` | string | optional | `reviewer` がある場合は常に必須。`produces[]` の中から必須の Markdown 項目を 1 つ指定する。これがレビュー対象の成果物となる。レビューの記録はそれに紐付けられ、ゲートはそれを `**Review:**` のパスとして示し、`--reject-finding` / `--reopen-finding` / `--fix-finding <artifact>#R-NN` がその指摘事項を扱う。レビュアーがそれに書き込むことは決してない（その中にある旧来の `## Review` セクションは移行のためにのみ読み込まれる）。ユニットごとのステージでは、必須の出力が適用されるすべての種類のユニットについて、引き続き適用可能でなければならない。プラグインによる追加で出力を増やすことはできるが、このスカラーの所有者を暗黙に変更することはできない。 |
| `reviewer_max_iterations` | integer | optional | 正の整数。`reviewer` が必要。デフォルトは `2` |
| `review_class` | string | optional | `adversarial` \| `advisory`。`reviewer` が必要。デフォルトは `adversarial`。`advisory` は通常フローでの 1 回のパスであり、その指摘事項は人間のゲートがエンジン所有の指摘事項リストから表示する。後続の出力書き込みによって無効化された終端の受領証は、次の序数で 1 回だけ範囲を限った回復リクエストを受ける。`none` はステージの値ではない。スコープの `review_cap` と実行ごとの `--review` は、実効的なクラスを下げることしかできない |
| `summary_confirmation` | string | optional | `required` \| `if-present`。`directive.ceremony.summary_confirmation` が `on` のときに、成果物の生成前に統合された回答レビューを使用するステージを示す。`off` はこのチェックポイントと受領証をスキップするが、その他の必須の人間による決定はスキップしない。`required` では、質問ファイル、`Looks correct` と完全に一致する回答、人間による裏付けのある受領証、変更されていない質問ファイルのダイジェスト、およびその受領証以降のネイティブな成果物書き込みが必要である。`if-present` は、条件付きの質問フローが質問ファイルを作成した場合にのみ同じチェックを適用する |
| `for_each` | string | optional | 成果物 slug。ステージはその成果物のインスタンスごとに 1 回実行される。ワークフローごとに 1 回のステージでは省略する。Doctor は、その成果物が上流のステージによって生成されることを検証する |
| `workspace_requires` | boolean | optional | デフォルトは `false`。`true` は、インテントごとの記録ディレクトリ配下の計画ドキュメントだけでなく、ワークスペースのルートにソースコードを書き込まなければならないステージを示す。その場合、ステージ完了時の成果物ガード（`aidlc-state.ts` の approve/advance/finalize/complete-workflow）は、ステージが完了する前に、`aidlc/` ワークスペースツリーとハーネスディレクトリの外にあるファイルを、ワークスペース内またはインテントの記録済みリポジトリのいずれかに要求する。`produces[]` の Markdown だけを書いてコードを書かなかったステージは拒否される。このフラグは、終端のレビューをワークスペースのソース状態にも結び付ける（#629）。ユニットごとのステージでは、各終端の受領証にさらにエンジンが検証した `source-manifest.json` が必要となり、そのユニットが主張するパスを結び付け、完了時には新しい主張をステージ開始時のソースのベースラインと比較するため、主張されていないアプリケーションソースの変更はフェイルクローズする（#662）。宣言しているステージが現在のステップである間にエージェントがプロジェクトファイルに書き込んだ内容は、承認時に Code Generation より前に変更されたファイルについて述べる行で名指しされることは決してない。現在これを宣言しているのは `code-generation` のみである |
| `produces` | string[] | yes | 空も可。小文字ケバブケースの成果物名 — ルールと実際のレジストリツールについては [成果物の語彙](../../../../docs/reference/16-artifact-vocabulary.md) を参照 |
| `consumes` | object[] | yes | 空も可。各要素は `{artifact, required, conditional_on?, cited?}` |
| `consumes[].artifact` | string | yes per entry | 小文字ケバブケース |
| `consumes[].required` | boolean | yes per entry | アクティブなプランに限定される。`true` は「生成元のステージが実行される場合、この consume は満たされなければならない」ことを意味する — その成果物が常に存在するという全体的な表明ではない。生成元をスキップするスコープ（例: `units-generation` をスキップする `bugfix`）では、その consume は意味を持たなくなり、ステージ本文が穏当な縮退を処理する。予約済みの `when:` プリミティブにより、将来は作成者がより豊かな述語を表現できるようになる |
| `consumes[].conditional_on` | string | optional | `brownfield` \| `greenfield`。無条件の consume では省略する — `always` という値はない |
| `consumes[].cited` | boolean | optional | `true`: 成果物は、この入力が存在する場合は常にそれを引用し、`upstream-coverage` チェックは必須の consume と並べてそれを読み込む。コンパイルはこれらの consume を、コンパイル済みの `consumes` の外側に、ステージの `cites` として列挙するため、これを指定しても完了済みのステージの記録は変わらない |
| `requires_stage` | string[] | yes | 空も可。各要素は既知のステージ slug。2 つの役割がある: (1) 意味的なデータ依存、(2) 意味的なつながりはないが表示順が固定されているステージのための、表示順のエッジ。算出される `display_order` の主要な入力である |
| `scopes` | string[] | optional | 各要素は、対応する `{{HARNESS_DIR}}/scopes/aidlc-<name>.md` ファイルを持つスコープ名。スコープを指定すると、そのスコープでこのステージは EXECUTE となり、指定がなければ SKIP となる。スコープ所属マトリクスをステージごとに転置したものであり、`aidlc-graph compile` はすべてのステージの `scopes:` を読み込み、コンパイル済みの EXECUTE/SKIP グリッド（`tools/data/scope-grid.json`）を出力する。3 つの initialization ステージはすべてのスコープを指定する（常に EXECUTE）。指定なしと `[]` は同一に扱われる |
| `ars` | object | optional | 出荷されている `tools/data/ars-priors.json` が名前を挙げていないステージ（プラグインのステージ）のための、コンポーザーのスクリーニング事前情報: `targets`（ARS 要素の記号 `iae` \| `csu` \| `ve` \| `r` \| `ua` のリスト。空も可）、`cost`（整数 `1`..`5`、または数値的にスクリーニングされることのないステージでは `null`）、任意の `role`（`initialization` \| `core` \| `phase-gate` \| `structural`）、および任意の `project_types`（`brownfield` \| `greenfield` の空でない部分集合で、ステージを 1 種類のプロジェクトに限定する `condition:` を反映する）— 事前情報ファイルの 1 エントリと同じ 4 つの事実である。そのままステージノードにコンパイルされる。`aidlc-graph ars` は、事前情報ファイルにその slug のエントリがない場合にこれを読み込む（両方ある場合はファイルが優先される）。どちらもないステージは `no-prior` の行となり、スクリーニングできない |
| `inputs` | string | yes | 人間向けの散文（現在の `**Inputs**:` 行を保持する） |
| `outputs` | string | yes | 人間向けの散文（現在の `**Outputs**:` 行を保持する）。**ランタイムでは意味を持たない** — エンジンはパス解決のために `outputs:` を決して読まない。出力時に、ノードの `produces[]` の成果物「名」を**アクティブなインテントの記録ディレクトリ**に照らして解決する（後述の「成果物のパスはエンジンが解決する」を参照）。`outputs:` は相対的な成果物「名」（または `<phase>/<stage>/<name>.md` の形）として記述する。ワークスペースのルート（`aidlc-docs/…` や `aidlc/spaces/…`）をハードコードしてはならない — 記録がインテントごとにルートを変えた瞬間に、その記述は誤りになる |

---

## Computed fields

数値の表示順はコンパイルステップによって導出されるものであり、YAML に記述するものではない。

| Field | Derivation |
|-------|------------|
| `display_order` | `<phase-prefix>.<sequence>`。フェーズ接頭辞: `initialization=0`、`ideation=1`、`inception=2`、`construction=3`、`operation=4`。シーケンス: このフェーズに絞り込んだ `requires_stage` エッジのトポロジカルソート。並列のステージは slug のアルファベット順で順位を決める |

`name` のデフォルトは、slug をタイトルケースにしたものである。明示的な `name:` は、
そのデフォルトでは頭字語、接続詞、または確立された表示ラベルが失われる場合にのみ記述する。

---

## Worked example

`scope-definition` ステージの YAML frontmatter。新しいステージを作成するときは、
これをコピー＆ペースト用のテンプレートとして使用する。
`stage-schema.ts` のスキーマは同じ形に照らして検証する。

```yaml
---
slug: scope-definition
phase: ideation
execution: ALWAYS
condition: Always executes — defines the scope boundary and prioritized backlog
lead_agent: aidlc-product-agent
support_agents:
  - aidlc-delivery-agent
mode: inline
summary_confirmation: required
produces:
  - scope-document
  - intent-backlog
  - scope-definition-questions
consumes:
  - artifact: intent-statement
    required: true
  - artifact: feasibility-assessment
    required: false
  - artifact: constraint-register
    required: false
requires_stage:
  - intent-capture
scopes:
  - enterprise
  - feature
  - mvp
inputs: Intent statement, feasibility assessment, constraint register
outputs: scope-document.md, intent-backlog.md, scope-definition-questions.md (under this stage's record dir, engine-resolved)
---
```

注: `display_order` はない（算出される）。`for_each` もない（ステージは
ワークフローごとに 1 回実行される — フィールドは省略）。`outputs:` 行は成果物を、ルートを含むパスではなく
相対的な「名」で示している — ルートはエンジンが解決する（後述）。

---

## Artifact paths are engine-resolved (no stage `.md` hardcodes a root)

ステージは相対的な成果物の**名前**（その `produces[]`）を出力する。エンジンは
ディレクティブの出力時に、**アクティブなインテントの記録ディレクトリに照らして**、それらを正規の書き込みパスに
解決する — `aidlc/spaces/<space>/intents/<YYMMDD>-<label>/<phase>/<stage>/<name>.md`
（ワークスペース導入前のプロジェクトは、最初に触れた時点でこのレイアウトに移行される — 移行後に
フラットなルートで解決する経路はない）。リゾルバーは
`aidlc-orchestrate.ts` 内の `resolveArtifactPath` / `memoryPathFor` であり、アクティブなインテントの相対的な記録ディレクトリ
（`aidlc-record-core.ts` 内の `relativeRecordDir`）が渡される。**どのステージ `.md` もワークスペースの
ルートをハードコードしない** — `outputs:` frontmatter や "Create `…/…`" といった散文は、
人間向けのドキュメントにすぎない。エンジンはコンダクターに解決済みの
`produces[]` のパスを渡す。ステージファイル内のルートを含むパスのリテラルは、動作のコントラクトではなく
ドキュメントのバグとして扱う。

同じ出力時の解決処理によって、consume される入力は存在の有無で分けられる。
ディレクティブの `consumes` には、ディスク上に存在する解決済みのパスのみが列挙され、
ファイルが存在しない必須の宣言済み入力は `consumes_absent` に移される。
生成元のステージがアクティブなスコープの経路外にある場合、または経路上のすべての生成元が
条件付きのランタイムスキップの後に `[S]` と記録されている場合（生成元が実行されなかった — 上記の
`consumes[].required` を参照）、それには `expected: true` の注記が付く。
`expected: false` は、経路上の生成元がスキップされなかったにもかかわらず、ファイルが
依然として存在しないことを意味する。存在しない `required: false` の consume は単に除外される —
存在しない任意の入力は入力ではなく、決してギャップではない。
コンダクターが読めないパスを示されることは決してない。設計上の不在は、
読み取りの失敗としてではなく、データとして届く。

---

## Body compartments

3 つの区画を、この順序で宣言する。現在内容があるのは `## Steps` のみで、
`## Sensors` と `## Learn` は、将来のリリースで内容が入る予約済みの見出し枠である。

| Compartment | Today | Future | Parser rule |
|-------------|-------|--------|-------------|
| `## Steps` | 必須、内容あり | 変更なし | 常に存在する |
| `## Sensors` | 予約済み、存在しない | 内容が入る（決定的なセンサー） | パーサーは不在を許容する |
| `## Learn` | 予約済み、存在しない | 内容が入る（ループドライバー、オブザーバールール） | パーサーは不在を許容する |

**本文の構造ルール:** 既存の本文の内容はすべて
`## Steps` の下に置き、それ以外には置かない。パーサーは
`## Sensors` と `## Learn` の見出しが存在しないことを許容する。

---

## Compile + drift invariant

`aidlc-graph compile` は、すべてのステージの YAML ソースを読み込み、
`{{HARNESS_DIR}}/tools/data/stage-graph.json` を再生成する。利用側は引き続き、
`loadStageGraph()` を通じてコンパイル済みの JSON を読み込む。

`aidlc-graph compile --check` はメモリ上でコンパイルを再実行し、
チェックイン済みの JSON と差分を取り、異なる場合は非ゼロで終了する。CI はこれを
すべての変更で実行する。チェックに合格していれば、ドリフトは起こりえない。

`aidlc-graph` がこのコントラクトを実装している。ライブラリと CLI については、
ハーネスの tools ディレクトリにある `aidlc-graph.ts` を参照
（8 つのエクスポート: loadGraph、producersOf、consumersOf、topoSort、findCycles、
subgraphForScope、validateScope、artifactsRegistry。加えて compile、compile
--check、および 7 つのクエリサブコマンド）。

---

## Future extensions — reserved namespace

現在は有効ではないが、意図的に予約されているフィールド。どのステージも
これらを宣言せず、スキーマは未知のキーを拒否する。ここで名前を挙げておくことで、
将来の貢献者による追加が計画中のプリミティブと衝突するのを防ぐ。

| Key | Purpose |
|-----|---------|
| `when` | 散文の `condition` に代わる構造化された置き換え。`consumes[].conditional_on` に取って代わり、`consumes[].required` のスコープを考慮したセマンティクスを、より豊かな述語（例: `producer-in-plan`、`mode == brownfield`）で一般化する |
| `on_failure` | 宣言的なエラー回復（ジャンプバック、入力を調整してのリトライ）。改訂のセマンティクスを `stage-protocol-recovery.md` の散文から移す |
| `blocks_on` | データの読み取りを伴わない完了依存。現在の多義的な `requires_stage`（「あなたの出力を consume する」と「あなたの後に実行する」を混同している）を分割する |
| `timeout`, `retry` | 実行の予算。ステージの frontmatter ではなく、センサーのバインディングとループ設定に置かれる（Claude Code のタスク API の設計を踏襲しており、プリミティブ単位のリトライ/タイムアウトはない） |

予約済み名前空間のパターンの先例: `docs/reference/06-hooks-and-tools.md` の監査分類は、
イベント名を、それを所有するツールやフックが使う前に一元的に宣言している。`ERROR_LOGGED` と
`RECOVERY_COMPLETED` が、現在の所有済みの例である。

**`mode` の利用側のコントラクト:** `mode` フィールドを読み込むオーケストレーターのコードは、
`agent-team` を明示的に処理しなければならない。最低限、"mode agent-team not
yet implemented" をスローする。デフォルトの実行経路にフォールスルーしてはならない — enum の拡張時に
黙ってフォールスルーすることは、レビューで指摘されている既知の落とし穴である。

**スウォームのトリガーとの結合:** 自律的な Construction スウォームは、
フィールドの一致（`for_each: unit-of-work` + `mode: subagent`）で起動する。ユニットごとのビルドステージの
モードを他のトポロジーに変更すると、黙ってスウォームの経路から外れる
（ユニットは直列にビルドされる）。`aidlc-graph compile` は、その形を検出すると
stderr に注意を出力する。

---

## Cross-references

- `stage-protocol.md` — ランタイムの実行時の動作（承認ゲート、質問
  フロー、状態の追跡）。この仕様はファイル形式を扱い、stage-protocol は
  動作を扱う。
- `SKILL.md` — オーケストレーターのルーティングとディスパッチ。
- `docs/reference/15-stage-definition.md` — 貢献者向けの、説明的な対応ドキュメント。
