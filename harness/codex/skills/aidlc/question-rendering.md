# Question Rendering — Codex CLI harness annex

このファイルは、`{{HARNESS_DIR}}/aidlc-common/protocols/stage-protocol.md` の「Structured questions」セクションが
求める構造化された質問を、この**ハーネス**がどのように描画するかを定義する。
The protocol and stage files are harness-neutral: they say *present a
structured question* and carry a fenced ` ```question ` spec block. This annex
is the one place that binds that contract to a concrete mechanism.

## Never echo the spec (non-negotiable)

A ` ```question ` fenced block is **INPUT to this annex's rendering, never
output to paste**. The orchestrator MUST render every ` ```question ` spec
THROUGH the mechanism below (the `request_user_input` tool when available, else
numbered prose), and MUST NEVER echo, print, paste, or "quote back" the fenced
block, or any of its field lines (`prompt:`, `header:`, `multiSelect:`,
`options:`, `label:`, `description:`), into the chat transcript. The user must
never see the raw fence; they see only the native prompt or the numbered-prose
rendering.

Echoing the fence as literal text is a **protocol violation**, not a stylistic
choice. It:

- produces a non-interactive wall of text with no answerable prompt;
- drops the "None of the above" / "Other" escape the rendering provides;
- is inconsistent with every correct rendering elsewhere in the same session.

If you find yourself about to write a triple-backtick `question` block into your
reply, STOP: that content is a spec to render, not message body.

This applies to **every** structured-question site, including but not limited to:

- approval gates (every stage completion);
- the questions interaction-mode choice (Guide me / I'll edit the file / Chat),
  when the step asks for it (otherwise say the step's notice and take the
  questions the way it says; stage-protocol.md §3 Step 2);
- the ladder prompt (autonomy mode after the walking skeleton);
- halt-and-ask on Bolt failure (Retry / Skip / Abort);
- consolidated-summary confirmation before artifact generation;
- the section 13 learnings question after a correction (keep for this project, for the team, or not).

(Literal ` ```question ` のフェンスが、**この**ファイルや stage-protocol のような
フレームワークのドキュメントに残っているのは正当である。それらはチャットへの出力ではなく、
作成用の仕様だからである。stage-protocol では、これらの仕様は規範的なプロンプトテンプレートであり、
周囲の指示が質問を求める場合、その内容は必ずこの付属文書を通して描画しなければならない。
この付属文書の対応付けの例は説明用である。禁止されているのは、実際のオーケストレーションの
ターンで生のフェンスをエコーすることである。)

## Mechanism (two-track, D-3)

Codex CLI には構造化質問ツール `request_user_input` があり、同梱の設定フラグ
（`[tools] experimental_request_user_input` +
`[features] default_mode_request_user_input`）で有効になる。これが**主要な**トラックである。
下記の散文トラックは、ツールが利用できないセッション（フラグがオフ、古い Codex、
ヘッドレス実行）のための恒久的な最低保証である。

一部のモデルでは、Codex は `request_user_input_async` も提供する。AI-DLC の質問は
それではなく `request_user_input` を使う。質問のたびにターンを終了するが、Codex は
ターンの終了時に非同期の質問を画面から消してしまうため、利用者が回答する入力欄がなくなるからである。

### Track 1 — request_user_input (when the tool is available)

仕様のフィールドを 1:1 で対応付ける:

| 仕様のフィールド | request_user_input のフィールド |
|------------|--------------------------|
| `prompt` | 質問文 |
| `header` | 質問のヘッダー |
| `options[].label` | 選択肢のラベル |
| `options[].description` | 選択肢の説明 |

- 質問に推奨の選択肢がある場合は、それを**先頭**に置き、ラベルに
  "(Recommended)" を付け加える — ツールは推奨を先頭にしてネイティブに描画する。
- ツールは、メモ欄付きの "None of the above" という逃げ道を自動で追加する — ツール呼び出しに
  明示的な Other の選択肢を追加**しない**。利用者にこれについて伝えるときは、表示どおりの名前で呼ぶ:
  "Pick "None of the above" on any question and add
  your own words in its notes (Tab), or tell me what you'd like to talk
  through."（どの質問でも "None of the above" を選んでメモ欄（Tab）に自分の言葉を書き添えるか、
  話し合いたい内容を伝えてください。）（質問*ファイル*では引き続き、
  プロトコル §3 に従ってすべての質問の末尾に `X. Other (please specify)` を置く — この
  ファイル形式はハーネスに依存しない。）
- 制限: 1 回の呼び出しにつき質問は 1〜3 個、各質問の選択肢は 2〜3 個。質問ファイルの
  質問で選択肢がそれより多いものは、代わりに番号付きの散文で尋ねる
  （Track 2、stage-protocol.md Step 3a）。質問ファイルには、正式な記録として
  選択肢の全セットを残す。
