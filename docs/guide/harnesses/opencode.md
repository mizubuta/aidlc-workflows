# AI-DLC on opencode

opencode ランタイムは、オープンソースの **opencode** ハーネス（opencode.ai）向けの、
フレームワークのハーネスディストリビューションの 1 つです。1 つの決定的なコアに、
多数のハーネスという構成です。エンジン、ステートマシン、監査ログ、グラフ、
スウォームのレフェリー、学びのゲートはすべてのディストリビューションでバイト単位で同一であり、
異なるのはシェルだけです。ソース／開発用のツリーは、`bun scripts/package.ts opencode` によって
`core/` と `harness/opencode/` から、git で無視されるローカルの
`dist/opencode/` に**生成**されます。決して手で編集しないでください。

ハーネス固有のオンボーディングは `.aidlc/onboarding.md` にあり、`opencode.json` の
`instructions` リストによって読み込まれます。ルートの `AGENTS.md` ブロックは
ハーネス非依存で、エンジンのディレクトリが異なる他のインストール済みハーネスと共有されます。
opencode と Copilot は 1 つの `.aidlc/` インストールを共有できません。

## Layout: two dot-dirs, on purpose

opencode は `.opencode/tools/` と `.opencode/tool/` 配下のすべての `*.ts` を
カスタムツールの定義として自動インポートします。CLI 形式のエンジンスクリプト
（トップレベルでのディスパッチ、`process.exit`）をインポートするとセッションがクラッシュします
（opencode 1.17.18 で実際に再現）。そのため、このディストリビューションは次のように分かれています:

- **`.aidlc/`** — AIDLC エンジンのツリー（ツール、フック、スキル、エージェント、
  ナレッジ、スコープ、センサー、aidlc-common）。opencode がこれをスキャンすることはありません。
  同梱の `opencode.json` が `skills.paths: [".aidlc/skills"]` を登録するため、
  オーケストレーターのスキルと生成されたすべてのランナーはそこから検出されます。
- **`.opencode/`** — ネイティブに利用されるものだけを置きます: 14 のペルソナの
  サブエージェント（`agents/*.md`、`mode: subagent`）、`/aidlc` コマンド
  （`command/aidlc.md`）、フックアダプターのプラグイン
  （`plugin/aidlc-opencode-adapter.ts`。opencode が自動検出します）。
  このコマンドは入力されるたびに、インストール済みのスキルを自身の中に出力します
  （`aidlc engine skill-text --harness-dir .aidlc`）。これにより、エージェントはスキルを丸ごと受け取ります。opencode は
  自身のスキルツールで読み込んだスキルを 50 KiB で切り詰め、残りを
  プロジェクト外のデータフォルダーに保存してしまうためです。

## Prerequisites

