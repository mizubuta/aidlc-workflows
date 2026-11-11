# Ensemble Protocol Module

`directive.mode` が `subagent`、`pipeline`、`mob` のいずれかの場合、またはステージがサポートエージェントを宣言している場合に、このモジュールを読み込む。ハーネス別のサブセクションは、アクティブなハーネスに一致するものだけを使う。

## 5. Multi-agent stages (ensemble topologies)

一部のステージは複数のエージェントを使う（例: Feasibility は aidlc-architect-agent + aidlc-aws-platform-agent + aidlc-compliance-agent を使う）。サポートエージェントがどのように参加するかは、ディレクティブの `mode`（ステージのコミュニケーショントポロジー）によって決まり、`support_agents` が存在するというだけで決まることは決してない。役割はトポロジーを問わず一定である: **リードエージェント**がステージの `produces[]` 成果物を所有し、**サポートエージェント**は自分の作業を自分で書く実際の参加者として協働し、`reviewer`（stage-protocol-reviewer.md §12a。宣言されている場合）が事後に外部から検証する。どのトポロジーでもオーケストレーターがバスである: 参加者間のやり取りはすべて、オーケストレーターが行うディスパッチと、オーケストレーターが運ぶ戻り値である。エージェント同士が互いを呼び出すことはない（NOT）。委任するのはオーケストレーターだけである。

**ディレクティブの `support_agents` が正である: 正確にそれらのエージェントだけをディスパッチし、それ以外は決してディスパッチしない。** これは今回（THIS）の実行における有効なリストである: エンジンはすでに collaborators スイッチ（スコープが所有する `collaborators` セレモニー。`stage-protocol.md` を参照）を適用しているため、作成時の frontmatter でサポートエージェントを挙げているステージでも、`support_agents` が空で届くことがある。**空のリストは、どのトポロジーでもリードが単独で実行することを意味する。** その場合リードは単独で作業する: ステージの本文全体を自分で行い、`produces[]` 成果物を完成させる。`pipeline` ではリードが唯一のリンクとなり、本来チェーンが分担するスキャンと統合の両方を行う。`subagent`/`mob` では、ディスパッチや統合を行うスポークのラウンドがないだけである。単独で作業する場合、コントリビューションファイルは書かず、自分のもの以外に記録されたリンクも必要ない。`directive.support_agents` にないエージェントを決してディスパッチせず、ステージ本文の散文や作成時の frontmatter から協働者を推測してはならない。このルールが、誰が参加するかについての唯一の信頼できる情報源である。ステージ本文は全員がそろった場合を記述しており、このルールに従う。

**アンサンブルの実行中にユーザーが耳にすること。** これらの引き継ぎはステージの内部で起こり、ディレクティブの `narration` の届かないところにあるため、話す文言をここに記す。話すのは二重引用符で囲まれたテキストだけである。`[bracketed]` のスロットを埋め、角括弧は外す。

- 特定の質問を 1 人の専門家に渡すとき - **SAY:** "Let me bring in the [trade] on [the specific question, in plain terms]."（[専門職] に [具体的な質問を平易な言葉で] について加わってもらいます。）
- 各専門家が前の専門家の成果の上に積み上げるチェーンを始めるとき - **SAY:** "The [first trade] takes a look first, then the [next trade] builds on what comes back."（まず [最初の専門職] が確認し、その結果を受けて [次の専門職] が作業を進めます。）
- 複数の専門家を一度に集めるとき - **SAY:** "Getting the [trade] and [trade] to weigh in on this together."（[専門職] と [専門職] にこの件について一緒に意見をもらいます。）
- 専門家の作業が戻ってきて、それを取り込んでいるとき - **SAY:** 何も言わない。統合は作業そのものであり、イベントではない。

伝えるのは職種名であり、エージェント名、ファイル、slug は決して使わない: product manager, product lead, designer, delivery lead, architect, architecture reviewer, platform engineer, compliance specialist, security engineer, developer, quality engineer, release engineer, operations engineer。引き継ぎの仕組み、ブリーフ、コンテキストパス、ルールバンドル、コントリビューションファイル、識別マーカー、参加者間の相互不可視性、ラウンド、ステージがどのトポロジーを宣言しているかについては何も言わない。ユーザーは同僚に会っているのであり、その段取りはこちらの事情であって、ユーザーの事情ではない。該当しないトポロジーについても決して言及しない。

