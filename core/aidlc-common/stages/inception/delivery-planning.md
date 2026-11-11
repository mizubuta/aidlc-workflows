---
slug: delivery-planning
phase: inception
execution: ALWAYS
condition: Always executes — capstone Inception stage, produces the detailed execution plan for Construction and Operation
lead_agent: aidlc-delivery-agent
support_agents:
  - aidlc-architect-agent
mode: inline
summary_confirmation: required
produces:
  - bolt-plan
  - team-allocation
  - risk-and-sequencing-rationale
  - external-dependency-map
  - delivery-planning-questions
consumes:
  - artifact: requirements
    required: true
  - artifact: stories
    required: false
  - artifact: mockups
    required: false
  - artifact: components
    required: true
  - artifact: unit-of-work
    required: true
  - artifact: unit-of-work-dependency
    required: true
  - artifact: unit-of-work-story-map
    required: false
  - artifact: contract-summary
    required: false
  - artifact: team-practices
    required: false
requires_stage:
  - units-generation
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
  - mvp
  - classic
  - workshop
inputs: All Inception artifacts (requirements, stories, mockups, architecture, units)
outputs: bolt-plan.md, team-allocation.md, risk-and-sequencing-rationale.md, external-dependency-map.md, delivery-planning-questions.md (under this stage's record dir, engine-resolved)
---

# Delivery Planning

## Steps

### Step 1: Load Prior Context

Inception フェーズのすべての成果物を読み込む:
- `<record>/inception/requirements-analysis/` から要件
- `<record>/inception/user-stories/` からユーザーストーリー
- `<record>/inception/domain-design/components.md` からドメイン設計（コンポーネントカタログ）
- `<record>/inception/units-generation/` からユニット
- `<record>/inception/contract-design/contract-summary.md` からユニット間の契約（作成されている場合）— 契約のオーナーシップと未解決の契約上の質問は、Bolt の順序付けとウォーキングスケルトンに対応付けられる
- `<record>/ideation/team-formation/` からチーム編成（存在する場合）

**practices-discovery が実行された場合**は、
`aidlc/spaces/<active-space>/memory/{project,team,org}.md` から、最も具体的な
空でない記述を使って 3 つのセクションを解決する:
- `## Way of Working` — Construction のワークツリーのベース／ターゲットブランチとマージ戦略
- `## Walking Skeleton` — 最初の Bolt を最小限のエンドツーエンドのスライス（ゲート付きで、利用者の承認を別途得る）にするか、通常の Bolt にするか
- `## Deployment` — Bolt を並列と直列のどちらで実行するかの方針と、承認ゲートの好み

`bolt-plan.md` を作成する際は、これらの確認済みのプラクティスを使う。より限定的な
記述がない場合（practices-discovery がスキップされた場合を含む）は、
アクティブなスペースの `memory/org.md` のデフォルトを使う。

### Step 2: Generate Clarifying Questions

このステージは Bolt の順序、つまり Units of Work を Construction でどの順に実行するかを計画する。2.7 は依存関係の DAG（トポロジー）を生成し、このステージ（2.9）はその中を通る経路を選ぶ。経済的な価値は DAG からは導けない — それは人間の価値判断である。

**このステージでの定義:**
- **Bolt** — `stage-protocol.md` の Glossary による: このステージ（2.9）で計画される Construction の提供単位であり、Definition of Done、確信度の仮説、オーナーシップを持つ 1 つ以上のユニットから成る。エンジンはユニットのグループ化や実行順序に `bolt-plan.md` を使わない。ランタイムのバッチは `unit-of-work-dependency.md` から決まる。**Batch** とは、同時に構築されるユニットのグループである（ランタイム。2.7 のその成果物から決まる）。

これらの定義は**あなた**のためのものである。読み上げるために書かれたものではなく、利用者は
それを見ていない。どの定義も、これから行う質問や書く成果物に現れようとしている
ものを指しているので、ある用語が初めて利用者に届くときは、別の用語集としてではなく、
それを使う文の中で、一節の定義を伴わせる。最も重要なのは "Bolt" である。
次のフェーズ全体の語彙だからである: 利用者向けに初めて言及するときは、Bolt に加えて
Bolt とは何か（作業の一部を 1 回通して構築し、動くものを作って終わる単位）を述べ、
以降の言及は単に "Bolt" とする。名前を挙げて提案するスコアリングモデルと、ウォーキング
スケルトンも同様に扱う。一節に縮めると定義が成り立たない用語は、
代わりに平易な言葉に置き換えるべき用語である。
- **確信度の仮説（Confidence hypothesis）** — その Bolt を出荷することで検証または反証される、観測可能な振る舞い（例: "latency stays under 200ms under 1k-rps load," "users complete signup without support tickets," "the event pipeline survives a 10x burst"）。
- **WSJF**（Reinertsen / SAFe）— Weighted Shortest Job First。順序スコア =（ユーザー／ビジネス価値 + 時間的緊急性 + リスク低減価値）÷ ジョブサイズ。スコアの高いものから出荷する。
- **ウォーキングスケルトン（Walking skeleton）**（Cockburn）— DAG の最初のユニットが、関連する統合ポイントを通る、最小限の動作するエンドツーエンドのスライスを提供する。そのユニットの該当する設計ステージと Code Generation は、後続のユニットより先に完了する。実際の統合チェックと人間によるチェックポイントの承認によって、その結果を実証する。

質問を含む `<record>/inception/delivery-planning/delivery-planning-questions.md` を作成する。戦略的な質問（プロジェクトごとに 1 つの回答）:

- 最初に何を作るべきか: 最もリスクの高い部分、最も価値の高い部分、全体がつながることを証明する薄いエンドツーエンドのスライス、またはその組み合わせ？ 組み合わせの場合は、どの方針をどこに適用するかを示してほしい。
- 正式なモデル（WSJF 型: 価値と緊急度をサイズと比較する）で作業をスコア付けして順位付けすべきか？ そうする場合、リスク、価値、サイズにそれぞれどれだけの重みを置くか？
- 1 つの Bolt の大きさはどうするか: 単一の Units of Work、関連する複数のユニットをまとめたもの、またはユニットを横断する薄いスライス？
- 複数の Bolt を同時に構築できるか、それとも 1 つずつ順に進める必要があるか？
- このチームの外にあるもので、作業を止める可能性があるものはあるか（API、データ、承認、他チームからの引き渡し）？ それぞれについて、誰がオーナーか、どれくらい時間がかかるか、どの Bolt をブロックするか、遅れた場合にどうするかを記録する。
- この構築で最も心配なことは何か？ それに早めに取り組むために聞きたい。

Bolt ごとの質問（aidlc-delivery-agent が成果物の生成中にこれらを繰り返し、プラン内の Bolt ごとに 1 組の回答を得る）:

- この Bolt はどの Units of Work をまとめるか？
- この Bolt は薄いエンドツーエンドのスライス（ウォーキングスケルトン）か？ そうであれば、アーキテクチャのどの部分を実証するか？
- この Bolt が完了とみなされるには、何が成り立っている必要があるか？
- この Bolt を出荷することで、まだわかっていない何がわかるか？
- この Bolt を構築するのはどのモブの予定か？（1.5 が実行された場合は 1.5 のチームを参照する。1.5 が SKIP の場合 — mvp、classic — はすべての Bolt に aidlc-developer-agent を割り当てる。）誰がユニットを構築するかは、このステージの問いではない。

NOTE: Bolt の順序付けは経済的な根拠を記録するものであり、エンジンが使うのは
実際のユニットの DAG とイテレーションの選択である。スケルトンが有効な場合は、
解決された DAG の最初のユニットが最小限の動作する統合スライスであることを確認し、期待される
デモと、それをエンドツーエンドで証明する実際のプロジェクトのチェックを明示し、
その前提条件を明確にする。分解がそのスライスを支えられない場合は、
Construction の前に Units Generation に立ち返る。`bolt-plan.md` だけを並べ替えても、
エンジンが最初に構築するユニットは変わらない。最初の設計ステージの
レビューを、出荷済みのスケルトンとして説明してはならない。

NOTE: このステージは Bolt の順序を計画する。どの AIDLC ステージをどの深度で実行するかは決めない — それは `/aidlc` スキルのスコープ選択が扱う。

stage-protocol.md の質問フローに従う。

### Step 3: Collect and Analyze Answers

選んだ Bolt の順序が 2.7 の依存関係の DAG を守っていることを（aidlc-architect-agent の意見を得て）検証する。トポロジカル順序からの逸脱があれば指摘し、根拠の成果物でその正当性を説明できるようにする。

### Step 4: Generate Artifacts

`<record>/inception/delivery-planning/` に 4 つの成果物を作成する。これらは
利用者がゲートで開いて読む文書なので、質問が従うのと同じルールが
その中の散文にも適用される: 専門用語は、そのファイル内で初めて現れるときに一節の
定義を伴わせ、各ファイルは単独で読めるようにする（読者は
`bolt-plan.md` を読まずに `team-allocation.md` を開くかもしれない）。"Bolt"、
"mob"、"walking skeleton"、"Program Board"、そして頭字語で呼ぶスコアリングモデルは
すべて該当する。短く説明して先に進む。説明を中心に成果物を
組み立て直さない。

- `bolt-plan.md` — Bolt の順序付きの並び。各 Bolt の項目: 含まれるユニット(s) of Work、該当する場合はウォーキングスケルトンの印、その Bolt の Definition of Done、確信度の仮説（"what will shipping this Bolt prove?"）、期待されるデモ。
- `team-allocation.md` — 計画した Bolt とモブの割り当て。1.5 が実行された場合（enterprise、feature）は 1.5 のチームを参照する。1.5 が SKIP の場合（mvp、classic）は、すべての Bolt に aidlc-developer-agent（AI）を割り当てる。チームが 2 つ以上の場合、これは Program Board に相当する。
- `risk-and-sequencing-rationale.md` — Bolt の順序の理由: WSJF 型のスコアリング、リスク優先の論拠、ウォーキングスケルトン優先の論拠、または価値優先の論拠。使用したヒューリスティクス（Cohn、Reinertsen CD3、SAFe WSJF）を示す。
- `external-dependency-map.md` — ゲートとなる項目（外部 API、データが利用可能になる時期、承認のリードタイム、外部チームからの引き渡し）を、それを使う Bolt に対応付けたもの。完全に AI 内で完結する場合は、簡素または空でよい。

### Step 5: Phase Boundary Verification

Inception → Construction の完全性の監査を実行する。実行された Inception ステージが生成した
すべての `traceability.json` を読む:

- `<record>/inception/user-stories/traceability.json`
- `<record>/inception/domain-design/traceability.json`
- `<record>/inception/units-generation/traceability.json`

（Contract Design は `traceability.json` を生成しない — 担うのは正式な契約であり、
要件のカバレッジではない — ため、このフェーズ境界の
チェックには寄与しない。）`GAP`、`ORPHAN`、無効な
ターゲット、欠落した上流 ID を含め、未解決の指摘がないことを確認する。表を
`<record>/verification/phase-check-inception.md` に統合し、先頭に合否の判定を
記す。指摘が残っている場合は、遷移を止め、Construction の開始前に
その指摘を担うステージに立ち返る。

### Step 6: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage delivery-planning --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

**Construction のイテレーション。** 順序を推奨する前に、記録済みの選択を読む。
新しいワークフローは `Construction Checkpoints: enabled` と
`Construction Iteration: unit-major`、`Construction Execution: serial` で始まる:
各ユニットの該当する設計ステージと Code Generation が直列に実行され、
その後に検証済みの完了チェックポイントが続く。
明示的な stage-major の選択は引き続き有効である。後の Code Generation の
バッチを有効にするには、人間の実行方式の選択を得て、イテレーションを stage-major に設定し、その後
`{{INVOKE}} engine state set-construction-execution swarm` を実行する。これは
ゲート付きと自律的のどちらの完了承認でも機能する。自律性の回答が
実行順序を変えることは決してない。unit-major は直列のままであり、矛盾する swarm の
設定を拒否する。unit-major に戻る前に serial を選択する。チェックポイントが有効な作業では、
スケルトンが有効な場合、どちらのイテレーション順序でも、DAG の最初のユニットの統合スライス全体を、
後続のユニットより先に必ず完了させる。

既存の明示的な選択は維持する。利用者が順序の変更を求めた場合
（"let's do each stage for both units together first"）は、そのターンで
`{{INVOKE}} engine state set-construction-iteration <unit-major|stage-major>` を実行し、
その `notice` 行を一字一句そのまま伝える。利用者に確認を求めない。
レガシーなワークフローを黙って移行しない: チェックポイントのフィールドがない場合、
以前の最初のステージのレビューが維持され、後のステージの承認は 1 つの質問としてまとめて届く。
チーム所有の作業は、
独自のステージごと、またはユニット終了時の `unit_gate` ポリシーを維持する。Plan Approval、サマリー
確認、検証コマンドの選択は、どちらの順序と自律性の選択の下でも、
人間の判断のままである。

**Construction の検証コマンド。** チェックポイントが有効な作業では、人間が承認した
既存のコマンドを維持する。そうでなければ、イテレーション／実行方式の設定とあわせて、
プロジェクトのスキャンから実際のプロジェクトのチェック（`bun test`、`pytest`、`make check`、またはプロジェクトでの同等のもの）
を提案する。このインテント単位のコマンドは、
すべてのユニット／バッチのチェックポイントで再利用される。完了したユニットの動作結果をチェックし、
スケルトンが有効な場合は統合スライスをエンドツーエンドで実証しなければならない。プロジェクトの
テストルールが要求するすべてのテストレベル（Code Generation がメモリの Testing Posture から
生成する Testing Contract: 該当に応じて unit、integration、
end-to-end）とカバレッジのチェックを実行しなければならない。必要なレベルが
このマシンで実行できない場合（コンテナランタイムがない、サービスが動いていない）は、
質問の中でそれを伝える: Approve の選択肢の説明に、コマンドが実行しないテストと
その理由を加える。例: "Runs typecheck and unit tests. Does not run the
integration tests: they need Docker, which is not installed." 欠けているランタイムを避けるために、
そう伝えずにより狭いコマンドを選んではならない。また、プランが CI Pipeline をスキップしている場合や
リポジトリに CI の設定がない場合に、その欠落に "CI runs them" と答えてはならない。実行可能なチェックが
まだ存在しない場合（グリーンフィールド）は、人間が選択を先送りしてよい。その場合はフィールドを
未設定のままにし、最初のチェックポイントで検証の前に尋ねることを説明する。
プレースホルダを作り上げたり、先送りを承認とみなしたりしてはならない。

1 つのセッションで同時に開いていてよい保護された質問は 1 つだけである。保護された質問は
1 つずつ尋ね、他の何よりも先に回答を待つ。

前後の空白を取り除いた後で、空白でない最大 1024 文字の 1 行を使う。
ツールは制御文字（改行、CR、タブ、
NUL を含む）と表示を偽装する文字を拒否する: Unicode の書式文字（ゼロ幅文字や
双方向制御文字を含む）、行／段落区切り文字、ノーブレークスペース
（U+00A0）。空白を取り除いたコマンドは、変更されずに記録、ハッシュ化、実行される。
複数行のチェックはスクリプトにまとめ、その呼び出しを記録する。コマンドを提示する前に、
ハーネスのファイル書き込みツール（Write/edit）を使って、それを UTF-8 テキストとして
`<record>/verification-command.txt` に書き込む。シェルの `echo` やヒアドキュメントは決して使わない。リポジトリ由来の
コマンドテキストを、シェルの行に決して埋め込んではならない: シェルの置換が
人間の承認前に実行されるおそれがある。渡すのは下記の記録相対のファイルパスだけである。
コマンドは自分が実行されるセッションを自分で見つけるので、セッションを渡したり
調べたりしてはならない:

```bash
{{INVOKE}} engine log decision --stage "<directive.stage>" --checkpoint verification-command --command-file verification-command.txt --decision "Use this command to verify each completed Unit?" --options "Approve,Request Changes"
```

`log decision` ツールの JSON 出力の `command` フィールドから、正規のコマンド全体を
正確にコピーして、構造化された質問のコードスパンに入れる。決して
省略したり、要約、接頭辞、ダイジェストで代用したりしない。コマンド内のバッククォートを保持できるだけの
長さのコードスパン区切りを使う。人間は
`<record>/verification-command.txt` を開くこともできる。人間の回答を待つ:

```question
prompt: "Use this command to verify each completed Unit? `<full command>`"
header: Verification
multiSelect: false
options:
  - label: Approve
    description: Record this command for all Unit and batch checkpoints in this intent.
  - label: Request Changes
    description: Propose a different project check before running verification.
```

利用者の返信を読み、利用者が行った選択を記録する。
人間のターンのフックが、利用者がこの質問に返信したことと、その正確な言葉を保持する。
新しいチャットで入力された返信も、それが最新の質問である間はこの質問への返信として保持され、
別の質問への返信は数に入らない。利用者が
承認したら、その回答を記録し、対応するツール所有の
レシートを使ってコマンドを設定する:

```bash
{{INVOKE}} engine log answer --stage "<directive.stage>" --checkpoint verification-command --command-file verification-command.txt --details "Approve"
{{INVOKE}} engine state set-construction-verification-command --command-file verification-command.txt
```

**Request Changes** の場合は、同じ `log answer` を
`--details "Request Changes"` で記録し、状態は変更せずに、別の
コマンドを提案する。自動承認したり、レシートなしで状態フィールドを書き込んだり、
汎用の `state set` を使ったりしてはならない。後で変更するには、新たな人間の判断／回答のレシートと
型付きの設定コマンドが必要である。自律性の付与は、コマンドの選択を許可するものではない。

### Step 7: Present Completion & Request Approval

完了時の絵文字: :calendar:
レビュー対象のパス: `<record>/inception/delivery-planning/`
承認ゲート: Approve（エンジンが指定するステージ `<directive.next_stage>` に進む）/ Request Changes。

## Sensors

このステージの出力は `<record>/inception/delivery-planning/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `requirements`, `stories`, `mockups`, `components`, `unit-of-work`, `unit-of-work-dependency`, `unit-of-work-story-map`, `contract-summary`, `team-practices`.

## Learn

`directive.protocol_modules` に `learnings` が挙げられている場合は、
`stage-protocol-learnings.md` に従う: 作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを行う。そのモジュールが、
利用者にいつ何を尋ねるかを定めている。モジュールがない場合は、日誌と
そのステップの両方をスキップする。
