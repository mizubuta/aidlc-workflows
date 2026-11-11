# Construction Protocol Module

検証付きのユニットチェックポイントポリシーは、実在するユニット DAG を持ち、
ワークスペースのソースを生成するステージを含むソロワークフローに適用される。
設計のみのスコープやユニットのないスコープは、既存のステージ承認フローを維持する。
続行するためにソースマニフェストや動作するスケルトンをでっち上げる必要はない。

このモジュールは、セッションで最初の Construction フェーズのディレクティブを受けたときと、すべての `invoke-swarm` で読み込む。使うのはアクティブなハーネスに対応するハーネスのサブセクションだけとする。チェックポイント、Build-and-Test のループバック、ウェーブはそれぞれ専用のモジュール（`stage-protocol-checkpoint.md`、`stage-protocol-loopback.md`、`stage-protocol-wave.md`）で実行し、エンジンはそれらを必要とするステップでモジュール名を示す。

**適用範囲と互換性。** ソースを生成する新しいソロのユニットワークフローは、
`Construction Checkpoints: enabled`、`Construction Iteration: unit-major`、
`Construction Execution: serial` を記録する。チェックポイントは、
実在する空でないユニット DAG を持つソロ作業に適用される。チーム所有の `unit_gate`
ディレクティブは独自の承認ポリシーを維持する。ユニットがゼロの実行や単独実行は、
ユニット、スケルトン、スウォームの手続きを持たない。チェックポイントフィールドを
持たない既存のワークフローは、最初のステージでの承認を維持する。unit-major では、
後半のステージ承認は 1 つの質問（`directive.approve_together`）としてまとめて行う。
明示的なイテレーションの選択は保持する。散文の中で状態を編集して移行してはならない。
現在のディレクティブのメタデータに従う。

### Construction directive routing

通常の本体、質問、レビュアー、学び、`gate: true` の処理の前に、
このモジュールを読み込み、次の分岐を順に適用する。

`gate_only: true` を持つ開いたステージゲートは、下記の完了のみの記録処理より前に、
Stage Protocol の開いたゲートへの再入手順に従う。その本体とレビューは決着済みだが、
それはゲートを自動承認してよいという許可ではない。チームの `unit_gate` と
決着済みスウォームのフィールドがある場合は、それらのポリシーを維持する。

`unit_record_step` は、出力されたユニットの書き込み済みの作業について、まだ残っている
記録を示す。その正確なコマンドと、それが返すステップに従い、その後 `next` を実行する。
ステージ本体、通常の質問やレビュー、完了ゲートを繰り返してはならない。ステップに従う前に、
`plan_approval.notice` があればそれを伝える。
Plan Approval は引き続き
適用される。エンジンは、ユニットの計画を承認チェックに通した後でのみこのステップを送る。
保留中または変更された計画には、代わりにその計画のステップが送られる。

1. **`directive.unit_gate`** は、下記の既存のチーム所有ゲートの手順を使う。
2. **`directive.swarm_checkpoint`** は、通常の本体や決着済みスウォームの処理の前に、
   スウォームモジュールの **Batch checkpoint** 手順を使う。完了したバッチを承認または
   却下し、その後 `next` を実行する。バッチのためにステージ全体を承認してはならない。
3. **`directive.construction_checkpoint`** はチェックポイントモジュール
   （`stage-protocol-checkpoint.md`）を使う。ユニットの本体はすでに実行済みである。再生成したり、このユニットについて
   Code Generation ステージ全体の完了を報告したりしてはならない。
   `rereview` を持つチェックポイントは、そこに記載のとおり、まずユニットの変更された
   コードやドキュメントを再確認する。
4. **`directive.construction_policy.completion_only === true`** は、ユニットが承認された後に
   ステージを閉じる。ソロの unit-major の巡回では、最後のユニットが承認されると素の `next`
   がこれらのゲートを自ら記録し、次の実際のステップを返す。そのため、このディレクティブは
   それができない場合（伝えるべき変更通知、チーム所有のユニット、スウォームの決着）にのみ届く。
   また `human_completion_required: false` も持っていなければならない。本体、質問、レビュアー、
   学びの問いかけをスキップし、このステップについては何も言わない。出力された `directive.stage`
   について `awaiting-approval`、続いて `approved` を、どちらも `--user-input` なしで報告し、
   `next` を再実行する。メタデータが矛盾している場合や報告が拒否された場合は、
   エラーを説明して停止する。承認をでっち上げたり生成を再試行したりしてはならない。
5. **`directive.construction_policy.offer_autonomy === true`** は、次のステージ本体を行う前に、
   下記の **Autonomy choice** を使う。実際の回答を記録し、`next` を再実行して
   現在適用されるポリシーを取得する。
6. **その他の Construction 作業** は、本体の作業の前に、まずハーネスのスタンス分類バインディングで
   `gate: "unresolved"` を解決して報告し、新しい `next` を取得する。それ以外の場合は、
   出力されたユニット、ウェーブ、またはステージ本体と、その必須の質問、有効なサマリー確認、
   Plan Approval、レビュー、検証、レシートに従う。サマリー確認は
   `directive.ceremony.summary_confirmation === "on"` の場合にのみ適用される。完了時には、ポリシーの
   `human_completion_required` によって、通常の完了ゲートに人間が必要かどうかが決まる。
   false の場合は、学びの質問と通常の承認の質問をスキップし、`awaiting-approval` と
   `approved` を `--user-input` なしで報告して、`next` を実行する。これによって、有効なサマリーでの停止、
   有効な Plan Approval、検証コマンドの選択が免除されることは決してない。
   未完了のユニットごとのイテレーションでも、ユニットのレシートを完了させて
   `next` を呼び、ステージ全体は報告しない。ポリシーがない場合は、
   旧来のゲートルールを使う。検証やツールの失敗はすべてフローを停止させる。

`construction_policy` は `iteration`（`unit-major` または `stage-major`）、
`execution`（`serial` または `swarm`）、`autonomy`（`unset`、`gated`、`autonomous`）、`offer_autonomy`、
`human_completion_required`、`completion_only` を持つ。これらのフィールドは、`gate: true` の
一般的な解釈より優先される。本体レベルでの人間の判断は変えない。
`single: true` の既存の単独実行の分岐は維持する。

### Execution is separate from approval

新しいワークフローの既定値は `Construction Execution: serial` である。自律性を設定しても、
実行方式やイテレーション順序は決して変わらない。スウォームには、まず stage-major、
次に実行方式の設定が必要である。利用者が並列のユニットを求めたときは、下記の
**Changing Construction policy** の説明どおりに両方のセッターを実行する。

```bash
{{INVOKE}} engine state set-construction-iteration stage-major --message <id>
{{INVOKE}} engine state set-construction-execution swarm --message <id>
```

`<id>` は利用者が依頼したメッセージである。スウォームは、ガイド付き（`gated`）または自動（`autonomous`）のバッチ完了承認を
使える。unit-major は直列のままで、矛盾するスウォーム設定を拒否する。
unit-major に戻す前に serial を選択する。実行方式フィールドを持たない旧来の
ワークフローは、既存の自律性ベースのスウォームルーティングを維持する。チーム所有の
作業は、そのクレームとユニットゲートのポリシーを維持する。自律性の回答からバッチを導いたり
順序を変えたりせず、常にエンジンが出力した作業に従う。

### Changing Construction policy

保護された質問は、セッションごとに 1 つしか開けない。保護された質問は
1 つずつ尋ね、回答を待ってから他のことを行う。

利用者が自分の言葉で `Construction Checkpoints`、
`Construction Execution`、`Construction Iteration` の変更を求めたとき（"turn checkpoints off"、
"from here on, build one unit at a time"、"run the Units in parallel" など）は、
そのターンで実行する。依頼されたメッセージを指す `--message <id>` を付けて型付きセッターを実行し、
その `notice` 行を一字一句そのまま利用者に伝える。利用者に確認を求めてはならない。例:

```bash
{{INVOKE}} engine state set-construction-checkpoints disabled --message <id>
{{INVOKE}} engine state set-construction-iteration unit-major --message <id>
{{INVOKE}} engine state set-construction-execution swarm --message <id>
```

Construction 中、セッターは指定されたメッセージに基づいて変更を行い、
そのメッセージの文言を変更とともに保持する。これを自分の判断で行ってはならない。拒否を解消するためでも、
自律性がオンだからでも、別のゲートの回答を根拠にしてもならない。セッターが、先に変更が必要な
設定を示した場合（並列には stage-major とチェックポイントのオンが必要、
1 ユニットずつには serial が必要）は、そのセッターを先に実行し、両方の行を伝える。
1 つのメッセージで Construction の変更と自動承認が同時に求められた場合は、
`set-autonomy` の前に Construction のセッターを実行する。1 つのメッセージに複数の変更がある場合は、
変更ごとに専用のセッターと専用の行を使う。開いたゲートでの承認を含まない依頼は、
利用者の回答を待つためにゲートを開いたままにする。

### Unit and skeleton checkpoints

チェックポイントが有効な場合、skeleton-on では、周囲のイテレーションの選択が stage-major であっても、
**DAG の最初のユニットを、後続のユニットより先に、Code Generation を含む適用可能なすべてのユニットごとのステージに通す**。
そのユニットは、動作する最小の統合スライスとして計画する。最初の設計ステージのレビューは、
出荷されたウォーキングスケルトンではない。unit-major は直列のままで、スウォームを決して起動しない。
明示的に `Construction Execution: swarm` を指定した stage-major は、スケルトンのチェックポイントが承認された後、
gated と autonomous のどちらの承認ポリシーでも、対象となる Code Generation の作業を
ファンアウトできる。出力されたユニットに従う。チェックポイントですでに承認されたインラインのユニットを、
後のスウォームで再びビルドしてはならない。

### Autonomy choice

チェックポイントワークフローでは、**`offer_autonomy: true` だけが自動化の
選択をトリガーする**。skeleton-off では Construction の開始時に提示する。skeleton-on では、
オンデマンドの選択がすでに記録されていない限り、実際のスケルトンのチェックポイントの後に提示する。
ハーネスの通常の質問バインディングを使って質問を表示する。

```question
prompt: "How should I continue building the remaining work?"
header: Autonomy
multiSelect: false
options:
  - label: Continue automatically
    description: Continue through ordinary completion checkpoints; still ask for plans, enabled summaries, verification command selection, and failures.
  - label: Review each checkpoint
    description: Wait for your approval at each ordinary completion checkpoint.
```

**Continue automatically** を `autonomous` に、**Review each checkpoint** を
`gated` に対応付ける。明示的な回答は
`{{INVOKE}} engine bolt set-autonomy --mode autonomous --message <id>`（利用者が
選択したメッセージ）または `--mode gated` だけで記録する。先に `log decision` や
`log answer` で記録してはならない。その後 `next` を再実行する。
権限の引き上げには新たな人間のターンが必要だが、`gated` への取り消しには必要ない。

明示的なオンデマンドの依頼は、Construction 中のどの時点でも有効である。
沈黙から許可を推測したり、選択がわかった後に再度提示したりしてはならない。
autonomous の許可は、どのイテレーションの選択でも一貫して通常の完了の質問を免除する。
一方、ユニットごとの Plan Approval、有効な生成前のサマリー確認、検証コマンドの選択、
スケルトンの承認、失敗には引き続き人間が必要である。下記の Grouped Plan Approval は
表示方法を変えるだけである。どのユニットにも、それぞれ有効なレシートが必要である。

チェックポイントフィールドを持たない旧来のワークフローでは、最初の
Construction ステージのレビューを維持する。unit-major では、後半のステージ承認は 1 つの
人間への質問（`directive.approve_together`）になる。
skeleton-on では、選択がない場合、その最初のステージのレビューの後に旧来の段階的な選択肢を提示してよい。
skeleton-off はオンデマンドの経路を維持する。その旧来のレビューは
最初のステージの承認として説明し、動作する統合スケルトンの証明として説明してはならない。

**失敗時の停止と確認**

Code Generation が失敗を返したときは、**自律モードにかかわらず、必ず停止して停止と確認のプロンプトを提示する**。Build-and-Test の失敗ループバックの第 4 段階も、上限に達した場合や特定可能な修正がない場合に停止する。必須の Plan Approval、サマリー確認、検証コマンドの選択は、それぞれ独立した人間による停止点のままである。

- ソロのユニットの失敗: 直ちに停止する。スウォーム / worktree の経路では `BOLT_FAILED` を出力し（停止と確認の対応付けのため `--slug` を付ける）、retry / skip / abort を提示する。
- 並列バッチの部分的な失敗: すべての並列タスクが戻るのを待ち、成功したユニットの成果物を保持し、失敗したユニットについて `Succeeded=[names]` を付けて `BOLT_FAILED` を出力し、`"Units [X, Y] succeeded, Unit [Z] failed with: [error]. Options: retry Z, skip Z, abort Construction."` を提示する。
- Retry: 失敗したユニットだけを既存の worktree 内で再実行する。
- Skip: 理由を添えて状態に `[S]` を付け、次のバッチに進む。`<path>` の worktree は保持される。
- Abort: Construction を停止する。ユーザーは後で再開できる。`<path>` の worktree は保持される。

この通常の Abort は、チェックアウトを破棄せずに Construction を一時停止する。
古いレビューからの復旧コマンドが明示的に `--discard` を含む場合、abort は代わりに、
Bolt の追跡対象ファイルと無視されていない未追跡ファイル（チェックアウトがない場合は残っているブランチの先端）
およびレビュー済みのソース参照を退避し、その後ライブのチェックアウトとブランチを削除する。
返された未変更のコマンドを実行する前に、人間の選択を得る。返された `restore_operation` がある場合、
それは退避した作業を隔離された復元用チェックアウトに復元する。ファイルを復元しても、
中止した Bolt が再開されたり、そのレビューの権限が復活したりはしない。レビューの証拠しか
残っていなかった場合は、復元すべき保存済みの作業ファイルがないため、
`restore_operation` も `restore_hint` も返されない。

**破棄が成功した後。** `--discard` の abort が成功し、試行が退避されたことが
確認された後で、かつ代わりの試行を開始する前に限り、次の SAY 行を使う。`[reason]` は
返された `abort_reason` から、平易なプロジェクトの言葉で埋める。`On your go-ahead I` は、人間が対処方法を選んだ場合にのみ使う。
人間による対処方法の選択が関与しなかった場合（自動ループバックを含む）は `I` を使う。
abort が失敗した場合や試行を退避しなかった場合は、保存したスナップショットを告知してはならない。

`[saved-files text]` は、返された `parked_mode` に応じて選ぶ。

- `snapshot`: "I saved a snapshot of its tracked files and non-ignored untracked files. Ignored files are not saved, and the snapshot may normalize line endings."（追跡対象ファイルと無視されていない未追跡ファイルのスナップショットを保存しました。無視されたファイルは保存されず、スナップショットでは改行コードが正規化される場合があります。）
- `branch-tip`: "I kept its committed work; there were no uncommitted files to save."（コミット済みの作業を残しました。保存すべき未コミットのファイルはありませんでした。）
- `evidence-only`: "Nothing of its working files remained to save; only its review evidence was kept."（保存すべき作業ファイルは残っていませんでした。レビューの証拠だけを残しました。）
- `null`: `[saved-files text]` を省略する。フォールバックの記述子からは、何が保存されたかは確定しない。

**SAY:** "[On your go-ahead I|I] set aside the previous attempt at [Unit] because [reason], and I'm starting a new attempt. [saved-files text] If you want the previous attempt back, ask me to restore it."