- **`mode: inline`** — サポートエージェントは、オーケストレーターが自身のコンテキストで採用する視点である: `inline_context_paths` から各サポートエージェントのファイルとナレッジを読み込み（`stage-protocol.md` の「For inline stages」を参照）、まずリードの出力を作り、次に各サポートの視点を重ね、最後に統合する。inline ステージではサポートエージェントをディスパッチしてはならない（NOT）。ディスパッチは他のモード専用である。コントリビューションファイルはない。
- **`mode: subagent`** - ハブアンドスポーク。草案のためにリードをディスパッチする。ステージが `support_agents` を宣言している場合は、返ってきた草案に対して各サポートエージェントをディスパッチする（成果物は §11 のコンテキスト予算に従ってパスで渡し、ルールは `stage-protocol.md` §「For subagent stages」のステップ 2 に従う。スポークは相互に見えない状態にする - どのサポートエージェントのブリーフにも他のエージェントのコントリビューションを含めない）。各スポークは自分のコントリビューションファイルを書く。その後、もう一度リードをディスパッチして、コントリビューションを成果物に統合させる。
- **`mode: pipeline`**: チェーン。`directive.pipeline.links` は宣言されたリード→サポートの順序であり、`directive.pipeline.completed` は現在の試行の復旧台帳である。開始時または再開時には、完了済みのエントリをすべて飛ばし、欠けている最初（FIRST）のリンクをディスパッチする。各リンクが返ったら、次をディスパッチする前に、`directive.pipeline.record` にあるそのリンクのコマンドで記録する。各コマンドには、必要な引き継ぎパス、リポジトリ、`--single` がすでに含まれている。マルチリポジトリのステージでは、登録済みのリポジトリごとに独立したチェーンを 1 つ実行し、リポジトリで修飾された `directive.pipeline.completed` のエントリ（`<repo>:<agent>`）をそのチェーンの復旧状態として扱う。リポジトリが 9 つを超える場合、`completed` は最初の 9 つを挙げ、`directive.pipeline.completed_other_repos` はリンクが記録済みのその他のリポジトリの数を示す。`directive.pipeline.record` からディスパッチすること。その最初のエントリが、欠けている最初のリンクである。現在の試行のリポジトリ単位の再利用レシートがあれば、そのリポジトリのチェーン全体がディスパッチなしで完了済みとなる。各リンクは上流のすべてを見て、作業成果物を直接前進させる: ステージ本文に従い、進化中の成果物をその場で編集する（逐次実行なので衝突しない）か、次のリンクが積み上げるためのコンテキストとして結果を渡す。最後のリンクが `produces[]` 成果物を完成させる。順序が肝心である。コントリビューションファイルは不要である。
- **`mode: mob`** - メッシュ。サポートエージェントとの境界付きラウンドとして実行する。ディスパッチするサポートエージェントがいる場合、エンジンは collaborators モジュール（`stage-protocol-collaborators.md`）を指定し、そのモジュールがラウンド、コントリビューションファイル、異議の進め方を定める。いない場合は、リードが単独で実行する。

どのトポロジーでも、レビュアーの NOT-READY（stage-protocol-reviewer.md §12a のステップ 3）は、指摘事項とともにリードだけ（LEAD alone）を再起動する — アンサンブルが招集されるのは 1 回だけであり、修正ループはリードとレビュアーの往復である。

**完了の証拠（決定論的）。** `mob` またはサポート付きの `subagent` のステージでは、コントリビューションファイルが、エンジンがチェックする決定論的かつ構造的な完了の証拠である: ディレクティブの有効な `support_agents` のいずれかのサポートエージェントについて、コントリビューションファイルが欠けているか、そのファイルの 1 行目に識別マーカーがない間、エンジンはゲートへの進入と完了を拒否する。単独で作業するリード（collaborators スイッチがオフで、有効なリストが空）は、コントリビューションの証拠を負わず、同様に自分のもの以外に記録されたリンクも必要ない。`pipeline` では、現在の試行の `PIPELINE_LINK_COMPLETED` レシートが証拠である: スキャンされた各リポジトリには宣言されたすべてのリンクが必要だが、`Decision: keep` を持つ現在の試行のリポジトリ単位の `ARTIFACT_REUSED` 行があれば、その再利用されたリポジトリは免除される。分離された再利用行とリンクのレシートは同じ `single-stage:<slug>` のワークフロー識別子を持ち、メインのワークフローを満たすことは決してなく、正規の CodeKB 成果物一式が権威ある通常ファイルのパスの下に完全に存在し、そのソーススコープが `CURRENT` のままである間だけ受け入れられる。ジャンプや後続ステージの開始はメインワークフローの証拠をリセットし、Guard Policy strict での却下も同様である（relaxed と off では却下された試行のレシートが保持される）。成果物ファイルだけでは pipeline の証拠を満たさない。共通の非常口は `AIDLC_DISABLE_ENSEMBLE_EVIDENCE=1` であり、正当に実行されたステージの証拠がアップグレードや中断の間に失われた場合の復旧にのみ使う。

