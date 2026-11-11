{{SLOT:frontmatter}}
{{SLOT:title_block}}

## Prerequisites

{{SLOT:prereq_bullets}}
- **ロック**: 監査ログファイルのロックは、システムの一時ディレクトリでの mkdir ベースのロックにより、移植性のある方法で処理される（外部依存なし）。
- **フックの権限**: 17 個のフックはすべて TypeScript（`.ts`）で、`bun` を介して実行される。実行ビットは不要であり、macOS、Linux、ネイティブの Windows PowerShell で同じように動作する。
{{SLOT:hook_permissions_note}}
{{SLOT:prereq_bullets_tail}}

## AI-DLC Structure

- **スキル**: `{{HARNESS_DIR}}/skills/aidlc/` — オーケストレーター（`SKILL.md`）、ステージプロトコル、各フェーズディレクトリにまたがるステージファイル（有効なセットは合成されたプラグインによって決まる: コンパイル済みの `{{HARNESS_DIR}}/tools/data/stage-graph.json` を参照するか、`{{INVOKE}} --doctor` を実行する）
- **ドキュメントスキル**（ユーザーが呼び出し可能）: `{{HARNESS_DIR}}/skills/aidlc-knowledge/`。`{{SKILL_INVOKE}}-knowledge` と入力して使う。フレームワークの CLI も `{{INVOKE}} engine knowledge <verb>` を公開している。これもライフサイクルグラフの外にある独立したスキルだが、以下の読み取り専用のセッションスキルとは異なり `read-write` に分類される: ドキュメントカタログを変更し、ドキュメントの監査イベントを出力する。ワークフローのステージポインターを進めることも、ゲートを承認することも決してない。プロジェクトのルート `AGENTS.md` の "Where things live" の下にある "Document knowledge" を参照すること（Claude Code と Copilot の読者は、このファイルの "Shared AI-DLC onboarding" の下にある）。
- **セッションスキル**（読み取り専用、ユーザーが呼び出し可能）: `{{HARNESS_DIR}}/skills/aidlc-session-cost/`、`{{HARNESS_DIR}}/skills/aidlc-replay/`、`{{HARNESS_DIR}}/skills/aidlc-outcomes-pack/` — `{{SKILL_INVOKE}}-session-cost`、`{{SKILL_INVOKE}}-replay`、`{{SKILL_INVOKE}}-outcomes-pack` と入力して使う。いずれもすべての数値を `{{INVOKE}} engine runtime summary --json` から取得する（LLM 側では数えない）。`read-only` に分類される: ワークフローのステージポインターを進めることも、監査イベントを出力することも決してない。`aidlc-session-cost` と `aidlc-replay` はターミナルに表示するだけであり、ファイルを書くのは `aidlc-outcomes-pack` だけである（`OUTCOMES.md`）。
- **ステージランナースキル**（ユーザーが呼び出し可能）: `{{HARNESS_DIR}}/skills/aidlc-<stage>/`: 実行可能なコアステージごとに 1 つあり、`{{SKILL_INVOKE}}-<stage>` と入力して使う（例: `{{SKILL_INVOKE}}-domain-design`、`{{SKILL_INVOKE}}-code-generation`）。プラグインが所有するステージは、プラグイン接頭辞付きのコマンド名をそのまま使う。各スキルは、エンジンの `--single` モード（`aidlc-orchestrate next --stage <slug> --single`）でそのステージだけを単独で実行し、**メインワークフローの `Current Stage` を決して進めない**: `next --single` は合成された開始境界だけを記録し、`report --single` は同じ試行を閉じる。これらはオプトインのパッケージングである: ランナーがなくても、同じステージには `{{SKILL_INVOKE}} --stage <slug> --single` で到達できる。ランナーのセットは `{{INVOKE}} engine gen runners` によってコンパイル済みのステージグラフから生成され、その `check` ドリフトガードによって同期が保たれるため、ステージファイルを追加して再生成すればそのランナーも追加される。ブートストラップ用の 3 つの **initialization** ステージにはステージごとのランナーがない（単独では意味を持たないため）。initialization フェーズ全体は `{{SKILL_INVOKE}}-init` としてパッケージ化されている: `--scope <name>` と説明を付けると、ワークフローの記録とその開始状態を 1 ステップで作成する。`--scope <name>` だけを付けると、まず何のための作業かを尋ねる。説明だけを付けると、まず作業内容を説明したときと同じ計画の提案を示す。（これはオプトインのパッケージングである: 作りたいものを説明すれば、通常は最初の作業が自動的にセットアップされる。別の初期化コマンドは必要ない。）
- **エージェント**: `{{HARNESS_DIR}}/agents/` — ベースのフレームワークは 14 のエージェントを同梱する: 11 のドメインエキスパートのペルソナ（product, design, delivery, architect, aws-platform, compliance, devsecops, developer, quality, pipeline-deploy, operations）、2 つのレビュー専用エージェント（product-lead, architecture-reviewer）、そして adaptive-workflows のコンポーザーである。プラグインのインストールによって追加されることがある。有効なセットは、そのディレクトリ配下に存在するファイルから検出される。{{SLOT:agents_note}}
- **センサー**: `{{HARNESS_DIR}}/sensors/`: 一致する書き込み時、または承認ゲートで既存の成果物ごとに 1 回実行される自動チェック。ゲートで発火するセンサーは、助言的なものとブロックするものがある。ブロックするセンサーが失敗した場合、ゲートが開く前に、監査対象となる明示的なオーバーライドが必要である。フレームワークのデフォルト（`aidlc-claim-sources.md`、`aidlc-required-sections.md`、`aidlc-upstream-coverage.md`、`aidlc-traceability.md`、`aidlc-linter.md`、`aidlc-type-check.md`）を同梱する。フォークは独自の `aidlc-<id>.md` マニフェストを追加できる。ステージは frontmatter の `sensors: [<id>]` リストで、どのセンサーを発火させるかを宣言する — コンパイル時に解決されるプル型のインポートである。
- **ナレッジ**: `{{HARNESS_DIR}}/knowledge/` — 方法論のリファレンス。エージェントごとのものは `aidlc-<agent>-agent/` サブフォルダーの下にあり、`aidlc-shared/` にはエージェント横断の資料がある。フレームワークに同梱される。
- **ツール**: `{{HARNESS_DIR}}/tools/`: 判断ではなく正確さが求められる部分を担う、小さなコマンドラインプログラム（TypeScript、bun で実行）。ワークフローがどこにいるかの追跡、決定ログの書き込み、次に何を実行するかの決定（`aidlc-orchestrate.ts`。サブコマンドはちょうど 6 つ: `next`、`continue`、`report`、`park`、`team-board`、`wait`。`continue` は内部のステアリング用の転送手段、`team-board` は読み取り専用の Team Construction クエリ、`wait` はディスパッチ済みの作業を待つ、上限付きの読み取り専用の待機である）、自動チェックの実行、チームが学んだことの記録（`aidlc-learnings.ts`）、並列の Construction 作業のレフェリー（`aidlc-swarm.ts`）を行う。フレームワークのファイルはすべて `aidlc-*.ts` という接頭辞を持つ。
- **フック**: `{{HARNESS_DIR}}/hooks/`: CLI が決まったタイミングで自動的に実行するスクリプト。これにより、誰かが更新を覚えていなくても、決定ログ、保存された進捗、ステータス表示が正しく保たれる。フレームワークのファイルはすべて `aidlc-*.ts` という接頭辞を持つ。