`restore_operation` がない場合は、最後の申し出 "If you want the previous attempt back, ask me to restore it." を省略する。証拠のみの試行に対して復元操作をでっち上げてはならない。

人間が後でその試行を戻すよう求めた場合は、保存された abort 結果の
`restore_operation` を使う。その `worktree` ルートを
`{{INVOKE}} engine worktree <args...>` で呼び出し、列挙された `args` の各要素を正確に
個別の argv 引数として渡す。それらの引数を連結してシェルコマンドにしたり、
slug だけの選択を組み立て直したりしてはならない。`restore_hint` は人間向けの表示テキストにすぎず、
実行の入力にしてはならない。安全な表示に失敗した場合（たとえば無効なハーネス
ディレクトリ）は、ヒントは省略され、`restore_hint_error` がその理由を説明する。
操作は引き続き利用でき、復元の申し出も引き続き有効である。
復元が成功したら、返された復元先パスを平易に伝える:
**SAY:** "I restored the previous attempt at [returned restored path]."
これによって古い試行が再開されたり、そのレビューが最新になったりはしない。

オーケストレーターは、停止と確認の質問を組み立てる前に `{{INVOKE}} engine worktree info --slug <slug>` を実行し、worktree の `<path>`、`<branch_name>`、`intent_id8` を決定論的に取得する。`info` は、監査された `Branch name` と `Worktree path` を、選択されたインテントの正規の Bolt 識別子と照合して検証し、正規の値（新しい Bolt では `bolt-<id8>_<slug>`、アップグレード前の Bolt では旧来の `bolt-<slug>`）を返す。そのため `path` と `branch_name` は検証済みの表示値であり、監査からの自由記述ではない。返されたとおりに正確に埋め込み、slug からブランチを再構築してはならない。終了コードが 0 以外の場合は、下記と同じ Retry/Skip/Abort の質問を表示し、"Worktree at [path] on branch [branch_name]." の節を語り口の規約に沿った言い換えに置き換える。表示できなかったものとその理由を述べる平易な 1 文と、次のステップを述べる 1 文である（たとえば "I couldn't confirm this Bolt's worktree from the audit record, so I'm not showing a path or branch. You can still choose Retry, Skip, or Abort below."）。監査からブランチやパスを引用したり、slug から自分で組み立てたりしてはならず、拒否のテキストはツールの結果の中に残しておく。JSON と命名の規約については [Bolt の識別子](../../knowledge/aidlc-shared/worktree-info-schema.md#bolt-identity) を参照。

```question
prompt: "Bolt [Z] failed during code generation: [short error]. Worktree at [path] on branch [branch_name]. How would you like to proceed?"
header: Bolt Failure
multiSelect: false
options:
  - label: Retry
    description: Re-run Bolt [Z] in the existing worktree.
  - label: Skip
    description: Mark Bolt [Z] skipped; worktree preserved.
  - label: Abort
    description: Stop Construction; worktree preserved.
```

---

### Per-Unit iteration

**エンジン主導のユニットごとのイテレーション。** オーケストレーションエンジンは、インラインのユニットごとの設計ステージ（functional-design、nfr-requirements、nfr-design、infrastructure-design）についても、code-generation で従来から行ってきたのと同じ方法でユニットごとのループを駆動するようになった。進行中のユニットごとのステージに到達する `next`（スウォームの経路以外）では、エンジンはユニットごとに 1 つの `run-stage` ディレクティブを Bolt のビルド順に出力し、解決済みのユニット名を `directive.unit` に、その成果物のパスとともに載せる。エンジンは `next` のたびに、次の未決着のユニットに差し替える。stage-major または旧来のステージゲートでは、まだ決着していないすべてのユニットでステージのユニットごとのゲートが**抑止**され（`gate: false`）、ステージの実際のゲートは最後のユニットが決着した後の再入時にちょうど 1 回だけ提示される。そのため、1 回のステージレベルの承認がすべてのユニットを対象とし、すべてのユニットがビルドされるまでその承認には到達できない（「ユニットごとのゲートは抑止し、単一のゲートがそれを置き換える」という同じルールを、5 つのユニットごとのステージすべてに適用し、決定論的に強制する。未完了のユニットごとのステージに対する `report --result approved` は、未決着のユニットがある間は拒否される）。units-generation の依存成果物がディスク上にないワークフローは、単一イテレーションのディレクティブ 1 つに縮退する（動作は変わらない）。成果物がある場合、エンジンはコンパイル済みの `bolt_dag` をそれと照合して検証し、キャッシュがないか古い場合はその場でユニットのバッチを再計算する。そのため、ユニットごとのループが古いユニット集合に黙って縮小することはない。ユニットのブロックを解析できない成果物は、代わりにエラーとして示される。

**ユニットのライフサイクルのレシート。** `directive.wave`、`directive.questions_only`、`construction_checkpoint`、`construction_policy.completion_only` のいずれも持たないユニットごとの本体ディレクティブでは、ユニットの作業をレシートの動詞で挟む。本体の前に `{{INVOKE}} engine state unit start --stage <slug> --unit <name>` を、ユニットの成果物を書き込んだ後に `... unit complete --stage <slug> --unit <name>` を実行する（complete は、必須の成果物がすべてディスク上の通常ファイルであることを検証し、ディレクトリや存在しないパスを拒否する。また、ユニットのレビュー依頼が判定なしで開いている間は、最後の質問以降に利用者が発言したか、復旧の確認でレシートを選んだ場合を除いて拒否する。レシートが完了の合図であり、成果物はそれが確認する証拠である）。エンジンが出力した `directive.stage` + `directive.unit` の組をそのまま渡す。`unit start` は、読み取り専用のエンジン観測としてルートを再実行し（ディレクティブを公開せず、状態、レシート、承認の証拠も書き込まない。その経路からの永続的な書き込みは黙って失敗せず、明示的に失敗する）、依存先や同じバッチ内の先行ユニットが決着していない DAG メンバーを拒否する。新しいユニット名は小文字の kebab-case を使う。安全な旧来の単一セグメントの名前（数字で始まる名前、大文字、アンダースコア、ドットを含むもの）は、既存の DAG と autonomous のスウォームで引き続き受け付けられ、それらはユニットの識別子を変えずに決定論的な内部の Bolt slug を使う。後方へのジャンプがインラインのユニットごとのステージにルーティングされた場合、自律性の許可があってもこれらのレシートは無効にならない。レシートを拒否するのは、現在 autonomous のスウォームが所有しているステージだけである。ユニットが完了前に停止しなければならない場合（ブロックする質問、依存先の失敗、ユニットの途中でのセッション終了）は、1 行のテキストでチェックポイントを記録する: `... unit pause --stage <slug> --unit <name> --reason '<why>' --next-action '<the exact next step>'`。すべてのライフサイクル行は、ステージの試行を正確に示す `Run floor`（`<boundary-event>:<timestamp>#<ordinal>`）を持つ。異なる監査シャードにある秒精度で等しい境界が因果的に順序付けられない場合、エンジンはシャードのファイル名の順序を信用する代わりに、古いレシートを無効にする決定論的な `AMBIGUOUS:<timestamp>#<digest>` の floor を使う。レシートの有効性は、試行の floor と行の内容の紐付けによって決まり、シャードや行が書き込まれた順序によっては決して決まらない。あるステージにレシートが 1 つでも存在すると、以降の試行はすべてレシートモードのままとなり、ユニットごとに現在の試行の `UNIT_COMPLETED` レシートが必要になる。成果物ファイルだけではユニットは決着しなくなったため、古い、一時停止中、再オープンされた、または一部だけ書き込まれたユニットが完了と誤認されることはない。一時停止中のユニットは最初にルーティングされ、ループを強制停止する。エンジンはユニット、記録された理由、次のアクションを示す `ask` を出力し（`unit_state: paused`）、明示的な `... unit resume --stage <slug> --unit <name>` があるまで他の作業を開始してはならない。`unit start` は、そのステージの別のユニットが開いている間は拒否する（アクティブなユニットは一度に 1 つ。先に再開または完了させる）。唯一の例外は、利用者が別のユニットのために脇に置いたユニットである（`--set-aside-for <unit>` 付きの一時停止で、unit-major の再オープンや取り上げで示される）。その場合はエンジンのルートに従う。ルートは、脇に置く原因となったユニットを先に取り上げ、一時停止中のユニットのステージであってもそのユニットの開始を許し、その後、脇に置いたユニットの再開を求める。Construction のチェックポイントが適用される場合（Construction にソースを生成するユニットごとのステージを含むソロ作業で `Construction Checkpoints: enabled` のとき）は、最初のユニットからレシートが必要となり、成果物ファイルだけでユニットが決着することはない。チェックポイントがない場合、動詞を一度も呼ばないワークフローは成果物ベースのカバレッジを維持する。ユニットのファイルが書き込まれ、最終レビューも記録されているがレシートが記録されていない場合、`next` はステージ本体を繰り返す代わりにレシートのコマンドを示す。それらを実行し、その後 `next` を実行する。Code Generation では、Plan Approval の権限を最新に保つため、ユニットの公開されたディレクティブの `unit_record_step` にそれらが載る。他のステージではそれらが表示される。代わりにレビューが欠けている記録である場合、`next` は同じ条件のもとで、何も言わずにステージ本体を返すのではなく、この試行が待っているレビューのステップ（何も依頼されていなければ最初の依頼、依頼が待機中なら判定、レビュアーが修正を求めた後なら次のパス）を表示する。その条件とは、ファイルがこの試行のものであることである。つまり、それがユニットにとってそのステージの最初の試行であるか、Code Generation の場合は、この試行で承認された計画に基づいてビルドが開始され、すべてのステップに、そこに示されたファイルが存在する状態でチェックが付き、ユニットの `source-manifest.json` が読めることである。そのため、却下によって再オープンされ、全体を再ビルドしたユニットにはレビューが求められる。レビューがオフの場合は記録すべきレビューがないため、`next` は同じ条件のもとでレシートのコマンドを表示する。それ以外の場合、ジャンプや却下の後はステージ本体が戻ってくる。unit-major の巡回では、チーム所有のユニットの最初の質問、回答、サマリーの選択、レビュー依頼は、その `unit start` を待つ。それが記録されるまで、それらのコマンドは拒否し、その旨を示す。

**ユニットごとのバッチウェーブ。** `directive.wave` を持つステップは、ウェーブモジュール（`stage-protocol-wave.md`）で実行し、エンジンはそのステップでモジュール名を示す。

**unit-major のイテレーション。** 新しいワークフローの既定値は `Construction Iteration:
unit-major` である。記録済みの stage-major の選択は保持し、既存のワークフローを書き換えない。
エンジンは、1 つのユニットについて Code Generation を含む適用可能なすべてのユニットごとのステージを
巡回してから、次のユニットに進む。各ユニットの Plan Approval は人間による停止点のままであり、
unit-major はスウォームを決して起動しない。チェックポイントが有効なソロ作業では、
その後 `construction_checkpoint` を通じてユニットを検証して承認する。`completion_only: true` を持つ
後半のステージゲートは記録処理である。チェックポイントがオフの場合、またはフィールドを持たない
旧来のワークフローでは、最後のユニットの後に残っているステージ承認は 1 つの人間への質問になる
（Stage Protocol の承認ゲートの説明どおり `directive.approve_together`）。ディレクティブは
`Current Stage` より後のステージを示すことがある。条件付きスキップの報告
（`--stage "<directive.stage>" --unit "<directive.unit>"`）を含め、常に `directive.stage` と
`directive.unit` を使う。その報告はそのユニットだけを対象とし、他のユニットは引き続きそのステージを通る。
ライフサイクルとレビューの証拠は、現在の
ワークフロー/ジャンプ/却下の試行に紐付いたままとなるため、後のステージの開始で承認済みの作業が繰り返されることはない。

**チーム所有の Unit Progress とゲート（オプトイン）。** `Unit Ownership: team` は
unit-major でのみ有効である。そのモードでは、すべての `next` が、ユニット DAG、成果物のカバレッジ、
ライフサイクル/レビューのレシート、ゲートイベントから `## Unit Progress` を書き直す。
この表はエンジンが所有する投影にすぎない。手で編集しても上書きされ、ルーティングは決して変わらない。
`owner` セルには、git ネイティブのライブなクレームが入る。
`Unit Gate Rhythm: per-stage` が既定値である。1 つの `(stage, Unit)` が決着すると、
そのユニットが進む前に、エンジンは `gate: true` と `unit_gate: per-stage` を付けてそれを再出力する。
`unit-end` は、アクティブなユニットごとの作業ビートをすべてゲートなしのままにし、
最後のアクティブでスキップされていないユニットごとの Construction ステージの後に
`unit_gate: unit-end` を 1 つ出力する。これらのゲートは、グリッドの終わりにある後半の 5 つの
ゲートを置き換え、Stage Progress のチェックボックスは完了したユニットの列から導出されるようになる。

**通常の `gate: true` の分岐より前に、`directive.unit_gate` で分岐する。**
本体、有効なサマリーのチェックポイント、ライフサイクルの完了、レビュアーはすでに
決着している。再生成や再レビューをしてはならない。学びのステップは `directive.protocol_modules` に `learnings` がある場合にのみ行い（いつ尋ねるかはモジュールに記載されている）、その後に承認ゲートを行う。チーム所有のユニット作業では、ゲート以外の
`aidlc-log.ts decision` / `answer` 呼び出しにもすべて
`--unit "<directive.unit>"` を付け、保留中の人間の判断を試行と
ユニットの範囲に限定する。このゲートのすべての report 呼び出しには
`--unit "<directive.unit>"` を付ける。まず `awaiting-approval`、次に `approved
--user-input "Approve"`（利用者が停止も求めた場合は `--park` を付ける）、または
`rejected --user-input "Request Changes"` と、その後の `revised` である。却下は、そのユニットのライフサイクル/レビューのレシートだけに floor を設定する。
`unit-end` の場合は、そのユニットのチェーン内のすべてのステージに floor を設定する。受け付けられた
各報告の後に `next` を再実行する。Unit Ownership がない場合や `solo` の場合は、上記のチェックポイントまたは旧来の
ポリシーに従う。チーム所有の `unit_gate` の経路は独自の承認リズムを維持し、
ソロのチェックポイントには変換されない。

**チームのユニットのクレームとスコープ付きチェックアウト。** `next` が
`ask_type: unit-claim` を出力したときは、その claimable/claimed/waiting のリストを提示し、選択されたユニットについて
`aidlc-unit.ts claim <unit> --team '<label>'` を実行する。クレームは
`claim/<intent-id8>/<unit>` 参照をアトミックなレジストリとして使い、gitignore された
チェックアウトローカルのスコープスタンプを書き込む。スタンプ付きのチェックアウトは、そのユニットの
アクティブなユニットごとの Construction ステージとゲートだけを実行する。ライフサイクル、レビュー、
ゲート、フォークの操作はすべて、スタンプされたユニットと現在の試行の世代を
指定しなければならない。終端の `notice` は、main がファンアウトのディスパッチャーとして動作していることを意味する。
それをそのまま表示して停止する。チームの引き継ぎ（計画の共有、ユニットの引き受け、
その公開、公開されたユニットのレビュー、land 後の main への push）は、
選択肢ごとにコマンドを持つ `ask_type: team-handover` の質問として届く。リリースはスコープなしの main からのみ
`aidlc-unit.ts release <unit>` で実行し、世代を進める tombstone 参照を残す。
スコープ付きの `next`、ライフサイクル、判断、レビュー、ゲートの書き込みは、ローカルで
検証されたクレーム時のスタンプを信頼し、ネットワークを必要としない。フォーク/リリースは
クレームに敏感な境界である。到達可能ならレジストリを再確認し、オンラインで古い試行は拒否し、
オフラインなら 1 回警告したうえでスタンプに基づいて続行する。

**ピン留めしたユニットのマージバック。** スコープ付きのユニットが完了すると、`next` は
その公開（`aidlc unit publish <unit> --commit`）を提案する。公開はクレーム参照を CAS 更新するが、
統合はしない。スコープなしの main で `aidlc unit pin <unit>` を実行し
（その提案に対するレビューの選択肢、または利用者自身のコマンド）、
同じターンで、返されたピン留め OID と証拠のサマリーを、その `merge_gate` の指示どおり
1 つのマージゲートとして提示する。利用者の返信を、入力されたとおりに、シェルではなくファイルツールで
`aidlc/.aidlc-request-text/gate-<unit>.txt` に書き込み、利用者の選択に対して示されたコマンド（`aidlc unit gate <unit>
--decision <approve|reject> --request-file aidlc/.aidlc-request-text/gate-<unit>.txt`）で記録する。
承認された場合、ゲートの `land` は `aidlc unit land
<unit> --target <branch>` を示す。これは追加の質問なしで同じターンに実行する。このコマンドが
トランザクションを担う。まず main が所有するメタデータを保持したままピン留めした git の内容を取り込み、
次にインテントのロックの下でユニット行を 1 つ反映し、最後に監査/確定処理を行う。クレーム参照が
移動した場合は再ピン留めが必要であり、レジストリが利用できない場合は gate/land が安全側に
失敗する。git のステップが land した後にのみ、その試行がリリースされた場合は、
マージを確認し、`aidlc unit land <unit>
--accept-released-attempt --user-input '<human acknowledgment>'` で明示的に続行する。後継の
クレームは決して受け付けない。ソースの競合は、状態の反映前に中止する。
クラッシュからの復旧では、同じコマンドが `--step git|state|audit` を受け付ける。各ステップは
冪等であり、`aidlc unit merge-status <unit>` がローカルのジャーナルを報告する。
回答は、ピン留めの後に人間が入力したターンが続いた場合に有効となる。トランザクションは
`merge` 戦略でピンの統合ブランチに land するため、レビュー済みの
ピン留め OID は直接の親のまま残る。これは利用者が選ぶものではなく、
それ以外のマージにはすべてチームのマージの既定値が適用される。
候補は、そのチーム自身の試行に紐付いたライフサイクル、チームゲート、レビュアーのレシートを含む、
クレームに紐付いた新しい監査シャードを 1 つ運べる。これらはチームの主張であり、
成果物/フィンガープリントと照合して再確認され、main のマージゲートで判断される。
`HUMAN_TURN`、`MERGE_DISPATCH_*`、`UNIT_MERGED`、ユニットのマージゲート、
他のユニットのレシート、兄弟ユニットの記録パス、追加のシャードは、
決して運べない。
マージゲートの行は、スコープなしの main チェックアウトの監査シャードからのみ受け付けられ、
その OID と世代に正確に一致しなければならない。ピン留め後の main シャードの `HUMAN_TURN` は、
時系列上の人間の存在の証明であり、意図的にトランザクションのフィールドを持たない。

各 Construction ステージファイル（3.1〜3.4）には、その実行モード（QUESTION-ONLY、ARTIFACT-ONLY、Full）とステップの分割点が記載されている。詳細は各ステージファイルを参照。

---

## Harness construction bindings

### Claude Code

- **`gate: "unresolved"`** — 最初の Construction のルートは**ウォーキングスケルトンのスタンス**に依存するが、チームの自由記述の `## Walking Skeleton` のプラクティスの散文からは、どのパーサーもそれを導出できない。これはあなたのナレッジ作業であり、結果をエンジンに返す。まだステージ本体を実行してはならない。代わりに、`## Walking Skeleton` セクションを読み（解決順は `aidlc/spaces/<space>/memory/org.md` → `team.md` → `project.md`。最も具体的な空でない記述が優先される）、スタンスを分類する。**"always"/"every greenfield feature"** → `on`、**"never"** → `off`、**"scope-dependent"/未指定/空** → `scope-dependent`（その場合、エンジンはアクティブなスコープファイルの `skeleton:` フィールドを使う）。`PRACTICES_OVERRIDE` の判断を尊重する（プラクティスと矛盾する bolt-plan のマーカーは負け、プラクティスが勝つ。先にオーバーライド行を出力する）。その後 `report --skeleton-stance <on|off|scope-dependent>` を実行する。次の `next` が解決済みの Construction ディレクティブを出力するので、本体やゲートの前にメタデータによるルーティングを適用する。前のステージが再出力されると決めつけず、スケルトン優先のユニットのルートを含め、そのステージとユニットに従う。分類ルールの全体はコンダクターのペルソナを参照。

**ユニットごとのイテレーション（`directive.unit`）。** Construction のメタデータによるルーティングでチェックポイントと完了のみのディレクティブを除外した後、`directive.unit` がある場合、この `run-stage` はユニットごとの Construction ステージ（`for_each: unit-of-work`。3.1〜3.4 の設計ステージと code-generation が対象）の 1 回のイテレーションである。このユニットについて質問フローを実行する。`directive.ceremony.summary_confirmation === "on"` の場合にのみ PRE-GENERATION SUMMARY STOP を適用し、両方のチェックポイントのログコマンドに `--unit "<directive.unit>"` を渡す。サマリー確認がオフの場合は、チェックポイントもレシートもなしで直接生成する。その後、本体を実行して成果物を `construction/<directive.unit>/<directive.stage>/` の下に書き込む。`directive.reviewer` がある場合にのみ、このユニットだけを対象に stage-protocol-reviewer.md §12a に従う。ループはエンジンが駆動する。ユニットごとのディレクティブで `directive.gate` が **false** の場合は、レシートに裏付けられた成果物の作業の後に `next` を再実行する（承認を報告してはならず、ゲートを提示してもならない）。エンジンが次の未カバーのユニットを渡し、すべてのユニットがビルドされると、`gate: true` を付けてこのステージを再出力する。残っているユニットごとのステージゲートについては、`construction_policy.human_completion_required` を使って、通常の人間による完了か自動の報告かを選ぶ。ポリシーのない旧来の経路だけが単一の人間によるステージゲートを維持し、`directive.protocol_modules` に `learnings` がある場合にのみ §13 の手順を実行する。チェックポイントと完了のみのディレクティブは、この本体の手順より前にすでに分岐していなければならない。レビューの計上と通常の予算がある場合はユニットごとである。無効になった終端のレシートには、そのユニットについて同じく 1 回限りの上限付きの古いレシートの復旧が適用される。コンパイル済みのユニット DAG がないために `directive.unit` がない場合は、Bolt やユニットごとの手続きなしで通常のステージのイテレーションを 1 回実行する。unit-major の Construction イテレーションが記録されている場合（`Construction Iteration: unit-major`）、エンジンは状態の Current Stage より後の Construction ステージ（unit-major の巡回が対象とする code-generation を含む）を示す `directive.stage` を出力することがある。常にディレクティブ自身の `directive.stage` + `directive.unit` に基づいて行動し、Current Stage に基づいて行動してはならない。

---

### Kiro CLI

- **`gate: "unresolved"`** — 最初の Construction のルートは**ウォーキングスケルトンのスタンス**に依存するが、チームの自由記述の `## Walking Skeleton` のプラクティスの散文からは、どのパーサーもそれを導出できない。これはあなたのナレッジ作業であり、結果をエンジンに返す。まだステージ本体を実行してはならない。代わりに、`## Walking Skeleton` セクションを読み（解決順は `aidlc/spaces/<space>/memory/org.md` → `team.md` → `project.md`。最も具体的な空でない記述が優先される）、スタンスを分類する。**"always"/"every greenfield feature"** → `on`、**"never"** → `off`、**"scope-dependent"/未指定/空** → `scope-dependent`（その場合、エンジンはアクティブなスコープファイルの `skeleton:` フィールドを使う）。`PRACTICES_OVERRIDE` の判断を尊重する（プラクティスと矛盾する bolt-plan のマーカーは負け、プラクティスが勝つ。先にオーバーライド行を出力する）。その後 `report --skeleton-stance <on|off|scope-dependent>` を実行する。次の `next` が解決済みの Construction ディレクティブを出力するので、本体やゲートの前にメタデータによるルーティングを適用する。前のステージが再出力されると決めつけず、スケルトン優先のユニットのルートを含め、そのステージとユニットに従う。分類ルールの全体はコンダクターのペルソナを参照。

**ユニットごとのイテレーション（`directive.unit`）。** Construction のメタデータによるルーティングでチェックポイントと完了のみのディレクティブを除外した後、`directive.unit` がある場合、この `run-stage` はユニットごとの Construction ステージ（`for_each: unit-of-work`。3.1〜3.4 の設計ステージと code-generation が対象）の 1 回のイテレーションである。このユニットについて質問フローを実行する。`directive.ceremony.summary_confirmation === "on"` の場合にのみ PRE-GENERATION SUMMARY STOP を適用し、両方のチェックポイントのログコマンドに `--unit "<directive.unit>"` を渡す。サマリー確認がオフの場合は、チェックポイントもレシートもなしで直接生成する。その後、本体を実行して成果物を `construction/<directive.unit>/<directive.stage>/` の下に書き込む。`directive.reviewer` がある場合にのみ、このユニットだけを対象に stage-protocol-reviewer.md §12a に従う。ループはエンジンが駆動する。ユニットごとのディレクティブで `directive.gate` が **false** の場合は、レシートに裏付けられた成果物の作業の後に `next` を再実行する（承認を報告してはならず、ゲートを提示してもならない）。エンジンが次の未カバーのユニットを渡し、すべてのユニットがビルドされると、`gate: true` を付けてこのステージを再出力する。残っているユニットごとのステージゲートについては、`construction_policy.human_completion_required` を使って、通常の人間による完了か自動の報告かを選ぶ。ポリシーのない旧来の経路だけが単一の人間によるステージゲートを維持し、`directive.protocol_modules` に `learnings` がある場合にのみ §13 の手順を実行する。チェックポイントと完了のみのディレクティブは、この本体の手順より前にすでに分岐していなければならない。レビューの計上と通常の予算がある場合はユニットごとである。無効になった終端のレシートには、そのユニットについて同じく 1 回限りの上限付きの古いレシートの復旧が適用される。コンパイル済みのユニット DAG がないために `directive.unit` がない場合は、Bolt やユニットごとの手続きなしで通常のステージのイテレーションを 1 回実行する。unit-major の Construction イテレーションが記録されている場合（`Construction Iteration: unit-major`）、エンジンは状態の Current Stage より後の Construction ステージ（unit-major の巡回が対象とする code-generation を含む）を示す `directive.stage` を出力することがある。常にディレクティブ自身の `directive.stage` + `directive.unit` に基づいて行動し、Current Stage に基づいて行動してはならない。

---

### Kiro IDE

- **`gate: "unresolved"`** — 最初の Construction のルートは**ウォーキングスケルトンのスタンス**に依存するが、チームの自由記述の `## Walking Skeleton` のプラクティスの散文からは、どのパーサーもそれを導出できない。これはあなたのナレッジ作業であり、結果をエンジンに返す。まだステージ本体を実行してはならない。代わりに、`## Walking Skeleton` セクションを読み（解決順は `aidlc/spaces/<space>/memory/org.md` → `team.md` → `project.md`。最も具体的な空でない記述が優先される）、スタンスを分類する。**"always"/"every greenfield feature"** → `on`、**"never"** → `off`、**"scope-dependent"/未指定/空** → `scope-dependent`（その場合、エンジンはアクティブなスコープファイルの `skeleton:` フィールドを使う）。`PRACTICES_OVERRIDE` の判断を尊重する（プラクティスと矛盾する bolt-plan のマーカーは負け、プラクティスが勝つ。先にオーバーライド行を出力する）。その後 `report --skeleton-stance <on|off|scope-dependent>` を実行する。次の `next` が解決済みの Construction ディレクティブを出力するので、本体やゲートの前にメタデータによるルーティングを適用する。前のステージが再出力されると決めつけず、スケルトン優先のユニットのルートを含め、そのステージとユニットに従う。分類ルールの全体はコンダクターのペルソナを参照。

**ユニットごとのイテレーション（`directive.unit`）。** Construction のメタデータによるルーティングでチェックポイントと完了のみのディレクティブを除外した後、`directive.unit` がある場合、この `run-stage` はユニットごとの Construction ステージ（`for_each: unit-of-work`。3.1〜3.4 の設計ステージと code-generation が対象）の 1 回のイテレーションである。このユニットについて質問フローを実行する。`directive.ceremony.summary_confirmation === "on"` の場合にのみ PRE-GENERATION SUMMARY STOP を適用し、両方のチェックポイントのログコマンドに `--unit "<directive.unit>"` を渡す。サマリー確認がオフの場合は、チェックポイントもレシートもなしで直接生成する。その後、本体を実行して成果物を `construction/<directive.unit>/<directive.stage>/` の下に書き込む。`directive.reviewer` がある場合にのみ、このユニットだけを対象に stage-protocol-reviewer.md §12a に従う。ループはエンジンが駆動する。ユニットごとのディレクティブで `directive.gate` が **false** の場合は、レシートに裏付けられた成果物の作業の後に `next` を再実行する（承認を報告してはならず、ゲートを提示してもならない）。エンジンが次の未カバーのユニットを渡し、すべてのユニットがビルドされると、`gate: true` を付けてこのステージを再出力する。残っているユニットごとのステージゲートについては、`construction_policy.human_completion_required` を使って、通常の人間による完了か自動の報告かを選ぶ。ポリシーのない旧来の経路だけが単一の人間によるステージゲートを維持し、`directive.protocol_modules` に `learnings` がある場合にのみ §13 の手順を実行する。チェックポイントと完了のみのディレクティブは、この本体の手順より前にすでに分岐していなければならない。レビューの計上と通常の予算がある場合はユニットごとである。無効になった終端のレシートには、そのユニットについて同じく 1 回限りの上限付きの古いレシートの復旧が適用される。コンパイル済みのユニット DAG がないために `directive.unit` がない場合は、Bolt やユニットごとの手続きなしで通常のステージのイテレーションを 1 回実行する。unit-major の Construction イテレーションが記録されている場合（`Construction Iteration: unit-major`）、エンジンは状態の Current Stage より後の Construction ステージ（unit-major の巡回が対象とする code-generation を含む）を示す `directive.stage` を出力することがある。常にディレクティブ自身の `directive.stage` + `directive.unit` に基づいて行動し、Current Stage に基づいて行動してはならない。

---

### Codex CLI

- **`gate: "unresolved"`** — 最初の Construction のルートは**ウォーキングスケルトンのスタンス**に依存するが、チームの自由記述の `## Walking Skeleton` のプラクティスの散文からは、どのパーサーもそれを導出できない。これはあなたのナレッジ作業であり、結果をエンジンに返す。まだステージ本体を実行してはならない。代わりに、`## Walking Skeleton` セクションを読み（解決順は `aidlc/spaces/<space>/memory/org.md` → `team.md` → `project.md`。最も具体的な空でない記述が優先される）、スタンスを分類する。**"always"/"every greenfield feature"** → `on`、**"never"** → `off`、**"scope-dependent"/未指定/空** → `scope-dependent`（その場合、エンジンはアクティブなスコープファイルの `skeleton:` フィールドを使う）。`PRACTICES_OVERRIDE` の判断を尊重する（プラクティスと矛盾する bolt-plan のマーカーは負け、プラクティスが勝つ。先にオーバーライド行を出力する）。その後 `report --skeleton-stance <on|off|scope-dependent>` を実行する。次の `next` が解決済みの Construction ディレクティブを出力するので、本体やゲートの前にメタデータによるルーティングを適用する。前のステージが再出力されると決めつけず、スケルトン優先のユニットのルートを含め、そのステージとユニットに従う。分類ルールの全体はコンダクターのペルソナを参照。

**ユニットごとのイテレーション（`directive.unit`）。** Construction のメタデータによるルーティングでチェックポイントと完了のみのディレクティブを除外した後、`directive.unit` がある場合、この `run-stage` はユニットごとの Construction ステージ（`for_each: unit-of-work`。3.1〜3.4 の設計ステージと code-generation が対象）の 1 回のイテレーションである。このユニットについて質問フローを実行する。`directive.ceremony.summary_confirmation === "on"` の場合にのみ PRE-GENERATION SUMMARY STOP を適用し、両方のチェックポイントのログコマンドに `--unit "<directive.unit>"` を渡す。サマリー確認がオフの場合は、チェックポイントもレシートもなしで直接生成する。その後、本体を実行して成果物を `construction/<directive.unit>/<directive.stage>/` の下に書き込む。`directive.reviewer` がある場合にのみ、このユニットだけを対象に stage-protocol-reviewer.md §12a に従う。ループはエンジンが駆動する。ユニットごとのディレクティブで `directive.gate` が **false** の場合は、レシートに裏付けられた成果物の作業の後に `next` を再実行する（承認を報告してはならず、ゲートを提示してもならない）。エンジンが次の未カバーのユニットを渡し、すべてのユニットがビルドされると、`gate: true` を付けてこのステージを再出力する。残っているユニットごとのステージゲートについては、`construction_policy.human_completion_required` を使って、通常の人間による完了か自動の報告かを選ぶ。ポリシーのない旧来の経路だけが単一の人間によるステージゲートを維持し、`directive.protocol_modules` に `learnings` がある場合にのみ §13 の手順を実行する。チェックポイントと完了のみのディレクティブは、この本体の手順より前にすでに分岐していなければならない。レビューの計上と通常の予算がある場合はユニットごとである。無効になった終端のレシートには、そのユニットについて同じく 1 回限りの上限付きの古いレシートの復旧が適用される。コンパイル済みのユニット DAG がないために `directive.unit` がない場合は、Bolt やユニットごとの手続きなしで通常のステージのイテレーションを 1 回実行する。unit-major の Construction イテレーションが記録されている場合（`Construction Iteration: unit-major`）、エンジンは状態の Current Stage より後の Construction ステージ（unit-major の巡回が対象とする code-generation を含む）を示す `directive.stage` を出力することがある。常にディレクティブ自身の `directive.stage` + `directive.unit` に基づいて行動し、Current Stage に基づいて行動してはならない。

---

### Cursor

- **`gate: "unresolved"`** — 最初の Construction のルートは**ウォーキングスケルトンのスタンス**に依存するが、チームの自由記述の `## Walking Skeleton` のプラクティスの散文からは、どのパーサーもそれを導出できない。これはあなたのナレッジ作業であり、結果をエンジンに返す。まだステージ本体を実行してはならない。代わりに、`## Walking Skeleton` セクションを読み（解決順は `aidlc/spaces/<space>/memory/org.md` → `team.md` → `project.md`。最も具体的な空でない記述が優先される）、スタンスを分類する。**"always"/"every greenfield feature"** → `on`、**"never"** → `off`、**"scope-dependent"/未指定/空** → `scope-dependent`（その場合、エンジンはアクティブなスコープファイルの `skeleton:` フィールドを使う）。`PRACTICES_OVERRIDE` の判断を尊重する（プラクティスと矛盾する bolt-plan のマーカーは負け、プラクティスが勝つ。先にオーバーライド行を出力する）。その後 `report --skeleton-stance <on|off|scope-dependent>` を実行する。次の `next` が解決済みの Construction ディレクティブを出力するので、本体やゲートの前にメタデータによるルーティングを適用する。前のステージが再出力されると決めつけず、スケルトン優先のユニットのルートを含め、そのステージとユニットに従う。分類ルールの全体はコンダクターのペルソナを参照。

**ユニットごとのイテレーション（`directive.unit`）。** Construction のメタデータによるルーティングでチェックポイントと完了のみのディレクティブを除外した後、`directive.unit` がある場合、この `run-stage` はユニットごとの Construction ステージ（`for_each: unit-of-work`。3.1〜3.4 の設計ステージと code-generation が対象）の 1 回のイテレーションである。このユニットについて質問フローを実行する。`directive.ceremony.summary_confirmation === "on"` の場合にのみ PRE-GENERATION SUMMARY STOP を適用し、両方のチェックポイントのログコマンドに `--unit "<directive.unit>"` を渡す。サマリー確認がオフの場合は、チェックポイントもレシートもなしで直接生成する。その後、本体を実行して成果物を `construction/<directive.unit>/<directive.stage>/` の下に書き込む。`directive.reviewer` がある場合にのみ、このユニットだけを対象に stage-protocol-reviewer.md §12a に従う。ループはエンジンが駆動する。ユニットごとのディレクティブで `directive.gate` が **false** の場合は、レシートに裏付けられた成果物の作業の後に `next` を再実行する（承認を報告してはならず、ゲートを提示してもならない）。エンジンが次の未カバーのユニットを渡し、すべてのユニットがビルドされると、`gate: true` を付けてこのステージを再出力する。残っているユニットごとのステージゲートについては、`construction_policy.human_completion_required` を使って、通常の人間による完了か自動の報告かを選ぶ。ポリシーのない旧来の経路だけが単一の人間によるステージゲートを維持し、`directive.protocol_modules` に `learnings` がある場合にのみ §13 の手順を実行する。チェックポイントと完了のみのディレクティブは、この本体の手順より前にすでに分岐していなければならない。レビューの計上と通常の予算がある場合はユニットごとである。無効になった終端のレシートには、そのユニットについて同じく 1 回限りの上限付きの古いレシートの復旧が適用される。コンパイル済みのユニット DAG がないために `directive.unit` がない場合は、Bolt やユニットごとの手続きなしで通常のステージのイテレーションを 1 回実行する。unit-major の Construction イテレーションが記録されている場合（`Construction Iteration: unit-major`）、エンジンは状態の Current Stage より後の Construction ステージ（unit-major の巡回が対象とする code-generation を含む）を示す `directive.stage` を出力することがある。常にディレクティブ自身の `directive.stage` + `directive.unit` に基づいて行動し、Current Stage に基づいて行動してはならない。

---

### opencode

- **`gate: "unresolved"`** — 最初の Construction のルートは**ウォーキングスケルトンのスタンス**に依存するが、チームの自由記述の `## Walking Skeleton` のプラクティスの散文からは、どのパーサーもそれを導出できない。これはあなたのナレッジ作業であり、結果をエンジンに返す。まだステージ本体を実行してはならない。代わりに、`## Walking Skeleton` セクションを読み（解決順は `aidlc/spaces/<space>/memory/org.md` → `team.md` → `project.md`。最も具体的な空でない記述が優先される）、スタンスを分類する。**"always"/"every greenfield feature"** → `on`、**"never"** → `off`、**"scope-dependent"/未指定/空** → `scope-dependent`（その場合、エンジンはアクティブなスコープファイルの `skeleton:` フィールドを使う）。`PRACTICES_OVERRIDE` の判断を尊重する（プラクティスと矛盾する bolt-plan のマーカーは負け、プラクティスが勝つ。先にオーバーライド行を出力する）。その後 `report --skeleton-stance <on|off|scope-dependent>` を実行する。次の `next` が解決済みの Construction ディレクティブを出力するので、本体やゲートの前にメタデータによるルーティングを適用する。前のステージが再出力されると決めつけず、スケルトン優先のユニットのルートを含め、そのステージとユニットに従う。分類ルールの全体はコンダクターのペルソナを参照。

**ユニットごとのイテレーション（`directive.unit`）。** Construction のメタデータによるルーティングでチェックポイントと完了のみのディレクティブを除外した後、`directive.unit` がある場合、この `run-stage` はユニットごとの Construction ステージ（`for_each: unit-of-work`。3.1〜3.4 の設計ステージと code-generation が対象）の 1 回のイテレーションである。このユニットについて質問フローを実行する。`directive.ceremony.summary_confirmation === "on"` の場合にのみ PRE-GENERATION SUMMARY STOP を適用し、両方のチェックポイントのログコマンドに `--unit "<directive.unit>"` を渡す。サマリー確認がオフの場合は、チェックポイントもレシートもなしで直接生成する。その後、本体を実行して成果物を `construction/<directive.unit>/<directive.stage>/` の下に書き込む。`directive.reviewer` がある場合にのみ、このユニットだけを対象に stage-protocol-reviewer.md §12a に従う。ループはエンジンが駆動する。ユニットごとのディレクティブで `directive.gate` が **false** の場合は、レシートに裏付けられた成果物の作業の後に `next` を再実行する（承認を報告してはならず、ゲートを提示してもならない）。エンジンが次の未カバーのユニットを渡し、すべてのユニットがビルドされると、`gate: true` を付けてこのステージを再出力する。残っているユニットごとのステージゲートについては、`construction_policy.human_completion_required` を使って、通常の人間による完了か自動の報告かを選ぶ。ポリシーのない旧来の経路だけが単一の人間によるステージゲートを維持し、`directive.protocol_modules` に `learnings` がある場合にのみ §13 の手順を実行する。チェックポイントと完了のみのディレクティブは、この本体の手順より前にすでに分岐していなければならない。レビューの計上と通常の予算がある場合はユニットごとである。無効になった終端のレシートには、そのユニットについて同じく 1 回限りの上限付きの古いレシートの復旧が適用される。コンパイル済みのユニット DAG がないために `directive.unit` がない場合は、Bolt やユニットごとの手続きなしで通常のステージのイテレーションを 1 回実行する。unit-major の Construction イテレーションが記録されている場合（`Construction Iteration: unit-major`）、エンジンは状態の Current Stage より後の Construction ステージ（unit-major の巡回が対象とする code-generation を含む）を示す `directive.stage` を出力することがある。常にディレクティブ自身の `directive.stage` + `directive.unit` に基づいて行動し、Current Stage に基づいて行動してはならない。

---

### GitHub Copilot

- **`gate: "unresolved"`** — 最初の Construction のルートは**ウォーキングスケルトンのスタンス**に依存するが、チームの自由記述の `## Walking Skeleton` のプラクティスの散文からは、どのパーサーもそれを導出できない。これはあなたのナレッジ作業であり、結果をエンジンに返す。まだステージ本体を実行してはならない。代わりに、`## Walking Skeleton` セクションを読み（解決順は `aidlc/spaces/<space>/memory/org.md` → `team.md` → `project.md`。最も具体的な空でない記述が優先される）、スタンスを分類する。**"always"/"every greenfield feature"** → `on`、**"never"** → `off`、**"scope-dependent"/未指定/空** → `scope-dependent`（その場合、エンジンはアクティブなスコープファイルの `skeleton:` フィールドを使う）。`PRACTICES_OVERRIDE` の判断を尊重する（プラクティスと矛盾する bolt-plan のマーカーは負け、プラクティスが勝つ。先にオーバーライド行を出力する）。その後 `report --skeleton-stance <on|off|scope-dependent>` を実行する。次の `next` が解決済みの Construction ディレクティブを出力するので、本体やゲートの前にメタデータによるルーティングを適用する。前のステージが再出力されると決めつけず、スケルトン優先のユニットのルートを含め、そのステージとユニットに従う。分類ルールの全体はコンダクターのペルソナを参照。

**ユニットごとのイテレーション（`directive.unit`）。** Construction のメタデータによるルーティングでチェックポイントと完了のみのディレクティブを除外した後、`directive.unit` がある場合、この `run-stage` はユニットごとの Construction ステージ（`for_each: unit-of-work`。3.1〜3.4 の設計ステージと code-generation が対象）の 1 回のイテレーションである。このユニットについて質問フローを実行する。`directive.ceremony.summary_confirmation === "on"` の場合にのみ PRE-GENERATION SUMMARY STOP を適用し、両方のチェックポイントのログコマンドに `--unit "<directive.unit>"` を渡す。サマリー確認がオフの場合は、チェックポイントもレシートもなしで直接生成する。その後、本体を実行して成果物を `construction/<directive.unit>/<directive.stage>/` の下に書き込む。`directive.reviewer` がある場合にのみ、このユニットだけを対象に stage-protocol-reviewer.md §12a に従う。ループはエンジンが駆動する。ユニットごとのディレクティブで `directive.gate` が **false** の場合は、レシートに裏付けられた成果物の作業の後に `next` を再実行する（承認を報告してはならず、ゲートを提示してもならない）。エンジンが次の未カバーのユニットを渡し、すべてのユニットがビルドされると、`gate: true` を付けてこのステージを再出力する。残っているユニットごとのステージゲートについては、`construction_policy.human_completion_required` を使って、通常の人間による完了か自動の報告かを選ぶ。ポリシーのない旧来の経路だけが単一の人間によるステージゲートを維持し、`directive.protocol_modules` に `learnings` がある場合にのみ §13 の手順を実行する。チェックポイントと完了のみのディレクティブは、この本体の手順より前にすでに分岐していなければならない。レビューの計上と通常の予算がある場合はユニットごとである。無効になった終端のレシートには、そのユニットについて同じく 1 回限りの上限付きの古いレシートの復旧が適用される。コンパイル済みのユニット DAG がないために `directive.unit` がない場合は、Bolt やユニットごとの手続きなしで通常のステージのイテレーションを 1 回実行する。unit-major の Construction イテレーションが記録されている場合（`Construction Iteration: unit-major`）、エンジンは状態の Current Stage より後の Construction ステージ（unit-major の巡回が対象とする code-generation を含む）を示す `directive.stage` を出力することがある。常にディレクティブ自身の `directive.stage` + `directive.unit` に基づいて行動し、Current Stage に基づいて行動してはならない。

### Grouped Plan Approval

複数のユニットの計画が同時に準備できたときは、エンジンはそれらを 1 つの
質問で尋ねる。`plan_approval.targets` には各ユニットのサマリーと計画のパスが入り、
選択肢は **Approve all**、**Request Changes**、**I'll edit the files** である。
質問の下にすべてのユニットのサマリー行を表示し、ターンを終える。利用者の返信を読み、
1 つの計画の場合と同じように、利用者が選んだ選択肢を記録する:
`{{INVOKE}} engine log answer --stage code-generation --checkpoint plan-approval
--details "Approve Plan"`（または `'Request Changes: <what they asked>'`、または
`"I'll edit the files"`）。一部のユニットについて選択を記録するには `--units "<unit>,<unit>"` を付け、
その後残りを記録する。その後 `next` を実行する。"Approve all" はすべてのユニットを承認する。
ユニットを指定した変更（"change billing: use Stripe"）の場合は、そのユニットに
`Request Changes: use Stripe` を、残りに Approve Plan を記録する。
ユニットを指定しない変更の場合は、どの計画を変更すべきかを 1 回だけ尋ねる。正確な
"Approve all" や "I'll edit the files" は自動的に記録される。単なる Request
Changes の場合は、どの計画かを知る必要がある。どのユニットも引き続き、
それぞれの計画に紐付いた個別の承認記録を持つ。グループ化は質問の表示方法を変えるだけであり、
後のバッチを承認することは決してない。
