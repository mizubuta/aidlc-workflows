# AI-DLC State Tracking

このドキュメントは、`aidlc-state.md` のセクションとフィールドの契約を定義する。
具体的な状態ファイルはエンジンが書き込み、ステージはコンパイル済みのステージグラフと
スコープグリッドから列挙する。このテンプレートで、出荷済みのステージを手作業で列挙してはならない。
初期説明の正確な内容は、状態ファイルの隣の `<record>/project-description.json` に
1つの文字列として JSON エンコードされて保存される。下記の `Project` フィールドは、その
安全な1行プレビューである。

正となる生成ビュー:
- ステージグラフ: `{{INVOKE}} engine gen stage-table`
- スコープグリッド: `{{INVOKE}} engine gen scope-table`

## Project Information
- **Project**: [プロジェクト説明の1行プレビュー]
- **Project Description Source**: project-description.json
- **Project Type**: [Greenfield/Brownfield。すべての読み手が比較に使う、その語のみ]
- **Project Type Source**: [`workspace scan` または `you`: 種別を決めたのは誰か。インテント作成時に書き込まれ（リクエストに `--project-type` が含まれていた場合は `you`）、`workspace reclassify`（利用者の `/aidlc --project-type` または平文での指示）によって `you` に書き換えられる。ルーティング用のフィールド: Construction を開始していない Greenfield の作業でこの値が `workspace scan` の間は、フォルダにコードが追加されたときに `next` が一度だけ確認する。`you` の場合は、スキャンによって覆されることは決してない。この行のない状態ファイルは `workspace scan` として読まれる]
- **Scope**: [コンパイル済みスコープグリッドのスコープ slug]
- **Plan**: [この作業のために組み立てたプランがある場合のみ記載する: ゲートに表示されたとおりのプラン名（`--plan-name`）、または `tailored plan`。そのステージの変更は Stage Progress のサフィックスとして表れ、スコープを変更するとこの行は削除される]
- **Start Date**: [ISO 8601 タイムスタンプ]
- **State Version**: 8
- **Active Agent**: [現在のリードエージェントの slug]
- **Worktree Path**: [worktree 内でない場合は空]
- **Bolt Refs**: [空リスト、またはカンマ区切りの bolt slug]
- **Practices Affirmed Timestamp**: [確認時の ISO 8601 タイムスタンプ]

## Scope Configuration
- **Stages to Execute**: [スコープに含まれるステージ番号と slug のカンマ区切り。例: 2.3 (requirements-analysis)]
- **Stages to Skip**: [ステージ番号と理由のカンマ区切り、または none]
- **Depth**: [Minimal/Standard/Comprehensive]
- **Test Strategy**: [Minimal/Standard/Comprehensive]
- **Guard Policy**: [strict/relaxed/off。その後に括弧で出所を付ける: `(from scope <name>)`、`(from <layer>.md)`、記録上利用者が求めた場合は `(set by you)`、または `(set by a command)`。インテント作成時に解決済みの値で書き込まれ、`/aidlc --guard-policy` または平文のチャットでの依頼によって書き換えられ、値のみで読み取られる。記録に廃止済みの `**Change Control**` 行しかない場合、すべての `/aidlc` 実行で、引き継がれた relaxed または off の値が誰も指示していない作業に対してもフェンスを下げるようになったこと、そしてすべての通過が監査証跡に記録されることを告知し、その値を維持するかフェンスを再び上げるかを利用者に尋ねる。この告知は、利用者が Guard Policy の設定を選ぶまで繰り返される。廃止済みの行が strict のみの場合は告知しない。両方の行が異なるポリシー語を持つ場合、メモリが strict を保持していない限り、出所を `conflicting state lines` として strict を適用する。これは、次の `next` が、記録上最後に設定されたほう（判別できなければ利用者自身のもの、スコープのもの、またはより厳しいもの）を唯一の行として残すまで続く。記録から判別できなかった場合にのみ、1行だけを残したことを伝える。両方が一致する場合は `Guard Policy` を使う。ポリシーを書き込むと廃止済みの行は削除され、`Guard Policy` だけが残る。廃止済みの行のみの場合、`next` は何も書き込まない]
- **Guards Off**: [この作業でフェンスがオフにされた後にのみ記載する: `plan-approval`、`review-freeze`、`state-transition`、`reviewer-scope` のカンマ区切りリストの後に、そのすべてのチェックが利用者によるものであれば `(set by you)`、そうでなければ `(set by a command)` を付ける。または `none`。`/aidlc config set guard.<fence> off|on` によって書き込まれる。human-presence には作業ごとのスイッチがないため、永続化された human-presence のエントリは無視される]
- **Guards On**: [この作業でフェンスがポリシー語より上に強制的にオンにされた後にのみ記載する: `plan-approval`、`review-freeze`、`state-transition`、`reviewer-scope` のカンマ区切りリストの後に、そのすべてのチェックが利用者によるものであれば `(set by you)`、そうでなければ `(set by a command)` を付ける。または `none`。`/aidlc config set guard.<fence> on|off` によって書き込まれる。永続化された human-presence のエントリは無視される。優先順位は、環境のキルスイッチ、Guards Off、Guards On、Guard Policy、その後にデフォルトのオンである]
- **Sensors**: [on/off。その後に括弧で出所を付ける: `(from scope <name>)`、`(set by you)`、または `(set by a command)`。インテント作成時にスコープのデフォルトで書き込まれ、`/aidlc --sensors` によって書き換えられ、値のみで読み取られる]
- **Learnings**: [on/off。その後に括弧で出所を付ける: `(from scope <name>)`、`(set by you)`、または `(set by a command)`。インテント作成時にスコープのデフォルトで書き込まれ、`/aidlc --learnings` によって書き換えられ、値のみで読み取られる]
- **Summary Confirmation**: [on/off。その後に括弧で出所を付ける: `(from scope <name>)`、`(set by you)`、または `(set by a command)`。インテント作成時にスコープのデフォルトで書き込まれ、`/aidlc --summary-confirmation` によって書き換えられ、値のみで読み取られる]