- **回答の取得**: 選択結果は選択肢のラベルそのものとして返る。それを一字一句そのまま
  記録する（プロトコル: User Input を決して要約しない）。
- **回答なし**: 入力欄は約 2 分で時間切れになるか、利用者が閉じたり消したりすると、
  回答なしで返る。何も回答されていないので、同じ質問を入力欄ではなく番号付きの散文で
  返信の中でもう一度尋ね、ターンを終了する。利用者の代わりに回答を選んでは決してならない。
  エンジンはこれを記憶任せにしない — record-human-turn フックが `QUESTION_UNANSWERED` を記録し
  （それ以前のターンを消費するため、回答やゲートの判断は残らない）、
  同じ質問をすでに番号付き散文として描画した状態で、ラベルとその順序を変えずに返す。
  それを提示してターンを終了する。

### Track 2 — numbered prose (the floor)

ツールが利用できない場合（エラーになる、ツール一覧にない、または
セッションがヘッドレスである場合）は、仕様を番号付きの散文の選択肢として描画し、
ユーザーが番号または自由記述で回答できるようにする:

```question
prompt: "[Stage Name] complete. How would you like to proceed?"
header: Approval
multiSelect: false
options:
  - label: Approve
    description: Continue to [next stage]
  - label: Request Changes
    description: Provide revision feedback
```

これは次のようになる:

```
**Approval** - [Stage Name] complete. How would you like to proceed?

1. **Approve** - Continue to [next stage]
2. **Request Changes** - Provide revision feedback
3. **Other** - describe what you want instead

Reply with a number (or just tell me).
```

## Mandatory consolidated-summary checkpoint

このチェックポイントは `directive.ceremony.summary_confirmation === "on"` の場合にのみ適用される。`"off"` の場合は、回答から直接生成する。サマリー確認のプロンプト、確認エントリ、レシートはいずれも不要である。必須のステージ質問や、Plan Approval やステージの承認ゲートを含むその他の人間による判断は変わらない。

ガイド付きまたはチャットによるファイルベースの Q&A の後（および、Requirements Analysis のように
ステージ定義が明示的に求める場合は常に）、ステージプロトコルは、ステージの成果物を
生成する前に別途確認を行うことを求める。質問ファイルの
`## Consolidated Summary Confirmation` に、サマリー、プロンプト、A/B のファイル用の文字接頭辞を
付けない 2 つの選択肢、および空の `[Answer]:` タグを追記または更新する。

プロトコルの **Confirm** 質問をアクティブなトラックで描画し、サマリーの箇条書きを
利用者が読める位置に置く。`request_user_input` の場合、質問文はサマリーの箇条書き、
空行、プロンプトの順とし、2 つの意味的な選択肢はそのまま対応付ける。逃げ道はツールが
自ら提供する。番号付き散文の最低保証トラックでは、同じメッセージ内でその直上に箇条書きを描画する:

```
- <each answer, as a summary bullet>

**Confirm** - Does this all look correct before I generate the artifact?

1. **Looks correct** - Generate the artifact from these answers
2. **Request changes** - Revise one or more answers before generation
3. **Other** - describe what you want instead

Reply with a number (or just tell me).
```

これは必須の人間によるチェックポイントであり、ステージの承認ゲートではない。描画する前に、
`SKILL.md` にあるこのチェックポイント専用の `aidlc-log.ts decision` コマンドを、
正確な `--questions-file` と、あれば `--unit` / `--single` の識別情報を付けて実行する。
提示したら**ターンを終了し**、ユーザーの応答を待つ。
その後、どちらのトラックで質問を描画したかにかかわらず、`[Answer]: Looks correct` または
`[Answer]: Request changes` を正確に保存し、対応するチェックポイント専用の
`aidlc-log.ts answer` コマンドを実行する。書き込む前に、元の文字、番号付き散文の番号、
句読点、選択肢の説明を取り除く:
`[Answer]: A. Looks correct`、`[Answer]: 1. Looks correct`、`[Answer]: A`、
`[Answer]: 1`、および自分で選んだ回答は無効である。Request changes の場合、利用者の返信に
何を変えるべきかがすでに書かれていれば、その言葉がフィードバックである。そうでなければ
**"What should change?"**（何を変更すべきですか？）と尋ねて再び**ターンを終了し**、
そのフィードバックが届くまでどの回答も更新しない。その後、フィードバックを記録し、影響を受ける
回答を更新し、このタグを空に戻して、統合サマリーをもう一度提示する。
ファイルに人間による明示的な `[Answer]: Looks correct` が含まれ、かつレシートコマンドが
成功するまでは、成果物を生成しない。このチェックポイントを、後続のレビュアー、学び、
承認の各ステップと決して統合してはならない。