---

## 11. Subagent Return Summary

サブエージェントは作業を完了したら、必ず（MUST）構造化されたサマリーをオーケストレーターに返す。これにより、サブエージェントの実行とオーケストレーターの継続の間でコンテキストが失われないようにする。

そのサマリーが存在する前にディスパッチが返った場合（エージェントをバックグラウンドで実行するハーネス）、シェルのループや sleep を書いてはならない。サポートエージェントをディスパッチした後は `{{INVOKE}} engine orchestrate wait --stage <directive.stage> --for collaborators` を、リードをディスパッチした後は `--for artifacts` を実行し（ユニット単位のステージでは `--unit <directive.unit>` を追加する）、`status: waiting` と応答する間は同じコマンドを再実行する。このコマンドは上限時間内に戻り、まだ欠けているものを正確に列挙する。

### Required return format:
```markdown
## Subagent Summary: [Stage Name]

### Produced
- [file path 1]: [brief description of content]
- [file path 2]: [brief description of content]

### Key Decisions
- [Decision 1]: [rationale]
- [Decision 2]: [rationale]

### Issues / Concerns
- [Any problems encountered, edge cases found, or risks identified]
- "None" if no issues

### Next Steps
- [What the orchestrator should do next based on this output]
```

### Rules:
- オーケストレーターは、次のステージに進む前に必ず（MUST）このサマリーを読む
- "Issues / Concerns" セクションが空でない場合、オーケストレーターは続行する前に必ず（MUST）それらをユーザーに提示する
- "Produced" セクションに挙げられたファイルがそのステージで期待される数より少ない場合、オーケストレーターはステージを完了とマークする前に必ず（MUST）調査する
- 実質的な引き継ぎはファイルである。返却サマリーには、パス、
  決定事項、懸念事項、次のアクションだけを記す。すでにディスク上にある成果物、
  コントリビューション、スキャン、ソース、テスト出力の本文は繰り返さない。

### Context budget for subagent prompts
サブエージェント呼び出しでのコンテキストのあふれを防ぐために:
- **現在のユニットのみ**: 実装中のユニットの設計成果物だけを渡し、すべてのユニットのものは渡さない
- **inception の成果物は要約する**: CONSTRUCTION のサブエージェントには、全文を埋め込むのではなく、各 inception 成果物について 1〜2 行の要約とファイルパスを渡す。サブエージェントは必要に応じて特定のファイルを Read できる。
- **常に含めるもの**: 具体的なタスク指示と、関連する状態/成果物のパス。ペルソナとナレッジのコンテキストはハーネスのエージェント設定が読み込むので、どちらもプロンプトに貼り付けない。
- **大きなナレッジセット**: 特に関連するファイルパスはブリーフで名前を挙げてよいが、読み込みはディスパッチされたエージェントに、設定済みのリソースを通じて任せる。
- **ファイルとコマンド**: すべてのサブエージェントのプロンプトには、次の文を一字一句そのまま含める: Write and edit files yourself with your file tools, never through the shell (no heredoc, no `echo`, `printf`, or `python3` writing a file, no `sed -i`, no `mkdir`; the file-write tool creates any missing folder). A command the person asks for, or one the plan names (a package install, a build, a scaffolder, a migration, a formatter, a code generator, even a `mkdir`), still runs as written. Do not stash, reset, check out over, restore or clean the person's working tree, and do not install what the plan does not name: for a before-and-after comparison, run the check on the unchanged tree before you edit. Every file you make on your own, a scratch file, a helper script, a command's output, stays inside the project (nothing in /tmp or any folder outside it), and a command's output is read from the tool result, never sent to a file; a file the person asks for, or one the plan names, goes where they say. Quote a pattern meant for the program, not the shell (`--include='*.ts'`), and never start an argument with `=`: zsh, the macOS default shell, stops on both. Read, list, and search (your own knowledge files included) with your file tools where you have them; where the shell is your only way to read, use one plain read command (no `cd` before it, no pipe or second command after it). Run every AI-DLC command exactly as written, as a command of its own (no `cd` before it, no pipe or second command after it), keeping its path as written (never a full path): a shell line can stop and ask the person to approve it.

