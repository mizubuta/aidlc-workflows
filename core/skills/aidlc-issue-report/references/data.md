# What safe.json holds

このスキルが作成したフォルダを読む人のための説明。`safe.json` は実行の記録をそのまま収めたもので、
人・会社・プロジェクトを特定しうるものはラベルに置き換えてある。中身に判断は一切含まれない。どのイベントが重要かは読み手が判断する。

## Labels

ラベルは、AI-DLC 自身の語ではない値を表す: `<custom-unit-3>`、`<custom-file-12>.ts`、`<custom-text-40>`。
1 つのフォルダ内では、同じ種類の同じ値には常に同じラベルが付くため、繰り返しは同じラベルとして現れる（同じ
返答が 2 回、同じファイルが 9 回書き込まれた、など）。呼び出しの `input` ラベルはツールとその入力を合わせたものを表し、
リダクションマップ由来のラベルは名前の変種（インテントのフォルダとその短い名前）を表すことがある。2 つのフォルダの
ラベルは互いに無関係である。

| 種類 | 表すもの |
|---|---|
| `unit`, `intent`, `space` | ユニット、インテント、スペースの名前（`<unit-N>`、`<intent-N>` も同じで、リダクションマップ由来） |
| `file` | プロジェクトのファイル（パス単位）。一般的な拡張子はその横に残す（`.ts`、`.md`） |
| `dir` | AI-DLC または AI ツール自身のフォルダ内にあるフォルダのうち、ユニット、インテント、スペースのものではないもの |
| `text` | 人またはエージェントが書いたもの全般: 返答、依頼、フィードバック、エージェントのメッセージ、質問の選択肢 |
| `value` | 名前を保持するフィールド内の、AI-DLC のものではない語（プラグインのステージ、未知のステータス） |
| `hash`, `id`, `session` | ダイジェストやフィンガープリント、レコードや呼び出しの ID、セッション ID（`audit` と `transcripts` で同じセッション ID） |
| `field` | AI-DLC のツールが書き込まない監査フィールドの名前 |
| `stage`, `agent`, `hook`, `sensor`, `tool`, `model`, … | その種類の名前のうち、AI-DLC 自身のものではないもの（プラグインやチームのもの） |
| `<project>`, `<aidlc-install>`, `<user>` | プロジェクトフォルダ、インストールフォルダ、マシンのユーザー |

## Sections

- `window`: 読み取った期間（`since`、`until`、UTC）と `lastAidlcActivity`。指定された時刻が使われなかった場合は `notes`。
- `environment`: OS の行、Bun、AI-DLC のネイティブインストール、各 AI ツールのバージョンとその読み取り元
  （`path`、`known-location`、`app` = 収集時に計測、`person` = 利用者が画面から読み取ったもの。ツール自身のログから得た
  バージョンは `transcripts[].toolVersions` にある）、`toolsUsed`、`versionsMissing`、そして `toolsNamed`（利用者が名前を挙げた場合のみ）: どのチャットログにも実行に使われたツールが示されていなかったときに、利用者が名前を挙げた `toolsUsed` 内のツール。そのために読んだチャットログが、このレポート自身のセッションしか含んでいなかった（`transcripts[].reportSessions`）り、失敗した（`error`）りした可能性もある。
  `kiroHomeEngine`: ホームフォルダの Kiro 設定ファイルにある `chat.agentEngine`（`file: false` = そのファイルがない）。2 つの
  どちらが優先されるかは Kiro CLI のバージョンによって変わってきた: 2.21.1 ではプロジェクトのもの、2.33.1 ではホームのもののみ
  （2026-10-22 に計測: プロジェクトの `v3` だけでは v2 セッションになり、ホームの `v3` とプロジェクトの `v2` では v3 になった）。その間のバージョンは
  計測していない。実際に動いたものは `transcripts[].engine` にある。
- `trees`: プロジェクト内の各 AI-DLC ツリー。
  - `row`: `kiro-ide` = "Kiro"（Kiro IDE、および v3 エンジン上の Kiro CLI。どちらかは Kiro のトランスクリプトの `tool` が示す:
    `kiro (v3: Kiro CLI)`、`kiro (v3: Kiro IDE)`）、`kiro` = "Kiro CLI (V2)"、`claude`、`codex`、`copilot`、`cursor`、
    `opencode`。
  - `stampVersion`, `buildVersion`: AI-DLC の `main` から作られたビルドは素のリリース番号を刻む。`-preview.` を含む
    ビルド ID はナイトリーのプレビューである。
  - `channel`: `native`（AI-DLC インストーラー、コマンド `aidlc`）または `copy`（プロジェクトにコピーされたファイルを Bun で実行）。
  - `kiroEngine`: プロジェクトの Kiro 設定ファイルにある `chat.agentEngine`。
- `intents`: 各インテントの現在の状態（フェーズ、現在と次のステージ、ステータス、スコープ、深さ、Guard Policy、オンと
  オフのガード、センサー、計画の承認、各ステージのマーク）。