- **opencode 1.18.29 以降、または OpenCode 2** — アダプタープラグインは
  両方の系統が読み込める 1 つのオブジェクトを default export します。opencode 1 はその
  `server()`（`tool.execute.before`、`tool.execute.after`、`chat.message`、
  `session.idle`、`experimental.session.compacting`、`shell.env` の各フック。
  オブジェクトのエントリーポイントには 1.18.29 が必要です）を呼び出し、OpenCode 2 はその `setup()` を呼び出します
  （[OpenCode 2 の場合](#on-opencode-2) を参照）。プロジェクトローカルのスキルとエージェントの検出は
  どちらでも動作します。`opencode --version` で確認してください。
- **bun** — ソース／開発用の `dist/` プロジェクションを生成または実行する場合にのみ必要です。
  ネイティブインストールとバージョン付きのリリースランタイムは、
  インストール済みの `aidlc` 実行ファイルを通じてディスパッチされます。
- **モデルプロバイダー** — 同梱のプロジェクトの `opencode.json` とエージェントの
  定義では、モデルを固定していません。プロバイダーとセッションのモデルは
  グローバルの opencode 設定で指定します。バランス型のレビュアーは、medium の推論
  バリアントのみを保持します。

## Install

### Native channel (recommended)

```bash
tmp="$(mktemp -d)"
curl -fsSL \
  https://github.com/awslabs/aidlc-workflows/releases/latest/download/install.sh \
  -o "$tmp/install.sh"
sh "$tmp/install.sh"
rm -rf "$tmp"
cd your-project
aidlc config
aidlc doctor
opencode
```

インストーラーは、リリースのメタデータ、実行ファイル、全ハーネス対応のランタイムアーカイブを、公開されている SHA-256 チェックサムと照合して検証します。インストールされたランタイムには Bun、Node.js、Git は不要です。ハーネスの選択は `aidlc config` で行います。

Windows では、`install.ps1` をダウンロードし、
`Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force` でその PowerShell ウィンドウでのスクリプト実行を許可してから、
`& $installer` を実行します。アカウントのスコープ、User PATH への自動登録、`-NoModifyPath` については
[Windows でのインストール](../18-install-and-lifecycle.md#windows-powershell)
を参照してください。
エアギャップ環境向けのパッケージでは、同じ
`Set-ExecutionPolicy` の行を実行した後、Unix では
`install.sh --from <release-directory> --offline` を、Windows では
`& $installer -From <release-directory> -Offline` を使用します。

`aidlc config` は `.aidlc/`、`.opencode/`、ワークスペースのシェル、
`AGENTS.md`、管理対象の `.gitignore` ブロック、そして
`opencode.json` 内の AI-DLC のエントリーを投影します。これらのエントリーは、スキルとメソッドのファイルを検出し、
AI-DLC 自身の直接的な `aidlc engine *` コマンドを確認なしで実行できるようにし、
`rm -r...` と `git push` の前に 1 回確認を求めます。それ以外のシェルコマンドはすべて、あなた自身の
opencode の設定に従います。すでに `opencode.json` があるプロジェクトでは、
モデル、プロバイダー、instructions、権限ルール、コメントはそのまま保持されます。AI-DLC は
自身のエントリーのみを追加し、その 2 つの確認ルールは `bash` マップの先頭に置かれるため、
その後に続くあなた自身のルールが最終的な判断を行います。グローバルの opencode 設定では、
シェルのルールをマップ形式（`"bash": {"*": "ask"}`）で書いてください。opencode は、グローバルの
短縮形式 `"bash": "ask"` を、このファイルを含む任意のプロジェクトのマップで置き換えてしまいます。
グローバルの `"git *": "allow"` があっても、AI-DLC の `git push` の確認は 1 回表示されます。"Allow
always" を選ぶとそれ以降は表示されません。`aidlc config` と `aidlc doctor` は、
グローバル設定で短縮形式を見つけると、そのことを 1 行で伝えます。プロジェクトで opencode を起動し、
`/aidlc --doctor` を実行した後、`/aidlc` に続けて作りたいものを入力してください。

### Versioned manual-copy alternative

特定のリリースの `aidlc-copy-runtime-X.Y.Z.tar.gz` を、
[インストールとライフサイクル: Copy Channel](../18-install-and-lifecycle.md#copy-channel)
の説明に従ってダウンロードして展開し、`RUNTIME_ROOT` に展開した `runtime/` ディレクトリを設定します。

1. ディストリビューションをプロジェクトにコピーします:

   ```bash
   cp -r "$RUNTIME_ROOT/opencode/.aidlc/"    your-project/.aidlc/
   cp -r "$RUNTIME_ROOT/opencode/.opencode/" your-project/.opencode/
   cp -r "$RUNTIME_ROOT/opencode/aidlc/"     your-project/aidlc/      # the workspace shell — a sibling of .aidlc/, not inside it
   ```

   コピー用ランタイムには `opencode.json` が含まれていないため、コピーによって既存のファイルが置き換えられることはありません。
   ステップ 2 で、AI-DLC にとって欠かせない 3 つの要素をそこに追加します（ファイルがなければ
   作成します）: `skills.paths`（`.aidlc/skills` からのスキル検出）、
   `instructions`（`.aidlc/onboarding.md` にあるネイティブのオンボーディングと、
   メソッドの glob `aidlc/active-memory/**/*.md` の両方。後者は、アクティブスペースのメソッドを
   エンジンが git で無視される場所にコピーしたもので、`/aidlc space <name>` によって書き換えられます）、
   そして AIDLC の bash エントリーポイントと、
   `.aidlc/tools/` および `.aidlc/hooks/` 配下の編集に対する権限ルールです。代わりに `opencode.jsonc`
   を使っている場合は、ステップ 2 でコメントを保持したままそこに要素を追加し、隣に
   `opencode.json` を作成することはありません。
   アダプターは権限の境界を強制します。対象は、パッケージ化されたツリーに埋め込まれた
   エントリーポイントでなければならず、連結、リダイレクト、展開、コマンド置換のない
   1 つの直接的なコマンドとして呼び出される必要があります。エンジンのコードの編集には
   承認を求めるプロンプトが表示されます。

2. プロジェクトから、コピーに含まれるセットアップを 1 回実行します:

   ```bash
   bun .aidlc/tools/aidlc.ts config --from "$RUNTIME_ROOT" --harness opencode
   ```

   これにより、最初のワークフローの前に、`AGENTS.md` と `.gitignore` の既存の内容の後ろに
   AI-DLC の行が追加され、`opencode.json` に AI-DLC のエントリーが追加（またはファイルが作成）されます
   （クローンごとの監査シャードは意図的にコミットされます。カーソルと
   マシンローカルのランタイムは無視されたままになります）。

3. プロジェクトで opencode を起動し、`/aidlc --doctor` を実行した後、`/aidlc`
   に続けて作りたいものを入力します。

opencode には session-start フックが注入するコンテキストを受け渡す経路がないため、
引数なしで呼び出された場合、`/aidlc` スキルは読み取り専用のステータス確認を 1 回行います。
既存のワークフローは、`/aidlc --resume` と同様に停止した箇所から再開します（やり直し、
ステージへのジャンプ、別の作業のために新しく始めることも依頼できます）。
`/aidlc --resume` は確認をスキップして直接続行します。

バージョン付きのランタイムはネイティブの `aidlc` コマンドを使用します。Bun 向けのプロジェクションが
必要なフレームワーク開発者は、リポジトリをクローンし、
`bun install --frozen-lockfile` と `bun scripts/package.ts` を実行してから、
git で無視されるローカルの `dist/opencode/` の出力を使用できます。

## Refresh and version skew

`aidlc update` はプロジェクトを書き換えずにマシンのランタイムを更新します。
`aidlc doctor` は、選択されたエンジンと異なるプロジェクトのスタンプを報告します。
ワークフローの合間に、リフレッシュをプレビューしてから適用します:

```bash
aidlc config --dry-run
aidlc config
```

config は管理対象のルートブロックと利用者が所有するファイルを保持し、フレームワークのファイルに対する
ローカルの編集を競合として報告します。`opencode.json` では、リフレッシュで更新されるのは
AI-DLC が書き込み、誰も変更していないエントリーだけです。あなた自身のエントリーと、あなたが編集した AI-DLC の
エントリーはそのまま残ります。
ワークフローが開いている間のリフレッシュは、設定の変更（`config
models`、`flags`、`providers`）と同様に実行され、開いている作業はそのまま続行される旨が表示されます。
アップグレードとロールバックはプロジェクトに触れないため、ワークフロー中でも安全に
行えます。

## What's different on this harness

- **質問は番号付きの文章の選択肢として表示されます**（構造化された質問の
  ウィジェットはありません）。`[Answer]:` タグ付きの質問ファイルが引き続き信頼できる唯一の
  情報源です。
- **フックはアダプタープラグインを経由して動作します。** opencode には hooks.json や settings によるフックの
  レジストリがありません。`.opencode/plugin/aidlc-opencode-adapter.ts` が opencode の
  プラグインのフックのタイミングを、`.aidlc/hooks/` にあるコアのフック本体（bun の
  サブプロセスとして実行）に対応付けます。ツール実行前のレビュアーの読み取り範囲と AIDLC の bash の境界、
  write/edit/apply_patch での監査とセンサー、
  bash での rebuild-stage-graph（エンジンのエラーの正確なメッセージを TUI のトーストとしても表示します）、
  todowrite でのステータスラインの同期、task でのサブエージェントのログ記録、人間の各ターンでの
  プレゼンスの発行、コンパクション前の状態検証です。
- **転送ループの強制は助言的です。** Stop の接続点は
  `session.idle` イベントで、反応的であってブロッキングではありません。コアの stop フックが
  `block` と応答すると、プラグインは促しのプロンプトを注入してループを再開させます
  （送信したパートの ID で促しを識別するため、促しがあなたのターンとして数えられることは決してありません）。促しは
  合成パートであり、エージェントはそれを読みますが、チャットには表示されません。Esc で
  ターンを止めた場合や、エージェントが実行を求めたコマンドを拒否した場合は、あなたが再び書き込むまで
  促しは行われません。チャット中や
  一時停止中の人間は、フックの対話上限によって解放されます。
- **`/aidlc` は入力した内容を表示します。** opencode はそのままではコマンドのテンプレート全体を
  あなたのメッセージとして表示しますが、プラグインはテンプレートをエージェント向けに保持し、
  代わりにあなたの `/aidlc ...` の行を表示します。
- **あなたの返答として扱われるのは入力した内容だけです。** opencode がエディターで開いている
  ファイルや選択範囲をメッセージに追加した場合（Zed のターミナル、またはエディター拡張機能を使った VS Code の
  ターミナル）、エージェントはそれも参照しますが、AI-DLC はあなたが入力した内容だけを
  あなたの言葉として扱います。
- **ペルソナはネイティブのサブエージェントです**（`mode: subagent`）。コンダクターは
  ほとんどのステージでそれらをインラインで採用し、2 つのサブエージェントステージ
  （2.1 reverse-engineering、3.5 code-generation）では `task` ツールを通じて委任します。ネイティブの
  権限マップで `task` が拒否されているため、委任されたエージェントがさらに委任することはできません。
  プラグインの構成でも、プラグインのペルソナ向けに同じ `.opencode/agents/` の対応ファイルが出力されます。
- **スペースの切り替えでは JSONC が保持されます。** `/aidlc space <name>` は、
  `opencode.json` と `opencode.jsonc` のどちらにあっても、コメントや末尾のカンマを取り除かずに
  メソッドの glob を更新し、明示的なペルソナのメモリパスの整合性も保ちます。
- **Construction のスウォームは task ツールによるファンアウトとしてのみ実行されます**（`AIDLC_USE_SWARM=1`
  は警告付きの no-op です。Workflow ツールは存在しません）。
- **セッション終了のタイミングはありません** — `SESSION_ENDED` の監査イベントは出力されません。
  コンパクション前の検証は実行されます（`experimental.session.compacting`）。
- **ステータスラインやウェルカムメッセージはありません** — `/aidlc --status` とゲートでの進捗の
  行を使用してください。
- **MCP サーバー**: 同梱されているものはありません。必要に応じて
  `opencode.json` の `mcp:` 配下で独自に設定してください。

## On OpenCode 2

OpenCode 2 は opencode 1 のプラグイン関数を実行しないため、アダプターは
V2 用のエントリーポイント `setup(ctx)` も備えています。これは V2 のプラグイン API から
同じフックのロジックを駆動します（2.0.24 で実際に検証済み）:

| OpenCode 2 のタイミング | 駆動するもの |
| --- | --- |
| `session.hook("prompt")` | プレゼンスの発行と session-start（`chat.message`） |
| `tool.hook("execute.before")` / `("execute.after")` | bash の境界、ガード、レビュアーのスコープ、監査、センサー、ステージグラフ、サブエージェントのログ |
| `session.hook("compaction")` | コンパクション前の状態検証 |
| `session.execution.succeeded` イベント | Stop フックとその促し（`session.idle`） |
| `session.execution.interrupted`、`permission.replied` イベント | Esc や拒否されたコマンドの後は促しを行わない |
| `shell.hook("create.before")` | 各シェルコマンドが自身のチャットの作業に対して動作する（`shell.env`） |

V2 では組み込みツールの名前が変わっています（`bash` → `shell`、`task` → `subagent`、
`apply_patch` → `patch`）が、アダプターがそれらを元の名前に対応付けます。V2 のツールイベントには
呼び出し元のエージェントが記載されているため、レビュアーの識別はイベント自体から行われます。促しは
合成メッセージとして送信され、V2 はそれをプロンプトフックに渡さないため、
人間のターンとして数えられることは決してありません。サブエージェントのプロンプトはプロンプトフックに届きますが、
アダプターは親セッションによって子セッションを識別し、それらをスキップします。

OpenCode 2 のシェルイベントはコマンドの環境を含みますが、セッションは含まないため、
アダプターはその直前のシェルツールの呼び出しからセッションを取得します。opencode 自身のターミナルで
入力されたコマンドにはそのような呼び出しがなく、どのチャットにも紐付きません。
これは opencode 1 と同じです。

同梱の `opencode.json`、エージェントファイル、`/aidlc` コマンドは
opencode 1 の形式のままで、OpenCode 2 はそれを変更なしで読み込みます。その形式のままにしておいてください:
`aidlc doctor` と `aidlc config` がその形式をチェックします。

opencode 1 と比べて機能が低下する点:

- **ステータスラインの同期がありません。** OpenCode 2 には todo ツールがないため、
  `aidlc-sync-workflow-state.ts` を実行する `todowrite` のタイミングが発生しません。ワークフローの
  状態はエンジン自身のコマンドによって引き続き進みますが、
  todo 駆動のステータスラインのミラーだけが欠けます。`/aidlc --status` を使用してください。
- **`/aidlc` はテンプレートを表示します。** OpenCode 2 には `command.execute.before`
  フックがないため、チャットには入力した行ではなくコマンドのテンプレートが表示されます。
  アダプターはテンプレートから入力した行を読み戻すため、
  `/aidlc --review none` のようなスイッチは保持され、引数なしの `/aidlc` も opencode 1 と同様に
  コマンドとして扱われます。
- **エンジンのエラーのトーストはありません。** OpenCode 2 はプラグインに TUI のトーストの経路を提供していません。
  コンダクターのスキルは引き続きエンジンのエラーメッセージをそのまま出力します。
- **デフォルトのシェルの方言。** OpenCode 2 は `shell` の設定をプラグインに公開していないため、
  bash の境界は、プラットフォームのデフォルトのシェルと同じ方法でコマンドを読み取ります
  （Linux と macOS では POSIX の `sh`）。

## Verifying an install

```bash
aidlc doctor                               # native install
bun .aidlc/tools/aidlc-utility.ts doctor   # source/development copy
opencode run --command aidlc -- "--status"  # /aidlc --status through the harness
```

doctor による opencode 固有のチェックは次のとおりです: アダプタープラグインが
`.opencode/plugin/` に存在すること、プロジェクトルートに `opencode.json` または `opencode.jsonc`
が存在すること、そして `.opencode/command/aidlc.md` が存在すること。