## Plugins

AI-DLC はオープンワールドである。`plugins/<name>/` 配下のプラグインはステージ、スコープ、エージェントを追加し、`select-plugins` がこのインストールでどれを有効にするかを選ぶ。上記の数はベースのフレームワークのものであり、有効なセットは異なる場合がある。ここで何が有効になっているかについては、コンパイル済みの `{{HARNESS_DIR}}/tools/data/stage-graph.json` と `{{INVOKE}} --doctor` が正となるライブの情報である。

## Guards

ガードは利用者のチェックである。利用者が自分の言葉で、自分のチェックのいずれかをオフまたはオンに戻すこと、あるいは Guard Policy を変更することを求めたとき（"turn the review freeze check off for this project"、"stop asking me to re-approve when files change"、"relax the change checks"、"turn it back on" など）は、同じターンのうちに利用者に代わってそれを行い、コマンドが表示した行を、同じターンの返信の中で、質問、ピッカー、次のステップよりも前に、一字一句そのまま伝える（コマンドの出力だけでは利用者に届かない）: 決して拒否せず、利用者に入力を求めず、先に調査もしない。ガードやチェック全体をオフにするよう求めること（"turn the guards off"）は、同じメッセージで 1 つのチェックの名前を挙げていても、Guard Policy `off` である。"Stop asking me to re-approve when files change" は 1 つのチェックではなく Guard Policy `relaxed` である。Guard Policy がすでに `off` の場合は、すでにオフであることを 1 行で伝え、何も変更しない。チェックをオンにすること、または Guard Policy `strict` は常に問題ない。チェックをオフにすること、または Guard Policy `relaxed` や `off` は、利用者が求めたときにだけ行い、自分の判断で行ったり提案したりすることは決してない。適用範囲は利用者の言葉どおりとする: この作業、このプロジェクト、またはこのマシン。どちらも言われなかった場合は、作業が開いている間はこの作業、そうでなければこのプロジェクトとし、その行がどこに適用したかを示す。行がより広い範囲を提案する質問で終わっている場合、その質問が唯一の質問である: その行を伝えることで尋ねるか、代わりに同じ質問を持つピッカー 1 つで尋ねるかのどちらかであり、両方は行わない。利用者が yes と答えたら、表のコマンドを使って自分でその範囲に設定する。質問で終わらない行は何も尋ねない。この作業については、表のキーと、利用者が依頼したメッセージの `<id>` を使って `{{INVOKE}} engine config set <key> <on|off> --message <id>` を実行する。このプロジェクトについては `{{INVOKE}} config flags --bypass <switch> --local --yes --message <id>` を実行する（このマシンについては `--global`）。オンに戻すには `{{INVOKE}} config flags --clear-bypass <switch> --yes` を実行する。利用者がこのマシンやプロジェクトの共有設定を挙げた場合は、そこでクリアするために `--global` または `--project` を追加する。表の `none` は、その範囲にはそのチェックのスイッチがないことを意味する: 利用者が範囲を挙げなかった場合は、スイッチがある範囲を使う。スイッチがない範囲を挙げた場合は、そこには設定できないことを 1 文で伝えてもう一方の範囲を提案し、利用者が同意したときにだけ変更する。Guard Policy は作業単位でのみ設定される。開いている作業がない場合は、その旨を 1 行で伝える。Guard Policy については `{{INVOKE}} engine config set guard-policy <strict|relaxed|off> --message <id>` を実行する。チームがメモリーで Guard Policy strict を保持している場合、コマンドはそのファイル名を挙げた 1 行で拒否するので、その行を伝える。入力された `{{SKILL_INVOKE}} --guard-policy off` や `{{SKILL_INVOKE}} config set guard.<fence> off` は、メッセージが届いた時点で適用済みである。ガードを変更するために `aidlc-state.md` を編集したりフックを実行したりしてはならない。`{{SKILL_INVOKE}} --status` は、現在の Guard Policy と、すべてのフェンスおよびその設定がどこから来たかを表示する。変更に注意することについての発言や、チェックについての質問は、変更の依頼ではない。判断できない場合は 1 回だけ尋ねる。