ルール（両トラック共通）:

- **承認ゲートの `[next stage]`**: 承認の質問では、
  `Continue to [next stage]` プレースホルダを `next_stage` フィールドの値でそのまま描画する
  （例: `Continue to NFR Requirements`）。使うのは、ゲートを開いた返信
  （`report --result awaiting-approval` または `revised`）の値であり、これにはステージ中に行われた
  プランの変更が反映されている。それがなければ run-stage ディレクティブの値を使う。
  `next_stage` が null の場合は `Complete workflow` と描画する。次のステージを決して推測しない。
- **承認ゲートの Accept as-is**: その返信、またはゲートを再表示するステップが、ゲートが
  3 つ目の選択肢を提供するかどうかを示す。
  `accept_as_is: "offered"` の場合は、3 つ目として `Accept as-is` を追加し、
  `revision_cycle` から何回目のラウンドかを示す。`"next-revision"` の場合は選択肢を 2 つのままにし、
  あと 1 回修正した後にそれが提供される旨の注記を添える。ラウンド数を
  自分で数えては決してならない。
- **場当たり的な選択肢を作らない**: 仕様の選択肢を正確にそのまま描画する（＋逃げ道）。
- **散文での回答キー**: Track 2 では、メッセージ内の前方にある番号付きの内容や、
  同じバッチ内の別の質問とは無関係に、すべての質問を `1` から始める。
  直前のサマリーには番号なしの箇条書きを使う。表示上の `1` は元の最初の選択肢ラベルに、
  `2` は 2 番目に、以下同様に対応する。
- **multiSelect: true** → 散文トラックでは "Reply with all numbers that apply
  (e.g. 1, 3)."（該当する番号をすべて返信してください（例: 1, 3）。）と伝える。
- ある選択肢に明らかに一致する自由記述の返信は、その選択肢として扱う。
  利用者が自分の言葉で質問に答えている返信は、それが利用者の回答である:
  利用者の言葉のまま "Other" の回答として記録する。質問について尋ねたり、
  話し合いたいと求めたりする返信の場合にのみ、先に話し合い、その後で
  利用者が決めた内容を採用する。
- いずれの場合もゲートの意味論は**エンジン**側にある - 描画が判断することは決してない。
  エンジンからの問いかけには必ず `ask_type` と `response_route` が含まれる。`"next"` ルートでは、
  選ばれた `confirm_command` / `compose_command`、または `scope` が選択されたプランと
  等しい `scope_commands` エントリを使う（エントリのない名前は有効なスコープではない）。
  `--request <8hex id>` はそのまま保持し、リクエストのテキストを決して付け加えない。
  問いかけはリクエストを ID でのみ示し（貼り付けられた
  `<document>` ブロックはデータとして質問ストアに残る）、
  質問は最大 240 文字とし、切り詰めた場合は `...` で終える。
  `intent-pick` では、選ばれた正確な `available_intents` セレクターを
  `select_commands[].selector` と照合し、そのエントリの `command` 全体を一字一句そのまま
  実行する。セレクターを補間して組み立てては決してならない。`new-work-routing` はルートを
  フィールドとして持つ: `new_intent_command`（または修正されたスコープ用の `scope_commands` エントリ。
  プランが提案されていない場合は存在しない）、`compose_command`、そしてアクティブなワークフロー用の `continue_command`、
  あるいは `available_intents` がある場合はレコードごとの `select_commands` と `reshape_commands` である。これらをそのまま実行し、
  選択や構成を通じて `--request` の ID を保持する。選択済みのカーソルがなく
  保留中の作業がある既存のインテントは、スコープの確認後も含め、どのハーネスでも
  この問いかけを受け取る。保留中の作業がない場合の選択にのみ `intent-pick` を使う。
  `"command"` ルートでは、人間が再開を選んだ場合にのみ `resume_command` を実行し、
  その後 `next` を再実行する。そうでなければ人間の指示を待つ。`"claim"` は
  Unit の claim フローに従う。`"execute-remedy"` は実行可能なガードの是正策のみを提示し、
  人間が選んだコマンドまたはアクションに従う。レポートを捏造しては決してならない。
  是正策が空の場合は終端のままである。再開時の redo・jump・start-fresh の要求は
  ステージ以外で唯一のレポート往復であり、
  `report --result resumed --choice <redo|jump|fresh>` を使う。これは汎用的な
  エンジン問いかけへの回答ルートではない。明示的なガード是正策によるステージレポートは変わらない。
