# Audit Event Taxonomy

**イベント名は必ずこの表と完全に一致させる。** 新しいイベント種別を作り出してはならない。ステージの完了には常に `STAGE_COMPLETED` を使い、"Requirements Analysis Complete" や "Code Generated" のようなステージ固有の名前で代用してはならない。

ステージのライフサイクルは report が担う。コンダクターは、ゲート・修正・完了・スキップの各結果を
`tools/aidlc-orchestrate.ts report` を通じて要求する。
以下の state ツールの項目はエンジン内部のアトミックな発行元であり、
ステージやコンダクターが直接呼び出すコマンドではない。

> 各イベントを発行する状態遷移については [`docs/reference/12-state-machine.md`](../../../../docs/reference/12-state-machine.md) を参照。`✓` の付いたイベントは必須（MANDATORY）であり、`tests/feature/t48-audit-event-emitters.sh` で検証される。

## Naming Convention

すべてのイベント名は `SUBJECT_PAST_VERB` 形式に従う。どのイベントも「何が起きたか？」に答える。

監査証跡の目的、それを担うコマンド、読み取り用コマンドについては
[監査証跡のルール](../../aidlc-common/protocols/stage-protocol.md#audit-trail-rules) で説明している。
このレジストリは各イベントの発行元とデータを示すものであり、ファイル書き込み用の
テンプレートではない。タイムスタンプとイベントの識別は発行元が担う。

## Emitter-Owned Fields

構造化レンダラーは、ブロックごとにちょうど 1 つの `Timestamp` 行と 1 つの `Event` 行を書き込む。
呼び出し側はどちらのフィールドも指定してはならない。互換性のため、汎用の
`audit append --field Timestamp=...` 形式は引き続き受け付けるが、その値は
意図的に無視される。過去のシャードは書き換えられない。ファイル全体を解析する読み取り側は、
`---` で分割して各ブロックの最初のタイムスタンプを使うか、
旧バージョンが生成したタイムスタンプフィールドの重複を除去しなければならない。

**Person Reply.** イベントが `Person Reply` を持つ場合、そこには質問が表示されてからの
利用者自身のメッセージが、ハーネスが届けたままの形でトリミングされて格納される。これは
human-turn フックが保持したものであり、コンダクターによるその解釈はイベントの選択フィールドに入る。
フックが保持する量には上限があるため、このフィールドは実際に発言された内容より
短くなることがある:
- ステージゲートは、ゲートが表示された後にそのチャットで入力されたメッセージを最大 8 件保持する。
  9 件目が届いた場合、または 1 件のメッセージが 8000 文字を超えた場合は、どれも
  添付されない。フィールドは存在せず、変更要求にはコンダクターの
  `--reason` が代わりに記録される。
- Plan Approval は最新の返信 8 件を、それぞれ 8000 文字に切り詰めて保持する。
- verification-command、Construction ポリシー、チェックポイントの質問は、
  返信を順に連結したものの末尾 8000 文字を保持する。
- ピッカーで選ばれた選択肢は、それがステージゲート自身の選択肢である場合にのみ保持される。
  それ以外の選択されたラベルはコンダクターのテキストであり、利用者の言葉ではない。

**Message Ids.** Person Reply（またはステージ変更要求の Feedback）が複数のメッセージの言葉を
含む行は、`Message Id`（最新のもの）に加えて、`Message Ids` にそのすべてを
古い順にカンマ区切りで列挙する。これにより、行の言葉と ID が対応する。
1 件のメッセージに基づく行、またはコンダクターが `--message` で指定したメッセージに基づく行は、
`Message Id` のみを持つ。

**Git Name.** 利用者の判断を記録する行は `Git Name` を持つ。これは、
このプロジェクトで利用者の git が報告する名前（`git config user.name`。リポジトリ
固有の設定を優先し、次にグローバル設定）であり、ユニットボードに表示される名前と同じである。
これは git が報告する範囲で、誰がキーボードの前にいたかを示すものである。署名でも
本人確認の証明でもない。git には誰でも任意の名前を設定できる。AI-DLC はこれを読み取るだけで、
設定したり、尋ねたり、メールアドレスを記録したりすることは決してない。git に名前がない場合や、
無人実行（`AIDLC_UNATTENDED=1`）の場合は、このフィールドは存在せず、行は
フィールドがない場合と同じ内容になる。このフィールドを持つ行:
- 利用者が判断した `GATE_APPROVED` と `GATE_REJECTED`: ステージゲート、チームの
  Unit ゲート、ユニットまたはバッチのチェックポイント、ユニットのマージゲート、および利用者の
  ジャンプによって書き込まれる再オープン。自律的な Construction の判断（その
  Build and Test のループバックを含む）や、修正バックストップの `Recovered`
  行には決して付かない。
- `PLAN_APPROVAL_RECORDED`、`PLAN_APPROVAL_OVERRIDDEN`、
  `SUMMARY_CONFIRMATION_RECORDED`、`VERIFICATION_COMMAND_RECORDED`、
  `CONSTRUCTION_POLICY_RECORDED`。
- `STAGE_JUMPED`（Construction が自律的に実行中にそこへ着地するものを除く）。
  Reason が `the person asked to skip it` で始まる `STAGE_SKIPPED` と `UNIT_SKIPPED`。
  利用者が求めたスキップ、または compose の質問での利用者の承認による `RECOMPOSED`。
- `Source: you` を持つ `GUARD_POLICY_SET`、`CEREMONY_SET`、`GUARD_DISABLED`、`GUARD_RESTORED`。
- 利用者の `Message Id` を持つ `CHANGE_ACCEPTED`。

## Event Registry (134 events, 27 categories)

### Workflow Lifecycle (9 events)

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| ✓ `WORKFLOW_STARTED` | スコープが決まり、ワークフローが始まる | Timestamp, Scope, Request。任意で Source Baseline（`sha256:<listing-hash>` または `unbindable`）。この作業のために組み立てたプランでは Plan, Stages skipped, Stages added。コンポーザーが調整したプランでは ARS Scores（`IAE 0.35, CSU 0.39, VE 0.40, R 0.35, UA 0.30`）と ARS Method（`codekb` または `fallback`） | `tools/aidlc-utility.ts intent-create` |
| ✓ `WORKFLOW_COMPLETED` | スコープ内のすべてのステージが完了 | Timestamp, Scope, Details | `tools/aidlc-state.ts complete-workflow` |
| ✓ `WORKFLOW_PARKED` | 後のセッションのためにワークフローを途中で中断（どのステージも前進しない） | Timestamp, Stage。任意で Person Reply（利用者が中断したときに言ったこと。フックが保持した場合）、Message Id（それを含むメッセージ）、Message Ids（上記 Message Ids を参照） | `tools/aidlc-state.ts park` |
| ✓ `WORKFLOW_UNPARKED` | 明示的な `--resume` による再開で中断マーカーが解除された | Timestamp | `tools/aidlc-state.ts unpark` |
| ✓ `WORKFLOW_ARCHIVED` | 進行中または完了済みのインテントが人によって退役された（`Status: Archived`、レジストリは `archived`）。記録と監査シャードは保持される | Timestamp, Stage。任意で Reason | `tools/aidlc-utility.ts intent archive` |
| ✓ `WORKFLOW_UNARCHIVED` | アーカイブされたインテントが `Running` / `in-flight` に戻された。完了状態でアーカイブされていた場合は `Completed` / `complete` に戻る | Timestamp, Stage | `tools/aidlc-utility.ts intent unarchive` |
| `STATE_UPGRADED` | 以前の State Version（7、2.1.x から 2.3.0）が書いた記録を、エンジンが初めて読み込んだときにその場で現行版へ引き上げた。名前が変わったステージはチェックボックスとプランのアクションを保持し、記録フォルダーは新しい名前へ移動し、追加されたステージは、新しいリリースで追加されたステージと同じ方法で配置される。何も尋ねず、何も告げない | Timestamp, From, To, Renamed（`<old> to <new>`）, Stages added, Moved（記録フォルダー、または `none`）, Original（元の状態。その横に保存される） | `tools/aidlc-state-upgrade.ts`（エンジンの状態読み込み） |
| `INTENT_REPOS_CHANGED` | すでにリポジトリを記録している作業に対し、任意のステージで兄弟リポジトリが追加または削除された（`intent add-repo <name>` / `intent remove-repo <name>`、またはセット全体に対する `intent set-repos <name,...>`）。まだここにクローンされていないリポジトリは先にクローンとインストールが行われ、クローンに失敗した場合は何も記録しない。そのインテント自身のシャードに書き込まれる | Timestamp, Stage, Repos（変更後のセット）, Added または Removed（対象リポジトリ。set-repos ではすべての名前を列挙）, 任意で Cloned From（クローン元のアドレス。userinfo はマスクされる）, 任意で Reason（エージェントの `--reason`）, 任意で Person Words（利用者の依頼。`--person-words`）, 任意で Source Baseline Addition（追加時点での追加リポジトリのファイルの `sha256:<listing-hash>`。その前に取得されたステージ開始時のソースベースラインに加わる）, set-repos の Named（`by the person`） | `tools/aidlc-utility.ts intent add-repo` / `intent remove-repo` / `intent set-repos` |
| `INTENT_CHECKOUTS_CHANGED` | 作業のコードが、レビューから通常は見えない場所にあるチェックアウト（ハーネスフォルダー内のワークツリー、またはプロジェクト外のもの）について、レビュー用に記録された、または再び除外された（`intent add-checkout <path>` / `intent remove-checkout <path>`）。そのインテント自身のシャードに書き込まれる。パスは記録用のテキストであり、設定として読まれるのはインテントのクローンごとの `.aidlc-engine/source-checkouts.json` のみである | Timestamp, Stage, Path（入力されたまま）, Checkouts（変更後のラベル、または `none`）, Added または Removed（その 1 つのラベル） | `tools/aidlc-utility.ts intent add-checkout` / `intent remove-checkout` |

### Phase Lifecycle (4 events)

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| ✓ `PHASE_STARTED` | フェーズが始まる（スコープ内の最初のステージが実行される直前） | Timestamp, Phase, Stage count, Scope | `tools/aidlc-utility.ts intent-create`（Init フェーズ）, `tools/aidlc-state.ts advance`（フェーズ境界） |
| ✓ `PHASE_COMPLETED` | フェーズ境界を越えた | Timestamp, From phase, To phase, Stages completed | `tools/aidlc-state.ts advance`, `tools/aidlc-state.ts complete-workflow` |
| `PHASE_VERIFIED` | 境界でのトレーサビリティ検査 | Timestamp, Phase boundary, Pass/fail, Issues | `tools/aidlc-state.ts advance`, `tools/aidlc-state.ts complete-workflow` |
| `PHASE_SKIPPED` | スコープがフェーズを除外している、または前方ジャンプがそのフェーズを飛び越えた | Timestamp, Phase, Scope, Reason | `tools/aidlc-utility.ts intent-create`（フェーズごとのスコープ評価）, `tools/aidlc-jump.ts execute`（前方ジャンプ） |

### Stage Lifecycle (9 events)

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| ✓ `STAGE_STARTED` | ステージが `[-]` Active に入る、または分離された試行が開始される | Timestamp, Stage, Agent。入ったステージが `workspace_requires` を宣言している場合は任意で Source Baseline。分離された行には、完了を選択されたセレモニーポリシーに結び付けるための Workflow（`single-stage:<slug>`）と Scope が含まれる。ステージが回答モードを尋ねる代わりに利用者の以前の回答モードを再利用する場合は任意で Answer Mode（例: `guide (reused from requirements-analysis)`） | `tools/aidlc-state.ts advance`, `tools/aidlc-utility.ts intent-create`（init ステージ）, `tools/aidlc-orchestrate.ts next --single`（分離された試行） |
| `STAGE_AWAITING_APPROVAL` | ステージが `[?]` に入る（ゲートが開く） | Timestamp, Stage, Artifacts, `Next Stage`（ゲートの返信が計算する、承認後に続くスコープ内のステージ、または `none`）, `Brief Digest`（このゲートについてこれまでに保持されたレビューブリーフの `sha256:<hex>`。保存先は `.aidlc-engine/reviews/<stage>/.../briefs/`。エージェントはこの行の後にブリーフを描画するため初回のオープンでは通常 `none` で、再入時に設定される。承認行には利用者が判断したときのダイジェストが入る）, `Sensor State`（適用されるセンサーと宣言された成果物ごとの `<id>@<path>=passed|failed(<n>) [detail: <path>]|not-run`。失敗を先頭に、最大 20 件まで明記）, `Open Decisions`（成果物が `aidlc-decisions` ブロックで利用者に委ねた判断の件数。`Decisions` にはその ID を列挙。エンジンが読めなかったブロックは `unreadable (<file>)`。ここではどれもゲートを拒否しない）, ゲートの質問記録（`Question Id`, `Question Type: gate`, `Question Shows`: ブリーフのダイジェストとドキュメントのフィンガープリントに対する `sha256:`, コマンドがチャットを指定する場合は `Question Chat`。Question records を参照）, 任意で `Recovered=true`（後から補完されたゲート行）または `Revalidated=true`（すでに開いているゲートがブロッキングセンサーのオーバーライドを消費した）, 再入時は任意で `Review Kept`（戻ってきた元の Request Changes のタイムスタンプと、その要求の対象となったレビュー。その後ステージに何の変更もなかったため。その場合、その却下は新しいレビュー試行を開始しない）, Approves Together（ユニットチェックポイントがオフの unit-major で全ユニットが構築された後、ゲートの 1 つの質問が名指ししたユニットごとのステージ。カンマ区切り）。ブロッキングセンサーのオーバーライドには、Blocking Sensor Override, Blocking Sensor IDs, 任意で Blocking Sensor Detail Paths, Blocking Sensor Reasons も含まれる。その権限は、直前の正確な `DECISION_RECORDED` → `HUMAN_TURN` → `QUESTION_ANSWERED` の組である。 | `tools/aidlc-state.ts gate-start`（通常、`--recovered` による補完、またはオーバーライドの再検証）, `tools/aidlc-state.ts revise`（ゲートへの再入）, `tools/aidlc-state.ts approve`（オープン時のガードを通過した後のバックストップ再入） |
| `STAGE_REVISING` | ステージが `[R]` に入る（ユーザーがゲートで却下した） | Timestamp, Stage, Revision count, Feedback, 任意で `Recovered=true`（承認時の修正バックストップによる補完） | `tools/aidlc-state.ts reject`, `tools/aidlc-state.ts approve`（バックストップによる補完） |
| ✓ `STAGE_COMPLETED` | ステージが完了する（`[x]`） | Timestamp, Stage, Details, Artifacts | `tools/aidlc-state.ts approve`（ゲート付きステージ。次へも自動で前進する）, `tools/aidlc-state.ts advance`（ゲートなしのステージ）, `tools/aidlc-utility.ts intent-create`（init ステージ） |
| `STAGE_JUMPED` | 前方・後方・やり直しのジャンプ先に到達した | Timestamp, Direction, Source, Target, Scope。任意で Source Baseline。後方ジャンプは Changed Upstream Artifacts, Invalidated Downstream Artifacts, Invalidated Downstream Reviews の JSON 配列も持つ。`Unit Ownership: team` の下で、ユニットごとのブロックより前のステージへ後方ジャンプする場合は `Unit Work: kept` も持つ（各ユニットのユニットごとの作業はそのクレームとともに残る） | `tools/aidlc-jump.ts execute` |
| `STAGE_SKIPPED` | 現在のステージが正当なスキップを報告した、unit-major の下でもはやどのユニットもそのステージを必要としなくなった（各ユニットがスキップ済みか、種類により空）、またはジャンプがそのステージをスキップした（`[S]`） | Timestamp, Stage, Reason | `tools/aidlc-state.ts skip`（内部的には `aidlc-orchestrate.ts report --result skipped` から呼ばれる）, `tools/aidlc-jump.ts execute` |
| `STAGE_DEFINITION_CHANGED` | 完了済みステージ自身の定義が変わり（AI-DLC の更新、プラグイン、ステージの編集）、利用者の作業には何の変更もなかった。次のステップが告げられた（ステージが以前は不要だった入力を必要とするようになった場合にのみ、その旨の一文を含む） | Timestamp, Stage, Validation Basis（新しい定義に対するレシート）, Details（告げた一文、または何も告げなかったこと） | `aidlc-orchestrate.ts` |
| `STAGE_OUTPUT_AMENDED` | 完了済みステージ自身のドキュメントが（内容のみ、ステージのそれ以外は何も変わらずに）、その承認が指すメッセージの次の利用者メッセージより前に変更され、その後に別のメッセージを指す判断や何も指さない判断がなかった。次のステップが告げられ、何も言わなかった | Timestamp, Stage, Validation Basis（現状のファイルに対するレシート）, Message Id（承認が指すメッセージ）, Details（`changed before the person's next message after <id>; receipt renewed`） | `aidlc-orchestrate.ts` |
| `SOURCE_WORK_NOT_FOUND` | コードを生成するステージ（`workspace_requires`）が、探した場所で新しいコードを見つけられなかった。探す場所は、プロジェクトフォルダー、インテントに記録されたリポジトリ、検査時に見つかった子リポジトリとワークツリー、それぞれのリンクされたワークツリー、プロジェクトルートの git が追跡していないプロジェクト内のチェックアウト、およびステージ試行開始以降のソース変更である。Outcome が `told the agent` の場合: この試行でのエージェント自身の最初の報告が拒否された。`put to the person` の場合: 承認の質問が注記付きで利用者に提示された（後の報告、利用者自身の承認、または記録や読み戻しができなかった最初の拒否）。`stood aside` の場合: Guard Policy が off のため作業が続行された。試行ごと・結果ごとに 1 行。承認行には同じ注記が `Source Work` として入る | Timestamp, Stage, Run floor, Outcome, Looked In, Details（告げた一文、またはエージェントに伝えた内容） | `tools/aidlc-state.ts`（`verifyStageArtifacts`） |

### Session Events (6 events - hook-owned, independent of workflow lifecycle)

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `SESSION_STARTED` | 新しいセッションが始まる（source=startup または clear。Kiro IDE ではチャットの最初のプロンプト） | Timestamp, Source | `hooks/aidlc-session-start.ts` |
| `SESSION_RESUMED` | 既存のセッションが再開される（source=resume。Kiro IDE では以前のチャットに戻るプロンプト） | Timestamp, Source | `hooks/aidlc-session-start.ts` |
| `SESSION_COMPACTED` | コンテキストの圧縮が発生した | Timestamp, Current Stage, State Validity | `hooks/aidlc-validate-state.ts`（PreCompact） |
| `SESSION_ENDED` | Claude Code のセッションが終了する | Timestamp, Reason, Session（ホストがチャットに名前を付けた場合） | `hooks/aidlc-session-end.ts` |
| `HUMAN_TURN` | サポートされているプロンプト送信またはウィジェット回答の接点が観測された（承認・インタビューのゲートは、前回のゲート解決以降に 1 件を必要とし、ステージゲートは、ステージ開始以降かつゲート表示後に送られた 1 件、またはエージェントが `--message` で指定したメッセージを必要とする）。ドライバーが `AIDLC_UNATTENDED=1` を宣言している場合や、プロンプトがスウォームワーカーのワークツリーに届いた場合は省略される（ワーカーのブリーフはコンダクターのものであり、利用者のターンではない）。ホストが作成したプロンプトについては決して書き込まれない（`HOST_TURN` を参照） | Timestamp, Session, `Origin: person`, `Source: typed` または `Source: picker`（ターンの到着経路。旧形式のペイロードにはない）, 任意で `Message Id`（`aidlc/.aidlc-sessions/messages/<id>.json` にあるフックによるメッセージの記録。記録を書き込めなかった場合はない）, 任意で `Reply: command`（そのターンが AIDLC へのコマンドのみだった。AIDLC 自身のコマンド構文、エンジンが読むフラグ・スコープ・動詞・名詞を伴う `/aidlc ...`、`/aidlc-<runner> ...`、`$aidlc ...`、または入力されたスイッチ。`/aidlc` や `$aidlc` の後に言葉だけが続くもの、たとえば `/aidlc approve the code plan` は、このエントリなしで保持される返信であり、スラッシュで始まるその他のテキストも同様である）（古い記録の `Reply: question` は "skip plan approval?" のようなスイッチについての質問を示していた。現在はこれを書き込むものはなく、エージェントが質問に答え、読み取り側はそれをコマンドのターンとして扱う）。開いている質問への判断では、コマンドのターンを返信として数えない。任意で `Open Question`（メッセージが届いたときに画面上にあった別のエンジン質問の質問種別。既定では、その下にある質問への判断はこれを数えない。ただしコンダクターが `--message` でメッセージを指定した場合を除く）。任意で `Picked`（ピッカーによる返信の選択内容。利用者が選んだラベルの JSON リスト）。任意で `Open Questions`（返信した場所で開いていた質問の ID。このチャットで表示されたもの、ステージの質問ファイル内のもの、またはチャット不明のコマンドによるもの）と `Questions Digest`（JSON: `file` はエンジンが最後に提示したステージの質問ファイル、`entries` はエントリごとの 16 桁 16 進ダイジェストで、`[Answer]:` 行を空にし改行を正規化したもの）。いずれも内容がある場合にのみ書き込まれる。現時点では、これらに基づいて何かを判断するものはない | `hooks/aidlc-record-human-turn.ts`（UserPromptSubmit + PostToolUse AskUserQuestion）+ ハーネスごとのプロンプト送信アダプター |
| `HOST_TURN` | 人のターンではない: ホストがエージェント向けに作成したプロンプトがプロンプト送信の接点に届き、利用者のターンとして数えられなかった（Kiro Workflows のブリーフや完了通知、Kiro サブエージェントの統合プロンプト、Claude Code のバックグラウンドタスク通知、サブエージェントの引き渡し、スケジュールされた起動（`/loop` のティック、CronCreate タスク、Claude Code 自身が設定するループ）、別の Claude セッションのメッセージ、Claude デスクトップアプリ自身のイベント、終了メモ、セッションブリーフ）。利用者のメッセージとの区別は、ホスト自身の記録（Kiro のチャット記録の `syntheticUserMessageReason`、Claude Code のトランスクリプトの `turnOrigin`）か、アンカー付きのホストの定型文全体によって行う。不明なものはすべて利用者のターンとする。利用者に関するものは何も載らない。メッセージ記録も、保持された言葉も、返信も、入力されたスイッチも、会話上のマーカーもない。Claude Code はプロンプトフックが戻った後にのみターン自身の行を書き込むため、起動は最初 `HUMAN_TURN` として記録される。次のエンジンコマンドまたはフックがその行を読み、そのターンの `Message Id` を指す `HOST_TURN` を追記する。それ以降、その `HUMAN_TURN` はどの読み取り側にとってもターンではなくなり、そのメッセージ記録は削除され、そのために保持されていた返信は取り下げられる | Timestamp, 任意で Session, `Origin: host`, Reason（ホスト自身による呼び名、または定型文の名前）, Source（`record` または `template`）, 任意で Message Id（取り消す対象のターン）。プロンプトのテキストは決して含まない | `hooks/aidlc-record-human-turn.ts`（UserPromptSubmit。Kiro CLI と Kiro IDE のアダプターがそれぞれのホストについて判断する）; `tools/aidlc-claude-turn-origin.ts`（`settleClaudeTurns`。取り消すターンについて） |

`HUMAN_TURN` は時系列上の在席の証拠であり、認証された判断内容ではない。
権限を伴うツールが記録する `--user-input`、`--feedback`、`--details` の各フィールドは
呼び出し側が与える散文であり、人が書いたテキストと区別できない。
監査シャードは運用上の証拠であり、改ざん防止された人間による作成の境界ではない。

### Initialization Events (3 events — fire IN ADDITION TO `STAGE_COMPLETED`)

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `WORKSPACE_SCAFFOLDED` | ディレクトリツリーが作成された | Timestamp, Details | `tools/aidlc-utility.ts` handleInit |
| `WORKSPACE_SCANNED` | ワークスペースの検出が完了した | Timestamp, Project type, Details | `tools/aidlc-utility.ts` handleInit |
| `WORKSPACE_INITIALISED` | 状態ファイルが作成された | Timestamp, Details | `tools/aidlc-utility.ts` handleInit |

### Navigation Events (9 events)

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `SCOPE_CHANGED` | `--scope` が既存のスコープを変更した | Timestamp, Old scope, New scope | `tools/aidlc-utility.ts` |
| `PLUGIN_SELECTION_CHANGED` | `select-plugins` が有効なプラグインを変更した | Timestamp, Previous Selection, New Selection | `tools/aidlc-utility.ts select-plugins` |
| `DEPTH_CHANGED` | `--depth` が深度レベルを変更した | Timestamp, Old depth, New depth | `tools/aidlc-utility.ts` |
| `TEST_STRATEGY_CHANGED` | `--test-strategy` がテスト戦略を変更した | Timestamp, Old strategy, New strategy | `tools/aidlc-utility.ts` |
| `REVIEW_CLASS_CHANGED` | `--review` が実行ごとのレビューオーバーライドを変更した | Timestamp, Old Override, New Override | `tools/aidlc-utility.ts` |
| `SCOPE_DETECTED` | 自由記述テキストに対してスコープが記録された | Timestamp, Detected scope, Input text, Source, Matched keywords（任意。`Source=keyword` の場合に存在） | `tools/aidlc-utility.ts detect-scope` |
| `RECOMPOSED` | 適応型コンポーザーが実行中のワークフローの保留中ステージを組み直した（`recompose` による後続部分の切り替え） | Timestamp, Scope, Stages skipped, Stages added, Stages in Scope | `tools/aidlc-utility.ts recompose` |
| `WORKSPACE_RECLASSIFIED` | 利用者が、作業は新規プロジェクトである、または既存コードであると述べた（`workspace reclassify --project-type`）。フォルダーが再スキャンされた | Timestamp, Old Project Type, New Project Type, Scanned As, Languages, Frameworks, Build System, Nested Root（任意）, Repos Recorded（任意）, Reverse Engineering。任意で Message Id と Person Reply（利用者がそう述べたメッセージとしてエージェントが `--message` で指定したもの）、または Conductor Summary（`--message` 以前の形式で渡された言葉。エージェント自身のもの） | `tools/aidlc-utility.ts reclassify` |
| `SCOPE_SAVED` | 利用者が作業の現在のプランを再利用可能なスコープとして保存した（`scope save`） | Timestamp, Scope, Saved as, Stages in Scope | `tools/aidlc-utility.ts scope-save` |

### Guard Policy Events (5 events)

Guard Policy（旧称 Change Control）はインテントごとの 1 つの設定で、`strict`、`relaxed`、`off` のいずれかをとり、ある作業に対してガードがどこまで道を譲るかを決める。具体的には、人が何かを承認または確認した後に入力が変わったときに管理対象のチェックポイントが何をするか、そしてどの権限フェンスを維持するかを決める。設定トランザクションと管理対象のチェックポイントは監査ライブラリを通じて来歴を発行する。公開の audit CLI はこれらの行を拒否する。

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `GUARD_POLICY_SET` | `config-change --guard-policy <strict\|relaxed\|off>`（入力された、または依頼されたスイッチに対してエージェントが実行するセッター）が状態行を書き換えた、`scope-change` がスコープ由来の値や明示的な設定を引き継いだ、または管理対象のチェックポイントが実効値を変えるメモリの編集を観測した | Timestamp, Old Value, New Value, Source（記録上利用者が依頼したことが示されている場合は `you`、そうでないセッター実行は `command`、`scope <name>`、または `<layer>.md`）。利用者による引き下げの場合は Person Reply と Message Id（エージェントが `--message` で指定したメッセージ）。設定とスコープの変更では、Old Value にはメモリ上の実効値ではなく、以前に永続化されたインテントの値を記録する（無効な場合は生のテキスト、行がなかった場合は `strict`）。チェックポイントでの観測は実効値の新旧を保持する。 | `tools/aidlc-utility.ts` が設定変更をまとめて処理する（`hooks/aidlc-record-human-turn.ts` からの `applyTypedGuardSwitchPrompt` を含む）; `tools/aidlc-lib.ts`（`appendGuardPolicySetRow` を介した `governedGuardPolicy`）がチェックポイントでの観測を記録する |
| `CHANGE_CONTROL_SET` | `GUARD_POLICY_SET` の旧名で、名前変更前のリリースが書き込んでいた。同じ設定履歴として読まれる。このリリースでは決して書き込まれない | Timestamp, Old Value, New Value, Source | なし（読み取り専用のレガシー行） |
| `GUARD_RESTORED` | 作業単位の `off` の後、`config-change --guard.<fence> on` がこの作業のフェンスを再び有効にした、またはそれを引き下げるポリシーの値より優先して強制的に有効にした | Timestamp, Guard（`plan-approval`, `review-freeze`, `state-transition`, `reviewer-scope`）, Scope, Source（利用者によるものなら `you`、それ以外は `command`） | `tools/aidlc-utility.ts` が設定変更をまとめて処理する |
| `GUARD_STOOD_ASIDE` | フェンスが、この作業について引き下げられていたため、アクションを拒否せずに通した。引き下げの手段は、Guard Policy の値、`guard.<fence>` スイッチ、またはその環境キルスイッチである。フェンスを引き下げられるのは、利用者が入力した、または依頼したスイッチに対してエージェントが実行するセッター、スコープの既定値、または信頼された環境設定（キルスイッチと、fixture/harness-launch の在席バイパス）のみである。人のターンだけ、あるいは選ばれた対処策だけでフェンスが引き下げられることは決してない。その設定は、この行の会話上の Authority とは別である。この行は、スイッチが適用されたときではなく、アクションが引き下げられたフェンスを通過したときに発行される。行を受け取れない台帳がアクションを止めることは決してない。その場合、plan-approval フェンスの一文は記録されなかったことを伝え、その欠落はフックのヘルスログに送られる。この行は拒否の代わりとなる証拠であり、人はそれと並んで一文を聞く | Timestamp, Guard（フェンス）, Authority（`grant`、`instruction`、または `none`: 何がフェンスを開けたかではなく、通過時に誰が作業していたか）, Grant（付与があった場合、その証明方法: `turn-marker`、`marker-sequence`、`dispatch-stamp`、または `none`）, Actor（`main`、`subagent`、または `unattended`）, 任意で Stage, Tool, Details | `tools/aidlc-lib.ts`（`recordGuardStoodAside`。フェンスのフックから呼ばれる） |
| `CHANGE_ACCEPTED` | 管理対象のチェックポイントが、人の承認または確認の後に入力が変わったことを検出し、`relaxed` または `off` の下で（あるいは、プラン承認後に動いたワークスペースのソースについてはすべてのポリシーの下で）、拒否する代わりに変更を記録して続行した。`relaxed` または `off` の下では、レビュー済みのどのユニットのクレームにも含まれない、ステージ中に変更されたアプリケーションのソースも記録する。すべてのポリシーの下で、このマシンに存在しないステージ開始時のソース記録（新しいクローン）も記録する。チェックポイント `units-generation`: Units Generation の承認後に作業のリポジトリが変わった。`relaxed` または `off` の下では変更時に書き込まれ、`strict` の下では、利用者がその質問に返信した（返信の内容を問わない）時点、または無人実行の場合に、次のステップによって書き込まれ、作業は承認済みのユニットで続行する。異なる変更ごとに 1 回書き込まれる。同じ Checkpoint、Stage、ユニットについて同じ Recorded と Current の値で 2 行目が生じることは決してない | Timestamp, Stage, 任意でユニット, Checkpoint（`plan-approval`, `review-receipt`, `summary-confirmation`, `swarm-batch`, `construction-unit`, `units-generation`）, Changed（上限付きのパスのリスト、または `(paths unavailable)`。`units-generation` では追加または除外されたリポジトリ）, Recorded（`units-generation` では、Units Generation が承認されたときのリポジトリのセットと承認日時）, Current, Details（人が聞く一文）, Message Id（利用者が `swarm allow-change` で依頼した `swarm-batch` の変更について: 依頼したメッセージ） | `tools/aidlc-lib.ts`（`recordAcceptedChanges`。チェックポイントの所有者から呼ばれる: `aidlc-log.ts decision` / `answer` / `review`、`aidlc-testing-posture.ts begin`、`aidlc-state.ts` のゲートと完了の検査、plan-approval ガードフック、`aidlc-swarm-checkpoints.ts` のバッチ `ask` / `approve`、利用者自身の編集の上にマージされたスウォームユニットについての `aidlc-worktree.ts merge`、および `units-generation` についてはインテントのリポジトリコマンドと `aidlc-orchestrate.ts next`） |

### Ceremony Events (1 event)

Sensors、Learnings、Summary Confirmation は、それぞれ独立したインテントごとの `on`/`off` 設定である。共有の設定適用処理は、他の要求された設定とともにそれらの来歴を `tools/aidlc-audit.ts appendAuditEntries` でまとめて記録する。公開の audit CLI はこのイベントを拒否する。環境キルスイッチは、インテントに保存された値を書き換えずに解決結果に影響する。

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `CEREMONY_SET` | `config-change --sensors\|--learnings\|--summary-confirmation\|--plan-approval <on\|off>` がインテントのオーバーライドを設定した、または `scope-change` がスコープ由来の設定を新しいスコープの既定値へ引き継いだ | Timestamp, Key（`sensors`, `learnings`, `summary_confirmation`, または `plan_approval`）, Old, New, Source（利用者が入力したスイッチ、またはそれを依頼した利用者の返信なら `you`、それがないセッター実行は `command`、または `scope <name>`）。利用者による off の場合は Person Reply と Message Id（エージェントが `--message` で指定したメッセージと、記録が保持したその言葉）。`--message` 以前の形式で依頼された off では Conductor Summary（エージェント自身のテキスト）。Old は環境上の実効値ではなく、以前に永続化されたインテントの値である（無効な場合は生のテキスト、行がなかった場合はスコープの既定値）。 | `tools/aidlc-audit.ts appendAuditEntries` を介した `tools/aidlc-utility.ts` の共有設定適用処理 |

### Interaction Events (19 events)

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `DECISION_RECORDED` | ゲート以外の構造化された質問を提示する前に、表示する選択肢を記録する。統合サマリー、verification-command、construction-policy のプロンプトはチェックポイントの識別情報も持つ。verification-command のプロンプトは正規化されたコマンドのダイジェストに、construction-policy のプロンプトは要求されたフィールドと値に結び付く。いずれも質問したセッションを記録する | Timestamp, Stage, Decision, Options, 質問記録（`Question Id`, `Question Type`: チェックポイント名または Question Kind、それ以外は `stage-question`, `Question Shows`, 任意で `Question Chat`）。任意で Recommended（渡されたラベルに末尾の "(Recommended)" マークが付いている場合: マークが付いた選択肢を、マークを除き Options と同じリスト文法で記録する。これがない行は推奨を記録しない）, Checkpoint, Command SHA-256, Field, Value, Session, Questions File, Unit, Attempt Generation, Workflow | `tools/aidlc-log.ts decision`（ユニットチェックポイントとスウォームバッチのものは `aidlc-bolt.ts checkpoint` と `swarm-checkpoint` から、`Question Type` はそれぞれ `unit-checkpoint` と `swarm-batch`） |
| `GATE_APPROVED` | 人がゲートで承認した | Timestamp, Stage, User Input。`Brief Digest`（利用者が判断した時点でこのゲートについて保持されていたレビューブリーフの `sha256:<hex>`。ゲート行の書き込み後にエージェントが描画するブリーフ、または `none`）, 任意で `Decisions Accepted Open`（ゲートオープン行に列挙された ID: Approve は、すべての未決の判断を現状のまま受け入れて続行する）, 任意で Person Reply（質問が表示されてからの利用者のメッセージ。human-turn フックが保持したもので、上記 Person Reply の上限内。コンダクターが選択を報告した）, 任意で Message Id（エンジンがチャットを特定できなかったステージ承認の場合: 表示後の利用者の最新の返信について、このマシン自身のシャードに保持されたメッセージ。表示以降に返信したチャットが 1 つだけの場合に限る。Person Reply の代わりに書き込まれ、決して併記されない。ユニットまたはスウォームバッチのチェックポイントの承認では、その質問について保持された最新の返信のメッセージ記録で、Person Reply と併記される）, 任意で Message Ids（上記 Message Ids を参照）, Review Finding Dispositions（バージョン 1 の JSON。現在未解決のすべての指摘を、利用者が承認した場合は Accepted risk に、Construction が自律的に承認した場合は Approved autonomously に、レビュー対象の作業がそのレビュー後に変更され、完了したレビューがまだその変更を見ていない場合は Changed after review に対応付ける。承認が `--fix-finding` で指定したもの（利用者が修正を求めたもの）は、他の条件にかかわらず Fix requested に対応付ける（Accepted risk 以外はいずれも判断ではなく、指摘は次のレビューまで未解決のまま残る）。ユニットのチェックポイントとスウォームバッチの承認はそのユニットの指摘を記録し、誰も与えていないステージの承認は、ユニット自身の承認が記録したものを何も記録しない。キーはレビュー成果物、指摘 ID、指摘内容のフィンガープリント。任意の判断時の重大度と、レビューされた記録のパス/ダイジェストが、判断を対になるレビューに結び付ける）, Review（試行内で要求されたレビューに判定がないまま、または誰のレビュアーも出していない NOT-READY のフォールバックのまま利用者が承認した場合は `not finished`）, Approves Together（この 1 つの承認が対象とするステージ。ゲートからコピー）, Approved Together With（列挙された後のステージについて: その承認の拠り所となる最初のステージ）, Unit, Gate Scope, Gate Stages, Attempt Generation（チームのユニットゲート）。ユニットのマージゲートは Pinned OID, Strategy, Target branch も持つ | `tools/aidlc-state.ts approve`, `tools/aidlc-unit.ts gate` |
| `GATE_REJECTED` | 人が変更を要求した | Timestamp, Stage, Feedback。任意で Message Id と Message Ids（ステージゲートでは Feedback に含まれるメッセージ。上記 Message Ids を参照）, 任意で Person Reply（質問が表示されてからの利用者のメッセージ。human-turn フックが保持したもので、上記 Person Reply の上限内。コンダクターが選択を報告した）と Message Id（その最新のもののメッセージ記録）（ユニットとスウォームのチェックポイント）, Conductor Summary（Feedback が記録された利用者の言葉を含み、両者が異なる場合のコンダクターの `--feedback`/`--reason`）, Review Finding Dispositions（バージョン 1 の JSON。却下された報告またはチェックポイントの reject による、明示的な `Rejected: <reason>` の判断と、レビュアーが解決済みとした指摘に対する `Reopened: <reason>` の判断（スウォームバッチの却下では、その各ユニットの行に付く）。任意で判断時の重大度と、レビューされた記録のパス/ダイジェスト。判断を持たない行はどの指摘も変更しない）, `Recovered=true`（承認時の修正バックストップによる補完）, Prior Accepted Source Fingerprint（前回の試行で検証済みの最終スウォーム集約。完了時のベースラインの代わりには決してならない）, Prior Accepted Source Listing（その集約の着地が保持した `Source Listing`。ない場合や以前のリリースが書いた行にはない）, Unit, Gate Scope, Gate Stages, Attempt Generation（チームのユニットゲート）。ユニットのマージゲートは Pinned OID, Strategy, Target branch も持つ | `tools/aidlc-state.ts reject`, `tools/aidlc-state.ts approve`（バックストップによる補完）, `tools/aidlc-unit.ts gate`, `tools/aidlc-jump.ts reopen`（unit-major の Construction でユニットごとのステージへ戻るジャンプ、ユニットのサマリーまたはチェックポイントでの停止に再入したときに利用者が求めた Redo、または列挙されたステージについての 1 つの遅い質問で利用者が求めた変更: 再オープンしたユニットごとにユニットスコープの行が 1 行。Feedback はジャンプまたは Redo を示すか、変更の場合は利用者の言葉を含み、Reopen は `jump`、`redo`、`change` のいずれか。変更の場合は User Input Request Changes も持つ。後のステージから Current Stage を戻した再オープンは、そのステージを示す Moved From も持つ） |
| `QUESTION_ANSWERED` | ゲート以外の質問にユーザーが回答した | Timestamp, Stage, Details（選択肢のある質問では、コンダクターが返信から読み取った選択。フックが利用者の言葉を保持しなかった場合や、1 つの返信で複数の質問に答えた場合は返信そのもの）。任意でユニット, Attempt Generation。任意で Person Reply（チェックポイントの質問では、質問が表示されてからの利用者のメッセージ。その他の質問では、エージェントが指定したメッセージ、または質問がいつ記録されたかにかかわらず、まだどの判断にも使われていないこのチャットでの利用者の返信すべて。human-turn フックが保持したもので、上記 Person Reply の上限内。コンダクターが選択を報告した）。任意で Message Id（そのメッセージ、またはそれらの返信の最新のもの）と Message Ids（上記 Message Ids を参照）。任意で Answer Source `chosen by the agent, reading their message`（`answer --on-instruction`。エージェントが選択を任されたと読んだ利用者のメッセージに基づくもので、Message Id がそれを示す）。任意で Conductor Summary（`--message` 以前の形式でエージェントが `--on-instruction` とともに渡した言葉。エージェント自身のテキストであり、何も主張しない） | `tools/aidlc-log.ts answer` |
| `QUESTION_REPLIED` | ハーネスの質問ボックスが利用者の回答とともに戻ってきた。質問ごとに 1 行で、表示された質問と、返されたままの返信（選ばれたラベル、カンマで連結された複数のラベル、または入力された言葉）を記録する。何も判断せず、ターンも消費しないため、`QUESTION_ANSWERED` が記録されなくても回答は記録に残る。これを書き込むのはこのフックだけである（公開の `append` は拒否し、ワークツリーの監査マージも決してコピーしない） | Timestamp, Question, Reply。任意で Session, Options（ボックスがその質問に提示した選択肢。ラベルの JSON リスト）, Recommended（ボックスが末尾に "(Recommended)" マークを付けて表示した選択肢。マークを除いた JSON リスト）, Question Id（ボックスが表示した正確なテキストを持つ、そのチャットで開いている質問） | `hooks/aidlc-record-human-turn.ts`（Claude Code の `AskUserQuestion`、Codex の `request_user_input`） |
| `QUESTION_UNANSWERED` | ハーネスの質問ボックスが回答なしで戻ってきた（Codex のものは約 2 分で期限切れになる、または利用者が閉じた）。以前の人のターンを消費するため、利用者が返信するまで何も記録されない。フックは同じ質問を番号付きの散文（ラベルと順序は変えない）として描画し直してエージェントに返し、エージェントがそれを再提示する。したがって散文のフォールバックはエンジンのものであり、コンダクターが再構成するものではない。これを書き込むのはこのフックだけである（公開の `append` は拒否し、ワークツリーの監査マージも決してコピーしない） | Timestamp。任意で Session | `hooks/aidlc-record-human-turn.ts`（Codex アダプター経由） |
| `QUESTION_ASKED` | エンジンが、質問時に独自の行を持たない自身の質問の 1 つを提示した: コードプランの質問、`next` が尋ねる質問（作業の所属先、どのプラン、誰がユニットを構築するか、プロジェクトの種類、どのリポジトリ）、またはガードが保留したシステムインストールの許可（`SYSTEM_INSTALL_BLOCKED` に並び、`SYSTEM_INSTALL_ALLOWED` がそれに答える）。質問ごとに 1 回書き込まれ、同じ質問が再度提示されても ID は変わらない（Question records を参照） | Timestamp, Question Id, Question Type（質問種別）, Question Shows, Stage。任意で Question Chat, Unit またはユニット, Options（JSON リスト）, State Digest（エンジンの質問）, Ask Id（コードプランの質問自身の記録）, Questions File, Question Text SHA-256 | `tools/aidlc-orchestrate.ts`, `tools/aidlc-plan-approval-ask.ts`, `hooks/system-install.ts` |
| `QUESTION_SHOWN` | まだ開いている質問が、以前に質問・表示されたことのないチャット（新しいチャット、再開、クローン）で表示された、またはコードプランの質問がプランを変更した状態で表示された | Timestamp, Question Id, Question Type, Question Chat。任意で Question Shows（現在表示している内容）, Stage, Unit, Ask Id, Questions File（質問の元になったファイル） | `tools/aidlc-state.ts gate-start`, `tools/aidlc-orchestrate.ts`, `tools/aidlc-plan-approval-ask.ts` |
| `REQUEST_ROUTED` | 利用者が、ルーティングの質問が尋ねた言葉を、別の新しい作業へ、またはプランの組み直しへ振り向けた。その言葉は進行中の作業で開いているどの質問にも答えるものではなく、それについての判断には後の返信が必要である | Timestamp, Request。任意で Session, Message Id（ルーティングの根拠となる返信: 前回の判断以降のそのチャットの最新のもの） | `aidlc-orchestrate.ts` |
| `SUMMARY_CONFIRMATION_RECORDED` | 対応するプロンプトと新しい人のターンの後に、統合サマリーの選択が記録された。公開の audit CLI からは予約されている | Timestamp, Stage, Details, Checkpoint, Questions File, Questions SHA-256, Hash Scope（新しいレシートでは必須。レガシーの行では省略されることがある）, Summary Authorization Id（`Looks correct` のレシートの場合: 試行、ステージ、ユニット、ワークフロー、質問ファイルのパス、確認済み内容のハッシュ、選択に対する sha256。同一の確認は同じ ID を生成し、回答が変われば新しい ID になる。レガシーの行では省略される）。任意でユニット, Workflow, Feedback（返信で何を変えるかを述べた `Request changes` のレシートの場合: 利用者の言葉）, Person Reply, Message Id, Message Ids（上記 Message Ids を参照） | `tools/aidlc-log.ts answer --checkpoint summary-confirmation` |
| `VERIFICATION_COMMAND_RECORDED` | 人が、対応する保留中の判断の一回限りのチャレンジと、提示されたとおりの選択肢を通じて、インテントの Construction 検証コマンドを承認した。それに対する利用者自身の返信であり、質問したチャット、またはそれが最新の質問である間は新しいチャットでのものに限る。無関係な人のターンはこれを承認しない。現在のワークフローの最新のレシートは、一致する state コマンドのみを承認し、公開の audit CLI からは予約されている | Timestamp, Stage, Checkpoint（`Construction Verification Command`）, Command SHA-256（正規化しトリミングした 1 行のコマンドの 16 進 SHA-256）, Command Label（正規化されたコマンド全体: 最大 1024 文字、制御文字や表示偽装文字を含まない）, User Input（`Approve`）, Session。任意で Workflow。任意で Person Reply（質問が表示されてからの利用者のメッセージ。human-turn フックが保持したもので、上記 Person Reply の上限内。コンダクターが選択を報告した） | `tools/aidlc-log.ts answer --checkpoint verification-command` |
| `CONSTRUCTION_POLICY_RECORDED` | 人が、対応する保留中の判断のフィールド/値に対する一回限りのチャレンジと、提示されたとおりの選択肢を通じて、Construction ポリシーの変更を承認した。返信は質問したチャットまたは新しいチャットで行われた（この質問をしたワークフローの場合。現在は、利用者が求めるとコンダクターが入力型のセッターを実行する）。そのフィールドについての現在のワークフローの最新のレシートは、要求された値のみを承認し、セッターがそれを適用した時点で消費される。公開の audit append とワークツリーの監査マージからは予約されている | Timestamp, Stage, Checkpoint（`Construction Policy`）, Field（`Construction Checkpoints`、`Construction Execution`、または `Construction Iteration`）, Value, Session, User Input（`Approve`）。任意で Person Reply（質問が表示されてからの利用者のメッセージ。human-turn フックが保持したもので、上記 Person Reply の上限内。コンダクターが選択を報告した） | `tools/aidlc-log.ts answer --checkpoint construction-policy` |
| `CHECKPOINT_VERIFICATION_RECORDED` | Construction チェックポイントの検証器が承認済みのコマンドを完了し、監査ロックの下で証明を書き込んだ。検証には、現在の試行の最新のレシートが、証明 ID、証拠のフィンガープリント、現在のコマンドのダイジェスト、run floor と一致し、`Verified: true` であることが必要である。証明の JSON だけでは承認を与えられない。証明ファイルがまったくないチェックアウトでは、証拠が変わっていないすでに承認済みのユニットについて、レシートが証明の代わりとなる。公開の audit append とワークツリーの監査マージからは予約されている | Timestamp, Unit, Kind（`unit` または `skeleton`）, Stage（最後のチェックポイントステージ）, Stages, Verification Id, Fingerprint, Command SHA-256, Exit Code, Verified（`true` または `false`）, Run floor。任意で Attempt Generation。任意で Blocking Checks（`<stage>:<sensor>@<binding digest>=passed|overridden`）, Checked Outputs（JSON: 検査したファイルから、改行を正規化した SHA-256 への対応）, Blocking Sensor Override, Blocking Sensor IDs, Blocking Sensor Detail Paths, Blocking Sensor Reasons（ユニットのブロッキングゲート検査がチェックポイントで実行された場合） | `tools/aidlc-construction-checkpoints.ts verifyConstructionCheckpoint` |
| `PLAN_APPROVAL_RECORDED` | Code Generation が保護された Plan Approval のチャレンジ/レスポンスのレシートを消費したことの来歴。この Markdown の行は承認の証拠ではない。`Directive Epoch` は診断のために記録され、比較されることはない。承認は `Plan Target`、`Run floor`（ステージの試行）、`Approval Fingerprint`（プランの内容）に結び付く | Timestamp, Stage, Details, Checkpoint, Plan Target, Intent, Directive Epoch, Run floor, Approval Fingerprint, Questions File, Questions SHA-256, Prompt SHA-256, Session。任意で Person Reply（質問が表示されてからの利用者のメッセージ。human-turn フックが保持したもので、上記 Person Reply の上限内。コンダクターが選択を報告した） | `tools/aidlc-log.ts answer --checkpoint plan-approval` |
| `PLAN_APPROVAL_SKIPPED` | この作業ではプラン承認がオフ（スコープの既定値、利用者自身のスイッチ、またはマシンスイッチ `AIDLC_DISABLE_PLAN_APPROVAL_GUARD=1`）のため、誰にも承認を求めていないプランから Code Generation が開始された。構築されたプランのフィンガープリントを記録し、何が実行されたかを証跡に残すが、人が承認したとは決して主張しない。質問ファイルには `[Answer]: Plan approval off` と記載され、保護されたレシートはスキップとしてマークされる。公開の audit CLI からは予約されている | Timestamp, Stage, Details（`Plan approval off`）, Checkpoint, Plan Target, Intent, Directive Epoch, Run floor, Approval Fingerprint, Questions File, Questions SHA-256, Prompt SHA-256, Source（オフ設定の出どころ）, Unit（ユニットスコープの場合） | `tools/aidlc-plan-approval-ask.ts`（エンジンがビルドを引き渡すとき: `next` で、またはビルドの最後のルール部分を届ける `continue` で） |
| `PLAN_APPROVAL_OVERRIDDEN` | 人が非常手段を使った: それを提示する拒否が表示された後、返信（利用者自身の言葉または選択。human-turn フックが記録）でとにかくプランを構築するよう求め、通常のレシート経路が拒否した後に、コンダクターが利用者の理由を付けて `answer --override-file` を実行した。この行に付随するレシートは、プランの内容とステージの試行のみに結び付く。ソースの認証はスキップされる。常に `Override: yes` を持つ `PLAN_APPROVAL_RECORDED` 行と対になる。公開の audit CLI からは予約されている | Timestamp, Stage, Reason（原文のまま）, Failed Checks（通常経路の拒否理由。縦棒で区切る）, Session, Unit（または `stage-level`）, Fingerprint。任意で Person Reply（提示以降の利用者の最新の返信。ホストが言葉を渡さなかった場合はない） | `tools/aidlc-log.ts answer --checkpoint plan-approval --override` |
| `REVIEW_REQUESTED` | コンダクターが §12a のレビュアーサブエージェントを派遣した。公開の audit CLI からは予約されている。構造的に不正な行は権威を持たず、序数を消費しない | Timestamp, Stage, Reviewer, Iteration, Artifact Fingerprint（レビューのために派遣されたとおりの、宣言されたすべての成果物の 1 つの安定したスナップショットから得た `sha256:<hex>`。CRLF のテキストは LF として読む）, Request Id（`review:<32 lowercase hex>`。要求ごとに生成され、完了行とレビュー記録がそれを反映する）, Review File（この要求のレビューを読み取る、記録相対のファイル。これなしで記録された要求は要求ごとのレビューファイル導入以前のもので、そのレビューはそのパスの共有ファイル `<iteration>.review.md` から読み取られる）, 任意で Unit + Attempt Generation（権威ある DAG によるユニットごとのクレーム）, `workspace_requires` ステージでは Source Fingerprint, Unit ごとの `workspace_requires` ステージでは Unit Source Fingerprint（マニフェストのバイト列 + クレームされたソースの一覧）, 任意で Retry（`pending-request`。現代的なバインディングが存在した後に 1 回だけ受け付ける）, 任意で Upgrade（`legacy-request`。要求 ID やソースのバインディング導入以前に記録された有効な要求に対する、上限付きの 1 回の近代化）, 任意で Recovery（`stale-receipt`）と Recovery Cause（`artifact`、`source`、または `artifact+source`）, 任意で Person Asked（ステージのパス数またはその使用済みのリカバリーを超えるパスについて、コンダクターが `--person-asked` で利用者の依頼として記録した場合に `true`。そのレビューが最後に要求されて以降の利用者のターンが記録にあること）と Person Reply（まだどの判断にも使われていないこのチャットでの利用者の返信。フックが保持した場合）, Message Id（その最新のもの）, Message Ids（上記 Message Ids を参照）, 任意で Checkouts Recorded（ソースの走査が届かなかった場所にある、ユニットのクレームを含む各チェックアウト。この要求がバインディングの前に記録したもので、`<label> (<path>)` の形式）, 任意で Replaces Request Id（同じスコープとパスで保留中だった要求のうち、判定の前に出力またはソースが変わったものの Request Id。要求 ID 導入以前に記録されたものは `none`。新しい要求はそのパスを引き継ぎ、レビューの予算には数えられず、置き換えた要求がリカバリー要求だった場合はその Recovery と Recovery Cause を引き継ぎ、それ自体が中断された場合も同様に置き換えられる。Replaces Request Id がそのような要求を指していない行は通常の要求として数える）。廃止された付録プロトコルの下で書き込まれた行は、Review Appendix Artifact, Review Appendix Offset, Review Appendix Prior Digest, Review Appendix Prior Length, Review Challenge も持つ。これらは引き続き読み取り可能であり、そのような要求の完了はそれらを変更せずに反映するため、組は引き続き結び付く | `tools/aidlc-log.ts review` |
| `REVIEW_COMPLETED` | レビュアーの判定が記録された。承認のゲートとなり、公開の audit CLI からは予約されている。不正な行は、対応する保留中の要求を消費せずに無視される | Timestamp, Stage, Reviewer, Iteration, Verdict, Request Fingerprint（要求と一致しなければならない）, Artifact Fingerprint（同じ安定したスナップショット: レビュアーは成果物を書き込まないため、レビューされたバイト列は要求されたバイト列である）, Request Id（要求と一致しなければならない）, Review Record（記録相対パス `.aidlc-engine/reviews/<stage>/stage/<attempt>/<iteration>.json` または `.aidlc-engine/reviews/<stage>/units/<unit>/<attempt>/<iteration>.json`）+ Review Record Digest（記録のバイト列に対する `sha256:<hex>`。もはやこのハッシュにならない記録はそのレビューではない）, 任意で Unit + Attempt Generation（ユニットごとのクレーム）, Request Source Fingerprint + Source Fingerprint（`workspace_requires` ステージでの、要求時と同一のソース識別）, `workspace_requires` ステージでは Source Listing（レビュアーが見た、内容アドレス型の一覧。これにより完了時に、レビュー後の AI-DLC の更新とソースの変更を区別できる。レビュー中にソースが動いた場合、保持できなかった場合、以前のリリースが書いた行にはない）, Unit ごとの `workspace_requires` ステージでは Unit Source Fingerprint または Unit Source Binding Bypass, 非推奨の付録移行経路の場合のみ Review Challenge, 1 回の stale-receipt リカバリーパスの場合のみ Recovery, 分離された実行では任意で Workflow, 任意で Review Headings Made Level 3（ロガーが `###` として記録したレビューファイルの各見出し行。`line N: <a trimmed preview of that line, up to 120 characters>` の形式。レビューのそれ以外は何も変わらず、記録は全文を保持する）, 任意で Review Finished（再試行したレビューがなおレビュー全体を書かなかったときに記録される NOT-READY のフォールバックでは `no`: どのレビュアーも判定を出していない。どのフィンガープリントにも影響しない）, 任意で Review Verdict Dictated（`yes` または `no`。レビュアー自身が書いた `**Verdict Dictated:**` 行のとおり: 与えられたブリーフが、書くべき判定や優先すべき判定を指定していたかどうか。`yes` は判定とともに利用者に 1 回伝えられ、strict の下ではゲートでもう 1 回伝えられる。行がない、または読めない場合はフィールドを書き込まない。どのフィンガープリントにも影響しない）, 任意で Reviewer Finished Before Verdict（次の場合に `no`: ハーネスが `agentFinishRecorded` を宣言しているホスト（Claude Code）で、レビューファイルの `Review File Digest` を持つ要求以降、その要求のレビュアーの `SUBAGENT_COMPLETED` 行がないまま判定が記録された場合（別のエージェントの完了はレビュアーの完了ではない）。または他のホストで、要求以降の `SUBAGENT_DISPATCHED` 行が、補助エージェントが送られ、そのすべてが完了によって閉じられ、そのどれもレビュアーではないことを示し、委任されたワークツリーの外であり、それ以降バックグラウンドエージェントが送られておらず、エージェント名のない完了行もなく、行を残さないか失敗を記録した完了フックの実行もない場合: どのレビュアーも完了していなかった。判定とともに利用者に 1 回伝えられ、strict の下ではゲートでもう 1 回伝えられる。どのフィンガープリントにも影響しない）, 任意で Testing Contract（Code Generation のみで、スウォームワーカーのチェックアウトでは付かない: 判定が記録された時点で有効だった Testing Contract のハッシュ。Code Generation の完了は、最終レビュー後に動いたコントラクトを次のビルドのものとして読む。どのフィンガープリントにも影響しない） | `tools/aidlc-log.ts review --verdict` |
| `PIPELINE_LINK_COMPLETED` | 宣言されたパイプラインのリンクが順番どおりに戻った。現在の試行のレシートがパイプラインの承認のゲートとなり、公開の audit CLI からは予約されている | Timestamp, Stage, Link, Position（`k/N`）, 任意で Repo（複数リポジトリのチェーンではプロトコル上必須）, 任意で Workflow（分離された実行では `single-stage:<slug>`） | `tools/aidlc-log.ts link` |

`Hash Scope: confirmed-content-v2` は、新しく発行されるレシートが使う質問ファイルの
ダイジェストを識別する。ダイジェストのアルゴリズムは変わらない。CRLF と単独の CR を
LF に正規化し、保持されたセクションの元の順序と生のバイト列を維持し、
得られた内容から末尾の空白を 1 回トリミングしてから、UTF-8 を SHA-256 で
ハッシュする。可視性のビューは生のダイジェスト入力を置き換えない。保持された内容に含まれる
コメント、コード、HTML、先頭の BOM は引き続き結び付けられる。

見出しと回答の認識は現在、組み込みの `Bun.markdown` による
CommonMark/GFM パーサーに、`markdownBlocks` とその `visibleMarkdownLines` 投影を通じて従う。
生の HTML ブロック（CommonMark の種類 1〜7）内のテキストは、見出し、回答、制御
タグには決してならない。ダイジェストには、可視のすべての Q<n> セクションと、各
`Requested Changes Feedback` セクションが含まれ、前提に関する判断の後に追加された
フォローアップの質問も含まれる。可視のトップレベルの `Assumption Confirmation`
セクションは、サマリーの後にちょうど 1 つある場合にのみ有効であり、その内容と、
直前の空行で区切られた区切り線とともに除外される。
除外は、トップレベルの `## Q<n>` 見出しまたは
`## Requested Changes Feedback` 見出しとして綴られた行で終わる。これは生の HTML やコードの内部でも同様であり、
パーサーが認識しない見出しは確認済みの内容を広げる方向にしか働かない。
`Q<n>` または `Assumption Confirmation` の見出しは、claim-sources センサーと同じ規則により、
先頭の絵文字装飾の有無にかかわらず数えられる。
サマリー以前にある同名のセクションは、確認済みのダイジェストの一部のままである。
除外された前提と回答は、ステージの既存の判断/回答の検査とセンサーの検査の対象のままである。
サマリーの後にあるその他の認識された見出しは無効である。
コメント、コードスパン、フェンス付きまたはインデントされたコード、HTML の属性値に含まれる
見出しのようなテキストはセクションではない。

`confirmed-content-v1` はサポートされているレガシーのスコープである。同じ
生の内容のダイジェストアルゴリズムを、以前の手書きの Markdown 可視性
ルールとともに使っていた。パーサーのアップグレードの影響を受けないドキュメントでは、v2 のダイジェストは
v1 とバイト単位で同一である。既存のレシートは次のように移行する:

- 記録されたダイジェストが現在の v2 ダイジェストと等しい `confirmed-content-v1` のレシートは
  受け付けられる。v1 は現在の意味論で比較されるため、影響を受けないドキュメント上の
  進行中のワークフローに再確認は強制されない。
- その v1 ダイジェストが異なる場合、内容の意味論が変わった可能性があるため、完了は拒否される。
  レシートは Markdown パーサーのアップグレード以前のものであり、確認後に確認済みの内容が
  変わったか、v1 が確認済みのテキストとして扱っていた生の HTML の内容が
  もはやその一部ではないかのどちらかである。生の HTML の見出しと制御タグは現在、
  Markdown の認識から除外される。サマリーを再提示して再確認すると、
  v2 のレシートが記録される。拒否は生の HTML の除外を示し、
  編集が行われたとは決して断定しない。
- `confirmed-content-v2` のダイジェストの不一致では、既存の
  「確認後に変更された」（“changed after confirmation”）拒否とリカバリーが維持される。
- `Hash Scope` を持たないレシートでは、レガシーのファイル全体の SHA-256 と
  既存のリカバリーテキストが維持される。進行中のスコープなしのレシートは、許可された確認後の
  追記がリカバリーできるようになる前に、スコープ付きのレシートを作成するための新しい人の
  確認を必要とする。
- 不明なスコープは、引き続き無効なハッシュスコープとして拒否される。

重複確認 / 以前の同一確認の経路は、v1 のレシートを同じ v2 ハッシュ関数で解決する。
保存済みのレシートは書き換えられない。

サマリー確認の権限比較は、1 つの監査シャード内では追記順を維持する。
シャードをまたぐ場合は、異なるタイムスタンプによって順序が決まり、等しい
タイムスタンプは因果的に順序付けられない。そのような同着が、現在の試行、選択されるレシート、
または成果物が確認後に書き込まれたかどうかを変えうる場合、完了はフェイルクローズし、
より後のタイムスタンプを持つ新しい証拠を必要とする。

#### Question records

AI-DLC が尋ねるすべての質問は、それを尋ねる行に 1 つの `Question Id`（8 桁の 16 進数。
保存された要求の質問がすでに使っている ID 空間）を持つ。具体的には、
ゲートの `STAGE_AWAITING_APPROVAL`、すべての `DECISION_RECORDED`、そして独自の行を持たない
エンジン自身の質問についての `QUESTION_ASKED` である。
それと並んで: `Question Type`（`gate`。記録された質問ではチェックポイント名または
Question Kind、それ以外は `stage-question`。`unit-checkpoint`、`swarm-batch`。
エンジンの質問では質問種別で、コードプランの質問は `plan-approval`）、
`Question Chat`（コマンドがチャットを指定する場合、質問したチャット）、
`Question Shows`（利用者に判断を求める内容に対する `sha256:`）。同じ
質問が別のチャットで再度表示された場合、ID は変わらず `QUESTION_SHOWN` を書き込む。
クローンは、コミットされた記録上のスロット（種類、ステージ、ユニット）によって開いている質問を見つける。
質問ファイル内の質問はエントリごとに、`[Answer]:` 行を空にし改行を正規化してダイジェストされるため、
書き込まれた回答や Windows の改行は何も変えない。ID は記録されるだけで、
現時点ではそれらを読んで何かを判断するものはない。

### Unit Configuration and Lifecycle Events (9 events, unit-major Construction)

スウォームの `SWARM_UNIT_*` 台帳に対応する対話型の版である。`UNIT_COMPLETED` は、
ステージについて何らかのレシートが存在するようになると、エンジンのカバレッジ走査が
成果物の単なる存在よりも優先する完了レシートである。発行する動詞は、コミットする前に
ユニットの必須成果物がディスク上の通常ファイルであることを検証する。
`Run floor` は正確な境界トークン（`<event>:<timestamp>#<ordinal>`）であるため、
1 つのシャード内の同じ秒の試行がレシートを再利用することはできない。異なるシャードにある
同時刻の境界は因果的に順序付けられず、決定的な
`AMBIGUOUS:<timestamp>#<digest>` フロアを使う。以前のレシートがそれに一致することはない。
unit-major のステージは、作業がそのステージ自身の `STAGE_STARTED` より前に行われうるため、
フロアをワークフロー/ジャンプ/却下の境界に結び付ける。Construction ポリシーの変更は
境界ではない。stage-major のフロア設定が有効だった間（次の `CONSTRUCTION_POLICY_SET` が見つけたポリシー）に
記録された `STAGE_STARTED` は、unit-major のイテレーションやチェックポイントへの切り替え後も境界のままであり、
unit-major のフロア設定が有効だった間に記録されたものは、チェックポイントがオフの stage-major へ戻した後も
Unit レシートのフロアからは無視されたままである。したがって、どちらの切り替えの前に完了したユニットも
レシートを保持する。自身の再オープン（ユニットタグ付きの `GATE_REJECTED`）を持つユニットは、
どのモードでもそのユニットについてレシートのフロアが設定されるため、再オープンはどちらの切り替えをまたいでも境界のままである。unit-major の走査でユニットが完了した後に記録された、ステージの最初の `STAGE_STARTED`（遅れたゲートの連鎖。戻す切り替えの後の場合もある）は、そのステージの再開ではない。`UNIT_SKIPPED` は、
ステージの条件がそのユニットに当てはまらない場合に、1 つのユニットの拍を決着させる。
そのユニットはその試行においてステージに何も負わない（種類によってすべての出力が刈り込まれる
ユニットと同様）が、他のすべてのユニットは引き続き負う。ステージがスキップとしてマークされるのは、
どのユニットもそれを負わなくなった場合のみである。

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `UNIT_OWNERSHIP_SET` | ユニットの活動が始まる前に、unit-major の所有モードが設定された | Timestamp, Mode | `tools/aidlc-state.ts set-unit-ownership` |
| `UNIT_GATE_RHYTHM_SET` | 最初のチームユニットゲートの前に、チーム所有のゲートのリズムが設定された | Timestamp, Rhythm | `tools/aidlc-state.ts set-unit-gate-rhythm` |
| `CONSTRUCTION_POLICY_SET` | 入力型のセッターが、変更された Construction Iteration、Checkpoints、または Execution の値を適用した。試行のフロアはこれを読むため、どちらの方向への切り替え（unit-major のイテレーションまたはチェックポイントのオンへ、あるいはチェックポイントがオフの stage-major へ戻す）でも、完了済みのユニットのレシートは保持される。Construction 中の変更は、利用者が依頼したメッセージが `--message` で指定された場合（または現在の `CONSTRUCTION_POLICY_RECORDED` がある場合）にのみ適用され、無人実行では決して適用されない | Timestamp, Field, Value, Previous Value, Construction Iteration, Construction Checkpoints。任意で Person Reply と Message Id（そのメッセージとその言葉。human-turn フックが保持したもの） | `tools/aidlc-state.ts set-construction-iteration`, `set-construction-checkpoints`, `set-construction-execution` |
| `UNIT_STARTED` | インラインのユニットごとのステージで、あるユニットの作業が始まる。そのステージの別のユニットが開いている間は拒否される | Timestamp, Stage, Unit, Run floor。任意で Attempt Generation | `tools/aidlc-state.ts unit start` |
| `UNIT_PAUSED` | ユニットが完了前に停止する。チェックポイントはその理由と次に行うことを持つ | Timestamp, Stage, Unit, Run floor, Reason, Next Action。任意で Attempt Generation, Set Aside For（その間に利用者が求めたユニット。走査はそれを先に扱う） | `tools/aidlc-state.ts unit pause` |
| `UNIT_RESUMED` | 一時停止中のユニットが明示的に再開された（これが行われるまでエンジンは強制停止する） | Timestamp, Stage, Unit, Run floor。任意で Attempt Generation | `tools/aidlc-state.ts unit resume` |
| `UNIT_COMPLETED` | ユニットの作業が完了し、かつその必須成果物がディスク上の通常ファイルである（発行時に検証） | Timestamp, Stage, Unit, Run floor。任意で Attempt Generation | `tools/aidlc-state.ts unit complete` |
| `UNIT_SKIPPED` | unit-major の下で、ステージの条件がこのユニットに当てはまらない（または利用者がこのユニットについてスキップを求めた）ことが走査中の (stage, unit) の拍について報告された、または前方ジャンプがこのユニットをステージの先へ進めた | Timestamp, Stage, Unit, Reason, Run floor | `tools/aidlc-state.ts skip --unit`（内部的には `aidlc-orchestrate.ts report --result skipped --unit` から、またはジャンプがスキップする各ステップについて `aidlc-jump.ts execute --units` から） |
| `UNIT_MERGED` | main が固定された候補の内容を取り込み、このユニットの行を畳み込んだ。運ばれたレシートが main のフロアを満たすようになる | Timestamp, Unit, Owner, Pinned OID, Merge commit OID, Attempt Generation | `tools/aidlc-state.ts fold-unit-merge` |

### Artifact Events (3 events — hook-emitted)

成果物フックは、アクティブなインテントの記録ツリー、または
アクティブなスペースの共有 `codekb/<repo>/` ツリーへの書き込みについて発行する。

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `ARTIFACT_CREATED` | アクティブなインテントの記録、またはスペースレベルの codekb ツリーに新しい成果物が書き込まれた | Timestamp, Tool, File, Context, 任意で Summary Authorization Id（書き込み時点での、書き込まれたステージとユニットについての有効なサマリー確認。`<record>/.aidlc-engine/summary-authorization/<stage>/stage.json` または `<record>/.aidlc-engine/summary-authorization/<stage>/units/<unit>.json` から読む。有効な確認がない場合はない） | `hooks/aidlc-write-audit-log.ts`（PostToolUse。新規パスへの Write） |
| `ARTIFACT_UPDATED` | いずれかのツリーの既存の成果物が変更された | Timestamp, Tool, File, Context, 任意で Summary Authorization Id（`ARTIFACT_CREATED` と同様） | `hooks/aidlc-write-audit-log.ts`（PostToolUse。Edit、または既存ファイルを上書きする Write） |
| `ARTIFACT_REUSED` | 後方ジャンプでの再利用の判断、またはリポジトリごとのパイプライン再利用の証拠。パイプラインの免除を与えるのは `Decision=keep` のみ。公開の audit CLI からは予約されている | Timestamp, Stage, Decision, Artifacts, 任意で Repo, 任意で Workflow（分離された鮮度依存の再利用では `single-stage:<slug>`）, 任意でユニット（ユニットごとの Keep が指すユニット、または Source が `Redo on re-entry` の場合: 利用者の Redo がそのユニットのステップについての質問に答えた）。有効なサマリー確認の下での `keep` では、`Summary Authorization Id` と `Kept Outputs`（`<sha256> <record-relative path>; ...`） | `tools/aidlc-state.ts reuse-artifact`, `tools/aidlc-jump.ts reopen --via redo` |

### Subagent Events (4 events - hook-emitted)

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `SUBAGENT_DISPATCHED` | 補助エージェントが送られた。公開の audit CLI からは予約されている | Timestamp, Agent Type（クルーの役割を 1 つの値として。派遣が何も指定しない場合は `unknown`）, 任意で Dispatch Agent（Kiro のレビュアーのコピー。その要求のレビュアーの下に記録される） | `hooks/aidlc-deliver-stage-rules.ts`（PreToolUse。派遣を通す経路で）; Cursor と Kiro IDE のアダプターが転送する SubagentStart については `hooks/aidlc-log-subagent.ts` |
| `SUBAGENT_COMPLETED` | サブエージェントのタスクが完了した。公開の audit CLI からは予約されている | Timestamp, Agent Type, 任意で Agent ID, 任意で Message | `hooks/aidlc-log-subagent.ts`（SubagentStop） |
| `SUBAGENT_PROMPT_UNMATCHED` | 助言のみで、人のターンではない: 同じチャットでサブエージェントの開始から数秒以内に Copilot のプロンプトが届き、記録されたどのサブエージェントのブリーフにも一致しなかったため、数えられなかった | Timestamp, 任意で Session, Agent, Counted（常に `no`）, Reason（どのブリーフにも一致しなかった、またはブリーフの記録を読めなかった） | `tools/aidlc-audit.ts appendSubagentPromptUnmatched`（Copilot アダプターの `record-human-turn`） |
| `REVIEW_VERDICT_DICTATED` | もはや書き込まれない。以前のリリースでは、描画された `**Verdict:**` 行をブリーフに含むレビュアーの派遣を plan-approval ガードが記録していた。現在はレビュアー自身がレビューファイルでその旨を述べ（`**Verdict Dictated:** yes\|no`）、`REVIEW_COMPLETED` に `Review Verdict Dictated` として記録される。以前に書かれた監査とワークツリーの差分が引き続き読み取り・マージできるよう、登録は残している | Timestamp, Agent Type（レビュアー）, Verdict, Tool, Request Id | 予約済み（もはや書き込まれない。過去の行のみ） |

### Reviewer Enforcement Events (2 events - hook-emitted)

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `REVIEWER_SCOPE_BLOCKED` | Unit ごとのレビュアーのツール呼び出しが、兄弟ユニットの `construction/` パスに及んだため拒否された（§12a の読み取りスコープの境界） | Timestamp, Guard, Reason（`reviewer-read-scope`）, Tool, Target, Stage, Unit, ホストがチャットに名前を付けている場合は Session | `hooks/aidlc-reviewer-scope.ts`（PreToolUse） |
| `REVIEW_FREEZE_BLOCKED` | ファイルツールまたはシェルによる `produces[]` への書き込みが、ゲート前の新しい READY のレビューレシートを無効にするため拒否された（§12a の最終レシートの順序） | Timestamp, Guard, Reason（`review-receipt`）, Tool, Target, Stage, 任意でユニット, ホストがチャットに名前を付けている場合は Session | `hooks/aidlc-review-freeze.ts`（PreToolUse） |

### Guard Refusal Events (2 events - hook-emitted)

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `STATE_TRANSITION_BLOCKED` | 委任されたエージェントのライフサイクルまたはルーティングのコマンドが拒否された。ステージの状態やルーティングを変更できるのはメインのワークフローセッションだけであるため | Timestamp, Guard, Reason（`delegated-lifecycle`）, Tool, Target。ホストがチャットに名前を付けている場合は Session | `hooks/aidlc-state-transition-guard.ts`（PreToolUse） |
| `STAGE_RULES_BLOCKED` | ステージのルールがサブエージェントに届けられなかった: 組み立てたブリーフがフックの出力上限を超えた（`Reason: oversize`）、または必須のルールファイルを読めなかった（`Reason: rules-unreadable`） | Timestamp, Guard, Reason。ホストがチャットに名前を付けている場合は Session | `hooks/aidlc-deliver-stage-rules.ts`（PreToolUse） |

### Plan Approval Enforcement Events (2 events - hook-emitted)

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `PLAN_APPROVAL_BLOCKED` | code-generation の開発者エージェントの派遣またはワークスペースの変更が拒否された: 対象に、明示的に承認された現在のプランの契約がなかった（ステージの Step 2-3 は Step 4 より前でなければならない）、ステージのルールがまだ届いている途中だった、ビルドの開始を記録できなかった、またはランタイムのロックフォルダーが保持されている | Timestamp, Guard, Reason（`plan-approval`, `rules-arriving`, `start-record`）, Tool, Stage。拒否にそれらがある場合は Target, Unit, Session | `hooks/aidlc-plan-approval-guard.ts`（PreToolUse） |
| `GUARD_DISABLED` | ワークフローが存在する間に、決定的なオフスイッチ（hooks リファレンスに記載された無効化用の環境変数）が設定されていたためツール呼び出しが Plan Approval ガードを通過した（連続ごとに 1 行: アクティブなシャードの最新の行が同じガードについてのこのイベントでない場合にのみフックが追記する）、または CLI のセッターが、利用者が入力した、または依頼したスイッチに対して、あるいは fixture/harness-launch の在席バイパスを通じて、ガードを引き下げた | Timestamp, Guard（フックからは `plan-approval-guard`、スイッチからは `plan-approval`、`review-freeze`、`state-transition`、または `reviewer-scope`）, Tool（フックの行）または Scope と Source（利用者によるものなら `you`、それ以外は `command`。スイッチの行）。利用者によるスイッチの行では Person Reply と Message Id（エージェントが `--message` で指定したメッセージと、記録が保持したその言葉） | `hooks/aidlc-plan-approval-guard.ts`（PreToolUse）; `tools/aidlc-utility.ts` がスイッチの行をまとめて処理する（`hooks/aidlc-record-human-turn.ts` からの `applyTypedGuardSwitchPrompt` を含む） |

### System Install Events (2 events - hook-emitted)

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `SYSTEM_INSTALL_BLOCKED` | エージェントが、任意のステージまたは分離された実行で、OS のパッケージマネージャーによるインストール（`apt-get install`、`dnf install`、`brew install`、`winget install` など。`sudo` の有無を問わない）を実行したが、前回の判断以降に有効な利用者のターンがなく、そのパッケージについて指定された利用者のメッセージ（`SYSTEM_INSTALL_REQUESTED`）もなく、利用者が承認した、それを指定する 1 つのプランもない（プラン承認オフで構築されたプランは何も指定しない）: コマンドは利用者が許可するまで待機する | Timestamp, Guard（`plan-approval-guard`）, Reason（`system-install`）, Tool, Target（コマンド。1 行で、160 文字で切る）, Stage。ホストがチャットに名前を付けている場合は Session | `hooks/aidlc-plan-approval-guard.ts`（PreToolUse）、`hooks/system-install.ts` を通じて |
| `SYSTEM_INSTALL_ALLOWED` | 同種のインストールが通された。利用者のターンがそれを裏付けていた（利用者自身の依頼、または保留後の利用者の許可）、またはエージェントがそのパッケージを求めるものとして指定したメッセージ（`SYSTEM_INSTALL_REQUESTED`）がまだ有効であるため。この行はそのメッセージが有効だったことを示すものであり、利用者が承認したとは決して示さない。また、ホスト自身のプロンプトや `sudo` が引き続きコマンドを止めることがある | Timestamp, Stage, Command, Message Id（ターンのもの、または指定されたメッセージ）, Person Reply（記録が保持した場合、利用者の言葉）。ホストがチャットに名前を付けている場合は Session | `hooks/aidlc-plan-approval-guard.ts`（PreToolUse）、`hooks/system-install.ts` を通じて |
| `SYSTEM_INSTALL_REQUESTED` | エージェントが、システムパッケージのインストールを求めた利用者のメッセージを指定した。それは承認や別の判断がすでに使ったメッセージである（"Approve. Then install shellcheck"）。エンジンはそのメッセージが依頼のものであること（この作業について、利用者の言葉で、未消費であること）を証明し、その言葉は一切読まなかった。そのメッセージが有効である間、これらのパッケージのインストールはそれに基づいて実行される | Timestamp, Packages（指定されたとおりの JSON リスト）, Message Id, Person Reply（記録が保持した場合、利用者の言葉） | `tools/aidlc-log.ts install` |

### Documents (3 events)

DocumentKB は**スペースレベル**のストアであるため、インテントスコープのドキュメントであっても、3 つのイベントはすべてスペースレベルの監査シャード（`spaces/<space>/intents/audit/`）に記録される。インテントの UUID はシャードの選択には使われず、フィールドとして記録される。これにより、`associate`/`dissociate` によるスコープ変更をまたいでも、1 つのドキュメントの履歴が 1 か所にまとまる。これらのイベントは来歴であり、**バックアップではない**。`documentkb/` ツリー全体を削除するとドキュメントごとの記録が失われるため、ドキュメント ID、トゥームストーン、インテントへのリンクは残らない。次の `sync` は、残っている原本をまったく新しい行として再インデックスする。復旧できるのは `index.json` だけが失われた場合のみである（`sync` がドキュメントごとの `metadata.json` ファイルから再構築する）。

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `DOCUMENT_INDEXED` | 顧客ドキュメントが初めて DocumentKB にインデックスされた（`onboard` から、および `sync` の新規ドキュメントの分岐から） | Timestamp, Space, Document, Source, Digest, 任意で Intent | `tools/aidlc-knowledge.ts` |
| `DOCUMENT_UPDATED` | インデックス済みドキュメントの記録が変わった。新しいリビジョン、再抽出、移動、サマリー、またはインテントとの関連付けの変更（`associate`、`dissociate`、`rebind`、`summarize`、`onboard` の編集済み行の分岐、`sync` の移動/変更/再試行の分岐、および冪等な監査修復パスから） | Timestamp, Space, Document, Change, 任意で Intent | `tools/aidlc-knowledge.ts` |
| `DOCUMENT_REMOVED` | 原本がなくなった。行はメタデータのみのトゥームストーンになり、抽出された内容は削除された（`sync` から） | Timestamp, Space, Document, Last Path, Last Digest | `tools/aidlc-knowledge.ts` |

ドキュメントが `related_intent_ids` を持つ場合でも、3 つともインテントのシャードではなく
**スペースレベル**のシャード（`intents/audit/`）に書き込まれる。ドキュメントはどの単一の
インテントよりも長く存続し、`associate`/`dissociate` によってスコープが移りうるため、来歴を
アクティブなインテントの下に記録すると、1 つのドキュメントの履歴がシャードをまたいで分割されてしまう。
何も変えない `associate`/`dissociate` はイベントを**発行しない**。呼び出しごとにイベントを出すと、
台帳が変更でないもので埋まり、台帳からの再構築が壊れるためである。

### Utility Events (1 event)

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `HEALTH_CHECKED` | `--doctor` が完了した | Timestamp, Request, Details | `tools/aidlc-doctor.ts`（共有のユーティリティコレクター経由） |

### Error/Recovery Events (3 events)

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `ERROR_LOGGED` | ツールの CLI が `error()` によって非ゼロで終了した、または Stop フックが固有のエンジンエラーのディレクティブを初めて届けた | Timestamp, Tool, Command, Error。任意で Source, Exit Code, Observed By, Error Fingerprint | `tools/aidlc-lib.ts emitError`（各ツールの `error()` ヘルパーから呼ばれる）および `hooks/aidlc-continue-workflow.ts` |
| `RECOVERY_COMPLETED` | ユーザーがコンテキスト圧縮の確認プロンプトに回答した | Timestamp, Choice, Current Stage | `tools/aidlc-state.ts acknowledge-compaction` |
| `COORDINATION_STOOD_ASIDE` | 助言のみ: Copilot アダプターが AI-DLC コマンドの調整記録を見つけられなかった、または信頼できなかった（このプロジェクトとインテントの記録がない、読めない記録である、または記録が書かれた後にワークフローの状態が動いた）ため、コマンドを拒否せずにエンジンへ通し、エンジンがディスクから応答した | Timestamp, 任意で Session, Command（`next`、`continue`、`report`、または `park`）, Reason | `tools/aidlc-audit.ts appendCoordinationStoodAside`（Copilot アダプターの `guard-tool-call`） |

Stop フックによる配信は、直近 32 件のフィンガープリントを FIFO 順で重複排除する。
フィンガープリントには、インテント、セッション、状態、ステージ、および 2,000 UTF-8 バイトに
制限された診断内容が含まれる。繰り返しは保持期間を更新しない。32 件の新しいエラーによって
追い出された診断は、配信時に再び `ERROR_LOGGED` を生成しうる。

### Construction Bolt Events (4 events)

Phase 3（Construction）の間にのみ発行される。Bolt については `stage-protocol.md` の Terminology を参照。Bolt はスプリントのようなイテレーションであり、意図されたユニットのグループ分けは Delivery Planning（2.9）で記録され、実行時のバッチ分けはユニットの依存グラフから決まる。`BOLT_STARTED` / `BOLT_COMPLETED`（およびスウォーム経路の `BOLT_FAILED`）は、スウォーム / ワークツリー経路で `aidlc-bolt.ts` から発行される。既定のゲート付き実行ではこれらは記録されない。`AUTONOMY_MODE_SET` は、既定のゲート付き実行を含むどの走査でも、ラダーの後に `aidlc-bolt.ts set-autonomy` によって発行される。

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `BOLT_STARTED` | スウォーム / ワークツリー経路のみ: 1 つのユニットとそのワークツリーに対する `aidlc-bolt.ts start`。既定のゲート付き stage-major 実行では発行されない。 | Timestamp, Bolt names, Batch number, Walking skeleton (true/false), 任意で Bolt slug, Base commit, Base Source Listing（`--worktree` の場合。この一覧は、ワークツリーの作成時から引き継がれる、内容アドレス型で raw を考慮したソースベースラインである）, Attempt Generation（チームのユニットクレーム） | `tools/aidlc-bolt.ts start` |
| `BOLT_COMPLETED` | スウォーム / ワークツリー経路のみ: 同じユニット/ワークツリーに対する `aidlc-bolt.ts complete`。バッチを閉じる**ことはない**。閉じるのは `SWARM_COMPLETED` である。 | Timestamp, Bolt names, Batch number, 任意で Bolt slug（--merge の場合）, 任意で Attempt Generation（チームのユニットクレーム） | `tools/aidlc-bolt.ts complete` |
| `BOLT_FAILED` | code-generation 中に Bolt が失敗した、またはユーザーによって明示的に中止された | Timestamp, Failed Bolt, Error summary, 任意で Bolt slug（`aidlc-worktree info --slug` が読む、停止して尋ねるための相関の接点）, 任意で Reason（明示的な中止では `aborted`）, 任意で Succeeded siblings, 任意で Attempt Generation（チームのユニットクレーム） | `tools/aidlc-bolt.ts fail` および `tools/aidlc-bolt.ts abort`。`tools/aidlc-state.ts set-construction-execution serial` は、並行ビルドが準備したまま着地させなかった Bolt を `abort` と同じ方法で閉じる（ワークツリーは保持される） |
| `AUTONOMY_MODE_SET` | ウォーキングスケルトンの後のラダーのプロンプトにユーザーが回答した | Timestamp, Mode（`autonomous` または `gated`）。任意で Message Id（`autonomous` を許可した利用者のメッセージ。`--message` で指定され、許可によって消費される） | `tools/aidlc-bolt.ts set-autonomy` |

### Worktree (7 events)

Bolt が Bolt ごとの git ワークツリー内で実行されるときに、Phase 3（Construction）の間に発行される。ワークツリーのプリミティブが `WORKTREE_*` を、state の fork/merge サブコマンドが `STATE_*` を、audit の fork/merge サブコマンドが `AUDIT_*` を発行する。

新しい `Worktree path` の値はプロジェクト相対
（`.aidlc/worktrees/bolt-<id8>_<slug>`）であり、`Branch name` には
`bolt-<id8>_<slug>` がそのまま記録される。読み取り側はパスをプロジェクトルート基準で解決し、
レガシーの絶対パスの値や、来歴が一致するレガシーの Bolt とも互換性を保つ。
`Bolt slug` は人間/監査向けの素の識別子のままである。`<id8>` は、ユニットのクレームが使うものと同じ
インテントレジストリ UUID の接尾辞であり、保持されるソース参照は
`refs/aidlc/reviewed-source/<id8>/<slug>/<commit>` を使う。命名とレガシーの方針については
[Bolt の識別](worktree-info-schema.md#bolt-identity) を参照。

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `WORKTREE_CREATED` | Bolt の開始時に、main から Bolt ごとの git ワークツリーが作成された | Timestamp, Bolt slug, プロジェクト相対の Worktree path, Branch name, Base branch, Base commit, Base Source Listing（監査を先に書く作成処理の前に、不変のベースから計算された raw を考慮したソース一覧に対する `sha256:<hash>`）, Repo（記録されたセレクター、またはワークスペースルートの場合は `-`）, 任意で Intent record と、スウォームの Unit/Batch/Stage/Run floor の来歴 | `tools/aidlc-worktree.ts` (`create`) |
| `WORKTREE_MERGED` | ゲートの承認時に、Bolt のワークツリーが main へマージされた | Timestamp, Bolt slug, Worktree path, Target branch, Strategy | `tools/aidlc-worktree.ts` (`merge`) |
| `WORKTREE_DISCARDED` | Bolt の復旧可能な作業ツリーのスナップショット（または残っているブランチの先端）とレビュー済みのソース参照が、監査の発行前に `refs/aidlc/parked/<id8>/<slug>/<stamp>/` の下に退避され、その後ライブのチェックアウトとブランチが削除された | Timestamp, Bolt slug, Worktree path, Repo（記録されたセレクター、またはワークスペースルートの場合は `-`）, Reason, Parked ref（名前空間の接頭辞）, Parked commit（`<parked ref>/snapshot` でマークされたスナップショットのコミット、`<parked ref>/branch-tip` でマークされた残りのブランチの先端、またはコミットを退避できなかった場合は `-`。マークのないレガシーの head は、コミットの author が `AI-DLC <aidlc@localhost>` で、件名が `aidlc: parked bolt-<slug> at ` で始まる場合にのみスナップショットとみなす） | `tools/aidlc-worktree.ts` (`discard`) |
| `STATE_FORKED` | Bolt の開始時に、状態ファイルがワークツリーへフォークされた | Timestamp, Bolt slug, Worktree path, Source state hash, Target state hash, 任意で Attempt Generation（チームのユニットクレーム） | `tools/aidlc-state.ts` (`fork`) |
| `STATE_MERGED` | ゲートの承認時に、ワークツリーの状態が main の状態へマージされた | Timestamp, Bolt slug, Worktree path, Source state hash, Target state hash, Conflict resolution | `tools/aidlc-state.ts` (`merge`) |
| `AUDIT_FORKED` | Bolt の開始時に、監査ログがワークツリーへフォークされた（意図の監査: 発行がバイトコピーより先に行われる） | Timestamp, Bolt slug, Source Audit Hash, Fork Boundary, 任意で Attempt Generation（チームのユニットクレーム） | `tools/aidlc-audit.ts` (`audit-fork`) |
| `AUDIT_MERGED` | ゲートの承認時に、ワークツリーの監査エントリが main の監査に追記された。Bolt ごとのエントリ順は保たれ、Bolt をまたぐ順序はマージ完了の順序を反映する。差分内で対になる `REVIEW_COMPLETED` 行が指すレビュー記録は、ダイジェストを検証したうえで先に main のインテント記録へ運ばれ、それができなければマージは拒否される | Timestamp, Bolt slug, Entries Merged, Source Audit Hash, Fork Boundary, Fork Timestamp | `tools/aidlc-audit.ts` (`audit-merge`) |

### Practices (4 events)

Inception ステージ `practices-discovery` と、実行時の Construction オーケストレーターによって発行される。ステージは確認ゲートで発行し、オーケストレーターは実行時に `--type empty`（フォールバックの助言）と `--type override`（bolt-plan-marker-conflict 経路の識別フィールド）を通じて発行する。

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `PRACTICES_DISCOVERED` | グリーンフィールドまたはブラウンフィールドのリードの草案、3 つのサポートからの寄稿、人へのインタビュー、リードによる統合が完了した。草案は確認待ち | Timestamp, sources scanned, drafts produced | `tools/aidlc-state.ts` `practices-event --type discovered` |
| `PRACTICES_AFFIRMED` | チームが practices-discovery の確認ゲートでプラクティスを承認した。内容は `aidlc/spaces/<active-space>/memory/team.md` と `project.md` へ昇格された | Timestamp, affirming user, sections written, mandated/forbidden rules appended | `tools/aidlc-state.ts` `practices-promote` |
| `PRACTICES_OVERRIDE` | practices-discovery の確認中に行をまたぐ昇格が失敗した、またはアクティブスペースの `team.md` にあるウォーキングスケルトンの方針が、現在の Bolt についての bolt-plan のマーカーより優先された | Timestamp, Reason（識別子）。経路ごとのフィールドセット: write-failure 経路は Reason + Failure detail のみを発行する（Bolt のフィールドはない）。bolt-plan-marker-conflict 経路は Reason + Bolt slug + Practices Stance + Bolt-Plan Marker を発行する。2 つのフィールドセットは重ならないため、doctor は `Reason` で絞り込み、どちらかの名前の系統で振り分ける。確認の昇格経路には `write-failure-*`、オーケストレーターの実行時経路には `bolt-plan-marker-conflict` | `tools/aidlc-state.ts` `practices-promote`（write-failure 経路）; `tools/aidlc-state.ts` `practices-event --type override`（bolt-plan-marker-conflict 経路。識別フィールドによる区別で、別のイベントはない） |
| `PRACTICES_SECTION_EMPTY` | オーケストレーターが読んだプラクティスのセクションが空だったため、org の既定値にフォールバックする（助言のみ） | Timestamp, Section name, Fallback source | `tools/aidlc-state.ts` `practices-event --type empty` |

### Merge Dispatch (3 events)

Construction の Bolt マージのステップが、チームのプラクティスの散文からマージ戦略を決めるために、Task を介して aidlc-pipeline-deploy-agent を呼び出すときに発行される。`aidlc-bolt dispatch-event` サブコマンドを通じて発行される。オーケストレーターは aidlc-pipeline-deploy-agent の各派遣を前後で囲む。呼び出し前に INVOKED、解析に成功した場合は呼び出し後に RETURNED、タイムアウトまたは不正な YAML の場合は FALLBACK を発行する。意図の監査の意味論: INVOKED は LLM の Task 呼び出しの前に発行される（派遣自体にはディスクへの副作用がなく、照合は slug + タイムスタンプの時間枠で行う）。doctor は孤立した INVOKED 行を照合する。

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `MERGE_DISPATCH_INVOKED` | オーケストレーターが、現在のプラクティスのセクション + Bolt のコンテキストとともに aidlc-pipeline-deploy-agent を派遣した | Timestamp, Bolt slug, Practices section excerpt。任意で Pinned OID + Attempt Generation + Pin Transaction | `tools/aidlc-bolt.ts` `dispatch-event --event MERGE_DISPATCH_INVOKED` |
| `MERGE_DISPATCH_RETURNED` | エージェントが、strategy、target branch、confidence、notes を含む解析済みの YAML を返した | Timestamp, Bolt slug, Strategy, Target branch, Confidence, Notes。任意で Pinned OID + Attempt Generation + Pin Transaction | `tools/aidlc-bolt.ts` `dispatch-event --event MERGE_DISPATCH_RETURNED` |
| `MERGE_DISPATCH_FALLBACK` | エージェントがタイムアウトした、または不正な YAML を返したため、オーケストレーターが org の既定値にフォールバックした。可観測性にとって重要なフック | Timestamp, Bolt slug, Fallback reason, Defaults applied。任意で Pinned OID + Attempt Generation + Pin Transaction | `tools/aidlc-bolt.ts` `dispatch-event --event MERGE_DISPATCH_FALLBACK` |

### Sensor Events (6 events)

決定論的センサーのシステムによって発行される。センサーディスパッチャーは 5 つの `SENSOR_*` イベントを発行する。ゲートでの発火が判定を返さず結果行もない場合、ゲートはそれを `SENSOR_COULD_NOT_RUN` で閉じる。ディスパッチャーがそれを書いていなければ、その前にゲート自身の `SENSOR_FIRED` を書く（`tools/aidlc-sensor-record.ts` の `recordUnfinishedFire`）。対になるカバレッジを確認する doctor の行は `Scope: all` とともに `GUARDRAIL_LOADED` を発行する。doctor はアクティブなステージなしで解決済みのガードレール全体を読むためである（下の When 句にあるワークフローごとの org → project → phase → stage のスコープ設定は定常状態のローダーを説明するものであり、doctor のスコープなしの読み取りではない）。`fire_on: write` のセンサーは、一致する書き込みに対して PostToolUse フックから実行される。`fire_on: gate` のセンサーは、`gate-start` が最初のゲートを開く前、`revise` がゲートに再入する前、または承認時の修正バックストップが補完による再入を試みる前に、宣言された既存の成果物ごとに 1 回実行される（`upstream-coverage` と `claim-sources` は出力フォルダーごとに 1 回）。そのうち助言のセンサーは、試行のより早い時点でも 1 回、レビュー要求時、または（レビューが求められなかった場合は）ゲートを開く報告時に、スコープの出力に対して発火する。ユニットの要求についての行には `Unit` が付く。Construction の自律性の下でエンジン自身が承認するゲートでは、ブロッキングのゲートセンサーのみが実行される（そこでは助言の証拠を読む者がいない）。このリリースでは、ブロッキングの重大度はゲートで発火したセンサーにのみ適用され、書き込みで発火した失敗は助言のままである。

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `SENSOR_FIRED` | ディスパッチャーが、一致する PostToolUse の Write/Edit、またはゲート境界での成果物の派遣のいずれかから、ステージの出力に対してセンサーを起動した。ディスパッチャーが行を書かず判定も返さなかったゲートでの発火については、ゲートが対になる `SENSOR_COULD_NOT_RUN` とともにこれを書き込む | Timestamp, Fire id, Sensor ID, Stage slug, 任意で Unit（ユニットごとのステージが作業していたユニット。アクティブなディレクティブから取得し、終端行にも反映される）, Output path | `tools/aidlc-sensor.ts` `fire`。判定のないゲートでの発火については、`tools/aidlc-sensor-record.ts` の `recordUnfinishedFire` を通じてゲートが書き込む |
| `SENSOR_PASSED` | センサーが完了し、指摘を報告しなかった（tool-unavailable の場合と、詳細ファイルを書き込めなかった失敗の場合も含む。Note の脚注を参照） | Timestamp, Fire id, Sensor ID, Stage slug, 任意で Unit, Output path, Duration ms | `tools/aidlc-sensor.ts` `fire` |
| `SENSOR_FAILED` | センサーが完了し、指摘を報告した。詳細ファイルは `<record>/.aidlc-engine/sensors/<stage-slug>/<sensor-id>-<fire-id>.md` に書き込まれる | Timestamp, Fire id, Sensor ID, Stage slug, 任意で Unit, Output path, Detail path, Findings count, 任意で Failed outputs（ファイルの集合に対する発火の場合: 失敗したもの。最初の 20 件、その後に `and N more`） | `tools/aidlc-sensor.ts` `fire` |
| `SENSOR_COULD_NOT_RUN` | 検査を実行できなかった: スクリプトを開始できなかった、非ゼロ（127 以外）で終了した、シグナルで停止された、または判定を出力しなかった。何も検査していないため、合格にも失敗にも決してならない。ゲートはそれを 1 行で示し、何も止めない（ただし、ブロッキングのゲートバインディングは検証済みの合格を必要とするため、引き続き拒否する） | Timestamp, Fire id, Sensor ID, Stage slug, 任意で Unit, Output path, Duration ms, Note（`script-error: <reason>`: `spawn-failed: <code>`, `exit-<n>`, `external-sigterm`, `signal-<name>`, `bad-output`, `unknown`。またはゲートでの発火が判定を返さなかった場合は `dispatch-error: <reason>`: `timeout`, `spawn-failed: <code or unknown>`, `signal-<name or unknown>`, `exit-<n>`, `bad-output`） | `tools/aidlc-sensor.ts` `fire`。判定のないゲートでの発火については、`tools/aidlc-sensor-record.ts` の `recordUnfinishedFire` を通じてゲートが書き込む |
| `SENSOR_BUDGET_OVERRIDE` | センサーが設定された上限（3 層の上限モデルによる registry / binding / 深度由来の値）を超え、終了またはスキップされた | Timestamp, Fire id, Sensor ID, Stage slug, 任意で Unit, Output path, Cap layer, Cap value, Observed value | `tools/aidlc-sensor.ts` `fire` |
| `GUARDRAIL_LOADED` | ガードレールローダーが、アクティブなワークフローについてスコープ階層のガードレールセット（org → project → phase → stage）を解決した。doctor の対になるカバレッジの検査はこのイベントを読む | Timestamp, Scope, Path, Rule count | `tools/aidlc-utility.ts` |

> `SENSOR_PASSED` の `Note` フィールドは任意である。センサーごとのスクリプトが使う実行ファイルが PATH 上にない場合は `tool-unavailable` を、失敗した検査の詳細ファイルを書き込めなかった場合は `script-error: detail-write-failed: <reason>` を持つ。実行できなかったスクリプトは `SENSOR_COULD_NOT_RUN` であり、決して合格ではない。書き込みで発火するセンサーではこれらは助言のままだが、ブロッキングのゲートバインディングは検証済みの合格を必要とするため、`Note` があるもの、実行できなかった検査、ディスパッチャーの失敗、不正な判定、予算のオーバーライドはいずれもゲートへの進入を拒否する。対の相関は `Fire id`（`SENSOR_FIRED` から終端行へそのまま反映される）で行う。`Output path` だけでは、繰り返された書き込みの派遣や、同じセンサー + ステージ + パスの組に対する後のゲートの派遣を区別できない。

> **監査行のインデックスではなく `Fire id` で対応付ける。** 書き込みの派遣は 1 回のツール呼び出しを複数のセンサーへ展開することがあり、ゲートの派遣はゲートに結び付いた各センサーを既存のすべての成果物へ展開する。終端行は起動時間によって入り混じることがあるため、`findAllEvents("SENSOR_FIRED")[i]` はインデックスで `findAllEvents("SENSOR_PASSED")[i]` と対にはならない。監査を走査する利用側（`sensor_firings[]` の生成処理、doctor、designer）は、8 桁 16 進の `Fire id` 相関子によって終端行を FIRED 行に必ず対応付けなければならない。

### Learning Loop (3 events)

stage-protocol §13（Learnings Ritual）によって発行される。ランタイムグラフのコンパイルは、承認されたばかりのステージの memory.md が 4 つの標準見出しの下に空でないエントリを 1 つも持たない場合に `MEMORY_EMPTY` を発行する。学びのゲートツールは、ユーザーが提示された学びまたは自由記述の学びを残すと `RULE_LEARNED` を発行し（学びはプラクティスそのものであり、`{project,team}.md` の振り分け先の見出しの下にプラクティスの行として記録される）、学びがセンサーのバインディング（マニフェスト + 発生元ステージの `sensors:` frontmatter）を導入すると `SENSOR_PROPOSED` を発行する。doctor は `MEMORY_EMPTY` 行を時系列で読み、ステージをまたいだ日誌の体系的な省略を検出する。

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `MEMORY_EMPTY` | ステージの承認がランタイムグラフのコンパイルを引き起こし、そのステージの memory.md が §13 の 4 つの見出しのいずれの下にも空でないエントリを持たなかった | Timestamp, Stage | `tools/aidlc-runtime.ts compile` |
| `RULE_LEARNED` | 学びのゲートが、残された学びを `{project,team}.md` の振り分け先の見出しの下にプラクティスの行として永続化した | Timestamp, Stage, Candidate-ID, Content-Hash, Destination, Heading, Source | `tools/aidlc-learnings.ts persist` |
| `SENSOR_PROPOSED` | 学びのゲートがプロジェクト層のセンサーマニフェストの雛形を作成し、発生元ステージの `sensors:` frontmatter に結び付けた | Timestamp, Stage, Candidate-ID, Sensor ID, Manifest path, Matches, Destinations, Source | `tools/aidlc-learnings.ts persist` |

### Swarm (9 events)

6 つのイベントはスウォームのレフェリー `aidlc-swarm.ts`（コンダクターが参照する決定論的な判定の接点）から発行され、`SWARM_SOURCE_MERGED` はアプリケーションのソースが main に着地した後に `aidlc-worktree.ts` から発行される。レフェリーはステートレスである（イテレーションのカウンターを持たない）。`prepare` は正確なステージ試行のトークンを取得し、Unit/Batch/Stage/Run floor を `WORKTREE_CREATED` とワークツリーのメタデータに刻み、ユニットごとのワークツリーをフォークし、準備したバッチと、試行に結び付いたユニットの義務の全体集合の両方とともに `SWARM_STARTED` を発行する（コンダクターが明示的な格下げを報告した場合は `SWARM_DEGRADED` も発行する）。`finalize` は、マージの前に、準備したそのトークンが現在の試行とまだ一致していることを要求し、各収束行にそれを保持する。完了時には、現在の作成済み DAG をその永続的な義務の集合と比較し、DAG が元に戻されるかステージの試行が再開されるまで、縮小や拡大を拒否する。直前のスタンプなしのリリースが準備したワークツリーは、その凍結された監査プレフィックスが対応するレガシーの `SWARM_STARTED`、`BOLT_STARTED`、`STATE_FORKED`、`AUDIT_FORKED` の連続を証明し、フォークのハッシュがまだ一致し、3 つのワークツリーのミラーがすべて存在し、導出された凍結済みの試行がなお現在の試行と等しい場合にのみ finalize できる。この互換経路が、日付を遡った開始を発行したり、古いワークツリーを採用したりすることは決してない。finalize は、コンダクターが収束したと主張する集合を再検証し、宣言された正確な記録成果物と、結び付けられた `source-manifest.json` のスナップショットを取り、本当に合格したものについてそれらの記録と AIDLC のメタデータを直列化してマージし、ユニットごとの組（`SWARM_UNIT_CONVERGED` / `SWARM_UNIT_FAILED`。ただし、記録/メタデータの書き戻しマージが失敗した収束済みユニットは、finalize の再試行でマージされるまでどちらの行も得ない）、失敗したユニットごとのバトン行（`SWARM_BATON_RETURNED`）、およびバッチの集計（`SWARM_COMPLETED`）を発行する。続くソースのマージは、永続的なワークツリーの来歴と現在の Bolt/バッチ/ステージ試行に相関付けられた権限を必要とし、main のチェックアウトを、ステージ開始時のベースライン、または `GATE_REJECTED` に記録された前回の試行の検証済み最終集約から連結し、`SWARM_SOURCE_MERGED` を発行する。却下が完了時のベースラインを置き換えることは決してない。最初のリンクはその信頼された先行者に対して再検証され、以降のリンクは正確なフィンガープリントの連鎖をなさなければならない。収束が不変のソースフィールドの導入以前である、バインディング導入以前のワークツリーは、従来のブランチマージを維持する。現代的な形式で不完全な権限はフェイルクローズし、バッチのルーティングを前進させない。Git への変更が着地したのに `SWARM_SOURCE_MERGED` を追記できない場合、コマンドはワークツリーを保持し、再試行不可能な `[merge-succeeded:<sha>]` の失敗を報告する。認証されたリカバリーのレシートが存在しないため、マージを再実行してはならない。ステージの試行を再開するか、ソースの鮮度に関する明示的なオフスイッチを人の承認がある場合にのみ使う。`SWARM_SOURCE_MERGED` が着地し、その後のワークツリー、ブランチ、または保持された参照のクリーンアップだけが失敗した場合は、同じマージを再実行すると、ソースを再適用したり権限を重複させたりせずに、クリーンアップのみの照合が行われる。`check` サブコマンドは何も発行しない。これはコンダクターの再試行の判断に情報を与える助言の判定である。ループとその上限はドライバー側にあるため、ユニットごとの行は `Iterations` / `Cap value` フィールドを持たない。

strict な Guard Policy の下でユニットが構築されている間に、利用者が main のチェックアウト内のファイルの変更を求めた場合、`allow-change` が `MAIN_CHECKOUT_CHANGE_ALLOWED` を発行する。別のチャットが開始したバッチの構築を利用者が 2 つ目のチャットに求めた場合、`take-over` が `SWARM_TAKEN_OVER` を発行する。

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `SWARM_STARTED` | スウォームのレフェリー `prepare` が正確な試行を取得し、依存関係で結ばれたユニットのバッチをフォークした | Timestamp, Batch number, Unit names, Unit obligations, Concurrency cap, Stage, Run floor, Landing branch（ユニットのリポジトリでチェックアウトされているブランチ。ある場合）, Session（バッチを開始したチャット。ホストがチャットに名前を付けている場合） | `tools/aidlc-swarm.ts` |
| `SWARM_UNIT_CONVERGED` | スウォームのユニットが再検証でグリーン（かつ改ざんなし）となり、宣言された正確な記録成果物と結び付けられたソースマニフェストが着地し、AIDLC のメタデータの書き戻しマージが着地した。明示的にバイパスされない限り、finalize は現在のグローバル/ユニットのソースバインディングと、raw を考慮したベースからワークツリーへのフットプリントも、レビュー済みのソースマニフェストのクレームに対して検証した | Timestamp, Batch number, Unit name, Stage, Run floor, Command SHA-256（承認された Construction 検証コマンドのダイジェスト。以前のリリースの行にはないことがある）, 任意で Source Fingerprint と Source Commit（`aidlc-worktree merge` が受け入れる不変のレビュー済みソース）、または finalize が明示的に `AIDLC_SKIP_SOURCE_FRESHNESS=1` を使った場合は `Source Freshness Bypass: true`（鮮度/ユニット/フットプリントの保証は適用されず、merge でも同じスイッチを繰り返す必要がある） | `tools/aidlc-swarm.ts` |
| `SWARM_SOURCE_MERGED` | 現在の試行のスウォームユニット 1 つについて、不変のレビュー済みソースが main に着地し、集約ソースの連鎖を延長した | Timestamp, Batch number, Unit name, Stage, Run floor, Previous Source Fingerprint, Source Fingerprint, Source Listing（着地が残した内容アドレス型の一覧。保持できなかった場合や以前のリリースが書いた行にはない）, Source Commit, Merge commit, Repo（記録されたセレクター、またはワークスペースルートの場合は `-`）。着地がレビュー済みソースと異なる場合: Landed changes（着地したバイト列がレビュー済みのワークツリーと異なるパス。利用者のフックやフィルターによるもの、または先に着地したユニットと衝突した箇所を手作業で統合したもの）と Driver merges（利用者が設定したマージドライバーがマージしたパス） | `tools/aidlc-worktree.ts merge` |
| `MAIN_CHECKOUT_CHANGE_ALLOWED` | strict な Guard Policy の下で、ユニットが並行して構築されている間に、利用者が main のチェックアウト内のファイルの変更を求めた。plan-approval ガードはそれらの書き込みを通し、着地、次のバッチの prepare、ステージの完了はそれらを保持する（各保持は同じ Message Id を持つ `CHANGE_ACCEPTED` 行になる） | Timestamp, Stage, Run floor（保持の対象となる Code Generation の試行）, Paths（プロジェクト相対パスの JSON リスト）, Person Reply（メッセージにある場合、利用者の言葉）, Message Id | `tools/aidlc-swarm.ts allow-change` |
| `SWARM_TAKEN_OVER` | このクローン上の別のチャットが開始してまだ着地していない並行バッチの構築を、利用者が 2 つ目のチャットに求めた。`next` はそれ以降そのチャットにバッチを渡し、もう一方のチャットでは別のチャットが構築を引き継いだことを伝える | Timestamp, Batch number, Stage, Run floor, Session（引き継ぐチャット）, Person Reply（メッセージにある場合、利用者の言葉）, Message Id | `tools/aidlc-swarm.ts take-over` |
| `SWARM_UNIT_FAILED` | スウォームのユニットが `finalize` の再検証に失敗した（主張されていない、主張されたがレッド、または改ざんされている） | Timestamp, Batch number, Unit name, Reason | `tools/aidlc-swarm.ts` |
<!-- Reason for a CLAIMED-but-red / tampered unit is always the tool's own verdict (`error`); for a DECLINED (unclaimed) unit it is the conductor's typed attribution via `finalize --reasons` (`unsatisfiable` / `budget-exhausted` / `cap-exhausted`, defaulting to `cap-exhausted`) — the tool records the conductor's knowledge call, it does not judge unsatisfiability itself (D-I). -->
| `SWARM_BATON_RETURNED` | スウォームのユニットが、オーケストレーターを介した調整のためにバトンをコンダクターへ返した | Timestamp, Batch number, Unit name, Reason | `tools/aidlc-swarm.ts` |
| `SWARM_COMPLETED` | バッチ内のすべてのユニットが終了し（収束または失敗）、バッチが閉じられた | Timestamp, Batch number, Converged count, Failed count | `tools/aidlc-swarm.ts` |
| `SWARM_DEGRADED` | `AIDLC_USE_SWARM=1` が要求されたが Workflow ツールが利用できなかったため、コンダクターがサブエージェントによる下限の方式で実行した（明示的な格下げ） | Timestamp, Batch number, Requested driver, Fallback driver | `tools/aidlc-swarm.ts` |

### Commit Provenance (1 event)

レビュー済みソースのクレームを着地させたコミットが観測されたときに、`aidlc attest anchor` によって発行される。補強のみ: `aidlc attest resolve` はこれらの行を決して読まない。帰属は、コミットされた内容（REVIEW_COMPLETED のレシート + コミットされた `reviewed-source-<hash12>.tsv` の証拠）の純粋な関数のままである。したがって、アンカーされなかったコミットも同じように解決される。関係するインテントごと、（commit, repo）ごとに 1 行。再アンカーは重複排除され、同じ（commit, repo）について `SWARM_SOURCE_MERGED` のレシートをすでに持つコミットは、二重に記録されずスキップされる。

| Event | When | Required Fields | Emitter |
|-------|------|-----------------|---------|
| `SOURCE_COMMITTED` | コミットの変更パスがレビュー済みのユニットに帰属された（明示的な `anchor` の呼び出し、またはオプトインのセッション開始時の照合スイープ） | Timestamp, Commit, Repo（記録されたセレクター、またはワークスペースルートの場合は `-`）, Units, Attributed Paths, Observed（`session` \| `reconciled`） | `tools/aidlc-attest.ts anchor`（runAnchor。`AIDLC_SESSION_ANCHOR=1` の場合はセッション開始フックのスイープからも呼ばれる） |

## Hook-Generated Format

フックは、オーケストレーター駆動の発行と同じライブラリの発行関数（`tools/aidlc-audit.ts` の `appendAuditEntry`）を通じてイベントを発行する。フックが発行するイベントは分類体系の正式なメンバーである（`ARTIFACT_CREATED`、`ARTIFACT_UPDATED`、`SUBAGENT_COMPLETED`、すべての `SESSION_*`）。`cwd` にアクティブなワークフローがない場合、フックは何もしない。

公開の `aidlc-audit.ts append` CLI は診断用の非常口であり、正規の発行経路ではない。これは `CLI_PROTECTED_EVENT_TYPES`（`tools/aidlc-audit.ts`）にある権限を伴うレシートを拒否する。これらは、それを所有するツールまたはフックだけが発行できる: `STAGE_COMPLETED`, `STATE_UPGRADED`, `STAGE_DEFINITION_CHANGED`, `STAGE_OUTPUT_AMENDED`, `SOURCE_WORK_NOT_FOUND`, `HUMAN_TURN`, `HOST_TURN`, `QUESTION_UNANSWERED`, `QUESTION_REPLIED`, `REQUEST_ROUTED`, `QUESTION_ASKED`, `QUESTION_SHOWN`, `GATE_APPROVED`, `GATE_REJECTED`, `QUESTION_ANSWERED`, `PLAN_APPROVAL_RECORDED`, `PLAN_APPROVAL_OVERRIDDEN`, `PLAN_APPROVAL_SKIPPED`, `REVIEW_REQUESTED`, `REVIEW_COMPLETED`, `PIPELINE_LINK_COMPLETED`, `ARTIFACT_REUSED`, `SWARM_STARTED`, `SWARM_UNIT_CONVERGED`, `SWARM_SOURCE_MERGED`, `MAIN_CHECKOUT_CHANGE_ALLOWED`, `SWARM_TAKEN_OVER`, `AUTONOMY_MODE_SET`, `UNIT_OWNERSHIP_SET`, `UNIT_GATE_RHYTHM_SET`, `CONSTRUCTION_POLICY_SET`, `UNIT_STARTED`, `UNIT_PAUSED`, `UNIT_RESUMED`, `UNIT_COMPLETED`, `UNIT_SKIPPED`, `UNIT_MERGED`, `DOCUMENT_INDEXED`, `DOCUMENT_UPDATED`, `DOCUMENT_REMOVED`, `BOLT_STARTED`, `BOLT_COMPLETED`, `BOLT_FAILED`, `AUDIT_FORKED`, `AUDIT_MERGED`, `STATE_FORKED`, `STATE_MERGED`, `WORKTREE_CREATED`, `WORKTREE_MERGED`, `WORKTREE_DISCARDED`, `SOURCE_COMMITTED`, `GUARD_POLICY_SET`, `CHANGE_CONTROL_SET`, `CHANGE_ACCEPTED`, `GUARD_RESTORED`, `GUARD_STOOD_ASIDE`, `SYSTEM_INSTALL_ALLOWED`, `SYSTEM_INSTALL_REQUESTED`, `CEREMONY_SET`。フィールド名は、監査フィールドの文法に一致する印字可能な 1 行のラベルでなければならず、値はすべての行終端がエスケープされる。`append-raw` も同様に、分類体系のイベントを指す `**Event**:` 行や空の `**Event**:` 行を含む本文を拒否し、改行を含む見出しも拒否する。イベントの読み取り側は、`**Event**:` の値をその行自体からのみ取得する。

## Format Standards

- すべてのタイムスタンプ: ISO 8601 形式（YYYY-MM-DDTHH:MM:SSZ）
- すべてのエントリのタイムスタンプは、発行するツールまたはフックが付与する。手作業で日付を入れるものはない
- ツールが所有する: エージェントがシャードに直接書き込むことは決してない（PreToolUse ガードがそれを拒否する）。すべての行は所有するコマンドを通じて届く
- 追記のみ: 既存のエントリを決して変更・削除しない
- 機密データ（認証情報、PII、シークレット）を含めない。ツールは、作業の説明を含むすべての値で、キー、トークン、パスワードの形をしたものを `[masked]` としてマスクし、メッセージ全体は git で無視されるメッセージ記録に残る
- 人の判断はそのまま記録する: 決して要約しない

### Free-form note format (`append-raw`)

所有する分類体系イベントを持たないメモ（回避したエラー、リカバリー、
ワークフロー途中の変更要求）は、次のコマンド入力を使う:

```bash
{{INVOKE}} engine audit append-raw 'Error: <brief>' '**Severity**: <level>\n**Description**: <what happened>\n**Resolution**: <action taken>'
```

それらのメモには `Recovery: <brief>` または `Change Request: <brief>` を使う。
ユーザーの言葉をそのまま渡し、認証情報、PII、シークレットは除外する。整形と
タイムスタンプはツールが担う。Event や Timestamp のフィールドを指定してはならない。

## Validation basis on `STAGE_COMPLETED`

メインワークフローの `STAGE_COMPLETED` エントリには、コンパクトなスキーマ 3 のレシートを含めることができる:

```text
**Validation Basis**: {"schema":3,"graphContract":"sha256:...","projectType":"brownfield","inputs":[{"artifact":"code-summary","producer":"code-generation","required":true,"instanceCount":12,"presentCount":12,"structureHash":"sha256:...","contentHash":"sha256:..."}],"outputs":[...]}
```

リゾルバーはまず、Bolt の DAG、ユニットの種類、`produces_kinds`、正規のファイル名エイリアスを使って、
具体的な実行時の成果物インスタンスの集合を計算する。次にレシートは、ユニット/パスの
行をすべて保存するのではなく、決定論的なステージレベルの集約フィンガープリントを保存する。
任意の入力は、完了が報告された時点で少なくとも 1 つのインスタンスが存在した場合にのみ含まれる。
これは報告時のスナップショットであり、ステージが実際にどのバイト列を読んだかを証明するものではない。
"Observed" の依存関係とは、このレシートに記録されたという意味であり、実行時の読み取り計測ではない。
取得前の変更はベースラインとなり、取得後の変更は検出可能である。

レシートの取得が利用できない場合でも、所有する完了ツールは `Validation Warning` 付きで
レシートなしの `STAGE_COMPLETED` を追記する。その完了は追跡されない助言扱いであり、
状態遷移を中止する理由にはならない。

最新のレシートは、ステージが再オープンされた後も依存関係の証拠として引き続き有用だが、
ステージ自体の現在の追跡として数えられるのは現在の試行での完了のみである。以前のスキーマと
レシートなしの完了は、ステージがこのスキーマで完了するまでフェイルオープンする。スキーマ 2 は、
以前のゼロインスタンスの解決と、スキーマ 3 で導入されたステージレベルのゼロユニットの解決とを
区別できないため、意図的に追跡対象外としている。

完了後に変わったのがステージ自体だけで（グラフコントラクトに加え、新たに読み書きするようになった、
またはしなくなった入力や出力、あるいは `required` フラグが動いた入力や出力）、利用者の作業が何も
変わっていない場合、そのステージは古くなっていない。次のステップが告げられると（ステージが以前は
不要だった入力を必要とするようになった場合にのみ、その旨を述べる）、エンジンは `STAGE_DEFINITION_CHANGED` 行を追記する。
その `Validation Basis` は新しい定義に対するステージのレシートであり、ステージが再び開始されるまで
完了レシートの代わりとなる。

完了後の唯一の変更がステージ自身のドキュメントの内容に対するもので、かつ完了時の承認が
利用者の最新のメッセージを指している場合（その後、作業に関するどの会話でも利用者のメッセージがなく、
別のメッセージを指す判断や何も指さない判断もない場合）、どの Guard Policy の下でもそのステージは
古くなっていない。変更はそれを承認したターンで行われたからである。次のステップが告げられると
（それについては何も述べない）、エンジンは `STAGE_OUTPUT_AMENDED` 行を追記する。その `Validation Basis` は
現状のファイルに対するステージのレシートであり、上の行と同様に完了レシートの代わりとなるため、
後の編集はこれと比較される。利用者の次のメッセージより後の変更、または何のメッセージも指さない
承認での変更は、従来どおり古いものとみなされる。