| チェック | この作業: キー | このプロジェクトまたはマシン: スイッチ |
|---|---|---|
| 計画承認 | `plan-approval` | `AIDLC_DISABLE_PLAN_APPROVAL_GUARD` |
| レビュー凍結 | `guard.review-freeze` | `AIDLC_DISABLE_REVIEW_FREEZE_HOOK` |
| レビュアー読み取りスコープ | `guard.reviewer-scope` | `AIDLC_DISABLE_REVIEWER_SCOPE_HOOK` |
| 状態遷移 | `guard.state-transition` | none |
| サマリー確認 | `summary-confirmation` | `AIDLC_DISABLE_SUMMARY_CONFIRMATION` |
| サマリー確認チェック | none | `AIDLC_SKIP_SUMMARY_CONFIRMATION_GUARD` |
| 人間の存在 | none | `AIDLC_SKIP_HUMAN_PRESENCE_GUARD` |
| ステージ出力 | none | `AIDLC_SKIP_ARTIFACT_GUARD` |
| リビジョンバックストップ | none | `AIDLC_SKIP_REVISION_BACKSTOP` |
| パイプライン引き継ぎ | none | `AIDLC_DISABLE_ENSEMBLE_EVIDENCE` |

## Models and effort

誰かが平易な言葉でエージェントのモデルや effort の変更を求めたとき（"make the developer agent think harder"）は、`{{INVOKE}} config models --agent <name> --effort <low|medium|high|xhigh|max> --project --yes` を実行する。モデルについては `--effort` の代わりに `--model <id>` を使い、両方を求められた場合は両方を使う。利用者本人だけのためである場合は、`--project` の代わりに `--local` を使う。コマンドが `--harness` を求めた場合は、このツールのハーネスを指定してもう一度実行する。その出力を表示して終了する。このために `aidlc-*-agent` ファイルを編集してはならない。

{{SLOT:structure_extra}}

{{SLOT:sections_before_resumption}}
{{SLOT:sections_after_resumption}}