- `audit`:
  - `rowsTotal`, `rowsInWindow`。`counts`: イベントごとの行数。
  - `rows`: 実行のすべての行を時刻順に並べたもの: `seq`、`ts`、`event`、`intent`、そして `fields`（各 `[name, value]` を
    レコードが書き込んだ順に並べる）。フィールドが AI-DLC 自身の語（イベント、ステージ、フェーズ、エージェント、センサー、
    フック、ツール、ステータス）、数値、時刻のみを保持する場合、値は書かれたまま残す。AI-DLC 自身の文（エラー、拒否、ツールの
    コマンド、`phase > Unit > stage > file` のコンテキスト）は固定テキストを残し、埋め込まれた部分をラベルにする。パスは AI-DLC と
    AI ツール自身のフォルダ（`aidlc/spaces/default/intents/…`、`.aidlc-engine/reviews/…`、Kiro の steering フォルダ）を残し、その中の名前を
    ラベルにする。それ以外のパスは `file` ラベルになる。リスト
    （ステージ、変更されたファイル、選択肢、言語）は JSON 配列である。
  - このレポートの作成中に書き込まれた行は `rows` にも `counts` にも含まれない。
  - 行の `Session` とイベントの `session` は同じ種類のラベルである。同じラベルは同じ ID を意味する（Kiro のチャット
    フォルダ名と AI-DLC が記録する ID は異なることがある）。
- `hooks`: 各 AI-DLC フックの最後のハートビート（`last`）と、フックが破棄した行（`drops`: 時刻、フック、理由）。
- `sessionsFolder`: AI-DLC のセッションフォルダ: 未解決の質問、まだ紐づけられていない承認、保留中の Kiro IDE リクエスト、
  トランスクリプトのカーソル。
- `transcripts`: ログにこのプロジェクト名を含む AI ツールごとに 1 エントリ。
  - `events`: 起きたことを時刻順に並べたもの: `seq`、`ts`、`session`、`kind`、そして種類に応じて `tool`、`input`（呼び出しの
    入力をラベルにしたもの。同じラベルは同じ入力）、`verb`（AI-DLC のコマンド。AI-DLC の語のみ）、拒否の `hook` と
    `owner`（`aidlc`、`other`、`unknown`）、拒否または Stop フックのフィードバックの `reason`（そのテキストを、監査行の
    `Reason` と同じ規則で残す: AI-DLC 自身の言い回しは書かれたまま、それ以外はラベル。Kiro CLI v3 と Kiro IDE は
    何も記録しない）、
    拒否の `detail`、`reply`（利用者が入力したものをラベルにしたもの。Codex では、まだ実行中のターンに取り込まれた場合は
    `inRunningTurn`、そのターン内で直前の返答を繰り返した場合は `sameAsBeforeInTurn`）、そして `tool-error` の場合は呼び出しの終わり方。
    これはコマンド出力の前後にあるツール自身の行から、数値とフラグだけとして読み取る: `exitCode`（最大 4 桁。
    Kiro のトレーラー、Claude Code の 1 行目、Codex のヘッダー）、`timeoutMs`（Kiro の制限時間、最大 7 桁）、
    `interrupted`（Kiro の `^C` または中断結果、Claude Code の中断マーカー）。Copilot の失敗した呼び出しには
    何も付かない（その形式は計測していない）。種類: `person-turn`、`call`、`aidlc-command`、`tool-error`、`tool-denied`、`approval-prompt`、
    `stop-hook-feedback`、`aidlc-hook-block`、`other-hook-block`、`hook-block`（誰のものかがレコードに示されていない
    拒否）。`noTime`: ツールがそのイベント独自の時刻を記録しなかった（Kiro CLI v2）ため、`ts` はそのセッションの時刻である。Kiro CLI v2 の
    `repeats` も同様にセッションの時刻を持つ。
  - 同じレコードに対する件数: `sessions`、`prompts`、`toolCalls`、`toolErrors`、`denials`、`aidlcHookBlocks`、
    `otherHookBlocks`、`unattributedHookBlocks`、`stopFeedback`、`approvalPrompts`、`questionsAsked`、`aidlcCommands`、
    `repeats`（同じ呼び出し入力が 3 回以上）、`subagents`、`handRunChecks`（エージェントが自分で実行した AI-DLC のチェックと、
    それが出力した `pass`。こうした実行は監査行を書き込まない）。
  - 実行時の状況: `engine`（Kiro）、`toolVersions`（ツール自身のログから）、`models`、`efforts`、`agents`、
    `approvalModes`。`reportSessions`、`mixedSessions`、`reportModels`: このレポートを作成したセッション。これらは実行の
    件数とイベントから除外される。
  - `notes`: 読み手がレコードから判断できなかったこと。
- `otherStores`: このマシン上にあるが読み取らないチャットストア（VS Code Copilot Chat、Cursor）。
- `doctor`: AI-DLC の doctor が実行されたかどうか、`dir`（実行対象のハーネスフォルダ: このスキルの実行元）、その
  終了コード、エクスポート自身のレポートの各指摘（レポートがない場合は doctor が出力した行の各指摘）とその ID（ID が
  AI-DLC のものでない場合はラベル）および重大度、`checksFailed`。エクスポートのアーカイブはフォルダに含まれない。
- `secretRules`: シークレットルールがどのインストール済み AI-DLC ファイルから来たか。
- `knownGaps`: レコードでは示せないこと。