### Subagent failure recovery
Task ツールの呼び出しが失敗した場合（タイムアウト、エラー、または切り詰められた/不完全な出力が返った場合）:
1. **1 回だけ再試行する**。コンテキストを減らしたプロンプトで行う — inception フェーズの成果物は全文を含めずに要約し、現在のユニットの設計成果物だけを渡す
2. 再試行も失敗した場合は、**何が失敗したかをユーザーに平易に伝え**、構造化された質問で 2 つの選択肢を提示する:
   - "Run it here": ステージの作業を引き継がずにこの会話の中で行う。時間はかかるが、失敗している原因を回避できる
   - "Skip and revisit": ステージを未完了のままにして先に進み、後で戻ってくる
3. 失敗と解決策を `{{INVOKE}} engine audit append-raw 'Error: <brief>' '<body>'` で記録する（セクション 4 の監査証跡のルール）

---

---

## Harness topology bindings

### Claude Code

**Pipeline receipt rule:** pipeline の `Task` が返るたびに、次のディスパッチの前に `directive.pipeline.record` にあるそのリンクのコマンドを実行し、`directive.pipeline.completed` から再開する。

`directive.mode` は本文をどのように（HOW）実行するかを示す。これはステージのコミュニケーショントポロジー（誰が誰と話すか。このモジュールの §5「Multi-agent stages」セクションが契約である）である。ディスパッチされるすべてのトポロジーでの書き込みモデル: 全員が自分の作業を自分で書く。ディスパッチされた各サポートエージェントはコントリビューションファイル（`<record>/<phase>/<stage>/contributions/<agent-slug>.md`。1 行目に識別マーカーを持つ §11 の形式）を書き、ステージの `produces[]` 成果物を編集するのはリードだけである。`inline`（このセッションで、ステップが `directive.lead_persona` で渡すリードエージェントのペルソナを使って実行する。サポートエージェントは自分が採用する声であり、コントリビューションファイルはない）、`subagent`（ハブアンドスポーク: 名前付きエージェントへの `Task` 呼び出しでリードを実行する。エージェントはペルソナを自動的に読み込むので、プロンプトに注入しない。ステージが `support_agents` を宣言している場合は、リードが返した草案に対して各サポートエージェントを `Task` でディスパッチする: 1 つのメッセージ内での並列呼び出しとし、ブリーフには成果物をパスで、ルールは `stage-protocol.md` §「For subagent stages」のステップ 2 に従って含め、相互に見えない状態にする: 各エージェントは自分のコントリビューションファイルを書き、その後リードの最後の `Task` がそれらを成果物に統合する）、`pipeline`（チェーン: リンクが共同で成果物を作成する。まずリードの `Task`、次に宣言順でサポートエージェントごとに 1 つの `Task`。各リンクは上流のすべてを見て、作業成果物を直接前進させる。最後のリンクが成果物を完成させる。コントリビューションファイルはない）、または `mob`（境界付きラウンドとして実行するメッシュ: リードが草案を作り、その後すべての（ALL）サポートエージェントを草案に対する並列の `Task` 呼び出しで実行し、それぞれがコントリビューションファイルを書く。リードとして統合し、未解決の異議を §5 に従って振り分ける: 判断の必要な事項はステージの途中で構造化された質問として人間（HUMAN）に回し、ナレッジに関する論争は異議を唱えた者だけで第 2 ラウンドに回す。維持された反対意見はゲートで一字一句そのまま引用する）。どのトポロジーでも自分がバスであり、レビュアーの NOT-READY はリードだけ（LEAD alone）を再起動する。コントリビューションファイルはアンサンブルの完了の証拠である: mob/サポート付き subagent のステージでは、1 つでも欠けている間、エンジンは承認を拒否する。出荷時のグラフは 29 inline / 2 subagent / 1 pipeline / 1 mob である: practices-discovery と code-generation は `subagent`、reverse-engineering は `pipeline`（developer がスキャンし、architect が統合して書く）、user-stories は `mob` を持つ。

---

### Kiro CLI

**Pipeline receipt rule:** pipeline の委任が返るたびに、次の委任の前に `directive.pipeline.record` にあるそのリンクのコマンドを実行し、`directive.pipeline.completed` から再開する。