## Workspace State
- **Project Root**: [プロジェクト相対パス。通常は `.`。実行時に再導出され、絶対パスとして信頼されることは決してない]
- **Languages**: [検出された言語]
- **Frameworks**: [検出されたフレームワーク]
- **Build System**: [検出されたビルドシステム]

## Execution Plan Summary
- **Total Stages**: [EXECUTE ステージの数]
- **Completed**: [完了した EXECUTE ステージの数]
- **In Progress**: [現在のステージの slug]

## Runtime State
- **Revision Count**: [整数]
- **Construction Checkpoints**: [新しいワークフローでは enabled。レガシーのワークフローでは記載なし]
- **Construction Iteration**: [unit-major/stage-major。新しいワークフローのデフォルトは unit-major。明示的な選択は維持する]
- **Construction Execution**: [serial/swarm。新しいワークフローのデフォルトは serial。swarm には stage-major が必要]
- **Construction Verification Command**: [人間が承認したプロジェクトのチェックコマンド。対応する verification-command のレシートが記録されるまでは未設定]
- **Unit Ownership**: [solo/team。省略可能。値がちょうど `team` の場合に導出グリッドが有効になる]
- **Unit Gate Rhythm**: [per-stage/unit-end。省略可能。team オーナーシップではデフォルトで per-stage]

チェックポイントのポリシーは、実際に空でない Unit DAG を持つ solo の作業にのみ適用される。
チェックポイントのフィールドを持たない既存のワークフローは、レガシーの最初のステージでの承認と
後続のステージごとの承認を維持する。execution フィールドがない場合は、レガシーの
自律性ベースのスウォームのルーティングを維持する。実行方式の選択は承認とは独立している。
team オーナーシップは、その `unit_gate` ポリシーを維持する。

## Phase Progress
<!-- Status values: Pending, Active, Verified, Skipped -->

- **[Phase]**: [Pending/Active/Verified/Skipped]

## Stage Progress
<!-- Checkbox states: [ ] pending, [-] in-progress, [?] awaiting approval, [R] revising, [x] completed, [S] skipped -->

エンジンは、コンパイル済みのフェーズごとに1つのフェーズ見出しを出力し、その後に
そのフェーズ内のコンパイル済みステージごとに1つのチェックボックス行を出力する:

### [PHASE] PHASE
- [ ] stage-slug — [EXECUTE/SKIP: reason]

## Unit Progress

`Unit Ownership: team` かつ `Construction Iteration:
unit-major` の場合にのみ記載する。この表は、Unit DAG、成果物のカバレッジ、ライフサイクルのレシート、
ユニットのゲートイベントから導出された、エンジンが所有する投影である。
`next` のたびに書き換えられる。手作業での編集は、ルーティングや完了の根拠には決してならない。

| unit | owner | [per-unit Construction stage columns in graph order] | gate |
| --- | --- | --- | --- |
| [Unit name] | - | [[ ]/[-]/[?]/[R]/[x]/[S] per stage] | [[ ]/[-]/[?]/[R]/[x]] |

ステージの列は `## Stage Progress` と同じチェックボックスの語彙を使う。
`owner` は、claim のインクリメントがオーナーシップを提供するまで `-` のままである。`gate` は、
Unit Gate Rhythm に応じて、現在のステージごとのゲート、またはユニット終了時のゲートを要約する。
Stage Progress の行は、対応する Unit Progress の列と必須のチームゲートが完了している場合にのみ、
完了として導出される。

## Current Status
- **Lifecycle Phase**: [READY/INITIALIZATION/IDEATION/INCEPTION/CONSTRUCTION/OPERATION]
- **Current Stage**: [ステージの slug またはステータスのテキスト]
- **Next Stage**: [次のステージの slug、または none]
- **Status**: [Running/Completed/Archived]
- **Construction Autonomy Mode**: [unset/autonomous/gated]
- **Last Updated**: [ISO 8601 タイムスタンプ]
- **Archived From**: [Archived の間のみ: 置き換えられた Status（Running または Completed）]

## Session Resume Point
- **Last Completed Stage**: [ステージの slug]
- **Next Action**: [次に行うこと]
- **Pending Artifacts**: [未完了の成果物、または none]