`directive.mode` は本文をどのように（HOW）実行するかを示す。これはステージのコミュニケーショントポロジー（誰が誰と話すか。このモジュールの §5「Multi-agent stages」セクションが契約である）である。ディスパッチされるすべてのトポロジーでの書き込みモデル: 全員が自分の作業を自分で書く。ディスパッチされた各サポートエージェントはコントリビューションファイル（`<record>/<phase>/<stage>/contributions/<agent-slug>.md`。1 行目に識別マーカーを持つ §11 の形式）を書き、ステージの `produces[]` 成果物を編集するのはリードだけである。`inline`（このセッションで、ステップが `directive.lead_persona` で渡すリードエージェントのペルソナを使って実行する。サポートエージェントは自分が採用する声であり、コントリビューションファイルはない）、`subagent`（ハブアンドスポーク: `subagent` ツールでリードを名前付きのエージェント設定に委任する。エージェント設定は自身のペルソナを読み込むので、プロンプトに注入しない。ステージが `support_agents` を宣言している場合は、リードが返した草案に対して各サポートエージェントに委任する: 可能なら 1 回の委任の中で並列タスクとし、ブリーフには成果物をパスで、ルールは `stage-protocol.md` §「For subagent stages」のステップ 2 に従って含め、相互に見えない状態にする: 各エージェントは自分のコントリビューションファイルを書き、その後リードへの最後の委任がそれらを成果物に統合する）、`pipeline`（チェーン: リンクが共同で成果物を作成する。まずリードへの委任、次に宣言順でサポートエージェントごとに 1 回の委任。各リンクは上流のすべてを見て、作業成果物を直接前進させる。最後のリンクが成果物を完成させる。コントリビューションファイルはない）、または `mob`（境界付きラウンドとして実行するメッシュ: リードが草案を作り、その後すべての（ALL）サポートエージェントに草案に対して並列で委任し、それぞれがコントリビューションファイルを書く。リードとして統合し、未解決の異議を §5 に従って振り分ける: 判断の必要な事項はステージの途中で構造化された質問として人間（HUMAN）に回し、ナレッジに関する論争は異議を唱えた者だけで第 2 ラウンドに回す。維持された反対意見はゲートで一字一句そのまま引用する）。どのトポロジーでも自分がバスであり、レビュアーの NOT-READY はリードだけ（LEAD alone）を再起動する。コントリビューションファイルはアンサンブルの完了の証拠である: mob/サポート付き subagent のステージでは、1 つでも欠けている間、エンジンは承認を拒否する。出荷時のグラフは 29 inline / 2 subagent / 1 pipeline / 1 mob である: practices-discovery と code-generation は `subagent`、reverse-engineering は `pipeline`（developer がスキャンし、architect が統合して書く）、user-stories は `mob` を持つ。 委任先にはそれぞれ、`{{HARNESS_DIR}}/agents/` のエージェント設定と、コンダクターの設定内の `trustedAgents` エントリが必要である: 14 のペルソナはすべて両方を同梱している。

---

### Kiro IDE

**Pipeline receipt rule:** pipeline の委任が返るたびに、次の委任の前に `directive.pipeline.record` にあるそのリンクのコマンドを実行し、`directive.pipeline.completed` から再開する。

`directive.mode` は本文をどのように（HOW）実行するかを示す。これはステージのコミュニケーショントポロジー（誰が誰と話すか。このモジュールの §5「Multi-agent stages」セクションが契約である）である。ディスパッチされるすべてのトポロジーでの書き込みモデル: 全員が自分の作業を自分で書く。ディスパッチされた各サポートエージェントはコントリビューションファイル（`<record>/<phase>/<stage>/contributions/<agent-slug>.md`。1 行目に識別マーカーを持つ §11 の形式）を書き、ステージの `produces[]` 成果物を編集するのはリードだけである。`inline`（このセッションで、ステップが `directive.lead_persona` で渡すリードエージェントのペルソナを使って実行する。サポートエージェントは自分が採用する声であり、コントリビューションファイルはない）、`subagent`（ハブアンドスポーク: `subagent` ツールでリードを名前付きの Markdown エージェントに委任する。エージェントは自身のペルソナを読み込むので、プロンプトに注入しない。ステージが `support_agents` を宣言している場合は、リードが返した草案に対して各サポートエージェントに委任する: 可能なら 1 回の委任の中で並列タスクとし、ブリーフには成果物をパスで、ルールは `stage-protocol.md` §「For subagent stages」のステップ 2 に従って含め、相互に見えない状態にする: 各エージェントは自分のコントリビューションファイルを書き、その後リードへの最後の委任がそれらを成果物に統合する）、`pipeline`（チェーン: リンクが共同で成果物を作成する。まずリードへの委任、次に宣言順でサポートエージェントごとに 1 回の委任。各リンクは上流のすべてを見て、作業成果物を直接前進させる。最後のリンクが成果物を完成させる。コントリビューションファイルはない）、または `mob`（境界付きラウンドとして実行するメッシュ: リードが草案を作り、その後すべての（ALL）サポートエージェントに草案に対して並列で委任し、それぞれがコントリビューションファイルを書く。リードとして統合し、未解決の異議を §5 に従って振り分ける: 判断の必要な事項はステージの途中で構造化された質問として人間（HUMAN）に回し、ナレッジに関する論争は異議を唱えた者だけで第 2 ラウンドに回す。維持された反対意見はゲートで一字一句そのまま引用する）。どのトポロジーでも自分がバスであり、レビュアーの NOT-READY はリードだけ（LEAD alone）を再起動する。コントリビューションファイルはアンサンブルの完了の証拠である: mob/サポート付き subagent のステージでは、1 つでも欠けている間、エンジンは承認を拒否する。出荷時のグラフは 29 inline / 2 subagent / 1 pipeline / 1 mob である: practices-discovery と code-generation は `subagent`、reverse-engineering は `pipeline`（developer がスキャンし、architect が統合して書く）、user-stories は `mob` を持つ。 Kiro IDE は 14 のペルソナすべてを `{{HARNESS_DIR}}/agents/aidlc-*-agent.md` から直接解決する。各ファイルは空でない `tools:` の付与と、能力単位でスコープされた `permissions.rules` を持ち、agent-v1 JSON やコンダクターの `trustedAgents` エントリはない。

---

### Codex CLI

**Pipeline receipt rule:** pipeline の起動（spawn）が返るたびに、次の起動（spawn）の前に `directive.pipeline.record` にあるそのリンクのコマンドを実行し、`directive.pipeline.completed` から再開する。

`directive.mode` は本文をどのように（HOW）実行するかを示す。これはステージのコミュニケーショントポロジー（誰が誰と話すか。このモジュールの §5「Multi-agent stages」セクションが契約である）である。ディスパッチされるすべてのトポロジーでの書き込みモデル: 全員が自分の作業を自分で書く。ディスパッチされた各サポートエージェントはコントリビューションファイル（`<record>/<phase>/<stage>/contributions/<agent-slug>.md`。1 行目に識別マーカーを持つ §11 の形式）を書き、ステージの `produces[]` 成果物を編集するのはリードだけである。`inline`（このセッションで、ステップが `directive.lead_persona` で渡すリードエージェントのペルソナを使って実行する。サポートエージェントは自分が採用する声であり、コントリビューションファイルはない）、`subagent`（ハブアンドスポーク: リードのエージェントロールを起動（spawn）する: ハーネスが `{{HARNESS_DIR}}/agents/aidlc-<role>-agent.toml` を解決し、それが `developer_instructions` を介して自身のペルソナを読み込むので、プロンプトに注入しない。ステージが `support_agents` を宣言している場合は、リードが返した草案に対して各サポートエージェントを起動する: このハーネスでは逐次でよい。ブリーフには成果物をパスで、ルールは `stage-protocol.md` §「For subagent stages」のステップ 2 に従って含め、相互に見えない状態にする: どのスポークのブリーフにも他のスポークのコントリビューションを含めない: 各エージェントは自分のコントリビューションファイルを書き、その後リードの最後の起動がそれらを成果物に統合する）、`pipeline`（チェーン: リンクが共同で成果物を作成する。まずリードの起動、次に宣言順でサポートエージェントごとに 1 回の起動。各リンクは上流のすべてを見て、作業成果物を直接前進させる。最後のリンクが成果物を完成させる。コントリビューションファイルはない）、または `mob`（境界付きラウンドとして実行するメッシュ: リードが草案を作り、その後各サポートエージェントを草案に対して起動する（ブリーフに兄弟のコントリビューションを決して含めないため、逐次の起動でも相互不可視の契約は保たれる）。それぞれがコントリビューションファイルを書く。リードとして統合し、未解決の異議を §5 に従って振り分ける: 判断の必要な事項はステージの途中で構造化された質問として人間（HUMAN）に回し、ナレッジに関する論争は異議を唱えた者だけで第 2 ラウンドに回す。維持された反対意見はゲートで一字一句そのまま引用する）。どのトポロジーでも自分がバスであり、レビュアーの NOT-READY はリードだけ（LEAD alone）を再起動する。コントリビューションファイルはアンサンブルの完了の証拠である: mob/サポート付き subagent のステージでは、1 つでも欠けている間、エンジンは承認を拒否する。出荷時のグラフは 29 inline / 2 subagent / 1 pipeline / 1 mob である: practices-discovery と code-generation は `subagent`、reverse-engineering は `pipeline`（developer がスキャンし、architect が統合して書く）、user-stories は `mob` を持つ。

---

### Cursor

**Pipeline receipt rule:** pipeline のタスクが返るたびに、次のタスクの前に `directive.pipeline.record` にあるそのリンクのコマンドを実行し、`directive.pipeline.completed` から再開する。

`directive.mode` は本文をどのように（HOW）実行するかを示す。これはステージのコミュニケーショントポロジー（誰が誰と話すか。このモジュールの §5「Multi-agent stages」セクションが契約である）である。ディスパッチされるすべてのトポロジーでの書き込みモデル: 全員が自分の作業を自分で書く。ディスパッチされた各サポートエージェントはコントリビューションファイル（`<record>/<phase>/<stage>/contributions/<agent-slug>.md`。1 行目に識別マーカーを持つ §11 の形式）を書き、ステージの `produces[]` 成果物を編集するのはリードだけである。`inline`（このセッションで、ステップが `directive.lead_persona` で渡すリードエージェントのペルソナを使って実行する。サポートエージェントは自分が採用する声であり、コントリビューションファイルはない）、`subagent`（ハブアンドスポーク: `task` ツールでリードを名前付きエージェントに委任する。エージェントはペルソナを自動的に読み込むので、タスクに注入しない。ステージが `support_agents` を宣言している場合は、リードが返した草案に対して各サポートエージェントを `task` でディスパッチする: 1 ターン内での並列タスクとし、ブリーフには成果物をパスで、ルールは `stage-protocol.md` §「For subagent stages」のステップ 2 に従って含め、相互に見えない状態にする: 各エージェントは自分のコントリビューションファイルを書き、その後リードの最後のタスクがそれらを成果物に統合する）、`pipeline`（チェーン: リンクが共同で成果物を作成する。まずリードのタスク、次に宣言順でサポートエージェントごとに 1 つのタスク。各リンクは上流のすべてを見て、作業成果物を直接前進させる。最後のリンクが成果物を完成させる。コントリビューションファイルはない）、または `mob`（境界付きラウンドとして実行するメッシュ: リードが草案を作り、その後すべての（ALL）サポートエージェントを草案に対する並列タスクで実行し、それぞれがコントリビューションファイルを書く。リードとして統合し、未解決の異議を §5 に従って振り分ける: 判断の必要な事項はステージの途中で構造化された質問として人間（HUMAN）に回し、ナレッジに関する論争は異議を唱えた者だけで第 2 ラウンドに回す。維持された反対意見はゲートで一字一句そのまま引用する）。どのトポロジーでも自分がバスであり、レビュアーの NOT-READY はリードだけ（LEAD alone）を再起動する。コントリビューションファイルはアンサンブルの完了の証拠である: mob/サポート付き subagent のステージでは、1 つでも欠けている間、エンジンは承認を拒否する。出荷時のグラフは 29 inline / 2 subagent / 1 pipeline / 1 mob である: practices-discovery と code-generation は `subagent`、reverse-engineering は `pipeline`（developer がスキャンし、architect が統合して書く）、user-stories は `mob` を持つ。

---

### opencode

**Pipeline receipt rule:** pipeline のタスクが返るたびに、次のタスクの前に `directive.pipeline.record` にあるそのリンクのコマンドを実行し、`directive.pipeline.completed` から再開する。

`directive.mode` は本文をどのように（HOW）実行するかを示す。これはステージのコミュニケーショントポロジー（誰が誰と話すか。このモジュールの §5「Multi-agent stages」セクションが契約である）である。ディスパッチされるすべてのトポロジーでの書き込みモデル: 全員が自分の作業を自分で書く。ディスパッチされた各サポートエージェントはコントリビューションファイル（`<record>/<phase>/<stage>/contributions/<agent-slug>.md`。1 行目に識別マーカーを持つ §11 の形式）を書き、ステージの `produces[]` 成果物を編集するのはリードだけである。`inline`（このセッションで、ステップが `directive.lead_persona` で渡すリードエージェントのペルソナを使って実行する。サポートエージェントは自分が採用する声であり、コントリビューションファイルはない）、`subagent`（ハブアンドスポーク: `task` ツールでリードを名前付きエージェントに委任する。エージェントはペルソナを自動的に読み込むので、タスクに注入しない。ステージが `support_agents` を宣言している場合は、リードが返した草案に対して各サポートエージェントを `task` でディスパッチする: 1 ターン内での並列タスクとし、ブリーフには成果物をパスで、ルールは `stage-protocol.md` §「For subagent stages」のステップ 2 に従って含め、相互に見えない状態にする: 各エージェントは自分のコントリビューションファイルを書き、その後リードの最後のタスクがそれらを成果物に統合する）、`pipeline`（チェーン: リンクが共同で成果物を作成する。まずリードのタスク、次に宣言順でサポートエージェントごとに 1 つのタスク。各リンクは上流のすべてを見て、作業成果物を直接前進させる。最後のリンクが成果物を完成させる。コントリビューションファイルはない）、または `mob`（境界付きラウンドとして実行するメッシュ: リードが草案を作り、その後すべての（ALL）サポートエージェントを草案に対する並列タスクで実行し、それぞれがコントリビューションファイルを書く。リードとして統合し、未解決の異議を §5 に従って振り分ける: 判断の必要な事項はステージの途中で構造化された質問として人間（HUMAN）に回し、ナレッジに関する論争は異議を唱えた者だけで第 2 ラウンドに回す。維持された反対意見はゲートで一字一句そのまま引用する）。どのトポロジーでも自分がバスであり、レビュアーの NOT-READY はリードだけ（LEAD alone）を再起動する。コントリビューションファイルはアンサンブルの完了の証拠である: mob/サポート付き subagent のステージでは、1 つでも欠けている間、エンジンは承認を拒否する。出荷時のグラフは 29 inline / 2 subagent / 1 pipeline / 1 mob である: practices-discovery と code-generation は `subagent`、reverse-engineering は `pipeline`（developer がスキャンし、architect が統合して書く）、user-stories は `mob` を持つ。

---

### GitHub Copilot

**Pipeline receipt rule:** pipeline の委任が返るたびに、次の委任の前に `directive.pipeline.record` にあるそのリンクのコマンドを実行し、`directive.pipeline.completed` から再開する。

`directive.mode` は本文をどのように（HOW）実行するかを示す。これはステージのコミュニケーショントポロジー（誰が誰と話すか。このモジュールの §5「Multi-agent stages」セクションが契約である）である。ディスパッチされるすべてのトポロジーでの書き込みモデル: 全員が自分の作業を自分で書く。ディスパッチされた各サポートエージェントはコントリビューションファイル（`<record>/<phase>/<stage>/contributions/<agent-slug>.md`。1 行目に識別マーカーを持つ §11 の形式）を書き、ステージの `produces[]` 成果物を編集するのはリードだけである。`inline`（このセッションで、ステップが `directive.lead_persona` で渡すリードエージェントのペルソナを使って実行する。サポートエージェントは自分が採用する声であり、コントリビューションファイルはない）、`subagent`（ハブアンドスポーク: リードを名前付きのカスタムエージェントに委任する。エージェントはペルソナを自動的に読み込むので、ブリーフに注入しない。ステージが `support_agents` を宣言している場合は、リードが返した草案に対して各サポートエージェントをディスパッチする: 1 ターン内での並列の委任とし、ブリーフには成果物をパスで、ルールは `stage-protocol.md` §「For subagent stages」のステップ 2 に従って含め、相互に見えない状態にする: 各エージェントは自分のコントリビューションファイルを書き、その後リードへの最後の委任がそれらを成果物に統合する）、`pipeline`（チェーン: リンクが共同で成果物を作成する。まずリードへの委任、次に宣言順でサポートエージェントごとに 1 回の委任。各リンクは上流のすべてを見て、作業成果物を直接前進させる。最後のリンクが成果物を完成させる。コントリビューションファイルはない）、または `mob`（境界付きラウンドとして実行するメッシュ: リードが草案を作り、その後すべての（ALL）サポートエージェントを草案に対する並列の委任で実行し、それぞれがコントリビューションファイルを書く。リードとして統合し、未解決の異議を §5 に従って振り分ける: 判断の必要な事項はステージの途中で構造化された質問として人間（HUMAN）に回し、ナレッジに関する論争は異議を唱えた者だけで第 2 ラウンドに回す。維持された反対意見はゲートで一字一句そのまま引用する）。どのトポロジーでも自分がバスであり、レビュアーの NOT-READY はリードだけ（LEAD alone）を再起動する。コントリビューションファイルはアンサンブルの完了の証拠である: mob/サポート付き subagent のステージでは、1 つでも欠けている間、エンジンは承認を拒否する。出荷時のグラフは 29 inline / 2 subagent / 1 pipeline / 1 mob である: practices-discovery と code-generation は `subagent`、reverse-engineering は `pipeline`（developer がスキャンし、architect が統合して書く）、user-stories は `mob` を持つ。
