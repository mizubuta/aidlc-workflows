# CLI Commands

AI-DLC には、利用者が使うコマンドの系統が 2 つあります。ハーネスのチャットコマンドは
`/aidlc`（Codex では `$aidlc`）でワークフローを進めます。インストールされたネイティブの
`aidlc` コマンドは、プロジェクトを初期化し、機械向けのライフサイクル、
診断、ライフサイクルの各経路を提供します。

> **呼び出しの接頭辞はハーネスによって異なります。** Claude Code、Kiro IDE、Kiro CLI、
> Cursor、opencode、GitHub Copilot では `/aidlc` と入力します。Codex CLI では `$aidlc`（または
> `/skills` → aidlc）です。以下のフラグと動作はどちらでも同じで、
> 変わるのは接頭辞だけです。例では `/aidlc` を使っています。Codex では `$aidlc` に
> 読み替えてください。[Kiro](harnesses/kiro.md)、
> [Kiro CLI (V2)](harnesses/kiro.md#kiro-cli-v2)、[Codex CLI](harnesses/codex-cli.md)、
> [Cursor](harnesses/cursor.md)、[opencode](harnesses/opencode.md)、
> [GitHub Copilot](harnesses/copilot.md) の各ハーネスガイドを参照してください。

> **Cursor のショートカット。** Cursor では `/aidlc-status`、
> `/aidlc-jump --stage <slug|#>`（または `--phase <name|#>`）、
> `/aidlc-scope <name>` もネイティブのスキルとして使えます。これらは以下の対応する `/aidlc`
> の形をまとめたもので、同じエンジンを使います。別の状態経路ではなく、
> エイリアスです。

---

## Quick Reference

| コマンド | 説明 |
|---------|-------------|
| `/aidlc [scope]` | スコープを明示して新しいワークフローを開始します |
| `/aidlc [description]` | 新しいワークフローを開始します。エージェントが説明からプランを提案します（合うものがなければ組み立てを提案します） |
| `/aidlc compose "<task>"` | 適応型コンポーザーを強制します。タスクに合わせた EXECUTE/SKIP プランを提案します |
| `/aidlc compose --report <path>` | スキャンレポートから組み立てます（指摘をトリアージして、修正と出荷のコンパクトな実行にまとめます） |
| `/aidlc --new-scope '<task>'` | 既製のスコープが合う場合でも、コンポーザーにカスタムプランを合成させます |
| `/aidlc` | 開いているワークフローを再開します。まだ作業がなければ、エージェントが何を作りたいかを尋ねます |
| `/aidlc park` | アクティブなワークフローを現在のステージ境界で保留し、後のセッションや別の人に引き継げるようにします |
| `/aidlc team-board [--snapshot] [--space <name>] [--intent <name>]` | 読み取り専用の Team Construction ボード（ユニットの進捗、クレーム、マージ準備状況）を表示します |
| `/aidlc intent [name]` | アクティブなスペースのインテントを一覧表示します（`--all` でアーカイブ済みも含む）。または既存のインテントに切り替えます |
| `/aidlc intent archive <name>` | 進行中または完了済みのインテントを、記録を削除せずに引退させます。`unarchive <name>` で元に戻します |
| `/aidlc intent add-repo <name> [--url <address>]` | どのステージでも、アクティブな作業に兄弟リポジトリを追加します。まだ手元になければ先にクローンします。`remove-repo <name>` で外します |
| `/aidlc intent set-repos <name,...>` | アクティブな作業が対象とする兄弟リポジトリを正確に設定します。`--all` は記録済みのリポジトリをすべて残します |
| `/aidlc intent add-checkout <path>` | ハーネスフォルダー内やプロジェクト外の worktree にあるコードもレビュー対象にします。`remove-checkout <path>` で外します |
| `/aidlc space [name]` | スペースを一覧表示します。または既存のスペースに切り替えます |
| `/aidlc space-create <name>` | デフォルトスペースの組織ルールと、空のチーム・プロジェクトのプラクティスから始まる新しいスペースを作成します |
| `/aidlc knowledge <verb>` | 自分のドキュメントをインデックス化して読みます（`onboard`、`sync`、`list`、`show`、`associate`、`dissociate`、`rebind`、`summarize`） |
| `/aidlc <plugin command> [args]` | インストール済みプラグインが追加したコマンドを実行します（ターミナルでは `aidlc <name>`）。`aidlc --help` はそれらをプラグインごとに一覧表示します |
| `/aidlc --status` | 読み取り専用のステータス概要を表示します |
| `/aidlc --config [section]` | 対話的にプロジェクトのポリシーを設定し、正確で決定論的な設定フラグとして反映します |
| `/aidlc --claim <unit> [--team <label>] [--rhythm <per-stage\|unit-end>]` | チーム所有の空いているユニットをアトミックにクレームし、このチェックアウトをその試行に結び付けます |
| `/aidlc --release <unit>` | スコープ外の main から tombstone を公開して、有効なユニットのクレームを解放します |
| `/aidlc unit adopt <unit>` | 新しいクローンで、チェックアウトした有効なクレームブランチを引き継ぎます |
| `/aidlc unit participate` | このクローンを、ガイド付きのユニットクレーム選択の対象としてマークします |
| `/aidlc unit share` | ここでの作業をコミットして統合ブランチをプッシュし、チームがユニットを取れるようにします。または取り込み済みのものを確認します |
| `/aidlc unit publish <unit> [--commit [--include <path>]...]` | スコープ付きチェックアウトのクリーンなコミット済み候補を、クレーム ref に CAS で公開します。`--commit` は先にユニット自身の作業をコミットし、`--include` は漏れた新しいファイルやフォルダーを含めます |
| `/aidlc unit pin <unit>` | スコープ外の main から、完了した候補 OID を 1 つ固定して検証します |
| `/aidlc unit gate <unit> ...` | 固定した OID と世代に対するマージ判断を記録します |
| `/aidlc unit land <unit> ...` | 再開可能な git → state → audit の取り込みトランザクションを実行します |
| `/aidlc unit merge-status <unit>` | ローカルの固定マージのトランザクションジャーナルを読みます |
| `/aidlc unit status` | 現在のクレーム可能、クレーム済み、依存関係でブロック中のユニットの集合を読みます |
| `/aidlc --doctor [--check-updates]` | ヘルスチェックを実行します。明示的なフラグで更新メタデータを再取得します |
| `/aidlc --doctor --export` | 新たにヘルスチェックを実行し、共有用に秘匿化した小さな診断レポートを書き出します |
| `/aidlc --stage <slug\|#>` | 特定のステージにジャンプします |
| `/aidlc --stage <slug> --single` | ワークフローを進めずに、1 つのステージを単独で実行します |
| `/aidlc --phase <name\|#>` | フェーズの先頭にジャンプします |
| `/aidlc --scope <name>` | アクティブなスコープを変更します |
| `/aidlc --depth <level>` | 深さのレベルを上書きします（minimal、standard、comprehensive） |
| `/aidlc --test-strategy <level>` | テスト戦略を上書きします（minimal、standard、comprehensive） |
| `/aidlc --project-type <type>` | この作業が新規プロジェクトか既存コードかを指定します（greenfield、brownfield）。ワークフローの途中では再スキャンし、既存コードなら Reverse Engineering を実行します |
| `/aidlc --review <class>` | この実行のステージレビューを設定し、スコープの上限を置き換えます（adversarial、advisory、none） |
| `/aidlc --guard-policy <value>` | この作業でガードがどこまで退くかを設定します（strict、relaxed、off）。`--change-control` は廃止された旧名です |
| `/aidlc --sensors <on\|off>` | このインテントでのセンサーの自動実行とブロッキングセンサーのチェックを設定します |
| `/aidlc --learnings <on\|off>` | このインテントでの学びの日誌と学びのゲートの手順を設定します |
| `/aidlc --summary-confirmation <on\|off>` | このインテントでの統合サマリー確認のチェックポイントを設定します |
| `/aidlc --plan-approval <on\|off>` | このインテントで、各コードプランを構築前に承認のために提示するかどうかを設定します（オフにできるのは利用者だけです） |
| `/aidlc --collaborators <on\|off>` | このインテントで、ステージがサポートエージェントを呼び入れるか、リードエージェントだけで実行するかを設定します |
| `/aidlc --statusline <on\|off>` | ステータスラインに AI-DLC の進捗を表示するか、元のステータスラインに戻します（Claude Code: 自分だけ、Codex: プロジェクトの全員） |
| `/aidlc config get <key>` | アクティブなワークフローの設定を表示します（`depth`、`test-strategy`、`review`、`guard-policy`、`sensors`、`learnings`、`summary-confirmation`、`plan-approval`、`collaborators`、`guard.<fence>`） |
| `/aidlc config set <key> <value> [--key value ...]` | 共通の設定処理を通じてインテントの設定を変更します。フラグとして入力された設定や、普通の言葉で頼まれた設定に対して、エージェントがこれを実行します |
| `/aidlc config set guard.<fence> <on\|off>` | この作業で 1 つのフェンスをオフにするか、ポリシーの指定より優先してオンに戻します（review-freeze、state-transition、reviewer-scope。`guard.plan-approval` は `plan-approval` の別名です） |
| `/aidlc config list` | アクティブなワークフロー設定 13 項目をすべて一覧表示します（構造化出力は `--json`） |
| `/aidlc plugin select [names]` | このインストールで有効なプラグインの一覧を表示または設定します |
| `/aidlc plugin list` | インストール済みプラグインと有効状態を一覧表示します |
| `/aidlc plugin sync` | インストール済みプラグインのルートを現在のインストールに組み込みます |
| `/aidlc plugin validate [path]` | 作成したプラグインを検証します（`--with <base-root>` はベースプラグインのエージェントを確認し、`--json` は構造化された結果を出します） |
| `/aidlc plugin build <harness> [outDir]` | ホスト向けのプラグイン投影をビルドします（`--plugin-root <path>` でソースを選びます） |
| `aidlc plugin test [path] --install <project-root>` | ターミナルから実行します。作成したプラグインがプロジェクトのコピーに組み込めるかをテストします。元のプロジェクトは変更されません（`--harness <name>`、`--with <base-root>`、`--json`） |
| `/aidlc --version` | フレームワークのバージョンを表示します |
| `/aidlc --help` | 使い方を表示します |
| `bun .claude/tools/aidlc-utility.ts select-plugins [names]` | プラグイン選択のユーティリティを直接呼ぶ形です |
| `aidlc engine worktree restore --slug <slug> [--parked <stamp>] [--raw]` | 退避した Bolt の試行から、別のチェックアウトにファイルを復元します |
| `aidlc engine worktree purge --slug <slug> [--parked <stamp> \| --older-than <days>]` | 復元したチェックアウトがなくなった後で、選択したローカルの復旧用 ref を削除します |

`--status`、`--doctor`、`--help`、`--version` は、単独でも他のフラグと並べても
実行できます。説明文の中にある場合は、依頼内容の一部として扱われます。
`/aidlc add a --version flag that prints the version from package.json` は
その作業を開始し、AI-DLC のバージョンは表示しません。

---

## Terminal Color

公開されている 6 つのターミナルコマンド（`config`、`doctor`、`version`、`update`、
`use`、`uninstall`）は、人間向けの出力にだけ控えめな色を使います。JSON、quiet
出力、ファイル、監査記録、TTY でないストリームは色なしのままです。

色の選択は次の優先順位で決まります。

1. `--no-color` は色を無効にします。
2. 環境変数 `NO_COLOR` が設定されていれば、値に関係なく色を無効にします。
3. `FORCE_COLOR` が空でなく `0` 以外の値なら、色を有効にします。
4. それ以外の場合、色は TTY のストリームで、かつ `TERM` が `dumb` でないときだけ有効になります。

判定は stdout と stderr で別々に行われます。1 つのコマンドには `--no-color` を、
シェルやプロセスの環境には `NO_COLOR=1` を使います。ターミナルのラッパーが ANSI カラーに対応していても
TTY 検出を公開していない場合は `FORCE_COLOR=1` を使います。

---

## Command Decision Tree

```mermaid
flowchart TD
    START(["What do you want to do?"])

    Q1{"Start a new\nworkflow?"}
    Q2{"Check or manage\nan existing workflow?"}
    Q3{"Verify the\nproject?"}

    A1["/aidlc feature"]
    A2["/aidlc Build a payments API"]
    A3["/aidlc"]
    A4["/aidlc --status"]
    A5["/aidlc --stage code-generation"]
    A6["/aidlc --phase construction"]
    A8["/aidlc --doctor"]

    START --> Q1
    START --> Q2
    START --> Q3

    Q1 -->|"Know the scope"| A1
    Q1 -->|"Describe what you want"| A2
    Q2 -->|"Resume where I left off"| A3
    Q2 -->|"See progress"| A4
    Q2 -->|"Jump to a stage"| A5
    Q2 -->|"Jump to a phase"| A6
    Q3 -->|"Verify setup"| A8

    style START fill:#e1bee7,stroke:#7b1fa2,color:#000
```

<!-- Text fallback: Starting a new workflow: use /aidlc classic (known scope) or /aidlc Build a payments API (the agent proposes a plan; the first intent auto-creates). Managing an existing workflow: /aidlc (resume), /aidlc --status (view progress), /aidlc --stage (jump to stage), /aidlc --phase (jump to phase). Verify setup: /aidlc --doctor (health check). -->

---

## Detailed Reference

### `/aidlc [scope]` — Start with explicit scope

有効なスコープの 1 つを指定して新しいワークフローを開始します。コアには名前付きのスコープが 11 個あります。プラグインでさらに追加でき、`select-plugins` で無効にしたプラグイン／コアのスコープを実行時から隠せます。

**構文:**

```
/aidlc enterprise
/aidlc feature
/aidlc mvp
/aidlc poc
/aidlc bugfix
/aidlc refactor
/aidlc infra
/aidlc security-patch
/aidlc classic
/aidlc workshop
/aidlc express
```

**動作:** フレームワークはスコープのキーワードを認識し、何を作りたいかを尋ねます（あなたが答えるまで何も作成されません）。その後、あなたの言葉を依頼内容として、そのプランで Initialization フェーズを実行し、最初のドメインステージを開始します。すでにワークフローがアクティブな場合、スコープのキーワード単独ではその作業を続け、後ろに説明が続く場合はその説明が新しい作業かどうかを尋ねます。キーワードが長い説明の最初の単語でもある場合（`/aidlc classic car rental website`）は、決めつけずに読み取ります。アシスタントがどちらの意味かを判断し、一度だけ尋ねることがあるため、選んでいないプランにその単語が使われることはありません。キーワードと 1 単語の組み合わせ（`/aidlc feature flags`、`/aidlc bugfix typo`）も同じように読み取るため、`feature flags` は依頼内容のままとなり、`bugfix typo` はタイポに対する bugfix の作業を開始します。コロンを付けるとプランの指定であることが明示されます（`/aidlc bugfix: Fix duplicate todos`）。`--scope` も同じ意味になります（`/aidlc --scope bugfix Fix duplicate todos`）。11 個の選択肢の実践的な比較は [ワークフロープロファイル](workflow-profiles.md) を参照してください。

**例:**

```
/aidlc bugfix
> I'll use the bugfix plan. What should I work on? Tell me in your own words and I'll start it.
> The login API returns 500 when email contains a plus sign
```

---

### `/aidlc [description]` — Start from a description

作りたいものを説明すると、エージェントがそれに合うプランを提案し、エンジンが確認を求めます。

**構文:**

```
/aidlc Build a REST API for inventory management
/aidlc Fix the login timeout bug
```

**動作:** エージェントが説明を読み、それに合うプランを提案します（バグ報告なら通常は bugfix）。エンジンは、そのプランと実際の手順（ステージ数、承認ゲート数、ユニットごとのファンアウトの有無。いずれもコンパイル済みのグリッドから算出）を示した 1 行の確認を求めます。Greenfield の作業では Reverse Engineering は除外され、ユニットごとの記述は `units-generation` が実行されてユニットの DAG を作成する場合にだけ表示されます。既製のプランが合わない場合は、黙ってデフォルトを使うのではなく、組み立ての提案（後述の `/aidlc compose` を参照）が表示されます。ワークフローが始まる前に、確認または上書きができます。プラン名で始まる説明は、すべての単語が保持されます（`/aidlc classic car rental website` はレンタカーのウェブサイトについての依頼です）。その単語がプランを指しているかどうかは、エージェントが読み取ります。

**例**（新規プロジェクトの場合。既存のコードベースでは bugfix は Reverse Engineering も実行するため、6 stages と 6 approval gates と表示されます）:

```
/aidlc Fix the ProfileSerializer null pointer
> I'd run the "bugfix" plan for: "Fix the ProfileSerializer null pointer" - 5 stages, 5 approval gates. Do you want me to go ahead with it, use a different plan, or tailor one to this task?
```

**シェルで渡せない依頼。** Windows では、cmd.exe は改行でコマンドを終わらせ、
引用符の中でも `%NAME%` や `!NAME!` の組を置き換えます。また aidlc
ランチャーは cmd.exe によって再度読み込まれます。これらのいずれかを含む依頼は、
コマンドラインではなくファイル経由でエンジンに渡されます（Kiro IDE では、ダブルクォート、
`“` や `’` のような活字の引用符、`&`、`|`、`<`、`>`、`^` のいずれか、または末尾のバックスラッシュを
含む依頼も、Windows PowerShell や cmd.exe が変えてしまう可能性があるため、エージェントがこの方法で送ります）。エージェントは依頼を
`aidlc/.aidlc-request-text/request.txt` に書き込み、ほかのフラグとともに `aidlc engine orchestrate next
--request-file aidlc/.aidlc-request-text/request.txt` を実行します。
Kiro（Kiro IDE と v3 エンジン上の Kiro CLI）では、依頼に `$`、バッククォート、`>`、`<`、`&`、`@(`、`@{`、改行の
いずれかが含まれる場合、どのシェルでもエージェントがこの方法で送ります。これらを含む AI-DLC コマンドには Kiro が承認を
求めるためです。エンジンは `aidlc/.aidlc-request-text/` の直下にある通常のファイルだけを、
リンクを経由せず、64 KiB まで読み込みます。その言葉は書かれたとおりに使い、コマンドが進んだ時点で
ファイルを削除します。

---

### `/aidlc compose` - The adaptive composer

既製のスコープが合う場合でも、コンポーザーを強制します。次の 3 つの場面で使えます。

```
/aidlc compose "harden the deployment pipeline and add observability"
/aidlc compose --report sonar.json
/aidlc compose            (mid-workflow: re-shape the pending stages)
```

**動作:** コンダクターがコンポーザーエージェントを起動します。コンポーザーエージェントはタスク（またはスキャンレポート、実行中のワークフローの状態）を読み、読み取り専用の `detect` スキャンを実行し、実装エントロピーの 5 要素（インテントの曖昧さ、構造の不確実性、検証のエントロピー、リスク、未解決の前提。設定されていれば CodeKB MCP の分析に、そうでなければワークスペースのスキャンに基づきます）を見積もります。そして、スコアの内訳と、各 EXECUTE と SKIP の理由を添えて、必要最小限の EXECUTE/SKIP グリッドを提案します。あなたはゲートで承認、編集、却下のいずれかを選びます。承認すると、既製のスコープに一致する場合は AI-DLC がワークフローを直接作成します。カスタムプランの場合は、同じターンのうちに、最も近い既製スコープ（新規プロジェクトでは、合うものがあれば新規作業向けの最も近いスコープ）の上に、プラン独自のステージ変更（`--skip` / `--add`）を加えてワークフローを作成し、スコープファイルは書きません。プランはこの作業専用です。カスタムプランのゲートでは **Approve and save as scope** も選べます。これは名前を尋ね、作業の作成時にプランを再利用可能なスコープとして保存します（保存は [`scope save`](#aidlc-engine-scope-save-keep-a-plan-as-a-scope) が行います）。後から「save this plan as <name>」と言うと、`scope save` そのものが実行されます。前段／レポートからの提案には、必ず空でない `creationDescription` が付きます。元のタスクの文面があればそれをそのまま使い、なければレポートやプランに基づく説明を使います。タスクから始めた組み立てでは、作成時にゲートの `--request <id>` を渡し、エンジンが元の説明を取り出します。レポート起点やタスクなしの組み立てでは、その request id とともに、`--` の後ろに説明をシェルで安全な 1 つの引数として渡します。組み立ての承認は、スコープだけで説明がない状態では先に進めません。進行中の提案は、`recompose` 動詞によって保留中ステージの末尾の切り替えとして反映されます（監査ロックのもとで厳密に検証され、`RECOMPOSED` が監査記録されます）。完了済みの作業ではコンポーザーは実行されません。AI-DLC は最後のステージの後から作業を再開するプランを、それぞれが実行するステージとともに挙げ、選んだプランでそこから作業を再開します。`--new-scope` は合成を強制し、`--report <path>` はトリアージ済みの指摘をインテントに取り込みます。`/aidlc-compose` スキルは、同じ経路を入力しやすくしたショートカットです。ワークフローの途中では、チャットで言うだけでもかまいません。自由な依頼（「ほかに削れるものは？」）は認識されて同じゲートと動詞に送られるため、`compose` と文字どおり書く必要はありません。一方、ステージを名指しした依頼（「market research は飛ばせる？」）は、その場でプランを変更し、取り消し方を示します（Claude 以外のハーネスでは、文字どおりの動詞を使うのが文書化された確実な方法です）。

全体の流れは [スコープと深さ - 適応型コンポーザー](05-scopes-and-depth.md#the-adaptive-composer) を参照してください。

---

### `/aidlc` — Resume existing workflow

状態ファイルがあるときに引数なしで実行すると、再開します。

**構文:**

```
/aidlc
```

**動作:** `aidlc-state.md` を読み、`/aidlc --resume` と同じく保存されたチェックポイントから続けます。別のことをしたい場合は、やり直し、ステージへのジャンプ、最初からの開始などを伝えてください。詳しくは [セッション管理](11-session-management.md) を参照してください。

`/aidlc --resume` も保存されたチェックポイントから直接続けます。後ろに入力した言葉（`/aidlc --resume redo the scan`）は、保留中のワークフローの保留を解除した後、`/aidlc` 単独の場合と同じように読み取られます。明示した行き先を優先して通常のジャンプ動作で進めたい場合は、`--stage <slug>` を付けてください。

状態ファイルがない場合、フレームワークは新しいワークフローとして扱い、スコープ／説明を尋ねます。

---

### Workflow Initialization — automatic

手動コピーでのインストールには、スキャフォールド用のコマンドはありません。Bun 形式の
`aidlc-copy-runtime-X.Y.Z.tar.gz` に含まれるバージョン付きの `runtime/<harness>/` シェルは
ビルド済みの状態で届き、ネイティブの
`aidlc` 実行ファイルを必要としません（`.claude/` のエンジンと
`aidlc/spaces/default/memory/`）。
作りたいものを説明し、エンジンが提案するプランを確認すると、エンジンが最初のインテントを作成します
（作業がない状態で `/aidlc` だけを実行すると何を作りたいかを尋ね、
スコープだけを指定すると作業の目的を尋ねます）。作成時には、3 つの Initialization ステージ
（Workspace Scaffold、Workspace Detection、State Init）を 1 回の決定論的な
ツール呼び出しとして実行します。インテントの記録ディレクトリを
`aidlc/spaces/<space>/intents/<YYMMDD>-<label>/` に作成し（`audit/` のシャードディレクトリ、
スコープが実行するフェーズごとの成果物ディレクトリ、`verification/`）、スペースレベルの空の
`aidlc/spaces/<space>/knowledge/` ディレクトリを作成し、ルールベースのワークスペーススキャンを実行して、そのインテントの
`aidlc-state.md` をスコープのプランとともに書き込みます。
初期化シーケンスのイベント（`WORKFLOW_STARTED`、`WORKSPACE_SCAFFOLDED`、
`WORKSPACE_SCANNED`、`WORKSPACE_INITIALISED`、およびステージごとの
`STAGE_STARTED`/`STAGE_COMPLETED`）を記録します。スコープを指定すると（`/aidlc --scope feature`）
それが初期スコープになります。指定がなければ、実際の環境変数 `AWS_AIDLC_DEFAULT_SCOPE`、
次に記録済みの `aidlc config flags --default-scope` の値、
最後に `classic` の順に解決します。最初の実行の前にチームのナレッジや
ガードレールを追加するには、同梱の `aidlc/spaces/default/memory/`
ファイルを編集してください。スペースレベルの `aidlc/spaces/<space>/knowledge/` ディレクトリは、最初のインテントができた時点で
（空の状態で）作成されるので、それ以降は自由な形式のファイルを追加できます。

プロジェクトのインストールと更新には、ネイティブの config コマンドを使うのが推奨の方法です。
フレームワークの開発者は、無視対象の Bun 形式の `dist/`
投影を `bun scripts/package.ts` でローカルに生成できます。リリース版の利用者は、チェックアウトから
コピーしないでください。

ネイティブのマシンインストールでは、ハーネスを開く前に `aidlc config` を一度実行してください。
このコマンドは同じシェルを配置し、更新の基準点を記録します。ワークフローの
インテントの作成は、最初のチャットでの呼び出し時に自動で行われるままです。

Claude Code では、ウェルカムメッセージはセッション開始時に
`.claude/settings.json` の `companyAnnouncements` エントリによって表示されます。ほかのハーネスには
ウェルカムメッセージはなく、`/aidlc --status`（Codex では `$aidlc --status`）で
作業の状況を確認できます。

**マルチリポジトリのワークスペース。** ワークスペースのルートに兄弟のコードリポジトリが
複数ある場合（それぞれ `.git` を持つ直下の子ディレクトリ）、作成ステップは
インテントが扱うリポジトリの集合を `intents.json` の行に記録します。デフォルトでは
兄弟リポジトリをすべて**自動検出**します。ただし、隣に自前のコードを持つプロジェクトの
git サブモジュールは除きます。そのプロジェクトは単一リポジトリのプロジェクトで、それが
作業のリポジトリになります（サブモジュールだけでできたプロジェクトは、サブモジュールを記録します）。インテントを
特定の一部のリポジトリに絞る場合や、サブモジュールを作業のリポジトリとして指定する場合は、
作成ツールが `--repos a,b`（リポジトリのディレクトリ名のカンマ区切りリスト）を受け付けます。
これらはエンジンが代わりに実行する決定論的な `aidlc-utility intent-create` ステップのフラグであり、
あなたが入力する `/aidlc` のフラグではありません。Construction の間、各 git
操作（worktree、スウォーム、Bolt）は 1 つのリポジトリを対象にします。コンダクターは
`--repo <name>` を渡して対象を固定します。これはインテントが複数の
リポジトリにまたがる場合にだけ必要です。リポジトリが記録されていないインテントは、単一リポジトリのデフォルトです（git は
ワークスペース／プロジェクトのディレクトリで実行されます）。チーム所有のユニットは現在、この単一リポジトリの
デフォルトを必要とします。兄弟リポジトリが記録された作業では、ユニットを誰が構築するかは尋ねられず、
最初の Construction ステップで、このセッションが構築することが 1 行で伝えられます。
`set-unit-ownership team` は状態を変更する前に同じ 1 行で拒否します。
[成果物リファレンス](14-artifacts-reference.md) を参照してください。

---

### `/aidlc park` - Park the workflow

現在のステージ間の境界できれいに停止し、後のセッションで、または `aidlc/` ツリーをコミットしてプルした別の人がワークフローを引き継げるようにします。

**構文:**

```
/aidlc park
```

**動作:** エンジンはこの動詞を `aidlc engine orchestrate park --person-asked` に送ります（エンジン自身の park で、AI-DLC が事前承認しているエンジンコマンドに含まれるため、エージェントは許可プロンプトなしで実行します）。これは `WORKFLOW_PARKED` を発行し、状態ファイルに保留マーカーを記録し、保留したステージを報告します。ステージは進まず、何も完了扱いにはなりません。アクティブなワークフローがない場合や、ワークフローがすでに Completed の場合、保留は拒否されます。ユニットにスコープされたチームのチェックアウト（ユニットのスコープスタンプを持つ Construction の worktree）では、同じコマンドが代わりにそのユニットをローカルで保留します。チェックアウトローカルのユニット保留マーカーを書き込み、共有のワークフロー状態には触れず、ルーティングされたコマンドを通じて `parked` ディレクティブを発行します。そのチェックアウトで次に `/aidlc` を実行すると、そのユニットがそこで保留中であると報告されます。`/aidlc --resume` で再開すると、該当するマーカーが解除されて処理が続きます。この動詞は単独のトークンとしてのみ機能します。長い文の中の `park` は作業の説明として扱われるため、保留したい場合はコンダクターに文章で頼むか、動詞だけを入力してください。

ユーザーごとのカーソル `aidlc/spaces/<space>/intents/active-intent` は gitignore の対象なので、保留されたワークフローをプルしたチームメイトは、`/aidlc --resume` の前に `/aidlc intent <name>` でそれを選択します。

---

### `/aidlc team-board` - Team Construction board

チーム所有の Construction の読み取り専用ビューです。ユニットの進捗、観測されたクレーム、固定したマージの準備状況、クレーム可能なユニット、ブロッカーを表示します。`Unit Ownership: team` のときに `/aidlc --status` が末尾に付け加えるボードと同じものです。

**構文:**

```
/aidlc team-board
/aidlc team-board --snapshot
/aidlc team-board --space <name> --intent <name>
```

**動作:** エンジンはこの動詞を `aidlc team-board` に送り、状態、キャッシュ、監査に触れずに、その出力をそのまま表示します。受け付けるのは `--snapshot`、`--space <name>`、`--intent <name>` だけで、それ以外のトークンは使い方のエラーになります。`Unit Ownership: team` が必要です。

---

### `/aidlc intent [name]` — List or switch intents

`/aidlc intent` だけを実行すると、アクティブなスペースの進行中および完了済みのインテントを
一覧表示します。構造化出力（アーカイブ済みを含むすべての行）には `--json` を、
人間向けの一覧にアーカイブ済みのインテントを表示するには `--all` を付けます。`/aidlc intent <name>`
は、一意に特定できる slug または記録ディレクトリの完全な名前で、ユーザーごとのアクティブインテントのカーソルを
既存のインテントに切り替えます。インテントを作成したり、ワークフローを進めたりすることはありません。
`/aidlc` の後に名前だけを入力した場合（`/aidlc auth`）も、それがここにあるちょうど 1 つのインテントの名前であれば、
同じようにそのインテントを選択します。切り替えの行には、それまでのインテントと、元に戻るためのコマンドが示されます。
`intent` の後の単語が動詞でも既存のインテントの正確な名前でもない場合
（タイポや、インテントではできないこと）は、動詞の一覧と最近のインテント名とともにエージェントに渡され、
エージェントが意図を読み取るか一度だけ尋ねます。名前の後に入力したもの（依頼や、
`--guard-policy relaxed` のような設定）は、そのまま丸ごとエージェントに渡され、選択したばかりのインテントに対して実行されます。

### `/aidlc intent archive <name>` — Retire an intent you will not finish

`/aidlc intent archive <name> [--reason '<text>']` は、進行中または
完了済みのインテントを `archived` ステータスに移します。何も削除されません。記録ディレクトリ、その
成果物、監査シャードはそのままの場所に残り、アーカイブ自体は
そのインテントの監査証跡に `WORKFLOW_ARCHIVED` として記録されます（`--reason` を
指定した場合はその理由も）。レジストリの行は `archived` に、
状態ファイルの `Status` は `Archived` に切り替わり、デフォルトの `/aidlc intent`
の一覧ではその行が非表示になります。アーカイブしたインテントがアクティブだった場合は、ユーザーごとの
カーソルがクリアされるため、次の `/aidlc` では引退したステージを再開するのではなく、どのインテントに取り組むかを尋ねます
（残っているインテントがなければ新しい作業を作成します）。

完了済みのインテントもアーカイブでき、これで完了した作業をデフォルトの一覧から隠せます。
Bolt の worktree を持つインテントもアーカイブできます。出力には
worktree の名前が示され、インテントを元に戻すまでディスク上にそのまま残ります。
アーカイブが拒否されるのは、クレーム済みのユニットを持つチーム所有のインテントだけです。
それらのユニットは、まだ別のチェックアウトで作業中だからです。

`/aidlc intent unarchive <name>` はこれを元に戻します。行と `Status` は
元の値に戻り（停止していたステージでの `in-flight` と `Running`、または
完了済みの作業なら `complete` と `Completed`）、`WORKFLOW_UNARCHIVED` が
記録されます。理由を記録するのは `archive` だけです。`unarchive` に指定した `--reason` は
記録されず、出力でもそう伝えられます。カーソルは移動しません。続けたい場合は
`/aidlc intent <name>` で復活したインテントに切り替えてください。


### `/aidlc intent add-repo <name>`: add a sibling repo to work already under way

作業は、作成時に扱う兄弟リポジトリを記録します（その時点のワークスペースで
`.git` を持つ子ディレクトリ、または明示的な `--repos`）。後になって
作業が別のリポジトリも変更することがわかった場合は、エージェントが理由とともに
`intent add-repo <name>` を実行して 1 行で伝えます。あなたがそう伝えて、エージェントに実行してもらうこともできます。作業が読むだけのリポジトリは追加されません。リポジトリが
記録され、変更は監査証跡に `INTENT_REPOS_CHANGED` として残り、
次に何が起こるかを 1 行で確認できます。Reverse Engineering がすでに
実行済みの場合、Reverse Engineering とコードのナレッジを読むステージは、見直すまで新しいリポジトリについて
遅れている状態として表示されます。まだ実行されていなければ、実行時に新しいリポジトリも対象になります。
リポジトリが加わる前に確認してほしいチームは、スペースメモリの `## Mandated` の下に
1 行でそう書いておきます。

リポジトリがまだワークスペースにない場合は、まずほかのリポジトリの隣にクローンされます。
クローンには `aidlc system workspace-sync` と同じコードと、あなた自身の git
設定が使われます（認証情報ヘルパーや `insteadOf` ルールは、自分のシェルと同じように適用されます）。
アドレスは、`--url` で指定したもの、なければリポジトリの `repos.json` エントリ、
それもなければ `repos.json` の `org` です。エージェントが自分の判断で `--url` を渡すことは
決してないため、コード検索で見つけた名前は、チームの org からしかクローンできません。
クローンはステージング用のフォルダーで行われ、ワークスペースのロックのもとで所定の場所に移動されます。
その後 `repos.json` のエントリが書き込まれ（コメントとレイアウトは保持されます）、管理対象の `.gitignore` ブロックと `aidlc.code-workspace` が
更新され、その後で初めて変更が記録されます。出力の行にはアドレスが示されますが、
`https://user:token@host/...` のようなアドレスのユーザー部分は
`***` と表示されます。チームがコミットする `repos.json` には、認証情報を除いたアドレスが
書き込まれます。`repos.json` がない場合、
ホスト型の `--url`（ホスト、オーナー、名前を含むもの）を指定すると、オーナーを
`org` とするファイルが作成され、出力の行でもそう伝えられます。一覧を共有するにはこれをコミットしてください。ローカルパスや、
ほかの形式のアドレスでは、ファイルを作成せずにリポジトリを追加します。
クローンに失敗した場合は何も記録・変更されず、出力の行に git の理由と、
手動で実行するための `git clone` コマンドが示されます。`repos.json` の org も
`--url` もない場合は、出力の行でアドレスを尋ねます。

名前は単一のフォルダー名でなければならず、すでに存在するフォルダーは Git の
チェックアウト（実際の `.git` エントリを持つもの）でなければなりません。見た目がリポジトリのようなだけのフォルダーは
拒否され、区切り文字を含む名前も拒否されます。このコマンドは、すでに
リポジトリを記録している作業のためのものです。何も記録していない作業は、ワークスペースのフォルダー自体を
リポジトリとして扱います（最初のリポジトリは `workspace reclassify` が記録します）。最後に
記録されたリポジトリを削除することも同じ理由で拒否されます。ただし、それがプロジェクトの
git サブモジュールである場合（プロジェクトの `.gitmodules` に記載され、プロジェクトフォルダー自体が Git チェックアウトである場合）は例外で、
作業は何も記録しない状態に戻り、再びワークスペースのフォルダーで作業します。
何も記録していない作業からリポジトリを削除しても、何も変わりません。

締め切りはありません。Units Generation が承認された後でもリポジトリは追加
（または削除）でき、それが承認済みのユニットにとって何を意味するかは、作業の
Guard Policy に従います。`strict` では、次のステップで、変更後もユニットがそのままで
よいか、それとも Units Generation に戻るかを一度だけ尋ねます。
`relaxed` または `off` では、作業は承認済みのユニットのまま続行して
`CHANGE_ACCEPTED` として記録され、出力の行には "Units Generation was approved before this
change: carrying on with its Units and <name> added. You can go back to Units
Generation any time."（Units Generation はこの変更の前に承認されています。そのユニットのまま、そのリポジトリを追加して続行します。いつでも Units Generation に戻れます。）と表示されます。新しいリポジトリを変更するユニットは、Code
Generation のプランでそれを明示し、プラン承認がオンの場合はあなたがそれを承認します。追加時点の
新しいリポジトリのファイルは、それより前に取られた各ステージのソースの基準点に加えられるため、
既存のファイルがそのステージ中に行われた変更として報告されることはありません。

`/aidlc intent remove-repo <name>` は取り消しの操作です。リポジトリをこの
作業から外し、そのフォルダーと `repos.json` の行はそのまま残します。
`repos.json` は作業ではなくワークスペースの一覧であり、出力の行に手動で
外す方法が示されます。Units Generation の承認後に追加したリポジトリを削除しても、
新たに判断すべきことは残りません。

### `/aidlc intent set-repos <name,...>`: say exactly which repos the work covers

リポジトリが多いワークスペースでは、この作業に必要なものを伝えると（「orders
と payments だけ」）、エージェントが `/aidlc intent set-repos orders,payments` を実行します。
作業はちょうどそれらを対象とするようになり、各名前は `add-repo` と同じ方法で
確認されます。何が変わったか、残りは後から追加し直せることが 1 行で伝えられます。
`/aidlc intent set-repos --all` は作業が記録しているリポジトリをすべて残します。どちらの場合も、
その集合は作業の作成時に `--repos` で指定したものと同じく、あなたが決めたものとして扱われます。

AI-DLC がディスク上で見つけた 10 個以上のリポジトリを作業が記録していて、あなたがそれらを指定していない場合、
Reverse Engineering はどれかを読む前に、すべてを読むか一部だけにするかを一度だけ尋ねます。
多くのリポジトリを読むと時間がかかり、多くのトークンを使う可能性があるためです。
あなたが指定した集合と無人実行では尋ねません。ステップはコストを一度伝えて、
すべてのリポジトリを読みます。依頼の中ですでにどのリポジトリか（すべて、または名前を挙げたもの）を
伝えている場合は、尋ねる代わりにエージェントが設定します。ここにクローンされていない名前は、
それをクローンする `intent add-repo` コマンドを示して拒否されます。set-repos 自体がクローンすることはありません。Units Generation
の承認後は、`add-repo` と同じく、変更が反映されてGuard Policy に従います。

### `/aidlc intent add-checkout <path>`: have the review cover code kept in another checkout

AI-DLC のレビューは、プロジェクトフォルダー内のコードを対象にします。その中に入れ子になった worktree や
クローンも含みます。この作業のコードが、ハーネスフォルダー内の worktree
（Claude Code の `.claude/worktrees/<name>`）やプロジェクト外の worktree に置かれている場合、
レビューはそれを見ません。そのため承認の質問には
"AI-DLC found no source changes"（AI-DLC はソースの変更を見つけませんでした）と表示され、レビュー後にそこで行われた変更も
見逃されます。`/aidlc intent add-checkout <path>` は、そのチェックアウトをこの
作業用に記録します。以降はレビューの対象になり、レビュー後にそこで行われた変更も
プロジェクト内のどこで行われた変更とも同じように検出され、承認にはコードが見つかった場所が記録されます。
エージェントがコードをそこに置く場合はエージェントが代わりに実行します。また、ソースの一覧にハーネス
フォルダーの worktree 内のパスを挙げているユニットは、自動でそれを記録します。

パスは worktree またはクローンそのもので、プロジェクトからの相対パスでも絶対パスでもよく、
`.git` エントリが必要です。表示される行では `<label>@checkout` という名前で示されます。
これは、ユニットのソースの一覧がプロジェクト外のチェックアウトにあるコードを指すときの名前です。
質問をすることはなく、記録はこのマシンにだけ残ります。チームメイトの
プロジェクトのコピーには反映されません。`/aidlc intent remove-checkout <path>`
で再び外せます。

### `/aidlc space [name]` — List or switch spaces

`/aidlc space` だけを実行するとスペースを一覧表示します。構造化出力には `--json` を付けます。
`/aidlc space <name>` は、ユーザーごとのアクティブスペースのカーソルを切り替え、その
スペースのメソッドファイルを `aidlc/active-memory/` に書き込みます。これはハーネスネイティブのメソッドの
インクルードが読み込む、git の無視対象のコピーです。追跡対象のファイルは変わりません。スペースを
作成したり、インテントを進めたりすることはありません。インテントの場合と同じく、動詞でも
スペースの正確な名前でもない単語は、動詞の一覧とスペース名とともにエージェントに渡されます。
名前の後に入力した言葉は切り替えとともに渡され、エージェントが対応します。

### `/aidlc space-create <name>` — Create a space

`memory/`、`knowledge/`、`codekb/`、
`intents/` を完全に備えた新しいチームスペースを作成します。ほかのチームが学んだプラクティスではなく、デフォルトスペースの `org.md` と
フレームワークのフェーズルールのコピー、そして空の `team.md` と `project.md` から始まります。
スペースは自動では切り替わりません。ワークスペースのモデル、
切り替えの例、何がコミットされるかについては [スペースとインテント](03-spaces-and-intents.md) を参照してください。

### `/aidlc knowledge <verb>` — Index and read your own documents

自分のドキュメント（PDF、Word や Excel のファイル、Markdown、プレーンテキスト、CSV）を
`aidlc/spaces/<space>/knowledge/documents/` の下に好きな構成で置き、インデックス化してください。
そうすると、エージェントは推測ではなくそれらを引用できます。

| コマンド | 内容 |
|---|---|
| `/aidlc knowledge onboard [path]` | プロジェクト内の任意の場所から 1 つのファイルまたはフォルダーを追加します（`documents/` の外にあるものは先にコピーされ、結果にコピー先が示されます）。パスを指定しない場合は、`documents/` の下でまだインデックス化されていないファイルをすべて取り込みます |
| `/aidlc knowledge sync` | カタログをディスク上の内容と突き合わせます。削除されたインデックスを再構築します |
| `/aidlc knowledge list [--json]` | カタログです。すべてのドキュメントとその状態を表示します |
| `/aidlc knowledge show <id>` | 1 つのドキュメントの完全な記録と、抽出したテキストを表示します |
| `/aidlc knowledge associate <id> --intent [slug]` | ドキュメントを 1 つのインテントにスコープします |
| `/aidlc knowledge dissociate <id> --intent [slug]` | そのスコープを解除します |
| `/aidlc knowledge rebind <id> --to <path>` | 元のファイルが移動*し、かつ*変更された行を修復します |
| `/aidlc knowledge summarize <id> --text-file <path> --source-revision <sha256> [--tags <csv>]` | LLM が書いた要約（と任意のタグ）を保存します。ツール自身がテキストを生成することはありません |

`--space <name>` はアクティブ以外のスペースを対象にします。`onboard` は冪等です。
変更のないファイルに対して再実行すると、2 つ目の行を書き込むのではなく `already` と報告するため、
一括取り込みは何度繰り返しても安全です。すでにインデックス化されたパスでファイルが**変更**されている場合は
`edited` と報告してその行をその場で更新するため、1 つのパスが有効な行を 2 つ持つことはありません。
結果は `fresh`、`already`、`edited` のいずれかです。「出力が変わらなかった」と「何も起こらなかった」は
別の結果なので、確認する価値があります。

**一括処理の上限。** パスを指定しない `onboard` と `sync` は、20 ドキュメント／256 MiB の
上限を、新規・変更・再試行の作業に適用します。すでに最新のカタログ行には適用しません。
すでに突き合わせ済みのカタログは、これより大きくてもかまいません。作業の一括処理が上限を超えた場合は、
該当するファイルを個別に onboard してから、もう一度 sync してください。上限に達した場合は何もインデックス化されないため、
拒否が中途半端な状態で終わることはありません。32 MiB を超える単一のドキュメントは、
まったく読まれずに拒否され、メッセージでもそう伝えられます。大きなファイルでは、「拒否された」と「読んでから
拒否された」とでコストが大きく異なるためです。

**スコープ。** `--intent` を省略すると、ドキュメントはスペース全体が対象になり、すべてのインテントから
見えます。`--intent` だけを指定するとアクティブなインテントを意味し、カーソルがない場合は推測せずに
失敗します。`--intent <slug>` は明示的に 1 つを指定し、slug に一致するインテントが
0 個または 2 個以上の場合は失敗します（完了したインテントの間では slug が重複することがあります。
保存される関連付けは常に UUID なので、slug の名前を変えてもドキュメントの参照先が変わることはありません）。
完了したインテントへのスコープは、`--allow-inactive` を付けない限り拒否されます。
これは、閉じた記録に後から証拠を補うためのものです。

**テキスト抽出**: Markdown、プレーンテキスト、CSV はそのまま読み込まれます。Word
（`.docx`）と Excel（`.xlsx`）のファイルは、何もインストールせずに AI-DLC 自身が読み込みます。
Word ファイルの段落と表、ブックのすべてのシートを表として読み込み、
各数式は最後に保存された結果を使います（日付と数値はファイルに保存されている形のままです。
画像、グラフ、コメント、ヘッダーは読み込みません）。プロジェクトは任意の種類に対して
独自の抽出ツールを設定でき、設定したものが優先されます。PDF には、設定がなければデフォルトの
抽出ツール（`pdftotext`）が使われます。それ以外の種類は `unsupported_type` としてカタログ化され、
引用できます。抽出ツールを設定した後は、`sync` を実行すると、その種類として検出された変更のない行が再試行されます。
AI-DLC が開けない Word や Excel のファイル（パスワードで保護されたもの、破損したもの、
古い `.doc` や `.xls` 形式を新しい拡張子で保存したもの）は、理由とともに `extraction_failed` としてカタログ化されます。
AI-DLC がそれらを読めるようになる前に追加された Word や
Excel のファイルは、次の `sync` でテキストが取得されます。「設定した」抽出ツールがインストールされていない場合、
ドキュメントは `extractor_unavailable` としてカタログ化されます。これは `list` で確認でき、
ツールをインストールして `/aidlc knowledge sync` を実行すると解消します。変更のない同じパスに対して `onboard` を
再実行すると `already` と報告され、抽出は再試行されません。この状態の行を再確認するのは
`sync` だけです。黙って飛ばされるものはありません。

**抽出には上限があります**: デフォルトの抽出ツールでは PDF は 50 ページ
（`pdftotext -l 50` の後、51 ページ目があるかを確認します）、抽出ツールの出力は 200,000
文字です（Word や Excel のファイルのテキストも同様）。上限を超えるとテキストは切り詰められ、行には
`truncated` が記録されます。`show` は内容の上に `truncated  yes` の行を表示し（ページの上限で切られた場合は `after
page 50`）、`--json` のペイロードでは `extraction` の中にフラグが入り、
ページで切られた場合は `pages_kept` も含まれます。切り詰められた
抽出結果は部分的なものとして扱ってください。そこから「ドキュメントは X に触れていない」と
結論づけるのは安全ではありません。

各抽出ツールの確認には 5 分の安全上限があり、抽出にはドキュメントごとに 15 分の上限があります。
設定した抽出ツールの `timeoutMs` は、抽出の時間枠を上書きします。
抽出がタイムアウトすると `extraction_failed` が記録されます。ドキュメントが正常に読み込まれたことには
なりません。これらの時間枠は、上記のバイト数、ページ数、出力の上限を変えません。

**pdftotext がない環境**（初期状態の Windows と macOS）では、AI-DLC が
何もインストールせずに自分で PDF を読み込みます（`show --json` ではその読み込み処理は
`builtin-pdf` と示されます）。`PATH` に pdftotext があれば引き続きそちらが先に実行され、
AI-DLC が自分で読み込んだ PDF は、pdftotext をインストールした後の次の `sync` で
pdftotext によって読み直されます。プロジェクトが独自に設定した PDF の抽出ツールが置き換えられることはありません。
それがない場合、ドキュメントは前述のとおり `extractor_unavailable` になります。AI-DLC の読み込み処理は
独立したプロセスとして、同じ抽出の時間枠と、同じ 50 ページ・
200,000 文字の上限のもとで実行されます。印刷やコピーだけが制限された PDF は読み込まれます。
開くのにパスワードが必要な PDF は、その理由とともに `extraction_failed` になります。
pdftotext のほうが優れている既知の制限: タイ語、ヒンディー語など、文字が結合する
文字体系の単語は、途中に空白が入ることがあります。AES-256 で暗号化された PDF や、
テキストが古い LZW や RunLength 方式で圧縮された PDF は、理由とともに `extraction_failed` になります。
フォントにテキストの対応付けがない PDF は、読めないテキストになります。スキャンした PDF には、どちらの読み込み処理でもテキストがありません（`no_extractable_text`。
OCR はありません）。

設定した抽出ツールの `argv` には、**ちょうど 1 つの `$IN`** が含まれていなければなりません。これはドキュメントの
パスが代入されるプレースホルダーです。これがない設定は、受け入れられずにツールの
起動時に拒否されます。そうしないと、ファイルを一度も受け取らないプロセスが、自分が出力したものを
そこに送られた*すべての*ドキュメントの抽出テキストとして記録してしまい、抽出が成功したように見えても実際にはそうでない状態になるからです。
`$IN` が複数ある場合も同じ理由で拒否されます。意図が曖昧なので、安全側に倒して失敗させます。

**`remove` は意図的に用意していません。** ドキュメントを削除するには、自分の
ファイルを削除してから `sync` を実行します。こうすることで、ツールがあなたのファイルに対して
破壊的な動詞を持つことはありません。削除された元ファイルは tombstone の行を残します。これは
意図的に削除されたというカタログの記録であり、リンク先の元ファイルに一時的に到達できないことを意味する
`source_unavailable` とは区別されます。

> **ドキュメントのテキストはデータであり、指示ではありません。** `show` は内容とともに
> この警告を表示します。顧客の契約書の中にある命令文は、その顧客の
> エンジニアに向けたものです。AI-DLC のワークフローの向きを変えたり、許可を与えたり、
> コマンドを承認したりすることは決してありません。

`/aidlc-knowledge` スキルは、同じ機能をコマンドとして入力する形です。

---

### `/aidlc --status` — Read-only status

何も変更せずに、現在のワークフローの進捗を表示します。

**構文:**

```
/aidlc --status
```

**動作:** アクティブなインテントの `aidlc-state.md` を読み、次の内容を表示します。スコープ（この作業用に組み立てたプランの場合は、代わりにその `Plan:` 行）、作業が新規プロジェクトか既存コードか（`Project Type: existing code (you said so)`）、既存コードの場合は Reverse Engineering が最後にスキャンした時期と、それが自動で実行されたかどうか（`, scanned 2026-10-03 23:17 UTC`）、深さとその出どころ（`Depth: Standard (from scope feature)`、または `set for this piece of work`。この作業用に組み立てたプランでは、ベースとなる既製スコープから来た設定は、深さ、Guard Policy 、各設定の行のいずれでも `(from the approved plan)` と表示されます）、異なる場合はテスト戦略、現在のフェーズ、現在のステージ、プランのステージのうち完了した数（プランの提案時と同じ数え方で、セットアップは含みません。スキップしたステージは数から除かれて名前が示され、`Completion: 4/4 stages (100%), skipped: Deployment Pipeline` となり、作業がそのステージで終わった場合は `Completed (Deployment Pipeline skipped)` と表示されます。一時停止中の作業はその旨と再開方法が `Paused at Deployment Pipeline (you stopped it); pick it back up with /aidlc --resume` のように表示されます）、レビュアーの設定とその出どころ（`Reviewers: advisory (from scope bugfix)`、またはこの作業でオフにした場合は `off (set for this piece of work)`）、インテントのGuard Policy の値とその出どころ（`Guard Policy: strict (from project.md)`、`off (from scope classic)`、`strict (set by you)`、またはこのフィールドがない古いインテントでは `strict (not set)`）、あなたまたは環境変数のスイッチでチェックをオフにした場合にだけ表示される `Checks off:` 行（それぞれの理由付き。`Checks off: state-transition (set by you)`。低いGuard Policy によってオフになるチェックはGuard Policy の行に含まれます）、ステージの進捗の一覧、そしてこのスペースでほかの作業がまだ開いている場合はそれを示す `Also open:` 行です。その作業への切り替えを頼むこともできます（`Also open: 261003-lunch-poll (ask to switch to it)`）。Guard Policy のフィールドが不正な場合は、検証エラーと修復コマンドとともに利用不可と表示されます。完了したステージの検証レシートも確認します。完了したステージの入力が承認後に変わっている場合は、次のステップが使う行でそれを伝え、やり直すかどうかを尋ねます（`Something Practices Discovery used changed after it finished. I'm carrying on with it as it is. Do you want me to redo Practices Discovery with the change?`、作業が既存コードになった場合は `Practices Discovery ran before the code was here. I'm carrying on with it as it is. Do you want me to redo Practices Discovery with the code?`、そのステージ自身のドキュメントだけが編集された場合は `requirements.md changed after Requirements Analysis finished. I'm carrying on with it as it is. Do you want me to redo Requirements Analysis with your change instead?`）。影響を受ける後続のステージも示します（`Also affected: ...`）。あなたの側は何も変わっていないものの、AI-DLC 自身のやり方が変わった完了済みのステージは、別の行で示され、遅れている扱いにはなりません（`AI-DLC now does Functional Design differently from when yours ran; your work stands, and /aidlc --stage functional-design goes back to it (the stages after it reopen too).`）。このチェックは助言的なもので、ルーティングは変えません。読めないレシートや、レシートのない完了は表示されません。現在のステージが承認待ちの場合、ステータスはいつから待っているかをわかりやすい時刻で示します（`waiting since 2026-10-04 12:24 UTC, about 2 minutes`）。アクティブなワークフローがない場合は、進行中のワークフローがないことを報告します。作業はあるもののこの会話でどれも選択されていない場合（たとえばチームメイトのクローン）は、各作業とその状況（完了済み、または現在のステージ）を示し、どれかを続ける方法や新しい作業を始める方法を伝えます。

ステータスには、**Sensors**、**Learnings**、**Summary Confirmation** の
行も別々に表示され、それぞれの有効な値と出どころが示されます。たとえば `Sensors: on (from
scope classic)`、`Learnings: off (set by a command)`、`Summary Confirmation: off
(set by you)`、`Summary Confirmation: off (from env
AIDLC_DISABLE_SUMMARY_CONFIRMATION)` のようになります。保存された設定がない場合は、
現在のスコープ、次に `on (from default)` にフォールバックします。後述の手順の制御を参照してください。

`Unit Ownership: team` の場合は、明確にラベル付けされた **Team Construction
Snapshot** を末尾に付け加えます。内容はスコープ外の main が表示するボードと同じで、ユニットの進捗、ローカルで
観測されたクレーム ref（オーナー、世代、プッシュ時刻ではなく観測された動き）、
固定したマージの準備状況、クレーム可能なユニット、ブロッカーです。スコープ付きと
スコープ外のチェックアウトは同じボードを表示します。このコマンドはフェッチせず、
状態、キャッシュ、監査も変更しません。明示的な `--space` と `--intent` のセレクターは、
ヘッダー、ユニットの DAG、クレーム、マージのジャーナルを、選択した同じ対象に結び付けます。
ボードの最後には、空いている作業や解放された作業のクレーム、固定したマージゲートの記録、
`aidlc unit land` の再開など、具体的な次のアクションが示されます。

---

以下の各ユニットコマンドは、`--help` で使い方を表示し、何も変更しません
（ターミナルで入力する `-h` も同様です: `aidlc unit share -h`）。受け付けない単語
（綴りを間違えたオプション、2 つ目のユニット名）は、何かがコミット、プッシュ、書き込みされる前に、
使い方とともに指摘されます。

### `/aidlc --claim <unit>` and `/aidlc unit claim <unit>` — Claim a team Unit

チーム所有でユニット優先の Construction ワークフローで、空いているユニットを 1 つアトミックにクレームします。
クレームのレジストリは git の ref `claim/<intent-id8>/<unit>` です。コマンドは
compare-and-swap のセマンティクスで一意のクレームコミットを書き込み、勝った
nonce を検証してから、gitignore の対象となるチェックアウトローカルのスコープスタンプを書き込みます。同時に
クレームした人のうち、成功するのはちょうど 1 人です。

**構文:**

```
/aidlc --claim user-profile-api
/aidlc --claim user-profile-api --team "Alice"
/aidlc --claim user-profile-api --rhythm unit-end
/aidlc unit claim user-profile-api --team "Alice"
```

`--team` は、人が読める保持者のラベルを指定します。`/aidlc --claim` では、
AI-DLC がクレームのために実行するコマンドが、書き込んだファイル（`--request-file`）でラベルを渡すため、
途中のシェルがそれを変えることはありません。`O'Brien` や `Ops & QA`
は、PowerShell、Git Bash、cmd.exe のいずれでも入力どおりに記録されます。`--rhythm` は任意で、
このクレームを `per-stage` または `unit-end` に固定します。省略すると、ワークフローで確定済みの
ユニットのゲートのリズムが使われます。依存関係と、必要なウォーキングスケルトンが完了するまでは
クレームは拒否されます。また、有効なクレームを持つチェックアウトは、スタンプされたユニットだけを
進めます。main が Units Generation に戻ってユニットのプランを変更している間も
クレームは進められ、その結果には、承認されたプランがそのユニット自身のエントリを変更した場合は
pin が main へのリベースを求めることが一度だけ示されます。

### `/aidlc unit adopt <unit>` — Adopt a teammate's live claim

新しいクローンで、ローカルのクレームブランチを正確にフェッチしてチェックアウトし、次を実行します。

```bash
git fetch origin refs/heads/claim/<intent-id8>/user-profile-api:refs/heads/claim/<intent-id8>/user-profile-api
git switch claim/<intent-id8>/user-profile-api
/aidlc unit adopt user-profile-api
```

引き継ぎでは、チェックアウトローカルのスコープスタンプを書き込む前に、チェックアウトしたクレームの OID とペイロードを
有効な ref と照合して検証します。対象はスペース、インテントの UUID、ユニット、世代、nonce、
結び付けられた監査シャードです。以降の監査の書き込みでは
クレームの既存のシャードが維持され、`publish` は同じ試行を続けます。
オーナーのローカルの監査記録はクレームとともに届き（publish の引き継ぎコミットが、
main が取り込む範囲の外で、オーナーのコミットの隣にそれを載せます）、このチェックアウトがすでに書き込んだ
行に合流します。そのため、利用者がすでに答えた質問は答えたままとなり、
もう一度尋ねることなく作業が再開されます。

### `/aidlc --release <unit>` and `/aidlc unit release <unit>` — Release a claim

スコープ外の main のチェックアウトから、有効なクレームを解放します。解放では ref を削除するのではなく、
世代を進める tombstone を公開します。そのため、古いスタンプの付いた
試行はクレームに関わる境界で安全側に倒れて失敗し、クレームの履歴は引き続き
確認できます。

```
/aidlc --release user-profile-api
/aidlc unit release user-profile-api
/aidlc unit release user-profile-api --expect-nonce <current-claim-nonce>
```

ユニットが解放されて再びクレームされた後の解放には、`aidlc-unit.ts status` から得た
`--expect-nonce` を含める必要があります。これにより、コマンドが後継の試行に結び付けられ、
出力が失われた後の再試行がその試行を tombstone にしてしまうことを防ぎます。

### `/aidlc unit participate` — Enable the guided picker

このクローンに、gitignore の対象となる参加者マーカーを書き込みます。以降、スコープ外の main で
`/aidlc` だけを実行すると、クレーム可能、クレーム済み、依存関係でブロック中の行を含む、型付きのユニット選択が表示されます。
共有された作業の新しいクローンでは、これは必要ありません。その `/aidlc` が、
取れるユニットを自動で提示します。マーカーのないファシリテーターの
チェックアウトには、代わりにターミナルでのファンアウトの案内が表示されます。

```
/aidlc unit participate
```

### `/aidlc unit share` - Share main with the teams

ユニットのクレームを持たないチーム作業のチェックアウトで、統合ブランチ上から
実行します。

```bash
/aidlc unit share
```

git がここで変更ありと示すものをコミットし（git が追跡していないファイルは含め、
無視対象のファイルと、`.gitignore`、`AGENTS.md`、
`opencode.json` のうち AI-DLC 自身の部分は除きます）、統合ブランチをリモートに
通常のプッシュで送ります。強制プッシュは決してしません。これで、チームがプランからユニットを取ったり、
取り込まれたユニットを確認したりできます。その後、このチェックアウトはユニットを構築する代わりに
ボードを表示します。リモートがプッシュを拒否した場合はコミットを残して理由を伝えます。
ブランチをプルしてから、もう一度 share してください。このステップが必要なときは `/aidlc` が自ら提案します。
リードのチェックアウトでユニットが始まろうとする前と、リモートにまだない取り込みの後です。

### `/aidlc unit publish <unit>` — Publish a completed candidate

スコープ付きのチームのチェックアウトで、成果物、ソース、状態の
ミラー、監査シャードをコミットした後に実行します。または `--commit` を付けると先にそれらをコミットします。ユニットが完了して承認されると、
`/aidlc` がこのステップを自ら提案します。`--commit`
はユニット自身の作業をコミットします。追跡対象のファイルへのすべての変更、
`aidlc/` の下の記録、ユニットのソースマニフェストに載っている新しいファイル（Code
Generation が書いたと示したもの）です。AI-DLC 自身のインストールや、
`.gitignore` に載っていない `node_modules` などのほかの新しいファイルは、コミットされずにディスクに残り、
publish は除外した数を一度だけ伝えます。`--include <path>` は新しいファイル
やフォルダーも含めます（1 つにつき 1 回指定するか、すべてなら `.` を指定します）。publish の後に
これを付けて実行すると、それらを含めて再び publish します。

```bash
/aidlc unit publish user-profile-api
/aidlc unit publish user-profile-api --commit
/aidlc unit publish user-profile-api --commit --include docs/diagrams --include README.md
```

このコマンドは追跡対象のファイルがクリーンな worktree を必要とし、有効なクレーム ref を、
クレームと実装の両方の履歴を保持した候補コミットに CAS で更新します。
`.gitignore`、`AGENTS.md`、`opencode.json` のうち AI-DLC 自身の部分（新しいクローンで
セッションが始まったときに書き込まれます）は、ここでも取り込み時でも数に入りません。
コミットされないまま残り、それ以外の変更があれば引き続きどちらも止まります。
ユニットの `construction/<unit>/` フォルダーの外にある記録への変更（たとえば `contract-summary.md` のような Inception
の成果物）も publish はされますが、pin はその候補を拒否します。そのため publish はそのファイルを示し、
ブランチから外して、代わりに main で変更し、もう一度 publish するよう伝えます。pin と land もそれを拒否し、
ユニットをクレームしたチェックアウトで外すよう伝えます。
クレームしたチェックアウトは、自分の監査行（セッション、ターン、回答）を、
インテントの記録の `.aidlc-engine/audit/` の下にあるローカルの記録に保持します。これは AI-DLC の
`.gitignore` によってコミットから除外され、クレームの追跡対象のシャードには、pin が受け入れる
ユニットの証拠だけが入ります。そのため候補はコミットそのままであり、
ユニットの取り込み後、クレームしたチェックアウトで main を普通にプルすると、
書き込んだすべての行を保持したまま fast-forward されます。

### `/aidlc unit pin <unit>` — Pin candidate evidence

スコープ外の main から実行します。

```bash
/aidlc unit pin user-profile-api
```

固定（pin）では、クレーム ref をフェッチし、その正確な OID／世代と新しい pin の
トランザクション ID を記録し、
成果物、ユニットのレシート、チームのゲート、レビュアーの判定、Plan Approval、状態、
監査シャードの受け渡しを、そのコミットから直接読み込みます。マージしたり、
worktree を作成したりはしません。クレームに結び付いたチームのシャードには、そのユニットの試行の
レシートだけを含められます。main が権限を持つ行、別のユニットの記録／レシートのパス、余分なシャード、
ほかのワークフロー記録のパスは拒否されます。pin はまた、候補のベースにあるそのユニットの
エントリ（種類と依存関係）と、アクティブなユニットごとのステージ列を、現在の main と比較します。
Construction の契約が変わっている場合は、リベースと再 publish が必要です。
プランがほかの場所で追加、削除、変更したユニットについては必要ありません。
ユニットの記録ツリーの外にあるプロダクトのソースのパスは、人によるマージゲートの証拠に
一覧表示されます。
クレームしたチェックアウトに置かれたルール（スペースの `memory/project.md` や
`memory/team.md`）もそこに一覧表示され、ユニットとともに取り込まれます。

pin と land は、依存するすべてのユニットが main に取り込まれるまで、
そのユニットを拒否します。

### `/aidlc unit gate <unit>` — Decide the pinned merge

```bash
/aidlc unit gate user-profile-api \
  --decision approve \
  --user-input "Approve pinned candidate"
```

受け付ける判断は `approve` と `reject` です。このコマンドは、pin の後に人が入力したターンが
続く場合に、利用者の回答を記録します（pin の出力には、回答ごとにこのコマンドが示されます）。利用者の言葉は
`--user-input` の代わりにファイルで渡すこともできます。`--request-file aidlc/.aidlc-request-text/<name>.txt` です。
これは AI-DLC がコマンドを表示するときに書き込むもので（pin の後、エージェントがあなたの返答を
pin のコマンドが示すファイルに書き込みます）、途中でシェルがそれを読むことはありません。
ファイルは判断が記録された時点で削除されます。固定したユニットは、pin がフェッチした統合ブランチに
通常のマージで取り込まれるため、レビューした OID は直接の親のまま残ります。ref が移動した、世代が変わった、または HOLD-MERGE マーカーがある場合は、
承認の前に明示的な再 pin が必要です。approve すると、出力に
land コマンド（`land`）が示され、エージェントが同じターンでそれを実行します。

ゲートの監査行には、ステージのゲートの行と同じく、pin の後に利用者が言ったことと、それが届いた
メッセージ（`Person Reply`、`Message Id`）が、エージェントの説明の隣に保持されます。pin の前にそのチャットで
入力された利用者自身の言葉の行が、まだどの判断にも使われていない場合、監査行は pin の後のメッセージを示し、
その言葉は記録に残します。人の存在を確認するガードがオフの場合は、どちらも保持しません。

### `/aidlc unit land <unit>` — Land the pinned transaction

```bash
/aidlc unit land user-profile-api --target main
```

取り込み（land）では、まず現在の統合ブランチをフェッチし、承認された証拠を、ユニットの DAG にあるユニットの
現在のエントリ（種類と依存関係）と、アクティブなユニットごとのステージ列に照らして再検証します。契約のずれがあると
Git を変更する前に拒否され、リベース、再 publish、再 pin、新しいマージ
ゲートが必要になります。その後、main が所有するエンジンのメタデータを保持しつつ固定した内容をマージし、ユニットの行を反映し、
受け渡された監査レシートを確定します。クラッシュからの復旧には、
冪等なステップを個別に実行します。

内容のポリシーは、候補と完全に一致することです。main と候補の両方が
共有ファイルを変更している場合、結果が固定した候補の blob と一致しない限り、
きれいな自動マージでもコミット前に拒否されます。チームのブランチを現在の
ターゲットにリベースし、そこで解決してから、新しい pin のために再 publish してください。

```bash
/aidlc unit land user-profile-api --step git
/aidlc unit land user-profile-api --step state
/aidlc unit land user-profile-api --step audit
/aidlc unit merge-status user-profile-api
```

クレームのレジストリが利用できない間、gate と land は安全側に倒れて失敗します。`--step git` が
レビュー済みのマージコミットを取り込んだ後になって、その正確なクレームの試行が解放された場合は、
そのコミットを確認して、例外的な完了を承認します。

```bash
/aidlc unit land user-profile-api --step state \
  --accept-released-attempt \
  --user-input "I inspected the landed commit and accept completing this tombstoned attempt"
```

このコマンドは、直前の版が固定した OID である直後の tombstone だけを受け入れ、
承認を main の監査とトランザクションジャーナルに記録し、
後継のクレームは拒否します。

### `/aidlc unit status` — Inspect Unit claims

現在の統合の状態とクレームのレジストリを読み、クレーム可能、
クレーム済み、待機中のユニットの集合を JSON で表示します。これはクレーム時／ステータス用の機能で、
設定された git リモートに接続することがあります。

チームモードでは、`## Way of Working` の下にある 1 行
`- **Integration branch**: <name>`（または単なる `Integration branch: <name>`）から統合ブランチを読み取ります。
`project.md`、次に `team.md`、次に `org.md` の順に探し、その行を持つ最初のファイルが優先されます。
ブランチの名前は問わず、周りの文章は読みません。Practices
Discovery は、あなたが確定したブランチについてこの行を `team.md` に書き込みます。手動で追加した行は
すぐに有効になります。AI-DLC の `org.md` には `main` が同梱されています。Way of Working に
文章はあるもののこの行がないファイル（この行ができる前に書かれたもの）は、
ブランチを必要とする最初のチームステップの前にエージェントが読みます。文章がブランチを示していれば
それを行として記録し
（`aidlc engine state practices-branch --file <file> --branch <name>`）、
示していなければ、示していないことを記録します（`--none`。AI-DLC 自身の git の無視対象の
記録に保持されるため、同じ文章が再び読まれることはありません）。その場合は、より広い範囲のファイルが答えになります。
あなたに尋ねるのは、1 つの文が 2 つのブランチを示している場合だけです。git が
使えない名前のブランチや、リモートにないブランチは、別の層の選択で置き換えられるのではなく、
それが書かれていたファイルとともに報告されます。

```
/aidlc unit status
```

---

### `/aidlc --config [section]` - In-session project configuration

ハーネスの会話を離れずに、`models`、`runtime`、`providers`、`trust`、`flags`、
`project` のいずれかを設定します。セクションを指定しない場合は、
どのセクションを検討したいかをコンダクターが尋ねます。セクションを指定した場合は、そのセクションに問題がなくても、
必ずそこで何を変更したいかを尋ね、そのセクションの選択肢と、変更しないという選択肢を提示します。

コンダクターは
`aidlc config <section> --show --json` で現在の状態を読み（`aidlc config --show --json` はすべての
セクションを一度に読みます）、対話的に変更内容を尋ね、
列挙できる選択肢にはネイティブの質問ピッカーを使います。「leave it」と言うと
そのセクションを飛ばします。受け入れられた変更はすべて、正確な 1 つの
`aidlc config <section> <explicit value flags> --yes` コマンドで反映され、コマンドと
その出力が表示されます。このエイリアスが値を作り出すことはなく、引数なしの
`aidlc config --yes` を実行することもありません。

これはターミナルでの設定作業です。変更が反映された後、またはあなたが
断った後、コンダクターは `next` の実行、前進、再開、
ワークフローステージの実行をせずに停止します。

---

### `/aidlc --doctor` — Health check

この実装の前提条件、設定、ステージグラフの整合性がすべて整っていることを検証します。
問題なし、または警告だけのレポートは終了コード 0 で、
失敗したチェックがあると 1 で終了します。完全なレポートはどの場合も stdout に書き出されるため、
オーケストレーターはいずれの場合もそれを表示します。コアの doctor のチェックは、あなたのファイルを変更しません。
ただし、セッション開始時と同じく、アクティブなスペースのメモリの AI-DLC 自身のコピー
（`aidlc/active-memory/`、Kiro IDE ではそのステアリングのコピー）は最新の状態に更新します。
そのため、このコマンドは最初のインテントを作成する前でも安全に実行できます。
インテントが存在すれば、`HEALTH_CHECKED` の監査行も記録します。プラグインのチェックは、インストール済みのプラグインのコードを実行します。プラグインの作者は慣例としてそれらのスクリプトを読み取り専用に保つことを求められますが、ランタイムはその性質を強制できません。

ワークフローに問題がある場合、`--doctor` は Project セクションに構造化された指摘も一覧表示します。未解決のゲート、ランタイムグラフの欠落、動いていないフック、そのほか「先に進まない」原因について、1 件ごとに `warn [<id>] <summary>` の行と `fix:` の行を示します（例: `gate-unresolved`、`runtime-graph-missing`）。実行時のレポートと `--export` は同じ分析を共有するため、指摘はどちらでも同じです。

**構文:**

```
/aidlc --doctor
```

**チェック内容:**

| チェック | 検証内容 |
|-------|-------------------|
| Prerequisites | 自己完結型のバイナリ、またはコピーインストールの場合は PATH 上の `bun` |
| Installed runtime | バイナリチャネルを使う場合の、アクティブなマシンのバージョンとインストール済みのハーネス配布物 |
| Project stamp | プロジェクトの配布物／バージョンを、プロジェクトを提供するエンジンと比較します。リリースの `.aidlc-version` が固定しているもの、なければ実行中のリリースです |
| Hook presence | `settings.json` が接続しているフレームワークのフックがすべて `.claude/hooks/` に存在すること。接続されているのに存在しないフックは明確に失敗します。AI-DLC のステータスラインは、あなたがオンにした場合に、自分の `.claude/settings.local.json` から確認されます。これは進捗を表示するだけなので、誰も接続していない `aidlc-statusline.ts` が、オフになっている強制機能として報告されることはありません。期待するフックの一覧を `settings.json` から取得するため、そこにフックを追加すると自動でチェック対象になります |
| Hooks enabled (Claude Code) | Claude Code の設定の各層（エンタープライズの管理ファイルとアルファベット順の `managed-settings.d/` の断片 → `.claude/settings.local.json` → `.claude/settings.json` → `~/.claude/settings.json`。優先順位が最も高い定義が勝ちます）を通じて解決された値が `disableAllHooks: true` でないこと。解決された値が `true` だと、存在するすべてのフックが黙って飛ばされるため、明確に失敗し、その層を示します |
| Project structure | `.claude/settings.json` が存在すること（ファイルの存在のみ。内容は検証しません） |
| Kiro hooks switched off | Kiro ハーネスで、`.kiro/hooks/` にある AI-DLC のフックファイル（`aidlc-*.json`）のうち、`"enabled": false` を持つものすべて。これは Kiro IDE の Agent Hooks でフックをオフにしたときに書き込まれます（コミットされたファイルが持っていることもあります）。ガードフック（`aidlc-guard-tool-call`）、返答の記録フック（`aidlc-record-human-turn`）、プロジェクトに残っている可能性のある古いガードファイルは失敗となり、それぞれ何が保護されなくなったかを示します。それ以外の AI-DLC のフックがオフになっている場合は 1 件の警告です。修正するには、Kiro IDE の Agent Hooks でフックをオンに戻すか、Kiro CLI ではフックのファイルの `"enabled": false` を `true` に変更し、Kiro を終了してからそのフォルダーで `kiro-cli --v3` を再び起動します |
| Kiro ignore sources | Markdown のコンダクター（`.kiro/agents/aidlc-aidlc.md`。Kiro IDE と Kiro CLI v3 が使用）を使う Kiro ハーネスで、git のグローバルな除外ファイル（git リポジトリ内の場合）、`~/.kiro/settings/kiroignore`、プロジェクトの `.gitignore`、`.kiroignore` を、エンジンがエージェントに `fs_read` で行わせる読み込みに対して個別に評価します。対象は、コンパイル済みのグラフで `harness.json` が選択する各ステージについて、ステージファイルと、コンダクターがインラインで保持するペルソナとナレッジ（Standard と Minimal の深さでのエンジン自身の一覧で、ディレクティブの 8 KiB の `inline_context_paths` 上限の範囲内）、さらに `stage-protocol.md` とその `stage-protocol-<name>.md` モジュール、各スキルの `SKILL.md` の隣にあるファイルです。`stage-definition.md` のようなコントリビューター専用のプロトコルファイルは読み込まれないため、数に入りません。プラグインは、どのように組み込まれたかにかかわらず数に入ります。`SKILL.md` ファイル、コンダクターエージェント（`agents/aidlc-aidlc.md`）、`aidlc-common/conductor.md`、そして `tools/`、`sensors/`、`hooks/`、`scopes/`、`steering/` は、`fs_read` ではなく Kiro 自身またはエンジンが読み込むため、数に入りません。それらの一部だけを隠すルールは、件数と、影響を受けるフレームワークのフォルダーとともに報告されます。`.kiro/` を隠すグローバルなソースのルールは、ソースと行を示して失敗となります。Kiro の `fs_read` ガード（Kiro IDE と Kiro CLI v3）が、すべてのステージ、エージェント、プロトコルの読み込みを拒否するようになるためです。ワークスペースのソースでの一致は警告になります。doctor にはどの Kiro が実行されるかわからないためです。Kiro IDE では、`kiroAgent.agentIgnoreFiles` がそのファイルを指定している場合にだけ適用されますが（デフォルトは `.gitignore` を含み、`[]` はワークスペースのソースを無効にします）、doctor はその IDE の設定を読めません。Kiro CLI v3 は `.kiroignore` を適用し（そこでの一致はそれらの読み込みを拒否します）、`.gitignore` は適用しません。警告ではどちらかを示します。doctor が評価できないソース（たとえば `git` が PATH にない場合。git リポジトリでは独自の `core.excludesFile` も見えなくなります。または、ディスク上に存在するリポジトリを git が拒否する場合）は、合格ではなく `not evaluated` として警告されます。行ではソースを `~/.config/git/ignore` や `.gitignore` のような固定の名前で示し、パス、ルールの文面、git のエラー文面で示すことはありません。別のファイルにある `!.kiro/` は、拒否を取り消しません |
| Workspace shell | `.claude/` と `aidlc/spaces/default/memory/` が存在すること（同梱のシェル） |
| Active space method copy | `aidlc/active-memory/`（すべてのハーネスのインクルードが読み込む、git の無視対象のコピー）が、アクティブなスペースの `memory/` ファイルを保持していること。セッション開始時と同じく、チェックは先にこれを書き込むため、差分が残っていれば、そこに書き込めないファイルがあるということです（警告） |
| VS Code agent request cap | Copilot のみ。`.vscode/settings.json` が `chat.agent.maxRequests` を 100 以上に設定していること。未設定（VS Code のデフォルトは 50）、それより小さい値、数値でない値（引用符で囲んだ数値を含む）、読めない場合は警告になります。VS Code が長いステージを止めて "Continue to iterate?" と尋ね、誰かが答えるまでチャットが待機するためです。修正内容には、ファイルに書く行が示されます |
| Submodules | `.gitmodules` がある場合、宣言されているサブモジュールのパスの数と、初期化されていない数を報告し、未初期化のものがあれば `git submodule update --init --recursive` を示します（助言のみで、失敗にはなりません） |
| Env scope | `AWS_AIDLC_DEFAULT_SCOPE`（設定されている場合）が有効なスコープを指していること |
| Synced folder | プロジェクトが同期フォルダーにある場合のみ。macOS の `~/Library/CloudStorage` の下にある Google Drive、OneDrive、Dropbox、Box のフォルダー、iCloud Drive（`~/Library/Mobile Documents`）、Windows の OneDrive フォルダー、または Dropbox 自身の `info.json` が示す Dropbox フォルダーです。同期によって AI-DLC がチェックするファイルが変わる可能性があること、プロジェクトを同期フォルダーの外に移すか、その同期を一時停止すべきことを伝えます。レビューや承認の後に変わったドキュメント、プラン、ソースに対する strict での拒否も、同じことを伝えます。それ以外の場合は何も表示しません（助言のみで、失敗にはなりません） |
| Hook heartbeats | `.aidlc-engine/hooks-health/` に、フックの実行時のタイムスタンプがあること（自分のタイムスタンプを持たない記録については、その記録を作成したターンのハートビートが、記録ができる直前に書き込まれたものとして、最初の 5 分間は数に入ります）。ハートビートがないことは、ワークフローが進む前は助言にとどまりますが、作業が進むと失敗になります。最新のハートビートが最新のステージ／ゲートのイベントより 5 分以上遅れている場合は停止として失敗となり、フックを動かすためのハーネス自身の手順が示されます（Claude Code では `.claude/settings.local.json` の `"disableAllHooks": false`、または組織のポリシー。`.claude/settings.json` がない、有効な JSON でない、AI-DLC のフックを 1 つも登録していない場合は、代わりにそのファイルの修復）。ハートビートは、それを書き込んだフックのあるクローンに残り、監査記録は git で移動します。そのため、進捗を示すのがほかのクローンの監査記録だけの場合（プルしたチームメイトの作業）は、代わりに警告 `This workflow was run in another clone (<machine>); this clone has no hook record for it yet` となり、このクローンで直すものはありません |
| Claude managed hook policy | Claude ハーネスのみ。既存の管理設定のリゾルバー（`AIDLC_MANAGED_SETTINGS_PATH`、現在および旧来の Windows のパス、macOS、Linux/WSL）とアルファベット順の `managed-settings.d/` の断片を使い、有効な `allowManagedHooksOnly` が `true` の場合に失敗します |
| Your replies are not being recorded | ステージ／ゲートのイベントがあるのに監査に `HUMAN_TURN` がない場合、利用者の返答が記録されていない（そのため次の承認は拒否される）ことを警告し、修正としてハーネス自身のフックの手順を示します |
| Hook drops | `.aidlc-engine/hooks-health/<hook>.drops` のテレメトリがあれば表示します。それぞれ、ツール呼び出しを壊さないためにフックが握りつぶした失敗の記録です。フックごとに、ドロップの件数、最後のタイムスタンプ、最も多い理由、対処方法（確認してからファイルを削除）を示します。最新の失敗が 24 時間以内のフックは、`--verbose` なしでも警告として表示され（`Hook failures, the latest within the last day`）、すべての失敗が数えられます。1 日後、またはファイルを削除すると解消します。各理由は最初のコロンまで、つまりフック自身の要約だけが、秘密情報を伏せた状態で表示されます。その後の詳細はファイルに残ります。それより古い失敗と、プラグインの組み込みフックからの `[advisory]` の行は、合格の助言の行になります。フックの通常の判断（先にあなたが答える必要があるため Stop フックがターンを終わらせる場合など）は、代わりに `<hook>.trace` に書かれ、数に入りません。失敗となるのは `[degraded]` のドロップ（プラグインの組み込みが中途半端に適用された場合）だけです |
| Workspace source boundary binds | ワークフローの状態が存在する場合のみ。Plan Approval がプランを結び付けるのと同じワークスペースのソースの走査を実行します。合格時はフィンガープリントの先頭 16 進 12 文字を示します。失敗時は理由コードとパスを示し（例: `budget-entries at .`、`dangling-symlink at linked/src`、`excluded-path at node_modules/pkg`）、現在の設定でプランの質問がどうなるかを伝え、修復の文面を示します。問題のパスを `.aidlc-source-paths.json` の除外リストに追加する、除外されたディレクトリの下にある実際のソースをそのパスのリストで宣言する、または壊れたシンボリックリンクを削除してから、`next` を実行します。Guard Policy が strict でプラン承認がオンの場合は、Plan Approval の判断が拒否されることを伝え、あなた自身の言葉がある場合に限る最後の手段を付け加えます。チャットでアシスタントにそれでもプランを構築するよう頼むと、プロジェクトのファイルを確認せずにあなたの承認として記録されます。それ以外の場合は、プランの質問は通常どおり尋ねられ（プラン承認がオフの場合は、プランは書かれたとおりに構築されます）、プロジェクトのファイルを確認せずに構築が始まることを伝えます |
| Current step out of date | その状態が続いている間のみ。アシスタントが作業の拠り所にしていたステップが古くなった場合です（チャットが圧縮された、またはステップの発行後にワークフローの状態が変わった）。いつ、なぜ古くなったか、わかる場合はどの状態の行が変わり、どの AI-DLC コマンドがそれを書いたかを伝えます。修正は単独のコマンドとして `next` を実行することで、現在のステップを改めて渡し、まだ一致している承認は保持します（警告のみで、失敗にはなりません） |
| State drift | アクティブなインテントの `aidlc-state.md` が、監査の最後の `WORKFLOW_COMPLETED` と一致していること |
| Pending approval | 現在のステージが本来の承認ゲートで 24 時間以上待っている場合、止まっているのではなく人を待っている状態だと示し、`/aidlc --status` を案内します（助言のみで、失敗にはなりません） |
| Background subagents | `aidlc/.aidlc-subagent-inflight` にある、セッション単位の新しいエントリと古いエントリを報告します。新しいエントリは助言にとどまり、古いエントリや形式が不正なエントリは、正確な削除の案内とともに失敗となります。ない場合は何も表示しません |
| Set-aside Bolt attempts | 保存された試行の参考情報の一覧です。slug、スタンプ、経過日数、モード（保存された head では `snapshot`、`branch-tip`、`legacy`。レビュー済みのソースの ref だけが残っている場合は `evidence-only`）、復元したチェックアウトの所有者、型付きの restore/purge 操作と、任意の安全な表示用コマンドまたは表示エラー（purge は evidence-only のエントリのみ）を示します。これらのエントリは警告でも失敗でもありません |
| Cycle detection | `stage-graph.json` に循環がないこと |
| Orphan stage files | グラフのすべての slug に、対応する `<phase>/<slug>.md` がディスク上にあること |
| Uncompiled stage files | slug がコンパイル済みのグラフにないステージの `.md` がディスク上にあれば表示します。プラグイン所有のファイルには `plugin sync` を、それ以外の作成されたステージには `aidlc-graph.ts compile` を示します（助言のみで、失敗にはなりません） |
| Plugin selection | 有効なプラグインの一覧、プラグインごとの有効なステージ数、グラフ全体での `enabled:false` フラグの一致、選択が中途半端な状態からの復旧のヒント |
| Plugin composition | オフラインでの、インストール済みと組み込み済みのバージョン／ハッシュの状態。sync や修復の対処方法を含みます |
| Composed plugin surface | 有効なプラグイン所有のステージファイルがコンパイルされていること。有効なプラグインのコントリビューションのサイドカーがすべて読み込み可能で有効であり、記録された対象ステージがすべて存在し、記録された構造的な追加や文章の断片がすべて残っていて変更されていないこと |
| Plugin checks | 有効なプラグインについてだけ、任意の `tools/<plugin>-doctor.ts` スクリプトを実行します。エラーの指摘は doctor を失敗にし、助言の指摘は終了コードを変えずに表示・エクスポートされます |
| Scope validation | 有効なすべてのスコープ（プラグイン選択後の `.claude/scopes/*.md` から）が問題なく走査できること（スコープの切り詰めによる欠落についての助言は想定内です） |
| Composed scope durability | コンポーザーが作成したすべてのスコープが、実際のプランに解決されること。グリッドの列がないスコープファイル（空の全 SKIP プランとして解決されてしまいます）、まだハーネスのツリーに投影されていない永続的な `aidlc/scopes/<name>.md` の記録、記録された `Scope` に解決可能な定義がない実行可能なワークフローで失敗します。compile で原因に対処できる場合は、対処方法として `aidlc-graph.ts compile` を示します。**裏付けとなる記録がない**列の欠落は別に報告されます。compile は、いずれかのステージが `scopes:` frontmatter で宣言しているスコープについてだけ列を出力するためです。その行では、記録を復元する、スコープのステージへのタグ付けを完了する、またはスコープファイルを削除する、のいずれかを示します。記録の説明用の frontmatter を、投影されたファイルと比較することはしません。グリッドは常に記録から作られ、compile は手動の編集を上書きせず、既存のスコープファイルをそのまま残すためです |
| Schema validation | すべてのステージの YAML frontmatter が `validateStageFrontmatter` に合格すること |
| Graph references | すべての `consumes[].artifact` と `requires_stage[]` の対象が解決できること |
| Duplicate producers | 消費されるすべての成果物の生成元が 1 つであること。生成元が複数ある場合は、そのステージの slug とともに報告され、グラフの読み込み順で最初のものに解決されます（助言のみで、失敗にはなりません） |
| Keyword overlap | 複数のプランが持つヒントは警告となり、そのプランが示されます（プランの提案では、それぞれのプランの下に表示されます） |
| Rule drift | 設定済みの組織ポリシーと重なる、現行のチーム／プロジェクトの見出しを、矛盾の確認のために表示します。ライフサイクル上古くなった重なりは、古いものとして抑制された別の行で報告します（助言のみで、失敗にはなりません） |
| Paired sensor coverage | 対になるセンサーを示すすべてのルールが、いずれかのステージで実際に実行されるセンサーに解決されることを確認します（助言のみで、失敗にはなりません） |
| Workspace records | 共有の記録が 1 つのチェックアウトにだけ残ることがないよう、`aidlc/` の下のコミットされていない変更を報告します。ただし `aidlc/.aidlc-request-text/` と、変更が doctor 自身の実行について書き込むエントリだけの監査ログは除きます（それだけの場合、この行は合格となり、その旨を示します）。git が依頼ファイルを追跡または表示している間、あるいはフォルダーが無視していない削除済みの依頼ファイルを保持している間は、`git add aidlc/` があなたの依頼まで取り込んでしまうため、コミットのコマンドは示しません（助言のみで、失敗にはなりません） |
| Request folder | git から `aidlc/.aidlc-request-text/` 内のファイルが見える場合のみ。そこにはあなた自身の依頼が入っているため、手順を示すだけで、実行はしません。フォルダーがまだ無視されていなければ、プロジェクトフォルダーの `.gitignore` に `aidlc/.aidlc-request-text/` の行を追加し（後に続く自分のルールがその無視を解除していないかも確認してください）、git がそれらを追跡している場合はプロジェクトフォルダーで `git rm --cached -r -f -- aidlc/.aidlc-request-text/` を実行し（ファイル自体は残ります）、削除と `.gitignore` の変更（更新によってコミットされないまま残っている場合があります）をコミットしてプッシュし、チームメイトのクローンからも消えるようにします。削除がまだコミットされていなければ、この行は残ります（助言のみで、失敗にはなりません） |
| Declared workspace repos | `repos.json` がある場合、宣言された集合を、実行時の検出がディスク上で見つけた兄弟リポジトリと比較します（助言のみで、失敗にはなりません） |
| Workspace gitignore | `repos.json` がある場合、管理対象の `.gitignore` ブロックが、宣言されたリポジトリの集合と一致していることを確認します（助言のみで、失敗にはなりません） |

**出力例:**

```
AI-DLC doctor

Machine
  warn  Runtime hook PATH: bun is on this shell's PATH (/home/user/.bun/bin/bun) but not on the system-wide PATH
        fix: Nothing needs changing when you start Claude Code from a terminal: it hands that terminal's PATH to AI-DLC's hooks. If you start it from a desktop icon, the dock, or a service and AI-DLC's hooks do not run, add /home/user/.bun/bin to the PATH line in /etc/environment, ENV_PATH in /etc/login.defs, or a PATH= line in ~/.config/environment.d/*.conf, then restart Claude Code. Editing .bashrc or .zshrc does not change this check.
  warn  Update: update check unavailable while offline
        fix: run `bun .claude/tools/aidlc.ts update --check`
  ok    4 checks passed

Project (.claude, Claude Code)
  warn  Instruction file: block or file missing (.claude/CLAUDE.md)
        fix: run `bun .claude/tools/aidlc.ts config`
  ok    43 checks passed

Framework integrity
  ok    all 12 checks passed

0 problems, 3 warnings.
Warnings are advisory - if everything works, ignore them.
Add --verbose to see every check.
```

`--verbose` を使うと、Machine、Project、グラフ、スキーマ、ステージ、スコープ、
センサーのすべての行が展開されます。警告や失敗には、必ず後に `fix:` のアクションが続きます。

---

### `/aidlc --doctor --export` — Write a diagnostic report

`--doctor` に `--export` を付けると、秘匿化した小さな診断レポートを書き出します。これにより、
プロジェクトのディレクトリ全体を共有せずに、うまく動かないワークフローをデバッグできます。
まず**新たに** doctor を実行し（レポートがキャッシュされた診断を反映することはありません）、
その後レポートを書き出します。レポートの書き出しが doctor の終了コードを変えることはありません。

**構文:**

```
/aidlc --doctor --export
/aidlc --doctor --export --output <dir>
```

`--output <dir>` は出力先を上書きします。デフォルトはプロジェクトの下の
`aidlc/diagnostics/` です。

**生成されるもの:** システムに `tar` があればタイムスタンプ付きの `.tar.gz` を生成します。
ない場合は、共有する前に自分で圧縮するよう案内を添えて、レポートのディレクトリを残します
（新しいパッケージ依存や独自のアーカイブ書き出し処理は追加しません）。
レポートには次の内容が含まれます。

| ファイル | 内容 |
|------|----------|
| `report.md` | 人が読めるワークフローのタイムラインと指摘 |
| `report.json` | 機械が読めるタイムライン、指摘、概要 |
| `manifest.json` | レポートのスキーマバージョン、AI-DLC のバージョン、ハーネス、ハッシュ化したインテント ID、ファイルごとの SHA-256 チェックサム、適用した秘匿化、切り詰めの通知、除外したものの一覧 |
| `evidence/normalized.json` | 許可リストに載った、正規化されたフィールドのみ。生のファイルは決して含みません |

**診断内容:** レポートは監査証跡からワークフローの**タイムライン**を再構成し
（ステージの所要時間、ゲート、改訂、空白期間、長時間・未完了・スキップのフラグ。スキップしたステージはスキップした時刻を Completed として示します）、
よくある「先に進まない」原因について、**決定論的な**条件→対処のルール（LLM は使いません）を実行します。
対象は、未解決の承認ゲート、状態と監査のずれ、ランタイムグラフの欠落／動いていないか止まったフックのハートビートです。
指摘は、実行時の `--doctor` が使うのと同じ共有の `DoctorFinding` モデルから得られるため、
コマンドとレポートが食い違うことはありません。復旧のための回避策
（たとえば `AIDLC_DISABLE_*` 環境変数や、"archive your workspace" という指示）を示す対処方法には、
必ず自動化するのは安全でないという印が付きます。

`DOCUMENT_INDEXED`/`DOCUMENT_UPDATED`/`DOCUMENT_REMOVED` は、スペースレベルの
監査シャードにあります。`--doctor --export` はそのシャードを明示的に読み込み、
アクティブなインテントのシャードと組み合わせます。そのため、ワークフローの権限を持つ読み込み処理はインテント単位のまま、
レポートにはワークフロー開始後のドキュメントの履歴が含まれます。`list` と
`show` は引き続き DocumentKB のカタログを直接読みます。

**安全性。** レポートには、ワークスペースのソース、生の状態／監査／
ランタイムグラフのファイル、成果物／コントリビューション／質問／メモリの本文、環境
変数、コマンドの出力は決して含まれません。出力される文字列はすべて秘匿化されます。ホームディレクトリは
`~` に、プロジェクトのルートは `<project>` になり、インテント ID とユニット名は
ハッシュ化され、秘密情報らしき値は消去されます。実際のパスがプロジェクトのルートの外に出る入力は
拒否され（シンボリックリンクの末端や親をたどってツリーの外に出ることはありません）、
ファイルごとのサイズと合計サイズには上限があり（切り詰めはマニフェストに記録されます）、
プラットフォームが対応していれば、ファイルは所有者だけが使える権限で作成されます。

**出力例:**

```
Diagnostic report created:
  aidlc/diagnostics/aidlc-diagnostic-report-20260714-153000-3f9a1c22.tar.gz

Findings:
  ERROR gate-unresolved
  WARNING runtime-graph-missing

No source files or artifact bodies were included.
```

---

### `/aidlc --stage <slug|#>` — Jump to stage

slug または番号を指定して、特定のステージに直接ジャンプします。

**構文:**

```
/aidlc --stage code-generation
/aidlc --stage 3.5
/aidlc --stage requirements-analysis
/aidlc --stage 2.3
```

**動作:** ワークフローがアクティブな場合は、対象のステージにジャンプします（間にあるステージは警告付きでスキップされます）。ジャンプと一緒に入力した `--scope`、`--depth`、`--test-strategy`、`--guard-policy` は先に適用され、変更後の作業の中でジャンプが行われます（`--review` や `--sensors` などのセレモニースイッチは、先にそれ単独で設定してください）。ワークフローがない場合は、`--scope` と組み合わせて、そのステージからワークフローを開始できます。

```
/aidlc --stage code-generation --scope bugfix
```

プランがスキップするステージでも、行き止まりにはなりません。そのステージが現在のステージより後にある場合、ジャンプはまずそのステージをプランに戻し（`recompose --add`。ジャンプを理由として記録されます）、それからジャンプします。その旨と元に戻す方法が 1 行で示されます。まだワークフローがない場合、そのステージはほかのジャンプ先と同じように実行され、スコープのプランには含まれないことが 1 行で示されます。そのステージが現在のステージより前にある場合や、現在のステージそのものである場合は、戻ると後続のステージをすべて再実行することになるため、ジャンプは拒否されます。その際、プランや進捗に触れずに今そのステージを実行する方法が示されます: `/aidlc --stage <slug> --single`。

---

### `/aidlc --stage <slug> --single` — Run one stage in isolation

`--single` を付けると、メインのワークフローを動かさずに、1 つのステージだけを単独で
実行できます。ステージは実行され、作業がそのステージのドキュメントを保存する場所に
ドキュメントを書き込んで停止します。ワークフローの `Current Stage` は決して進みません（慣習ではなく
エンジンがこれを強制します）。フルのライフサイクルに踏み込まずに、方法論の一部（要件分析や
リバースエンジニアリングのスキャン）だけを適用したいときに使います。
作業にそのステージのファイルがすでにある場合（承認済みのステージや、ユニットの
ステップ）、実行の前にそのコピーを作業の記録の `archive/<time>-<stage>/` に保存し、
その旨とフォルダーが 1 行で示されます。作業は実行で書き込まれた内容から続き、
コピーが元に戻す手段になります。実行のたびに個別のコピーが保存されます。
スコープがスキップするステージも実行できます。プランに含まれないことが 1 行で示され、
プランはそのまま変わりません。
単独実行でも、そのステージに設定されたエージェントとレビュアーは使われますが、
ワークフローの学びの処理やワークフローの承認は行われません。合成された
完了が監査ログに記録され、コマンドは停止します。
一緒に入力した `--depth` や `--test-strategy` はその実行にだけ適用され、ワークフローの
設定は変わりません。`--review`、`--guard-policy`、
`--guard.<check>` スイッチ、セレモニースイッチ（`--sensors` など）は
`--single` と一緒には使えず拒否されます。これらを付けずにステージを実行してください。

```
/aidlc --stage requirements-analysis --single
/aidlc --stage reverse-engineering --single
```

実行可能なステージにはそれぞれ、入力しやすい 1 語のランナー `/aidlc-<slug>` も用意されています。
これは `/aidlc --stage <slug> --single` をまとめたものです。ランナー全体（スコープ
ランナー、ステージランナー、`/aidlc-init`、セッションビュー）については
[スキルとランナーコマンド](17-skills.md) を参照してください。

---

### `/aidlc --phase <name|#>` — Jump to phase

特定のフェーズの最初のステージにジャンプします。

**構文:**

```
/aidlc --phase construction
/aidlc --phase 3
/aidlc --phase ideation
/aidlc --phase 1
```

**動作:** `--stage` と同じですが、指定したフェーズの最初のステージが対象になります。`--scope` と組み合わせられます。

---

### `/aidlc --scope <name>` — Change scope

実行中のワークフローのアクティブなスコープを変更します。

**構文:**

```
/aidlc --scope bugfix
/aidlc --scope enterprise
```

**動作:** `aidlc-state.md` のスコープ設定を更新し、実行するステージとスキップするステージを再計算して、`SCOPE_CHANGED` 監査イベントを記録します。`--depth`、`--test-strategy`、`--review`、`--guard-policy`、`--sensors`、`--learnings`、`--summary-confirmation`、`--guard.<fence>` スイッチと組み合わせられます。あなたが自分の言葉で頼んだ設定をエージェントが追加するときは、頼んだメッセージを示す `--message <id>` も追加し、その設定は `config set` と同じく `set by you` と表示されます。スコープ由来の Guard Policy は、より厳しい新しいデフォルトには自動的に従います。より低いデフォルトに従うのは、スコープの変更を頼んだメッセージをエージェントが指定した場合（`--message <id>`）です。そうでない場合、実行中のワークフローはポリシーを維持し、その旨が出力に 1 行で示されます。セレモニーの値は新しいスコープのデフォルトに従いますが、人が明示的に上書きした値や、古い作業で行がないものは維持されます。メモリでの strict 指定は引き続き実効ポリシーを決めます。明示したフラグは人による設定という出どころを保ち、`config-change` と同じ引き下げルールに従います。現在のスコープを選んだ場合でも、指定した設定は同じ設定適用処理で適用され、不要なスコープ変更イベントは発生しません。無効または不明なフラグや、許可されていない `--guard-policy relaxed` または `--guard-policy off` があると、CLI の更新全体が拒否されます。ステージのチェックボックスは、未解決の承認（`[?]`）や修正（`[R]`）を含めてそのまま引き継がれます。ただし、新しいスコープが承認待ちのステージをスキップする場合や、開始前の現在のステージをスキップする場合は例外で、それらのステージは変更とともにスキップされます（`[S]`。それぞれ `STAGE_SKIPPED` 行が 1 つと出力行が 1 行あり、単独で実行するための `/aidlc --stage <slug> --single` が示されます）。実行中の作業で、あなたを待っているものがない場合（ステージが単に実行中だった場合）、エージェントは同じターンのうちに続行し、新しいスコープがそのステージを実行しなくなったときは、新しいスコープが実行する次のステージに進みます。承認、質問、ユニットのチェックポイントや要約があなたを待っている場合はターンが終わり、そのステージが新しいプランに残っていて作業がユニット単位で進んでいなければ、作業がどこから再開するかをエージェントが伝えます。同じメッセージが、新しいプランでも実行されるステージであなたを待っている承認にも答えていて、作業が一時停止中でもユニット単位でもない場合（「承認して、poc プランに戻して」）、エージェントは切り替えの直後にその回答を記録するため、作業は新しいプランから続きます。Practices Discovery のゲートでは、変更依頼は同じように記録されますが、承認は、承認したプラクティスの保存を待つため、次のメッセージまで待ちます。完了済みの作業（`Status: Completed`）では、最後のステージの後にあるステージを実行する新しいスコープは、同じ書き込みの中でそのステージから作業を再開します（`Status: Running`、ステージは `[-]`、`STAGE_STARTED` 行が 1 つ、`intents.json` の行は `in-flight` に戻ります）。エージェントは同じターンのうちにそのステージに進み、切り替えの行と、新しいスコープが変えた設定を、ステージの最初の行とともに伝えます。以前に承認したものは承認済みのままで、`doctor` は再開した実行を、以前の完了からのずれではなく 1 つの実行として読みます。最後のステージの後に何も追加しない新しいスコープでは、作業は完了したままです。切り替えと一緒にスキップしたステージ（完了済みの poc の作業での `/aidlc --scope feature --skip ci-pipeline`）は、再開するステージが選ばれる前にプランから外されるため、作業はそれより後のスキップしていない最初のステージから再開します。エージェントは何をスキップしたかと、それを単独で実行する方法（`/aidlc --stage <slug> --single`）を伝えます。切り替えで追加されるステージをすべてスキップした場合、作業はそのスキップをプランに残したまま完了した状態のままです。Unit Ownership が solo でも team でも、進行中のユニットの作業を理由に変更が拒否されることはありません。新しいスコープが Units Generation をスキップする場合、Units Generation がすでに作成したユニットはユニットごとに続行し、新しいスコープが外すステージでのユニットの承認やユニットの作業は 1 行で示されます（例: "The beta Unit's NFR Requirements approval is no longer part of the plan."（beta ユニットの NFR Requirements の承認は、もうプランに含まれません））。逆に、作業が Units Generation を実行せずに通過した後で、Units Generation を実行する新しいスコープを選んでも、ユニットは作成されません。その Construction のステージは作業全体に対して 1 回実行され、未解決のゲートは作業とともに開いたままになり、Units Generation に戻る（`/aidlc --stage units-generation`）と作業がユニットに分割されます。

応答の最初の行には、新しいプランで作業が実行するステージ（完了済みのものと、現在のステージ以降のもの。新しいプランが現在のステージより前に追加したステージは、そこに戻った場合にだけ実行されます）、そのうち完了した数、承認ゲートが示されます。また、元に戻しても残りのステージが同じになる場合は、元に戻す方法（`/aidlc --scope <old scope>`）も示されます。作業に合わせて組み立てたプランの後では、元に戻しても同じにはならないため示されません。続いて、変更でスキップしたステージごとに 1 行、外したユニットの作業を示す行が 1 行、値が変わった設定ごとに 1 行が示されます。新しいスコープで変わらなかったデフォルトは示されませんが、スコープと一緒に入力した設定は、変わったかどうかにかかわらず必ず報告されます。

作業中のステージの要約を確認するのと同じ返信で、そのステージを含まないプランへの切り替えを入力した場合（「1. あと、これを express プランに切り替えてくれる？」）、エージェントはまず、確認された内容からそのステージのドキュメントを書き、そのステージのそれ以外のことは何もしません。その後、切り替えによって上記のとおりステージがスキップされ、新しいプランにそのステージが含まれないことが示されます。後続のステージはそのドキュメントを読みます。確認する前や、ドキュメントが書き込まれた後であれば、切り替えは上記のとおりに行われます。

自律的な Construction（`Construction Autonomy Mode: autonomous`。利用者が "Continue automatically" を選んだ場合）では、利用者が頼んだスコープの変更はほかの変更と同じように行われ、残りの作業は自動で実行され続けます。利用者の依頼が記録されていない変更や、`AIDLC_UNATTENDED=1` で実行された変更は拒否されます（新しいプランを承認する人がいないためです）。拒否の際には `bun .claude/tools/aidlc-bolt.ts set-autonomy --mode gated` が示されます。これは Bolt ごとに承認のために止まる方式に切り替えるもので、これにより変更を行えるようになります。`recompose`（`/aidlc --skip` や `--add`、または承認された compose の提案）も同じルールに従います。

まだワークフローがない新しいプロジェクトでは、`--scope <name>` は代わりにワークフローを開始します。動作は `/aidlc <name>` とまったく同じです。作るものと一緒に指定すると、指定したスコープでワークスペースが初期化され、ワークフローは最初のステージから始まります。スコープだけを指定すると、まず何のための作業かを尋ねます。

---

### `/aidlc --depth <level>` — Override depth

現在のワークフローまたは新しいワークフローの深さを上書きします。

**構文:**

```
/aidlc --depth minimal
/aidlc --depth standard
/aidlc --depth comprehensive
```

**動作:** ワークフローがアクティブな場合は、`aidlc-state.md` の Depth フィールドを更新し、`DEPTH_CHANGED` 監査イベントを記録します。`--scope` と組み合わせると、新しいスコープのデフォルトの深さを上書きします。`--stage` や `--phase` と組み合わせると、ジャンプ先の実行コンテキストの深さを設定します。作業がまだないチャットで入力した場合は、そのチャットが次に始める作業のために保持され、エージェントがその旨を 1 行で伝えます。

**有効な値:** `minimal`、`standard`、`comprehensive`（大文字と小文字は区別しません）。

**例:**

```
/aidlc --depth minimal                            Change depth of active workflow
/aidlc --scope bugfix --depth comprehensive        Bugfix with comprehensive analysis
/aidlc --stage code-generation --depth minimal     Jump with minimal depth
```

---

### `/aidlc --test-strategy <level>` — Override test strategy

深さとは独立して、テスト量の戦略を上書きします。

**構文:**

```
/aidlc --test-strategy minimal
/aidlc --test-strategy standard
/aidlc --test-strategy comprehensive
```

**動作:** 指定しない場合は、スコープが独自の上書きを宣言していない限り、現在の深さと同じレベルになります。独立して設定すると、Standard の深さ（成果物はすべて作成）と Minimal のテスト（Nyquist モデル）のような組み合わせが可能になります。`aidlc-state.md` の `Test Strategy` フィールドを更新し、`TEST_STRATEGY_CHANGED` 監査イベントを記録します。

**有効な値:** `minimal`、`standard`、`comprehensive`（大文字と小文字は区別しません）。

**テスト戦略のモデル:**
- **Minimal (Nyquist):** 要件ごとに 1 テスト、ハッピーパスを最低限、ユニットテストのみ（合計約 5〜15）
- **Standard:** コンポーネントごとに 5〜8 テスト、ユニット + 結合
- **Comprehensive:** コンポーネントごとに 10〜15 テスト、すべての種類のテスト

各レベルの詳細、デフォルトの決まり方、よくある組み合わせについては [スコープ、深さ、テスト戦略](05-scopes-and-depth.md#the-3-test-strategy-levels) を参照してください。

**例:**

```
/aidlc --test-strategy minimal                         Minimal testing for active workflow
/aidlc --depth standard --test-strategy minimal        Full artifacts, minimal tests
/aidlc --scope bugfix --test-strategy comprehensive    Bugfix with thorough testing
```

---

### `/aidlc --project-type <type>` - New project or existing code

ワークスペースのスキャンに任せずに、この作業が何であるかを指定します。

**構文:**

```
/aidlc --project-type brownfield "add the hover tooltip"   Start on existing code
/aidlc --project-type greenfield "scaffold the new service"   Start as a new project
/aidlc --project-type brownfield                            Mid-workflow: this is existing code
```

いつでも普通の言葉で伝えることもできます（「これは既存のコードで、フロントエンドは ui-repo にある」）。エージェントが同じコマンドを実行します。

**動作:** 開始時には、指定した種別がスキャンの判定に代わって使われます。言語、フレームワーク、ビルドシステムは引き続きスキャンが埋めます。ワークフローの途中では `aidlc engine workspace reclassify --project-type <type>` を実行します。これはフォルダーを再スキャンし、`Project Type` を設定し、`Project Type Source: you` を記録して、`## Workspace State` を更新します。既存コードの場合は、作業の開始後にフォルダーに追加されたリポジトリも記録し（まだ何も記録されておらず、Construction が始まっていない場合）、新規プロジェクトとしてのスキャンが除外した Reverse Engineering や、空のフォルダー向けに組み立てたプランが除外した Reverse Engineering を元に戻します（`infra` のように Reverse Engineering をまったく実行しないスコープでは、除外されたままです）。ワークフローがすでに Reverse Engineering を過ぎている場合は、Reverse Engineering が次に実行され、その後ワークフローは元のステージに戻ります。完了したステージは完了したままで、コードが判明する前に完了していたステージが応答に示されるため、必要ならやり直せます。新規プロジェクトの場合、完了していない Reverse Engineering は、承認ゲートで待っているものも含めてスキップされます。その質問はスキップとして閉じられ、書き込まれたドキュメントは残ります。Construction が始まった後はプランはそのままで、Reverse Engineering を単独で実行する方法が応答に示されます（今すぐコードをスキャンするようにも頼んだ場合は、エージェントが実行します）。種別が変わった場合、応答の最後に元に戻す方法が示されます。`WORKSPACE_RECLASSIFIED` を記録します。

スキャンで新規プロジェクトとして設定された作業で、Construction の前にフォルダーにコードが追加された場合、`next` はそれが既存のコードかどうかを一度だけ尋ねます。どちらの答えでも、種別はあなたが指定したものとして記録されるため、再び尋ねられることはありません。指定した種別はその作業にだけ有効で、次の作業ではフォルダーが再びスキャンされます。

**有効な値:** `greenfield`、`brownfield`（大文字と小文字は区別しません）。

---

### `/aidlc --review <class>` - Set stage reviews for this run

実行ごとのレビューの上書きを設定します。これは、アクティブなワークフローで §12a のステージ
レビューをどこまで重く実行するかの上限で、スコープの `review_cap` に代わって使われます。

**構文:**

```
/aidlc --review adversarial
/aidlc --review advisory
/aidlc --review none
```

**動作:** レビュアーを持つ各ステージは、frontmatter でレビュークラスを宣言します。
`adversarial`（レビュアーが成果物に反論し、リードが最大 `reviewer_max_iterations` 回のパスで
指摘を修正します）または `advisory`（通常フローでのレビューを 1 回行い、承認ゲートが
エンジン管理の指摘リストからその指摘を示し、あなたが取捨選択します）です。ステージごとの実効クラスは、ステージの宣言を
1 つの上限で引き下げたものです。上限は、この上書きが設定されていればそれ、そうでなければスコープの
`review_cap` です（bugfix、poc、classic、workshop は `advisory` が上限、express
は `none` が上限）。つまり `--review advisory` は、残りの adversarial のループをすべて
通常フローでの 1 回の意思決定支援パスに変え、`--review none` はゲートのある
ステージのレビュアーの起動をスキップし、`--review adversarial` は、上限のあるスコープでも
各ステージ自身のクラスで実行します。スコープ自身のレベルを設定すると（たとえば
bugfix での `--review advisory` や、feature での `--review adversarial`）、代わりに上書きが
解除されます。スコープの上限が再び適用され、以降のスコープの変更にも追従します。
どの上書きも、ステージの宣言を超えてクラスを引き上げたり、ステージが宣言していない
レビュアーを追加したりはしません。
上書きがない場合、classic はゲートのあるフローで、レビュアーを持つステージごとに advisory の
パスを 1 回実行し、指摘を承認ゲートで示します。明示的な自律 Construction は
例外で、classic のもとでも、マージ前の 1 回のレビュアーを維持します。スコープの上限もセレモニースイッチも、そのレビューを無効にはしません。
`aidlc-state.md` の `Review Override` フィールドを更新し、
`REVIEW_CLASS_CHANGED` 監査イベントを記録します。ワークフローの作成時や
`--scope` と一緒に指定できます。現在と同じスコープを指定した場合は、レビューの
上書きを破棄せず、設定変更として適用します。どちらのクラスでも、後の
出力の書き込みによって最終レシートが無効になった場合は、次の序数で 1 回だけ、
上限付きの回復リクエストが許可されます。

**有効な値:** `adversarial`、`advisory`、`none`（大文字と小文字は区別しません）。

**例:**

```
/aidlc --review advisory              Single normal-flow pass, findings at the gate
/aidlc --review none                  No gated stage reviews this run
/aidlc --review adversarial           Each stage's own review class, above any scope cap
/aidlc --review advisory              On bugfix: back to bugfix's normal reviews
```

---

### Workflow configuration — one atomic setter

12 個のインテント設定すべて（`guard.plan-approval` は `plan-approval` の別名のため、キーは 13 個）が、1 つの CLI セッター `config-change` を共有します。
異なる設定のフラグを 1 つのアトミックな CLI コマンドにまとめられます。一緒に指定する設定を、
続けて呼ぶ別々のセッターに分けないでください。
入力された引き下げのコマンドについては、エージェントがスイッチと、それに伴う
インテント設定を 1 つのトランザクションでこのセッターに渡して実行します。フックはメッセージを記録するだけです。
これはアクティブなインテントの設定であり、`aidlc config flags` によるネイティブの
プロジェクト設定とは別のものです。

| 設定キー / スラッシュフラグ | 値 | 状態フィールド |
|-------------------------|--------|-------------|
| `depth` / `--depth` | `minimal`, `standard`, `comprehensive` | Depth |
| `test-strategy` / `--test-strategy` | `minimal`, `standard`, `comprehensive` | Test Strategy |
| `review` / `--review` | `adversarial`, `advisory`, `none` | Review Override |
| `guard-policy` / `--guard-policy` | `strict`, `relaxed`, `off` | Guard Policy |
| `sensors` / `--sensors` | `on`, `off` | Sensors |
| `learnings` / `--learnings` | `on`, `off` | Learnings |
| `summary-confirmation` / `--summary-confirmation` | `on`, `off` | Summary Confirmation |
| `plan-approval` / `--plan-approval` | `on`, `off` | Plan Approval |
| `collaborators` / `--collaborators` | `on`, `off` | Collaborators |
| `guard.plan-approval` / `--guard.plan-approval` | `on`, `off` | Plan Approval（`plan-approval` の別名） |
| `guard.review-freeze` / `--guard.review-freeze` | `on`, `off` | Guards Off / Guards On |
| `guard.state-transition` / `--guard.state-transition` | `on`, `off` | Guards Off / Guards On |
| `guard.reviewer-scope` / `--guard.reviewer-scope` | `on`, `off` | Guards Off / Guards On |

廃止されたキー `change-control` と廃止されたフラグ `--change-control` は、1 リリースの間は
引き続き `guard-policy` として解決され、非推奨の旨が 1 行表示されます。
`config-change` と `scope-change` のユーティリティ経路では、1 つのコマンドで両方の綴りを指定すると、
値が異なる場合は拒否され、一致する場合は受け付けられます。
`orchestrate next` では、両方の値が有効なら、2 つのフラグのうち後のものが優先されます。
`validate-grid` では、順序にかかわらず `--guard-policy` が `--change-control` より
優先され、選ばれたフラグの最初の出現が使われます。
どちらのフラグもスコープオブジェクトを上書きします。スコープオブジェクトでは、文字列の `guardPolicy` が
文字列の `changeControl` より優先されます。

以下の各行は設定を組み合わせたものです。エージェントが実行するセッターは、
引き下げも含め、指定されたインテント設定をすべてまとめて適用します。

```
/aidlc --depth minimal --review none --guard-policy relaxed --sensors off
/aidlc config set depth standard --test-strategy minimal --review advisory --guard-policy strict --sensors on --learnings on --summary-confirmation off
/aidlc --scope bugfix --guard-policy relaxed --sensors off --learnings on
```

ネイティブのディスパッチャーでの形式は、`aidlc engine config set <key> <value>` の後に
ほかの設定フラグを続けたものです。ターミナルで入力する場合は、`aidlc config set <key> <value>`、
`aidlc config get <key>`、`aidlc config list` で、`engine` なしで同じ動詞に
届きます。どのキーも同じユーティリティコマンドに送られ、
最初の設定が `--<key> <value>` になり、残りのフラグはそのまま渡されます。
あなたが自分の言葉で頼んだことのためにエージェントがこれを実行するときは、
`--message <id>`（それを頼んだあなたのメッセージの ID）を追加します。エンジンは、
そのメッセージがあなたのものであること、このチャットを通じて届いたこと、その後の決定で使い切られていないこと
（同じメッセージが行った 1 回の承認や回答では、使い切られたことになりません）を確認します。
確認できれば、設定は `set by you` と表示され、オフにした設定は
監査行にあなたのメッセージ全体を `Person Reply` として保持します。確認できないメッセージは拒否され、
拒否の際にはその理由が示されます。`--message` がない場合、設定が `set by you` と表示されるのは、
あなたが自分の行で入力した場合か、自分のターミナルでコマンドを実行した場合だけで、
それ以外は `set by a command` と表示されます（どちらの場合も適用されます）。ただし、
引き下げ（チェックのオフ、プラン承認や要約確認のオフ、Guard Policy の
`relaxed` または `off`）は、代わりに拒否されます。`--person-words '<words>'` と
`--person-words-file <file>` は引き続き受け付けられますが、何の証明にもなりません。エンジンは
その言葉をエージェント自身のテキスト（`Conductor Summary`）として保持し、
設定は `set by a command` と表示されます。

```bash
aidlc engine config set guard-policy relaxed --sensors off --intent login-fix --space platform
bun .claude/tools/aidlc-utility.ts config-change --depth minimal --review none --guard-policy relaxed --sensors off --intent login-fix --space platform --project-dir /work/shop
```

`config-change` が受け付けるのは、13 個の設定フラグ（と廃止された `--change-control` の綴り）、`--message`、
`--person-words` または `--person-words-file`（引き続き受け付けますが、何の証明にもならず、エージェント自身のテキストとして保持されます）、
そして `--intent`、`--space`、`--project-dir` のセレクターだけです。設定が少なくとも 1 つ必要です。
セレクターは、状態ファイル、メモリのポリシー、監査シャードを同じ対象に固定します。
インテントやスペースのセレクターを省略すると、アクティブなワークフローの選択が使われます。セレクターは
アクティブなインテントやスペースを切り替えません。スコープも変更する場合は、`config-change --scope` ではなく
`scope-change`（`/aidlc --scope` の経路）を使ってください。

入力された引き下げのスイッチについては、
`/aidlc config set guard-policy relaxed --intent <name> --space <name>` と
`/aidlc --guard-policy relaxed --intent <name> --space <name> ...` のどちらも、
エージェントのセッターが適用する対象の作業を指定します。
フラグ形式の末尾の `...` は、省略可能なタスクの説明を表します。
セレクターを省略すると、セッションのワークフローの選択が使われます。存在しないインテントを
指定すると拒否され、作業が選択されていない場合、セッターはその設定を、
チャットが次に取り上げるか開始する作業のために保持します。既存の作業を選ぶと（選択の
質問、`/aidlc intent <name>`、または作業が 1 つだけのスペースへの切り替え）、
そこに設定が適用され、作業を示す行の後に一度だけその旨が示されます。
セッターは、変更の前に、認識したすべての付随するインテント設定を検証します。
形式の誤ったコマンド、不明なフラグ、値の欠落、付随する設定の無効な値があると、
何も変更されず、CLI の経路は通常の検証エラーを報告します。

CLI は、状態を変更する前にすべてのフラグと値を検証します。無効な値や
不明なフラグがあると、その更新全体が拒否され、不明なフラグはエラーで示されます。
明示的な `--guard-policy relaxed` または `--guard-policy off` が、メモリ
レイヤーの `Mode: strict` によって拒否された場合、付随する設定もスコープの変更も
適用されません。エラーでは、
編集すべきメモリファイルが示されます。明示的な strict や無関係な設定は引き続き許可されます。
1 つのロックで、対象の状態の読み取り、すべての設定の適用、
監査バッチの追記、状態の 1 回の書き込みをまとめて行います。監査に失敗した場合、
状態は変更されません。変更と出力は表のキーの順序に従います。`Last
Updated` は、保存された状態が変わったときにだけ変わります。すでに保存されている
選択を繰り返しても何も起きませんが、スコープ由来の値を明示的な上書きに変えた場合は、
値が同じでもその出どころが記録されます。

`config get` は表のすべてのキーを受け付け、`config list` は 13 個すべてを
その順序で返します。Guard Policy、フェンス、セレモニーの読み取りには、status と同じく
実効値と出どころが含まれます。

```
/aidlc config get guard-policy
/aidlc config get plan-approval
/aidlc config get summary-confirmation
/aidlc config list
/aidlc config list --json
```

追加の読み取り専用の `guard.human-presence` の参照は、`on (default)` または
`off (env AIDLC_SKIP_HUMAN_PRESENCE_GUARD)` を報告します。これは作業ごとの設定ではなく、
`config list` には含まれません。

ネイティブの読み取りに相当するのは `aidlc engine config get <key>` と
`aidlc engine config list` です。`aidlc engine config --help` は config の動詞を
すべて一覧表示します。以下のセクションでは、この同じセッターが管理する Guard Policy、
フェンス、セレモニーのポリシーについて説明します。

#### `/aidlc --guard-policy <value>` - Guard Policy for this piece of work

インテントの Guard Policy の値を設定します。これは、目の前の作業に対して、フレームワークのガードが
どこまで退くかを表します。決めるのは 2 つのことです。人がすでに承認または確認したものが、
知らないうちに変わっていたときにどうなるか（コードプランの承認後にソースファイルが変わった、
レビュー後にレビュー済みのドキュメントが編集された、現在の要約確認なしに出力が保存された）と、
実行中のワークフローのどのステップも求めていない操作に対して、5 つのフェンスのうちどれが有効のままかです。

**構文:**

```
/aidlc --guard-policy strict
/aidlc --guard-policy relaxed
/aidlc --guard-policy off
```

**動作:** 入力が変わった場合、`strict` は承認をやり直します。実行は、何が変わったかを
わかりやすい文で示して止まり、改めて承認を求めます。
`relaxed` と `off` は、変更を `CHANGE_ACCEPTED` 監査行として一度記録し、
1 行で伝えて続行します。コードプランを承認した後にコードが変わった場合は
例外で、ビルドはその行と監査行を出して続行し、`strict` のもとで再び承認を求めるのは、
プランやそのテスト手順が編集された場合だけです。`strict` のもとでのユニットの並列
ビルドは、あなた自身の変更（編集、コミット、`git pull`）に対しても再び承認を求める唯一のケースで、
それを元に戻すか `relaxed` を設定するまで、次のユニットの land を止めます（
[Guard Policy](13-customization.md#guard-policy) を参照）。フェンスについては、`strict` は 5 つすべてを有効のままにし、
`relaxed` は `plan-approval`（プランの再承認: 編集されたプランについて再び尋ねません）と
`review-freeze` を下げ、`off` はその 2 つに加えて
`state-transition` と `reviewer-scope` も下げます。`human-presence` は、ポリシーの値によって
下がることはありません。

チャットから `guard-policy relaxed` や `guard-policy off` を設定するのは、利用者が
決めることです。利用者が `/aidlc --guard-policy relaxed` と入力すると（その値にするには `off` を使います）、
コンダクターの最初の `next` がセッターを示し、コンダクターは同じターンのうちにそれを
実行します。セッターは状態の行と監査行を書き込み、チャットで頼まれたとおりに何が変わったかを
伝えます。フックはメッセージを記録するだけです。
利用者が普通の言葉で頼んだ場合、コンダクターは頼まれた値で
`config-change --guard-policy <strict|relaxed|off>` を実行します。これは、最後の決定以降の
利用者の返信が記録されていれば引き下げを行い、出力をそのまま表示して止まります。
この Kiro IDE のビルドはプロンプトのテキストを渡さないとハーネスが示している場合は、
そのビルドではアクティブな作業の引き下げはできないことを伝えてください（そのアダプターは引き下げの
セッターも拒否します）。Kiro IDE を更新するか、デフォルトの低いスコープで新しい作業を始めてください。
config 形式でもフラグ先頭の形式でも、`--intent <name>` と `--space <name>` で
対象の作業を選びます。セレクターを省略すると、フックのペイロードのセッションの
ワークフローの選択が使われます。
存在しないインテントを指定すると拒否されます。作業がまだないときに入力した Guard Policy の `relaxed` または `off`
のスイッチは、このチャットが次に始める作業のために保持されます: `Guard Policy relaxed for the piece of work you start now (set by you).`
ほかに取り上げられる作業がある場合は、チャットが次に取り上げるか開始する作業に
適用されます（`... for the work you pick up or start next (set by you).`）。
後から入力した `strict` は、それより前にオフにしたチェックを元に戻します。
依頼と一緒に入力した場合は、その依頼に適用されます
（`Guard Policy relaxed for the work you are asking for (set by you).`）。新しい作業は
作成時にそれを受け取り、未完了の作業を続ける場合はそこに適用されます。メッセージだけで
未完了の作業が変わることはありません。`off` の形式では、`relaxed` の代わりに `off` が示されます。
フェンスのスイッチ（`guard.<fence> off`）も同じように動作します。
作業がまだないときは
`The review freeze check is off for the piece of work you start now (set by you).`、
依頼と一緒に入力したときは
`The review freeze check is off for the work you are asking for (set by you).` となります。唯一の例外は、コードプランの質問が
開いていて、利用者がプランのファイルを編集していないときです。この場合、言葉と一緒に
入力した設定はこの作業に適用され、その言葉はその質問への返信になります
（`/aidlc --guard-policy off approve the plan`）。引用符で囲まれていない `--` の後の言葉は、
引き続き新しい作業の説明になります。
フックは Windows でも動作するため、入力したスイッチは、セッター側でのセッションの参照なしに
プロンプトを渡すすべてのハーネスで機能します。
無関係な返信では何も開かず、`AIDLC_UNATTENDED=1` はプロンプト時の
適用を抑止し、CLI での引き下げを拒否します。
スコープのデフォルトは確認なしで適用されます。
すでにオフのフェンスや、すでに `set by you` となっている同じポリシーの値は、
CLI の更新が何もしないため、キーを必要としません。
メモリでの strict と無人実行のチェックの後、CLI のセッターが利用者の
プロンプトなしに引き下げるのは、次の 2 つの場合だけです。あなたが自分のターミナルで入力した場合（両端が
ターミナルで、コマンドにチャットの識別情報がなく、エージェントのためにターミナルを開くホストの
しるしもない場合）。これはあなた自身の行為です。もう 1 つは、`fenceKeyBypassed` が
テスト用のフィクスチャやハーネス起動時のプレゼンスのバイパスを認識した場合で、インラインの環境変数の
代入は含みません。
セッション開始フックは、`AIDLC_SKIP_HUMAN_PRESENCE_GUARD=1` で起動された有人のハーネスのために、
`presence-bypass-<session>` スタンプを Plan
Approval のランタイムディレクトリに保持します。
モデルのツールは、フックを呼び出したり、`aidlc/.aidlc-sessions/` や、
`.aidlc-plan-approval/`、`<record>/.aidlc-engine/gate-words/` の各ディレクトリに書き込んだりできません。これは
[状態遷移ガード](../reference/06-hooks-and-tools.md#pretooluse-aidlc-state-transition-guardts) によって強制されます。
メモリで保持された strict は最初に拒否し、ポリシーの値と、以前に下げたフェンスの
両方を上書きします。`/aidlc --status` では、マシン全体のキルスイッチが優先される場合を除き、
`on (guard policy strict (from <layer>.md))` と表示されます。

入力された `config set` の引き下げのスイッチは、`config set <key> <value>` の後に、
省略可能なほかの設定フラグ（`--depth minimal --sensors off`）と、
`--intent <name>` と `--space <name>` の組を受け付けます。不明なフラグや無効な
値があるとコマンド全体が拒否され、その名前が示されます。
[カスタマイズ](13-customization.md#the-five-fences) を参照してください。
Codex では `$aidlc` を使い、拒否のメッセージでも `/aidlc` の代わりに `$aidlc` が示されます。

利用者が頼んだメッセージ（`--message <id>`）なしに実行されたセッターがポリシーを `relaxed` に
変えようとすると、次のように拒否されます。

> Setting Guard Policy relaxed lowers fences, which is the person's call. Run it with --message <id>, the message they asked in, when they ask for it.

直接の `scope change --guard-policy relaxed|off` にも同じルールが適用されます。チャットからの直接の
`intent create --guard-policy relaxed|off` は、利用者がこのチャットでそれを頼んでいれば、
すでに進行中の作業に対するセッターと同じように実行されます。利用者のメッセージがその根拠となり、
フラグが利用者自身が入力した行にある場合（またはその人のターミナルで実行された場合）、新しい作業には
`Guard Policy: <value> (set by you)` が記録され、エージェントが追加した場合は `(set by a command)` が記録されます。作業がまだない段階では、そのメッセージは
フックが届いたときのまま保持したものです。作業ができた後は、最後の決定以降の利用者の
ターンです。利用者が自分のターミナルで実行するコマンドは、それ自体が根拠になります。作業がまだないときに利用者が入力した場合や、新しい作業と同じメッセージで入力した場合、Guard Policy の `relaxed` または `off` は、このチャットが次に始める作業のために保持され、その依頼に応えます。そのための `intent create --request <id>` は、フラグの有無にかかわらず `Guard Policy: <value> (set by you)` を記録し、未完了の作業はそれぞれのポリシーを維持します。作成時にスコープ自身のデフォルトを指定すると、別のプロンプトなしにスコープの
値が記録されます。実行中のワークフローが、デフォルトのより低いスコープに移る場合、
利用者がスコープの変更を頼んでいればその値を受け取ります。そうでなければ、
より厳しいポリシーを維持し、その旨を 1 行で伝えます。このチャットで利用者のメッセージが
記録されていない場合、作成は次のように拒否されます。

> Setting Guard Policy relaxed lowers fences, which is the person's call. Run it with --message <id>, the message they asked in, when they ask for it.

`off` の場合の拒否メッセージでは、`relaxed` の代わりに `off` が使われます。無人実行では、
ドライバー向けの案内も表示されます。利用者がガードからの回復の選択肢 `lower-fence`
を選ぶと、エージェントが代わりに `config set guard.<fence> off` を実行し、何が変わったかを 1 行で
伝えます。利用者が自分で入力する必要はありません。

compose による作成では、プランが実行されるスコープから Guard Policy を読み取り、
スコープファイルは書き込みません。既製のスコープに一致するプランは、そのスコープのデフォルトか、
あなたが頼んだより厳しい値を維持し、カスタムプランは、デフォルトが承認された値以下である
既製のスコープの上で実行されます。
コンダクターは `strict` または `relaxed` の場合に `--guard-policy` を渡し、これによって低いスコープの
デフォルトが引き上げられます。`off` の場合は決して渡しません。compose のゲートで、一致したプランを
既製のデフォルトより低い値に切り替えると、コンポーザーはそれを編集として扱います。
提案はその値を持つ既製のスコープ上のカスタムプランになり、
インテントはそこから作成されます。デフォルトより高い値への切り替えでは、プランは一致したままです。
後からあなたが入力することは何も残りません。
コンポーザーが進行中のインテントの値を変えることはありません。

どの値でもゲートはなくなりません。コンダクターは必ずすべての承認の質問をしなければならず、
フェンスを下げても、この散文上の義務が強制されなくなるわけではありません。レビュアーの判定が
変えられることも、証拠が削除されることもなく、エージェントが人の代わりに答えることも
決してありません。下げたフェンスを通過するたびに `GUARD_STOOD_ASIDE` 行が 1 つ書き込まれます。
その 1 行の通知がどう届くかはハーネスによって異なります。
[カスタマイズ](13-customization.md#what-you-see-when-a-guard-decides) を参照してください。

共有の `config-change --guard-policy <value>` セッター（入力されたスイッチや普通の言葉での依頼に対して
エージェントが実行します。人のターンのフックはメッセージを記録するだけです）は、
`aidlc-state.md` の `Guard Policy` 行を
`<value> (set by you)`（利用者が頼んだことを示す記録がない場合は `(set by a command)`）に書き換え、
監査バッチに `GUARD_POLICY_SET` 行を追加します。
ポリシーの行を書き込むと、廃止された `Change Control` 行は削除され、`Guard Policy` だけが残ります。
両方の行があり、ポリシーの値が異なる場合、メモリが strict を保持していない限り、次の `next` が
どちらかを残すまで、status には `strict (from conflicting state lines)` と表示されます
（[値の保存場所](13-customization.md#where-the-value-lives) を参照）。
両方が一致する場合は `Guard Policy` 行が使われ、次のポリシーの書き込みで
廃止された行が削除されます。廃止された relaxed または off の行だけを持つ記録は、
次のメッセージのときに名前が変えられます。値は維持されて `Guard Policy` として書き込まれ、
ポリシーの監査行は作られません。それがまだ行われていない場合は、すべての `/aidlc` の実行で
通知が表示されます。`/aidlc config set guard-policy relaxed` で値を維持するか、
`/aidlc config set guard-policy strict` でフェンスを上げると、どちらでも通知は止まります。
この通知を表示しても行は書き換えられず、廃止された strict の行だけの場合は通知されません。値は
インテントとともにコミットされ、セッションをまたいで保持され、チームメイトにも見えます。
同じセッターが、不正な行を修復し、古いテキストを記録します。「ファイルが変わったときに再承認を求めるのをやめて」
のような、普通のチャットでのポリシーの依頼は、
コンダクターが実行します。コンダクターは頼まれた値でセッターを実行し、最後の決定以降の
あなたの返信が記録されていれば引き下げを行います。
入力されたスイッチ（たとえば `/aidlc --guard-policy relaxed`）も同じセッターで、
コンダクターが最初の `next` から実行します。
設定とスコープの変更では、監査行の `Old Value` は、以前に保存されたインテントの値です
（不正な場合は生のテキスト、行がなかった場合は `strict`）。メモリを反映した実効値ではありません。
管理対象のチェックポイントの観測では、引き続き実効値の新旧が記録されます。
この行がない古いインテントは、設定されるまで strict のままです。新しいインテントは、
スコープのデフォルトから始まるか、メモリレイヤーに `Mode: relaxed` または
`Mode: off` が設定されていればそこから始まります。メモリレイヤーの `## Guard Policy`
セクションが `Mode: strict` を指定している場合、明示的な `relaxed` や `off` は、
ほかに指定した設定も含めてコマンド全体を拒否し、そのファイルを示します。全員に対して緩めるには、
そこにあるメモリの行を編集してください。明示的な strict の設定は引き続き
許可されます。作成時には、スコープと一緒にフラグを指定できます
（`/aidlc --scope poc --guard-policy strict "..."`）。

廃止されたフラグ `--change-control` と廃止された設定キー `change-control` は、
1 リリースの間は引き続き解決され、次のマイナーバージョンで削除されることを示す非推奨の行が 1 行表示されます。
`config-change` と `scope-change` の経路では、
1 つのコマンドで両方の綴りを渡すと、値が異なる場合は拒否され、
一致する場合は受け付けられます。`orchestrate next` では、2 つのうち後のものが優先されます。

**有効な値:** `strict`、`relaxed`、`off`。

**例:**

```
/aidlc --guard-policy relaxed        Record and announce input changes, keep going
/aidlc --guard-policy strict         Approve again whenever an approved input changes
/aidlc --guard-policy off            Lower the four lowerable fences too (each pass-through logged)
```

#### `/aidlc config set guard.<fence> <on|off>` - one fence, one piece of work

ポリシーの値には触れずに、目の前の作業で 1 つのフェンスだけをオフにします。また、ポリシーの値が
そのフェンスを下げている場合でも、オンに戻せます。作業ごとの 3 つの
スイッチは `review-freeze`、`state-transition`、`reviewer-scope` です。
`guard.plan-approval` は `plan-approval` 設定の別名なので（
[プラン承認](#aidlc-plan-approval-plan-approval) を参照）、`Guards Off` のエントリーは書き込みません。human presence
は鍵の持ち主であり、作業ごとのスイッチはありません。

**構文:**

```
/aidlc config set guard.review-freeze off
/aidlc config set guard.review-freeze on
```

**動作:** フェンスをオフにすると、`aidlc-state.md` に `- **Guards Off**: <comma list> (set
by you)` が書き込まれ、`Guard`、`Scope`、`Source` を持つ `GUARD_DISABLED` 監査行が 1 つ
書き込まれます。オンに戻すと、そのリストから外され、
`GUARD_RESTORED` が書き込まれます。`on` を設定すると、ポリシーによって下げられたフェンスが上がり、
`- **Guards On**: <comma list> (set by you)` に記録され、同じフィールドを持つ `GUARD_RESTORED` が
書き込まれます。Guard Policy の値を自分で設定すると、それと矛盾する行がクリアされ（同じコマンドで
指定したフェンスは除きます）、変わったフェンスごとに同じ監査行が書き込まれます。`off` は `Guards On` を、`strict` は `Guards Off` を
クリアし、`relaxed` はどちらもクリアしません。その後の
単一フェンスのスイッチは引き続き適用されます。すでに有効な設定を繰り返しても何も起きず、その旨が示されます。
ポリシーによって下げられたフェンスに `on` を設定するのは、何もしない操作ではありません。どちらの状態の行も
human presence は受け付けず、保存された human presence のエントリーは無視されます。
`/aidlc --status` は、あなたまたは環境変数のスイッチがオフにしたチェックと、その理由を
`Checks off:` 行に示します。低い Guard Policy によってオフに
なるチェックは、ポリシーの出どころを示す `Guard Policy:` 行に含まれます。優先順位は、環境変数のキル
スイッチ、メモリが strict を保持していなければ作業ごとのオフ、作業ごとのオン、Guard Policy の値、デフォルトのオン
の順です。5 つのうち 4 つにはキルスイッチがありますが、`state-transition` にはないため、
制御できるのはポリシーの値とこのスイッチだけです。

チャットから `guard.<fence> off` を設定するには、利用者が
`/aidlc config set guard.<fence> off` または `/aidlc --guard.<fence> off` と入力し、
エージェントが同じターンのうちにそのセッターを実行します。セッターは監査行を記録し、
何が変わったかを伝えます。
自分の言葉で頼んだり、ガードの選択肢 `lower-fence` を選んだりしてもかまいません。
エージェントがそのフェンスのセッターを実行し、何が変わったかを 1 行で伝えます。
どちらの形式も `--intent <name>` と `--space <name>` を受け付けます。セレクターを省略すると、
フックのペイロードのセッションのワークフローの選択が使われます。
存在しないインテントを指定すると拒否されます。状態ファイルがまだない場合、スイッチは
このチャットが次に始める作業のために保持されます。依頼と一緒に入力した場合
（`/aidlc --guard.<fence> off <description>`）は、その依頼に適用され、
未完了の作業が変わることはありません。
CLI のセッターは、最後の決定以降の利用者のターンが記録されている場合にフェンスを下げます
（空の台帳は数えません）。すでにオフのフェンスでは何も起きず、
キーも必要ありません。
フックは Windows でも動作するため、プロンプトを渡すすべてのハーネスが、
セッター側でのセッションの参照なしに、入力されたスイッチに対応します。
`AIDLC_UNATTENDED=1` はプロンプト時の適用を抑止し、CLI での引き下げを拒否します。
メモリでの strict と無人実行のチェックの後、利用者のプロンプトなしに CLI での引き下げを許可するのは、
`fenceKeyBypassed` がテスト用のフィクスチャやハーネス起動時のプレゼンスのバイパスを認識した場合だけです。
インラインの環境変数の代入では、これは成立しません。
メモリで保持された strict は最初に拒否し、以前に下げたフェンスを上書きします。
`/aidlc --status` では、マシン全体のキルスイッチが優先される場合を除き、`on (guard policy strict (from <layer>.md))` と
表示されます。保存された `Guards Off` のエントリーは
残り、メモリの行が strict でなくなった後にだけ、再び有効になります。

review-freeze のフェンスをオフにするセッターを、あなたが頼んだメッセージを指定せずに
エージェントがあなたの代わりに実行すると、次のように拒否されます。

> Turning the review-freeze check off is the person's call. Run it with --message <id>, the message they asked in, when they ask for it.

ほかのフェンスの拒否メッセージでは、そのフェンスの名前に置き換わります。無人実行では、
ドライバー向けの案内も表示されます。あなたが自分のターミナルで入力した同じコマンドは、
この理由で拒否されることはありません。それはあなた自身の行為なので実行され、1 行で
伝えられ（"The review freeze check (it stops edits to work you already
approved) is off for this piece of work, set by you. You can turn it back on any
time."（review freeze のチェック（承認済みの作業への編集を止めるもの）は、この作業ではオフになりました。あなたが設定しました。いつでもオンに戻せます。））、あなたが設定したものとして記録されます。エディターが自身の
エージェントのために開くターミナル（VS Code の Copilot、Kiro IDE、Cursor）は、あなたのターミナルではありません。その場合の
拒否メッセージは "To turn the review-freeze check off, ask for it in your Kiro
chat."（review-freeze のチェックをオフにするには、Kiro のチャットで頼んでください。）となり、あなたのツールの名前が示されます。このコマンドは、ポリシーの値が有効のまま
残しているものも含め、切り替え可能な 3 つのフェンスを制御します。切り替え可能なフェンスのメインセッションでの
拒否はコマンドを示し、human presence の拒否はスイッチを示しません。この作業でプロンプトの
フックが動作している間（ハートビートが新しい間）は、利用者がまだ答えていないことを示し、エージェントは
ターンを終えます。利用者に伝えることは何もありません。そうでない場合は、利用者がすでに送った返信が
どうなったかを示します。利用者が操作した後にだけフックを実行するハーネスでは、フックを
有効にする手順を示し、それ以外では利用者向けに 1 行を示します: "Your answer didn't reach
AI-DLC. Please give it once more. If
it happens again, type /aidlc --doctor."（回答が AI-DLC に届きませんでした。もう一度送ってください。再び起きた場合は /aidlc --doctor と入力してください。）Kiro CLI では、`/agent` で選んだ
別のエージェントに入力した返信は決して記録されないため、その行は "Your
answer didn't reach AI-DLC. Type /agent and pick aidlc, then give it once
more." で始まり、aidlc がその一覧にない場合（v3 エンジンの Kiro CLI で、
エージェントファイルがアップグレードされていない場合）に備えて、Kiro 自身のアップグレード
（`/upgrade-agent` でワークスペースのエージェントを選び、再び `kiro-cli --v3` を実行）、
または Kiro への切り替えと `kiro-cli --v2 --agent aidlc` での起動が続き、最後に ", and
give it there." となります。
エージェントは、利用者の代わりにチェックをオフにすると申し出ないよう指示されています。

利用者の返信の後にエージェントが記録した質問に対する `log answer` は、
ほかの回答がすでにその返信を記録していない限り、その返信に基づいて記録されます。利用者が
再び尋ねられることはありません。

human presence を下げられるのは `AIDLC_SKIP_HUMAN_PRESENCE_GUARD=1` だけで、マシン全体で設定するか、
`aidlc config flags --bypass` で記録します（その場合、AI-DLC はオフであることを伝えます）。
`AIDLC_UNATTENDED=1` は、これとは別に、人のターンの発行を保留します。フェンスを下げるわけでは
ありません。`guard.human-presence` を設定しようとすると、更新全体が拒否され、
0 以外の終了コードとともに次のメッセージが表示されます。
`Human presence cannot be switched off: it is how AIDLC knows an approval or an answer came from a real person, so reply in the chat yourself. For a supervised session where nobody can reply, launch the CLI with AIDLC_SKIP_HUMAN_PRESENCE_GUARD=1 set.`

**有効な値:** `on`、`off`。

#### `/aidlc --sensors`, `--learnings`, `--summary-confirmation` — Ceremony controls

アクティブなインテントについて、次の 3 つの独立したポリシーを `on` または `off` に設定します。

```
/aidlc --sensors off
/aidlc --learnings on
/aidlc --summary-confirmation off
```

| フラグ / 設定キー | 状態とステータスの行 | `off` でスキップされるもの |
|-------------------|----------------------|------------------|
| `--sensors` / `sensors` | Sensors | センサーの自動起動とブロッキングセンサーのチェック。診断用に明示的な `sensor fire` は引き続き使えます |
| `--learnings` / `learnings` | Learnings | 学びの日誌と学びのゲートのセレモニー |
| `--summary-confirmation` / `summary-confirmation` | Summary Confirmation | ステージの frontmatter が宣言する、統合された要約の `Looks correct` チェックポイントだけ。intent-capture の Assumption Confirmation は、必須の質問やステージの承認と同じく、別個の人による判断として残ります |

**デフォルトと優先順位:** `1` に設定されたキルスイッチは、そのポリシーを強制的に `off` にします。
そうでない場合は、インテントごとの明示的な設定が優先され、次に現在のスコープのデフォルト、
スコープに設定がない場合は `on` となります。要するに、**環境変数 → インテントごと →
スコープ → on** です。出荷されているスコープはすべて、3 つすべてを明示的に宣言しています。classic は
センサーと学びを `on`、要約確認を `off` に、bugfix は
センサーを `on`、学びと要約確認を `off` に、express は 3 つすべてを
`off` に、ほかの 8 つは 3 つすべてを `on` に設定しています。キーを省略したスコープファイルでは、
引き続き `on` にフォールバックします。新しいインテントは、
スコープのデフォルトを、たとえば Sensors なら `on (from scope classic)` のように保存します。
スコープを変更すると、スコープ由来の値は新しいデフォルトに引き継がれ、
明示的な値（`set by you` または `set by a
command`）は維持されます。これらのフィールドがない古いインテントは、スコープから解決され、それもなければ `on` になります。
スコープの選択と一緒に指定した明示的なセレモニーのフラグは、スコープ由来のデフォルトではなく、
明示的な上書きとして書き込まれます。
これらのフラグは、互いに組み合わせることも、深さ、テスト戦略、
レビュー、Guard Policy、フェンスのスイッチと 1 つの設定トランザクションで組み合わせることもでき、
スコープの変更の有無は問いません。
単独の `--single` 実行は、合成されたステージ開始イベントに記録された、選択したスコープのポリシーを
完了まで使い、メインのインテントのセレモニーの上書きは引き継ぎません。未完了の単独実行の試行は、
別のスコープでは再開できません。試行を完了するか、記録されたスコープで再開してください。
スコープが記録されていない古い単独実行の開始では、要約確認が維持され、
このスコープの比較は強制されません。

明示的なセレモニーの設定は、対応する状態の行に `<value> (set by a command)` を書き込みます。
ただし、セッターが利用者の入力したスイッチを適用する場合や、利用者がチャットで頼んだことを実行し、
エージェントが頼んだメッセージを指定する場合（`config set <key> <value> --message
<id>`。たとえば「ここからはリードエージェントだけで」の後）は、`<value> (set by you)` を書き込みます。そのメッセージがこのチャットからの利用者のものであることはエンジンが証明します。
さらに、共有の監査バッチに `CEREMONY_SET` 行を追加します。この行には `Key`、`Old`、`New`、`Source`、`Person Reply`（利用者の
メッセージ全体）、そして利用者がオフを頼んだ場合は `Message Id` が含まれます。
`Old` は以前に保存された値です（不正な場合は生のテキスト、行がなかった場合はスコープのデフォルト）。
環境変数のキルスイッチによって強制的にオフにされた値ではありません。
監査のキーは `sensors`、`learnings`、`summary_confirmation` です。コマンドは
`Source: command` を記録し、利用者が入力したスイッチは `Source: you` を記録します。
利用者自身の選択を繰り返すコマンドは何もせず、`set by
you` を維持します。`intent-create --learnings off` のような作成時のフラグも
`set by a command` を記録します。ただし、利用者がチャットで依頼と一緒にその設定を入力した場合
（`/aidlc --learnings off <request>`、または作業がまだないとき）は、
`set by you` を記録します。保存された上書きはインテントとともにコミットされ、
セッションをまたいで保持されます。環境変数のキルスイッチは、保存された選択を上書きせずに
優先されます。セレモニーをオフにしても、フックがアンインストールされたり
削除されたりすることはなく、必須のステージのゲートが外されることも、明示的に自律 Construction を選んだときに使われる
マージ前の 1 回のレビュアーが無効になることもありません。

要約確認をオフにすると、利用者の `Looks correct`
チェックポイントがなくなるため、フェンスと同じルールに従います。利用者が
`/aidlc config set summary-confirmation off` または `/aidlc --summary-confirmation
off` と入力し、エージェントがそのセッターを実行します。説明と一緒に指定した場合（`/aidlc
--summary-confirmation off build the export`）、フラグは新しい作業のものとなり、
その作成時に適用されるため、すでに進行中の作業はチェックポイントを維持します。利用者が自分の言葉で頼んだ場合は、エージェントがセッター（`config
set`、`config-change`、`scope-change`）を実行します。利用者が頼んだメッセージを指定せずに
実行すると、次のように拒否されます。

> Turning summary confirmation off skips the person's `Looks correct` check before a stage writes its output, so it is their call. Run it with --message <id>, the message they asked in, when they ask for it.

明示的な選択としてすでに保存されているオフでは、何も起きません。スコープによるオフ（たとえば
`off (from scope classic)`）でも利用者が必要です。明示的な値として保存すると、
後のスコープの変更の後まで残ってしまうためです。`on` にすること、スコープ自身の
デフォルト、作成時のフラグには、利用者の入力したターンは必要ありません。オフにするというコンポーザーの提案は、
それを含むプランを利用者が承認した時点で利用者のものとなり、
コンダクター自身がセッターを実行します。フェンスと同じく、`AIDLC_UNATTENDED=1` は変更を拒否し、
利用者なしに許可するのは、テスト用のフィクスチャやハーネス起動時のプレゼンスのバイパスだけです。

設定コマンドでは、深さ、テスト戦略、レビュー、Guard Policy、フェンスのスイッチと並んで、
この同じ 3 つのキーを扱えます。たとえば、次のコマンドは
3 つのセレモニーすべてと Guard Policy をまとめて設定します。

```
/aidlc config set guard-policy relaxed --sensors off --learnings on --summary-confirmation off
```

**環境変数のキルスイッチ:**

| 変数 | 値がちょうど `1` のときに強制的に `off` になるポリシー |
|----------|---------------------------------------------------|
| `AIDLC_DISABLE_SENSORS` | Sensors |
| `AIDLC_DISABLE_LEARNINGS` | Learnings |
| `AIDLC_DISABLE_SUMMARY_CONFIRMATION` | Summary Confirmation |
| `AIDLC_DISABLE_PLAN_APPROVAL_GUARD` | Plan Approval（メモリの Guard Policy の strict ロックよりも優先） |

それ以外の値では、ポリシーは強制的にオフにはなりません。これらのスイッチは、
ネイティブの config のバイパスのインターフェースで明示的に記録することもできます。

```bash
aidlc config flags --bypass AIDLC_DISABLE_SENSORS --local --yes
aidlc config flags --bypass AIDLC_DISABLE_LEARNINGS --local --yes
aidlc config flags --bypass AIDLC_DISABLE_SUMMARY_CONFIRMATION --local --yes
aidlc config flags --show
```

記録したスイッチをプロジェクトで共有するには、`--local` の代わりに `--project` を使います。
実際の環境変数は、記録された config のフラグより優先されます。
スイッチの変数が `1` 以外の値に設定されている場所（チェックが実行されるエディターや CLI）では、
記録したスイッチでそのチェックはオフになりません。`config flags --bypass` を実行する場所で
その変数が設定されている場合、コマンドは、センサー、学び、使用量の記録以外の各チェックについて、
チェックがオフになったと伝える代わりにその旨を伝えます。
スイッチの記録やクリアで変わるのはその設定ファイル（と、どのように設定されたかを示す、AI-DLC の
gitignore 対象のメモ
`aidlc/.aidlc-sessions/recorded-switches.json`）だけで、ハーネスのファイルは更新されません。
そのため、ワークフローの実行中でも機能し、次のチェックが再起動なしにそれを読み取ります。
AI-DLC が管理する `.gitignore` のブロックには、すでに
`aidlc.settings.local.json` が含まれています。それより前のインストールでは、最初の `--local`
の記録で、`.gitignore` を編集する代わりに、クローン自身の `.git/info/exclude` によって
そのファイルを git の対象から外します。どちらの設定ファイルもあなたのコードとは見なされないため、
Code Generation の途中で記録しても、ステージの完了が止まることはありません。
コマンドは何も尋ねず（`--yes` は省略可能）、複数のハーネスがあるプロジェクトでも `--harness` は不要で、
記録またはクリアした内容を、それを元に戻すコマンドとともに表示します。`--local`、`--project`、`--global` の
いずれもない場合、`--bypass` はあなた自身の `aidlc.settings.local.json` に記録され、
自分のターミナルで入力した `--clear-bypass` は、そのスイッチを記録しているすべてのファイルからクリアします。
チャットからエージェントが実行した場合、同じクリアは、セッターと同じ方法でチェックをオンに戻します
（後述）。あなた自身のエントリーはクリアされ、プロジェクトの共有の
`aidlc.settings.json` とこのマシンのファイルは、名前が示されるだけで変更されることはありません。スイッチは、
いずれかのファイルが記録している間はオンです。無人実行（`AIDLC_UNATTENDED=1`）では、
頼む人がいないため `--bypass` は記録されません。`--clear-bypass` は常に
実行されます。チャットでエージェントが実行する場合、あなたからチェックを取り去る 9 つのチェック
（後述）のいずれかの `--bypass` は、あなたが頼んだときにだけ記録されます。
これは、チェックをオフにするすべてのセッターが従うルールによるものです。エージェントは、あなたが頼んだ
メッセージを `--message <id>` で指定し、エンジンがそれを証明します。作業がまだないときも同じです。
そうでない場合は何も記録せず、それはあなたが決めることだと伝えます。たとえば "Turning the
human presence check off is the person's call. Run it with --message <id>, the
message they asked in, when they ask for it."（human presence のチェックをオフにするのは利用者が決めることです。利用者が頼んだときに、その頼んだメッセージを指定して実行してください。）のようになります。ほかのフラグも
変更するコマンドも設定の変更であり、部分ごとに 1 行ずつ示しながら同じように実行されます。
ほかのリリースのファイルを取り込むコマンド（
`--download` や、別のリリースに固定されたプロジェクトが先に必要とする更新）も同様です。

スイッチは、どのように設定されたかにかかわらず（このコマンド、
ターミナル、ファイルの編集）、記録された時点で有効になります。あなたからチェックを取り去る 9 つ
（プラン承認、review freeze、レビュアーの読み取り範囲、human presence、要約
確認とそのチェック、ステージ出力のチェック、修正のバックストップ、
パイプラインの引き渡しチェック）は、必ず伝えられます。エージェントが次に伝えるステップには、
チェックの名前、その目的、いつからか、どのように設定されたか、
そしてオンに戻すかどうかの提案を示す 1 行が含まれます。たとえば次のとおりです。

> The review freeze check (it stops edits to work you already approved) is off for this project since 10:42, because you said: "turn the review freeze check off for this project". Do you want it back on?

あなたが自分でチェックをオフにした直後は、その行は、今決めたことについて尋ねる代わりに、
元に戻す方法を示します: "You can turn it back on any time."（いつでもオンに戻せます。）
より広い範囲で変わることがある場合は、その範囲を一度だけ提案します。この作業の後には "Do you
want it off for the whole project instead?"（代わりにプロジェクト全体でオフにしますか？）、このプロジェクトの後には "Do you
want it off for this whole machine instead?"（代わりにこのマシン全体でオフにしますか？）です。その範囲ですでにオフになっている
チェックを頼んだ場合は、その旨が同じ提案とともに示されます。このマシン、プロジェクトのスイッチがない
チェック、Guard Policy には、それより広い範囲がないため、
何も尋ねられません。自分のターミナルで実行したコマンドについても何も尋ねられません。
この作業の要約確認と学びのステップを自分でオフにしている間は、この作業のスイッチの後にも
何も尋ねられません。これは、エージェントが「何も尋ねないで」を記録する方法です。どちらかをオンに戻すと、
提案が再び表示されます。

エンジンがチャットでのあなたのメッセージと結び付けられない場合、その行は
チェックがオフであることと、いつからかだけを示します。チェックがオフの間は、新しいチャットはすべて
同じ行で始まり（セッション開始時のコンテキストを表示しない opencode を除きます）、
`config flags --show` と doctor の Flags 行（警告であり、doctor の終了コードは変わりません）に
表示されます。その提案に自分の言葉で答えると、エージェントがコマンドを実行します。環境変数が
引き続きチェックをオフにしている場合は、コマンドがその旨を伝えます。チャットでの「はい」によって、
プロジェクトの共有の `aidlc.settings.json` やこのマシンの設定ファイルが変わることは決してないため、
どちらかがスイッチを記録している間は、その行は尋ねる代わりにそのファイルを示して終わります。
たとえば "The project's shared
settings in aidlc.settings.json keep it off for everyone on this project.
Changing that file changes it."（aidlc.settings.json にあるプロジェクトの共有設定が、このプロジェクトの全員に対してオフにしています。そのファイルを変更すると変わります。）です。`--show` と doctor の行は、実行した場所でそれを行うコマンドを
示します。自分のターミナルではファイルを指定しないクリア、
それ以外（IDE のターミナルを含む）では、スイッチを記録しているファイルごとに、
その `--local`、`--project`、`--global` を付けたクリアです。スイッチはクリアされたものの、
未完了の作業が独自に（スコープや Guard
Policy によって）チェックをオフにしている場合は、その行がその旨を示し、その作業でもオンにするかどうかを提案します。
たとえば次のとおりです。

> The review freeze check (it stops edits to work you already approved) switch is cleared for this project, but it stays off for this piece of work: guard policy off (set by you). Do you want it on for this piece of work too?

逆に、スイッチがまだチェックをオフにしている間に、この作業でチェックをオンに戻すと、
あなた自身の `aidlc.settings.local.json` にあるそのエントリーが
クリアされ、設定自体の行の代わりにその旨が示されます。たとえば次のとおりです。

> The review freeze check (it stops edits to work you already approved) is back on: aidlc.settings.local.json had kept it off for this project, and that entry is cleared.

プロジェクトの共有の `aidlc.settings.json` とこのマシンの設定ファイルは、
その依頼では変更されません。その行はファイルを示し、そのファイルを変更すれば変わることを伝えます。
たとえば次のとおりです。

> The project's shared settings in aidlc.settings.json keep the review freeze check (it stops edits to work you already approved) off for everyone on this project. Changing that file changes it.

自分のターミナルでは、その行は代わりにクリアするコマンドを示し、
エディターや CLI がその変数付きで起動された場合は、変数なしで起動するよう伝えます。センサーと学びも
同じように表示され、チャットからこのプロジェクトに対してエージェントが実行する
`config flags --clear-bypass` も同様です。共有のファイルやこのマシンのファイルの変更を頼んだ場合は、
エージェントが `--project` または `--global` でそこからクリアします。

`config get` は、スイッチがチェックをオフにしている場所を示します。設定ファイルが記録している場合は
`off (AIDLC_DISABLE_REVIEW_FREEZE_HOOK in aidlc.settings.local.json)`、
エディターや CLI がその変数付きで起動された場合は `off (env AIDLC_DISABLE_REVIEW_FREEZE_HOOK)` です。
プラン承認のロックアウト中は、最後の決定以降にあなたが発言していれば、エージェント自身の `config flags
--bypass` は通ります（無人実行からは決して通りません）。`--clear-bypass` は常に通ります。

#### `/aidlc --plan-approval` - Plan approval

アクティブなインテントについて、コード生成がビルドする前に、各コードプランを承認のために
表示するかどうかを設定します。

```
/aidlc --plan-approval off
/aidlc config set plan-approval on
```

| フラグ / 設定キー | 状態とステータスの行 | `off` でスキップされるもの |
|-------------------|----------------------|------------------|
| `--plan-approval` / `plan-approval`（`guard.plan-approval` も可） | Plan Approval | Plan Approval の質問。プランが書かれると、利用者にはそれを示す 1 行が伝えられ、ビルドが始まります。`PLAN_APPROVAL_SKIPPED` 行が、ビルドされたフィンガープリントを記録します |

express と poc ではオフ、それ以外の出荷されているスコープではすべてオンで提供されます。
優先順位はほかのセレモニーと同じですが、1 つ追加があります。メモリの
`## Guard Policy` セクションが `Mode: strict` を保持していると、スコープや
インテントがオフを指定していてもオンのままになり、`AIDLC_DISABLE_PLAN_APPROVAL_GUARD=1` は
マシン上でオフにし、そのメモリのロックよりも優先されます。この変数が有効になるのは、
ハーネスのセッションがその変数付きで開始された場合、`config flags --bypass` で記録された場合、
またはプロジェクトにハーネスのセッションが記録されていない場合だけです。
セッション内の 1 つのコマンドにインラインで設定しても無視されます。そのとき status は、たとえば
`Plan Approval: on (guard policy strict (from project.md))` と表示されるか、
セッションがその変数付きで開始された場合は `Plan Approval: off (from env AIDLC_DISABLE_PLAN_APPROVAL_GUARD)`、
`config flags --bypass` が記録した場合は `Plan Approval: off (from AIDLC_DISABLE_PLAN_APPROVAL_GUARD in aidlc.settings.local.json)`
と表示されます。

これは利用者が決めることです。利用者が `/aidlc --plan-approval off` または
`/aidlc config set plan-approval off` と入力するか（エージェントがセッターを実行します）、
自分の言葉で伝えると（「この作業ではプラン承認を飛ばして」）、エージェントが
セッターを実行します。利用者が頼んだメッセージを指定せずにセッターを実行すると、
次のように拒否されます。

> Turning plan approval off lets code generation start without the person approving the plan, so it is their call. Run it with --message <id>, the message they asked in, when they ask for it.

メモリが strict を保持している場合、拒否メッセージは代わりにメモリファイルを示します。作業がまだないときに
（compose のゲートやスコープの確認で）伝えた場合、利用者の言葉はその依頼に応えます。
そのための `intent create --request <id>` は、フラグの有無にかかわらず
`Plan Approval: off (set by you)` を記録し、そのチャットでの次の作成が、何を作成するかにかかわらず
その言葉を使い切ります。そのようなターンのないフラグや、別の依頼を指定するか何も指定しない作成は、拒否されるか、
スコープの値を維持します。`on` にするには、利用者の入力したターンは必要ありません。スコープを変更すると、スコープ由来の値は
どちらの方向にも新しいスコープのデフォルトに引き継がれ、このインテントに設定された値は維持されます。
`guard.plan-approval` は同じ設定を指すため、
`/aidlc config set guard.plan-approval off` は `Guards Off` のエントリーではなく、
`Plan Approval` の行を書き込みます。オフの間にプランのレビューを頼むと、
設定を変えずにそのプランだけが承認のために表示されます。詳しい動作については
[プラン承認](13-customization.md#plan-approval) を参照してください。

---

### `/aidlc --statusline <on|off>` - AI-DLC's status line

Claude Code と Codex CLI で、ターミナル下部のステータスラインに AI-DLC の進捗を表示するか、
自分で設定したステータスラインに戻します。これは
`aidlc config --statusline <on|off>` を実行し、何も尋ねずに、何が変わったかと
元に戻す方法を 1 行で表示します。作業には何の影響もありません。

- **Claude Code:** あなた個人の選択です。AI-DLC のステータスラインは、あなた自身の
  `.claude/settings.local.json`（git の対象外）に追加または削除されるため、
  チームメイトには影響しません。そのファイルにあったあなた自身のステータスラインは、
  AI-DLC のものをオフにすると元に戻ります。
- **Codex CLI:** プロジェクトの選択です。Codex にはプロジェクト内の個人ごとの
  設定ファイルがないためです。共有の `.codex/config.toml` にある AI-DLC の `status_line`
  行だけを、プロジェクトで作業する全員に対して追加または削除します。
  各自は代わりに、Codex の `/statusline` で自分のステータスラインにタスクの進捗を追加できます。

初回のセットアップでも同じ質問が一度だけ尋ねられます。何も尋ねないセットアップでは、ステータスラインは
追加されません。[ステータスライン](13-customization.md#status-line-claude-code-and-codex) を参照してください。

```
/aidlc --statusline off
aidlc config --statusline on
```

ほかのハーネスでは、そのツールにはステータスラインがないことと、
`/aidlc --status` で作業の状況がわかることを伝えます。

---

### `/aidlc --version` — Framework version

フレームワークのバージョン（`aidlc <X.Y.Z>`）を表示して終了します。読み取り専用で、ワークフローがなくても動作し、ワークフローの再開を促すことはありません。

`aidlc config --pin` でこのリリース以降に pin されたプロジェクトでは、
ワークフローを実行する pin されたリリースと、それと異なる場合はこのマシンでアクティブなリリースが
答えに示されます: `this project is pinned
to aidlc 2.10.0, which runs its workflow (machine active: 2.9.0)`。それより前の
リリースへの pin では、このマシンのバージョンの行が答えになります。

**構文:**

```
/aidlc --version
```

---

### `/aidlc --help` — Usage information

使用できるコマンドとフラグの概要を表示します。

**構文:**

```
/aidlc --help
```

プラグインがコマンドを追加しているプロジェクトでは、ターミナルでの `aidlc --help` が
それらを `PLUGIN COMMANDS` の下にプラグインごとに一覧表示します。

---

### `/aidlc <plugin command>` - Run a command a plugin adds

プラグインは、レポートやダッシュボードなど、独自のコマンドを追加できます。AI-DLC 自身の
コマンドと同じように実行します。

```
/aidlc test-pro-report --since 7d                          # in the chat ($aidlc in Codex)
aidlc test-pro-report --since 7d                           # in a terminal, native install
bun .claude/tools/aidlc.ts test-pro-report --since 7d      # in a terminal, copy install (your harness folder)
```

AI-DLC は、独自のランタイムでプラグインのスクリプトを自ら起動するため、このコマンドは
ネイティブインストールでもコピーインストールでも、PowerShell、cmd、POSIX シェルでも動作します。
名前の後ろにあるものはすべてスクリプトに渡され、その出力と終了コードは
そのまま返されます。ほかのコマンドと同じく、ツールは独自の許可設定をこれに適用します。
AI-DLC はプラグインのコマンドを事前承認しません。

- 実行されるのは、このプロジェクトにインストールされ、オンになっているプラグインだけです。オフの場合、
  コマンドはその旨を 1 行で伝え、オンに戻す `aidlc config project --plugins`
  コマンドを示します。
- AI-DLC 自身のコマンドは、常にその名前を保ちます。それらと同じ名前、プランと同じ名前、
  別のプラグインのコマンドと同じ名前のプラグインのコマンドは追加されません。
  それに出会ったプラグインの同期は、両方の名前を挙げてその旨を一度だけ伝え、`aidlc doctor
  --verbose` でも再び表示されます。
- 名前が長い文の先頭にある場合（`/aidlc test-pro-report on the login
  bug`）、エージェントは、コマンドを意図したのか依頼なのかを読み取り、
  判断できない場合にだけ尋ねます。

---

## Deterministic CLI Tools

ネイティブのディスパッチャーは、利用者の操作のための安定した公開ルートを提供します。
バージョン付きのリリースのランタイムは、それらのルートを使います。ローカルで生成されたソースの
プロジェクションは、ハーネスのディレクトリにある Bun/TypeScript のツールで同じ操作を実装しており、
公開ルートのない内部処理には、ツールを直接呼び出すことが引き続き役立ちます。以下にルートが記載されている場合は、`aidlc` を優先してください。

### Read Earlier Questions and the Audit Timeline

エージェントは、これらのコマンドで以前の回答とタイムラインを読みます。ワークフローを調べるために
あなたが実行することもできますが、ほかの `aidlc engine` のルートと同じく、これらは
ハーネスの内部機構であり、あなた自身のスクリプトのための安定したインターフェースではありません。

```bash
aidlc engine log answers --stage requirements-analysis
aidlc engine audit history
```

`log answers` は、対になった回答、未回答の質問、まれにある曖昧な回答（前に質問がないもの、または 2 つの書き込みが同じ秒に重なったもの）を含む JSON を返します。1 つの返信で、それが答えたメニューについて記録されたすべての質問が閉じられます。
`audit history` は、イベントと自由形式のメモの JSON タイムラインを返します。
`aidlc engine audit turns` はあなたのためのものです。コミットされる監査には言葉が
残らないため、このマシンのメッセージ記録から読み取った、あなたの各ターンと送った内容を一覧表示します。
記録は、書き込んだマシンに 1 日だけ残るため、それより古いターンや、
別のクローンのターンでは、言葉がここにないことが示されます。

シェルが解釈する文字（`$`、バッククォート、引用符、
`%`、`&`、`|`、`<`、`>`、`^`、`!`、改行）を含む回答は、コマンドラインではなく
ファイルを通じて `log answer` に渡されます。エージェントはそれを
`<record>/.aidlc-engine/answer-text/answer.txt` に書き込み、`--details-file
.aidlc-engine/answer-text/answer.txt` を渡します。プロジェクトフォルダーからの指定や
フルパスで指定した同じファイルも読み取られます。エンジンはそのフォルダーだけを、リンクをたどらずに、
64 KiB まで読み取り、読み取った後にファイルを削除します。`log decision` は、質問を
同じように `--decision-file` で、選択肢を 1 行に 1 つずつ `--options-file` で受け取り
（カンマを含む選択肢は、個別の `--option '<choice>'` として渡すこともできます）、`learnings surface` は修正を
`--correction-file` で受け取ります。エージェントが `next --request-file` のために書いたファイルは、
回答にも使えます。`log answer --request-file aidlc/.aidlc-request-text/request.txt` は、
`next` と同じようにそれを読み取り、回答が受け付けられた後に削除するため、拒否された回答では
ファイルが残ります。これは `--details` や `--details-file` と一緒には渡しません。

通常の `log answer` には、まだどの決定にも使われていない、このチャットでの利用者の返信が必要です。
質問が記録される前に送られたものでも後に送られたものでもかまいません。どの
質問に答えるものかは、エージェントが読み取ります。選択肢のある質問では、
`--details` はエージェントが返信から読み取った選択肢で、フックが言葉を保持しなかった場合や、
返信が複数の質問に答える場合は、返信そのものです。回答の行には、そのような
返信の言葉がすべて残り（Person Reply）、最新のもの（Message Id）と、
複数ある場合はそのすべて（古い順、Message Ids）が示されます。ただし、以前の回答がすでに記録した返信は、
それが最新で、かつ質問がその前に記録された場合を除いて含まれません。エンジンが
チャットを特定できない場合は、1 つのチャットが返信したときは ID だけを残し、複数のときはどちらも残さず、
いずれの場合も回答を記録します。`log decision --message <id>` と `log answer --message <id>` は、
代わりに 1 つのメッセージを指定します。たとえば、エンジンの別の質問が開いている間に送られたメッセージです。
エンジンは、その記録が存在すること、このチャットを通じて届いたか別のチャットでこの
作業について送られたこと、後の承認や決定で使い切られていないことを証明し、
その言葉は一切読みません。ID は
エンジンからエージェントへのテキストであり、利用者には決して表示されません。
対応付けのルール、順序、フィルターについては [フックとツール](../reference/06-hooks-and-tools.md#read-only-audit-commands) を参照してください。

### `aidlc engine log install` - a system install the person asked for

```bash
aidlc engine log install <package>... --message <id>
```

システムパッケージのインストール（`sudo apt-get install`、`brew install`、`winget
install` など）は、利用者が頼んだ場合を除き、利用者を待ちます。承認やほかの決定にすでに使われた
メッセージで利用者が頼んだ場合（「承認。
それから shellcheck をインストールして実行して。」）、インストールのチェックの拒否メッセージは、利用者のメッセージとともにこの
コマンドを示します。エージェントは頼んだメッセージを指定し、エンジンはほかの依頼と同じように
それを証明します（存在すること、この作業について利用者自身のプロンプトを通じて届いたこと、
利用者の言葉を含むこと、後の決定で使い切られていないこと）。その言葉は一切読まず、
パッケージを `SYSTEM_INSTALL_REQUESTED` として記録します。
その後、利用者の次の決定までの間、それらのパッケージのインストールが実行され、その `SYSTEM_INSTALL_ALLOWED` 行には
そのメッセージと利用者の言葉が示されます。

### `aidlc engine bolt set-autonomy` - change Construction approvals

Construction の間に、自動で続行するか、チェックポイントごとにレビューするかを明示的に
頼みます。コンダクターは **Continue automatically** を `autonomous` として、
**Review each checkpoint** を `gated` として記録します。

```bash
aidlc engine bolt set-autonomy --mode autonomous --message <id>
aidlc engine bolt set-autonomy --mode gated
```

どちらも `Construction Autonomy Mode` を更新し、`AUTONOMY_MODE_SET` を発行します。自律を
許可するには `--message <id>`（それを選んだ利用者のメッセージの ID）が必要で、
許可によってそのメッセージは使い切られます。ただし、同じメッセージの中での承認
（「プランを承認して、ここからは Construction を自動で進めて」）や、そのメッセージの中でエージェント自身の質問に
答えた回答では、使い切られたことになりません。取り消しにはメッセージは不要です。新しいチェックポイント方式の
ワークフローでは、skeleton-off の場合は Construction の開始時に、skeleton-on の場合は
最初の動作する統合済みユニットがスケルトンのチェックポイントを通過した後に、この選択肢が示されます。
すでにわかっている選択は再び尋ねられず、必要に応じた変更も引き続き有効です。

自律が制御するのは、通常の完了の承認です。プラン承認がオンの間は、どのユニットにも引き続き Plan
Approval が必要です（オフの場合、各ユニットのプランは 1 行の通知の後にビルドされ、
スキップとして記録されます）。検証コマンドの選択とスケルトンのチェックポイントの承認には、
常に人が必要です。生成前の要約確認に人が必要なのは、
`directive.ceremony.summary_confirmation === "on"` の場合だけです。失敗すると止まります。
`Construction Checkpoints` のない既存のワークフローは、従来の最初のステージと
後半のステージでの承認を維持し、チームが所有するユニットのゲートは独自のポリシーを維持します。

### Checkpoint verification timeout

`aidlc engine bolt checkpoint --unit <unit> --kind unit --action verify`
は、承認されたプロジェクトのチェックを、デフォルトで 30 分の上限で実行します。
より長いチェックには、`--timeout-ms 3600000`（1 時間）を追加します。値は
1〜2,147,483,647 ミリ秒の整数でなければならず、
`--action verify` と一緒の場合にだけ受け付けられます。証拠を更新するための再試行を含め、各実行が
その時間枠を使います。

タイムアウトすると、失敗した検証を記録する前に、POSIX では所有するプロセスグループを、
Windows ではプロセスツリーを終了させます。終了を確認できない場合は、
クリーンアップが不完全であることを報告します。POSIX では、SIGINT、
SIGTERM、SIGHUP で検証を止めると、そのシグナルが所有するグループに転送されます。
それを無視するプロセスは、短い猶予期間の後に強制終了されます。クリーンアップには最大 4 秒
かかることがあります。転送できない SIGKILL で検証プロセスが強制終了された場合も、
所有する POSIX グループ内の監視プロセスがグループを終了させます。中断された検証プロセスは、
合格の証明を記録できません。強制終了では、開始した証明は未完了かつ未検証のままになります。
独立して切り離された POSIX プロセスは、そのグループの対象外です。

### Construction order and execution

ユニットの分解がスコープに含まれる、ソースを生成する新しい solo のユニットワークフローでは、
`Construction Checkpoints: enabled`、`Construction
Iteration: unit-major`、`Construction Execution: serial` が記録されます。1 つのユニットが、
該当する設計ステージと Code Generation を終えてから、次のユニットに進みます。
設計のみのワークフローやユニットのないワークフローは、既存のステージの流れを維持し、チームが所有する
ユニットは独自のゲートのリズムを維持します。既存のワークフローと明示的なイテレーションの
選択は維持されます。swarm 実行を明示的に選ぶには、まず stage-major
を選択します。

```bash
aidlc engine state set-construction-iteration stage-major
aidlc engine state set-construction-execution swarm
```

既存のワークフローで検証付きのチェックポイントを使うには、できればユニットの
作業が始まる前に、`aidlc engine state set-construction-checkpoints enabled` を使います。
`disabled` は従来のチェックポイントの流れを維持します。これらの型付きのセッターは、
ランタイムの設定を更新します。汎用の `state set` は、`Construction Checkpoints`、
`Construction Execution`、`Construction Iteration`、`Construction Verification
Command` を拒否します。それぞれ `set-construction-checkpoints`、`set-construction-execution`、
`set-construction-iteration`、またはレシートに紐付いた
`set-construction-verification-command` を使ってください。
Construction の間は、これらを自分の言葉で変更できます（「チェックポイントを
オフにして」「ここからはユニットを 1 つずつビルドして」「ユニットを並列で
実行して」）。エージェントは、先に質問することなくそのターンで変更を行い、何が変わったかと、
元に戻せることを 1 行で伝えます。たとえば "Construction
checkpoints (a stop after each Unit for you to check and approve it) are off for
this work now (they were on). You can switch back any time."（Construction のチェックポイント（各ユニットの後に止まってあなたが確認・承認するもの）は、この作業では今オフになりました（以前はオンでした）。いつでも元に戻せます。）です。変更は、あなたの言葉とともに `CONSTRUCTION_POLICY_SET` として記録されます。
セッターが変更を行うのは、あなたが頼んだメッセージをエージェントが指定した場合
（`<value> --message <id>`。`config set` と同じように証明されます）か、エージェントが提案した
変更をあなたが選んだ場合だけです。そのため、エージェントが独断でこれらを変えることはできず、
無人実行で変わることも決してありません。先に別の変更が必要な変更（並列のユニットには
stage-major とチェックポイントのオンが必要）では、両方を行い、両方を伝えます。

ステージの途中で `unit-major` に切り替えたり、チェックポイントをオンにしたりしても、
すでに完了したユニットはそのままです。`/aidlc` は、まだ作業が残っている次のユニットから続行し、
変更は `CONSTRUCTION_POLICY_SET` として記録されます。
逆方向に、チェックポイントをオフにした `stage-major` に切り替える場合（イテレーションを
元に戻す場合や、stage-major のままチェックポイントをオフにする場合）も、それらはそのままです。切り替えの前に
再開したユニット（前に戻るジャンプ、Redo、チェックポイントでの Request Changes）は、
引き続きやり直しが行われ、完了後もその結果が保たれます。

並列のビルドが準備された後に serial に切り替えた場合（たとえば、並列の開始が止まった後の
「代わりに 1 つずつビルドして」）、承認したプランは維持されます。
各ユニットは、新たなプランの質問なしに、プロジェクトそのものの中でプランからビルドされます。
そのビルドが準備した worktree はそのまま残り、それらの Bolt はクローズとして記録されます。

実行方式は承認とは別のものです。swarm は、ガイド付き（`gated`）の完了とも
自動（`autonomous`）の完了とも組み合わせられます。unit-major は serial のままで、
矛盾する swarm の設定を拒否します。unit-major に戻る前に `aidlc engine state set-construction-execution serial` を
実行してください。既存の明示的な選択は維持してください。実行方式のフィールドがない
ワークフローは、従来の自律に基づく swarm のルーティングを維持します。

チェックポイントが有効な solo の作業で、実際に空でないユニットの DAG があり、ソースを生成する
ステージが含まれている場合、skeleton-on では、stage-major を選択していても、
後のユニットより前に、DAG の最初のユニットを最小限の動作する統合済みスライスとして
必ずビルドします。最初の設計ステージのレビューだけでは、動作するスケルトンの証明には
なりません。すでに承認されたインラインのユニットは、後の swarm のバッチから除外されます。

### `aidlc engine swarm prepare` - prepare a reproducible batch

最初の保護された Code Generation の準備の前に、すでに承認された親のアプリケーションの
ソースをコミットし、選択したベースからそれを再現できるようにします。これには、並列のバッチに切り替える前の、
承認済みのインラインのスケルトンのソースも含まれます。`.env` やログなど、Git が
無視して追跡しないファイルは、`.aidlc-source-paths.json` でソースとして登録しない限り、
これに含まれません。それらはその場所に残り、ユニットの worktree には、ほかの Git のチェックアウトと
同じく含まれません。そのような無視されたリンクの先も含まれず、サブモジュールのファイルは
サブモジュール自身の無視ルールで読み取られます。それらのファイルの変更（プランの承認後に
書かれたテスト実行のログや、ビルド中の開発サーバーのログ）によって、プランの承認が再び
求められることはなく、land やステージの完了でも、
メインのチェックアウトへの変更とは見なされません。このルールは、従来の自律方式のワークフローにも、
新しいチェックポイント方式のワークフローにも適用されます。自律の許可によって、
自動コミットが許可されることは決してありません。

```bash
aidlc engine swarm prepare --batch <N> --units "<exact emitted Units>"
```

このツールは、子の worktree を作成する前に、すべてのユニットについて、読み取り専用のソースと承認の事前チェックを
行います。ソースがコミットされていない場合は、コミットして再試行するよう指示を返し、
その拒否によって子が残ることはありません。明示的な許可がある場合にだけ
コミットし、その後、現在の承認の証拠で再試行してください。
アプリケーションのソースやプランが変わった場合は、必要な Plan Approval を改めて提示してください。
この要件はアプリケーションのソースに関するもので、無関係なフレームワークの記録やほかのファイルを
まとめてコミットすることではありません。

同じ事前チェックは、ユニットが実際に開始するコミットで gitlink に `.gitmodules`
のエントリーがないために、ソースファイルがユニットの worktree で空のフォルダーになってしまう
入れ子のリポジトリを示します。それらを Git のサブモジュールとして登録し、
そのエントリーを含む開始コミットを選ぶか、そのユニットを serial でビルドしてください。この
拒否では、子の worktree は作成されず、親のコミットも行われません。登録された
サブモジュールは、ネットワークにアクセスせずにメインのチェックアウトからコピーされます。
選択したソースの境界の外にある gitlink は、準備を妨げません。選択した
ベースに承認済みの親のソースが必要な場合、準備はその親のコミットから始まり、
その登録を確認します。明示的な破棄の後の回復では、代わりに、親がその後変わっていても、
記録された Approval Source Commit を確認します。

### Construction verification command — record human authorization

チェックポイントが有効な作業では、Delivery Planning が、プロジェクトのスキャンから、`bun test`、`pytest`、
`make check` などの実際のプロジェクトのチェックを提案します。構造化された
**Approve** / **Request Changes** の質問は、**Use this command to verify each
completed Unit?**（このコマンドで、完了した各ユニットを検証しますか？）と尋ねます。コマンドを提示する前に、
ハーネスのファイル書き込みツール（Write/edit）で、それを UTF-8 テキストとして
`<record>/verification-command.txt` に書き込んでください。シェルの `echo` やヒアドキュメントは決して使わないでください。リポジトリ由来のコマンドの
テキストを、シェルの行に埋め込んではなりません。シェルの置換が
承認の前に実行されるおそれがあるためです。記録からの相対パスだけを渡してください。コマンドは、
自身が実行されるセッションを自分で見つけます。

```bash
aidlc engine log decision --stage "<directive.stage>" --checkpoint verification-command --command-file verification-command.txt --decision "Use this command to verify each completed Unit?" --options "Approve,Request Changes"
```
`decision` ツールの JSON 出力の `command` フィールドから、完全な正規化済みのコマンドを
そのまま質問のコードスパンにコピーしてください。要約、先頭部分、ダイジェストで省略したり
置き換えたりしては決してなりません。コマンド内のバッククォートを保てるだけの長さの
コードスパンの区切りを使ってください。人は
`<record>/verification-command.txt` を開いて確認することもできます。


そのチャット、またはそれが最も新しい質問である間は新しいチャットで、人の **Approve** / **Request Changes** の
返信を待ってください。レシートを許可するのは **Approve** だけで、
無関係な返信、**Request Changes**、別の質問への返信では許可されません。人が選んでいない限り、
`--details "Approve"` を決して書かないでください。そのときにだけ、次を実行します。

```bash
aidlc engine log answer --stage "<directive.stage>" --checkpoint verification-command --command-file verification-command.txt --details "Approve"
aidlc engine state set-construction-verification-command --command-file verification-command.txt
```

これらは `aidlc-log` の decision/answer のチェックポイント形式です。どちらも同じ
`--stage`、`--checkpoint verification-command`、正規化済みのコマンドを必要とします。それぞれが
自身が実行されるセッションを見つけ、どのセッションか判断できないと報告した場合にだけ、
`--session <id>` でそれを上書きします。どちらも `--command-file <path>`
または `--command` のちょうど一方を受け付け、両方またはどちらもない場合は拒否されます。コンダクターの
シェル呼び出しではファイル形式を使ってください。直接の引数が安全なのは、シェルの
展開なしに渡される場合だけです。ファイルは作業の記録の中にある通常のファイルで、記録からの名前
（`verification-command.txt`）、プロジェクトフォルダーからの名前
（`aidlc/spaces/<space>/intents/<intent>/verification-command.txt`。スラッシュはどちらの向きでも可）、
またはフルパスで指定します。記録の外の名前、`..` を含む名前、経路上のシンボリックリンクは
拒否され、ファイルは 16 KiB 以下でなければなりません。ファイルは
UTF-8 としてデコードされ、直接の引数とまったく同じように正規化されます。
前後の空白は、記録、ハッシュ化、実行の前に取り除かれます。
結果のコマンドは、空でなく、1024 文字以下で、1 行で
なければなりません。ツールは、制御文字（改行、CR、タブ、NUL を含む）と、
表示を偽装する文字（Unicode の書式文字（ゼロ幅文字や
双方向制御文字を含む）、行／段落区切り文字、ノーブレークスペース（U+00A0））を拒否します。
複数行のチェックはスクリプトにまとめ、その呼び出しを記録してください。
`decision` は、`Checkpoint: Construction Verification
Command` と `Command SHA-256` を持つ `DECISION_RECORDED` を記録します。その JSON 出力には、
`challengeId` と `challengeFile` に加えて、完全な正規化済みの `command` と `command_sha256` が含まれます。
`answer` も `command_sha256` を表示します。`answer` には、一致する保留中の
decision と、`AIDLC_SKIP_HUMAN_PRESENCE_GUARD=1` の場合でも、利用者がそのコマンドとセッションの
現在のチャレンジに返信したという、人のターンのフックによる記録が必要です。`--details` は
エージェントが返信から読み取った選択肢を示し、レシートには利用者の言葉がそのまま
含まれます。後の `HUMAN_TURN` だけでは不十分です。新しい decision を記録すると、
セッションの以前のチャレンジと応答が置き換えられます。成功した answer は、両方を消費する前に
監査イベントを追記します。追記に失敗した場合、同じ応答は再試行できます。
古い応答、一致しない応答、すでに正常に消費された応答は拒否されます。
承認する返信（エージェントが利用者の返信を読んで
`--details "Approve"` を渡します。数字だけや別の単語は、選択肢を示していないとして拒否されます）は、
ツールが所有する `VERIFICATION_COMMAND_RECORDED` レシートを発行します。変更を求める返信は
`QUESTION_ANSWERED` だけを記録し、状態を設定せずに別のコマンドを提案することを意味します。
どちらも選んでいない返信は、尋ねるべき 1 つの追加質問とともに拒否されます。レシートには、
ステージ、チェックポイント、セッション、その正規化済みのコマンドの SHA-256、
`Command Label` としての完全な正規化済みのコマンド（決して切り詰められません）、そして人が選んだ
選択肢そのものが含まれます。
`aidlc-audit append` は、この予約されたレシートを発行できません。

型付きのセッターは、`--command-file <path>`（同じ方法で指定）か、位置引数の
コマンド 1 つを受け付け、両方は受け付けません。最新の現在のワークフローの
承認レシートがそのダイジェストと一致する場合にだけ、`aidlc-state.md` の `## Runtime State` の下に
`- **Construction Verification Command**: <cmd>` を書き込みます。その JSON 出力には、`command_label`
（完全な正規化済みのコマンド）と `command_sha256` が含まれます。人の追加のターンは求めません。
実行を許可するのは状態のフィールドではなくレシートです。検証も同じ
結び付きを確認します。古いワークフローや単独実行のステージのレシートでは許可されず、
別のコマンドに対する後のレシートは、以前のものに優先します。一致するレシートのない
フィールドや、一致するフィールドのないレシートは、許可にはなりません。このフィールドを直接、または汎用の `state set` で書き込まないでください。

記録されたコマンドは、このインテントのすべてのユニット／バッチのチェックポイントで再利用されます。
選択とその後の変更には、自律的な完了のもとでも、常にこの decision/answer/セッターの流れが必要です。
実行可能なチェックがまだない場合（greenfield）、
人は Delivery Planning の間に先送りできます。フィールドは未設定のままにしておけば、最初の
チェックポイントで尋ねられます。仮のコマンドを作り出したり、自動承認したりしては決してなりません。

### `aidlc engine bolt checkpoint` - verify and approve a completed Unit

エンジンは、ユニットとチェックポイントの種類（`unit` または `skeleton`）を示します。本体、
レビュー、レシートはすでにあります。作り直すのではなく、チェックポイントに従ってください。
レビューの後にユニットのコードやドキュメントが変わった場合、チェックポイントの `rereview` が
1 つの再チェックのリクエストを示し、エージェントは検証の前にそれを実行します。これは
Guard Policy が `strict` の場合です。`relaxed` と `off` では、代わりに変更が受け入れられ、
承認済みのユニットは承認済みのままで、`verify` はその 1 行を
`change_notices` として返します。レビューがオフの場合も同様で、次のユニットの `verify`、または
Construction ステージ自身のチェックがその行を返します。ユニット自身のレビューが
完了していない場合（まだ判定がない、またはパスが残っている NOT-READY）、`rereview` は
`unfinished` を持ち、それを完了させるリクエストを示します。利用者がユニットを
そのまま承認するよう伝えた場合は、`verify` に `--over-unfinished-review --message <id>` を追加します。`<id>` は
利用者がそう伝えたメッセージです。Guard Policy が `off`、`relaxed`、またはその作業に設定された `strict` で、そのメッセージが
証明され、レビューが依頼されていた場合、ユニットは検証され、利用者の言葉がその
承認となり（`review_not_finished.approved_in_words`: `approve` はそれ以上何も尋ねません）、
ステージのゲートと同じく、レビューは未完了として記録されます。そのレビューの判定が後から届いた場合は、
記録され、ユニットは承認されたままです。チームがロックした `strict` のもとでは、利用者に尋ねずに、
まずレビューが完了します。再試行したレビューでも完全なレビューが書かれなかったときにコンダクターが記録する
NOT-READY のフォールバックでレビューが終わったユニットは、
一度だけ尋ねられ（`review_not_finished.question`）、同じように承認されます。スキップする
選択肢はありません。ユニットの作業のこの実行で一度も依頼されていないユニットのレビュー
（前に戻るジャンプや再開の後は、それ以前のレビューは数えません）は、どの Guard Policy でも必須です。`rereview` は
`first` を持ち、その実行の最初のリクエストを示します。

```bash
aidlc engine bolt checkpoint --action status --unit "<Unit>" --kind <unit|skeleton>
aidlc engine bolt checkpoint --action verify --unit "<unit>" --kind <unit|skeleton>
aidlc engine bolt checkpoint --action verify --unit "<unit>" --kind <unit|skeleton> --over-unfinished-review --message <id>
aidlc engine bolt checkpoint --action verify --unit "<unit>" --kind <unit|skeleton> --override-blocking-sensors --user-input "Override blocking sensors"
```

検証は、記録され人が許可した `Construction Verification
Command` を実行し、現在の成果物、ソース、試行に紐付いた証明を保存します。コマンドの
引数は受け付けません。`construction_checkpoint.command_authorized`
が false の場合は、`verify` を実行しないでください。
[記録されたコマンドの流れ](#construction-verification-command-record-human-authorization) を完了してから、
`next` を再実行してください。スケルトンのコマンドは、統合済みのスライスをエンドツーエンドで
証明し、通常のユニットの動作結果を確認するものでなければなりません。ユニット単位でビルドする solo の作業では、`verify` はまず、
ユニット自身の出力に対して、チェックポイントのステージのブロッキングゲートのチェックを実行します。
失敗したチェックがあると、そのチェックの Fix または Override の質問のためにユニットを示して拒否し、
利用者が Override を選んだ場合は `--override-blocking-sensors --user-input
"Override blocking sensors"` で再試行します（[ユニットのチェックポイントでのブロッキングチェック](../reference/07-sensor-system.md#default_severity)）。承認には、現在の
検証済みの証明が必要です。人への承認の質問では、"Verified with `<full command>` (exit 0)" と表示し、
現在のツールの出力にある完全な `verification_command` を、省略せずにコードスパンに
コピーしてください。この表示ラベルは、先頭部分ではなく、完全な
正規化済みのコマンドです。`verify` が `verified: true` を報告し、
現在のチェックポイントが `ready: true` を持つようになってから、`ask` を実行してください。これは、準備ができていない
チェックポイントや未検証のチェックポイントを拒否します。**Approve** / **Request Changes** を提示する前に、
現在のユニット、種類、フィンガープリント、検証の証明の
ID、許可されたコマンドのダイジェストについて、1 回限りの質問を開きます。各 `checkpoint` アクションは、
自身が実行されるセッションを自分で見つけます。どのセッションか判断できないと報告した場合にだけ、
`--session <id>` でそれを上書きします。

```bash
aidlc engine bolt checkpoint --action ask --unit "<unit>" --kind <unit|skeleton>
```

`ask` が `earlier_reply` を返した場合、利用者はユニットのレビューの実行中に、そのセッションで
すでに答えています。レビューが依頼された後、ほかの質問（学びの質問など）が
尋ねられる前に利用者が入力した言葉が、
この質問への返信です。
それを読み、質問を表示せずに、利用者が選んだアクションを実行してください。何も
答えていない場合は、質問を提示して、後述のとおり待ちます。`ask` が
`left_out`（利用者がユニットのビルドから外すよう頼んだもの）を返した場合は、質問の直前に
それが伝えられます。たとえば "Left out of this build: AC2.1.1 and AC2.2.1."（このビルドから外したもの: AC2.1.1 と AC2.2.1。）です。
`changed_files`（利用者が持ち込んだファイルのうち、ユニットのビルドが変更または
削除したもの）も、元に戻す方法とともに同様に伝えられます: "The build changed README.md, a file you
had before this work started. To put it back, run `git checkout HEAD -- README.md`."（ビルドが、この作業の開始前からあったファイル README.md を変更しました。元に戻すには、上記のコマンドを実行してください。）
`check_offer` も同様で、ステージのレビューのパスを使い切り、レビュー済みの作業が
最後のレビューの後に変わった場合に伝えられます: "Do you want me to have the new approach checked
before you decide?"（決める前に、新しいやり方をチェックしてもらいますか？）

このチェックポイントの質問に対する人の **Approve** / **Request Changes** の返信を、
そのチャットか新しいチャットで待ってください。独自の質問がない新しいチャットの
返信は、別のチャットが尋ねたチェックポイントの質問が最も新しい質問で、まだ開いている間は、
その質問のために保持されます。返信が許可するのは、一致するアクションだけです。
無関係な返信、エージェント自身の言葉、別の質問への返信では
許可されません。人が選んでいない `--user-input` を決して渡さないでください。利用者が選んだ
アクションだけを実行してください。

```bash
# Only after the human chose Approve:
aidlc engine bolt checkpoint --action approve --unit "<unit>" --kind <unit|skeleton> --user-input 'Approve'
# Only after the human chose Request Changes and supplied feedback:
aidlc engine bolt checkpoint --action reject --unit "<unit>" --kind <unit|skeleton> --user-input 'Request Changes' --reason '<human feedback>'
```

利用者が判断の際に、レビューの指摘の修正を頼んだり、指摘を却下したりした場合は、
ステージのゲートと同じく、指摘ごとに、承認には `--fix-finding "<review-artifact>#R-NN"` を、
却下には `--reject-finding '<review-artifact>#R-NN=<their reason>'` を追加します
（レビュアーが修正済みとしたものの、利用者が修正されていないと言う指摘には `--reopen-finding`）。

エンジンの別の質問が画面に表示されている間に利用者が答えた場合（その言葉が
新しい作業の提案に対するものである場合）、その返信はチェックポイントのためには保持されません。
拒否のメッセージはそのメッセージを示し、承認または却下で `--message <id>` を指定すると、
それが利用者の返信として記録されます。アクションは応答を消費します。`verify` を再実行すると、
どのセッションのものでも、このインテントの開いているチェックポイントの質問と、取得済みのチェックポイントの応答が
すべて取り下げられます。新しい検証が `verified: true` を報告してから、改めて尋ねてください。
古い証明に対する応答では、フィンガープリントとコマンドが変わっていなくても、
新しい証明を承認できません。`human_required: false` の検証済みの通常のユニットは、
`--user-input` なしで承認され、`ask` も不要です。人による却下には、常に
上記の検証済みの質問と回答の流れが必要です。スケルトンには常に
人が必要です。欠落した証拠や古い証拠は `errors` で説明されます。検証をでっち上げたり、
チェックポイントの承認の質問を早く開いたりせずに、必要に応じて人に相談しながら、
示されたレビューやレシートを修復してください。検証、承認、却下の後には `next` を再実行し、
1 つのユニットのチェックポイントを、Code Generation ステージ全体の承認として報告しては決してなりません。
検証プロセスは、証明ファイルと並んで、ツールが所有する `CHECKPOINT_VERIFICATION_RECORDED` レシートを記録し、
承認にはそのレシートが必要です。手書きの
証明ファイルでユニットを検証することはできません。証明ファイルがまったくないチェックアウト（
新しいクローンや別のマシン）では、証拠が変わっていない承認済みのユニットについて、
そのレシートが証明の代わりとなるため、何も再実行されません。

1 つのセッションで開ける保護された質問は 1 つだけです。保護された質問は
1 つずつ尋ね、ほかのことをする前に回答を待ってください。

バージョン 4 の証明と CLI の JSON には、`command_sha256` と、`command_label` に入った完全な正規化済みの
コマンドに加えて、終了ステータス、取得した stdout/stderr 全体のバイト数と
SHA-256 ダイジェスト、各ストリームの末尾 2 KiB（`stdout_tail` と
`stderr_tail`）が残ります。末尾部分は、先頭にある不完全なマルチバイトシーケンスを取り除いた後、
UTF-8 としてデコードされ、改行とタブ以外の制御文字は
U+FFFD に置き換えられます。出力全体は保持されません。チェックは出力をメモリではなく
一時ファイルに書き込むため、長時間実行されるテストスイートの出力によって、合格するチェックが失敗することはありません。
実行中にユニットのファイルを変更するチェック（フォーマッターやジェネレーター）は、
その時点のファイルに対してもう一度実行されます。再びファイルが変わった場合は、
証明の `error` が、ファイルをそのままにするチェックを使うよう伝えます。プロジェクトの
チェックコマンドは、秘密情報を出力してはなりません。これらの診断用の末尾部分は、
秘密情報の秘匿化が行われません。承認では、`GATE_APPROVED` の `Verification Command SHA-256` が
証明の `command_sha256` に結び付けられます。以前に許可された
コマンドで承認されたユニットは、利用者が新しいコマンドを承認しても承認を維持します。新しいコマンドは
まだ承認されていないユニットを検証し、セッターは
`Using <command> from here on.` という行を `notice` として返します。
失敗の説明には末尾部分を使ってください。さらに診断が必要な場合は、新しく選んだコマンドではなく、
同じ許可済みのプロジェクトのチェックを使ってください。バージョン 1〜3 の
証明は、アップグレード後は未検証になります。承認の前に、記録されたコマンドを許可し、
`checkpoint --action verify` を再実行してください。

### `aidlc engine swarm check` / `finalize` - verify native worktrees

どちらのコマンドも、Construction
Checkpoints の有無にかかわらず、準備された各ユニットの worktree で、インテントに記録され人が許可した Construction
Verification Command を実行します。

```bash
aidlc engine swarm check <Unit> [--test-file <protected spec>] [--show-output]
aidlc engine swarm finalize --batch <N> --units "<all Units>" --claimed "<converged Units>"
```

`--check-cmd` は、チェックポイントの有無にかかわらず省略可能です。指定する場合、その
正規化済みのダイジェストは許可されたコマンドと一致しなければなりません。許可がない場合は
実行が拒否されます。合格するコマンドに置き換えるのではなく、[記録されたコマンドの流れ](#construction-verification-command-record-human-authorization)
と `set-construction-verification-command` を完了してください。ワークフローがない場合、どちらのコマンドも拒否されます。`check --show-output` は、
実行中のコマンド自身の出力を stderr に送り、stdout には判定の 1 行だけが残ります。
並列のビルドでは、各ワーカーへの指示と各レビュアーへのレビューのリクエスト
（`test_command`）が、プロジェクトのメインフォルダーから実行する `check <Unit> --show-output` を示すため、
あなたが承認したテストは AI-DLC 自身のコマンドを通じて実行されます。
`check` は助言的なものです。`finalize` は、クレームされた各ユニットをマージする前に、コマンドを再実行し、
レビューの証拠を検証します。`finalize` を再実行すると、どのセッションのものでも、
このインテントの開いているチェックポイントの質問と、取得済みのチェックポイントの応答がすべて取り下げられます。
新たな検証、ソースの land、バッチのステータスが `ready: true` になった後でのみ、
改めて尋ねてください。`SWARM_UNIT_CONVERGED` を受け取るのは、
検証済みのネイティブの合格だけで、許可された `Command SHA-256` が付きます（以前のリリースの行には
ないことがあります）。`next` の前に、ネイティブの worktree のマージでそれらのソースを land してください。
バッチのビルド中に利用者がメインのチェックアウトに加えた編集は、strict の Guard Policy のもとでは
そのマージを止めます（拒否のメッセージは手順を示します。利用者がその
変更を頼んだ場合は、利用者のメッセージを付けた `swarm allow-change` の後、同じマージで変更が維持されます。
そうでなければ、編集を元に戻してマージを再実行するか、`guard policy relaxed` と伝えて
維持します）。relaxed または off のもとでは、編集は維持され、`CHANGE_ACCEPTED`
（`swarm-batch`）として一度記録され、マージは `note:` 行を 1 行表示します。最後のマージの後に
行われた編集についても、ステージの完了時に同じことが当てはまります。

### `aidlc engine swarm allow-change` - a change the person asked for in the main checkout

```bash
aidlc engine swarm allow-change <file>... --message <id>
```

strict の Guard Policy のもとでは、ユニットを並列でビルドしている間、メインの
チェックアウトはユニットの land によってだけ変わるため、land のたびにそれを確認できます。
利用者がそこで別のファイルの変更を頼むと（「README のタイポも直して」）、
plan-approval ガードによるその書き込みの拒否が、利用者が頼んだメッセージとともに
このコマンドを示します。このコマンドはメッセージを証明し、現在の Code Generation の試行について、ファイルを利用者の言葉とともに
`MAIN_CHECKOUT_CHANGE_ALLOWED` として記録し、利用者向けに `notice` 行を 1 行
表示します。その後、書き込みが行われ、land、次のバッチの準備、ステージの完了では
その変更が維持され、それぞれがそのメッセージとともに `CHANGE_ACCEPTED` として記録されます。
AI-DLC 自身のファイルや、ビルド中のユニットがプランで指定しているファイルは拒否されます（その変更は
ユニット自身のチェックポイントを通じて行います）。チームがメモリで保持する strict の Guard Policy のもとでは、
何も変更せず、それがどこで設定されているかを 1 行で伝えます。

### `aidlc engine swarm take-over` - build in this chat the Units another chat started

```bash
aidlc engine swarm take-over --batch <n> --message <id>
```

1 つのチャットからユニットを並列でビルドしている間、同じマシンの別のチャットでの `next` は
"Your other chat started building these Units."（別のチャットがこれらのユニットのビルドを始めました。）と伝え、それらについて実行するものを
何も渡しません。利用者がその 2 つ目のチャットに代わりにビルドするよう頼むと、
`next` はこのコマンドのために利用者のメッセージを示します。このコマンドはそれを証明し、
このチャットとともに `SWARM_TAKEN_OVER` として記録し、その後 `next` はこのチャットに
バッチを渡します。最初のチャットには "Your other chat took over building these
Units."（別のチャットがこれらのユニットのビルドを引き継ぎました。）と伝えられます。利用者がチェックポイントで差し戻したバッチは、
再び準備されるまで、どのチャットにも保持されません。引き継げるのは、もう一方のチャットがビルドを開始したか、
最後に引き継いだ後に送られたメッセージだけなので、持ち主が変わる前の言葉で
元に戻ることはありません。ツールがチャットを区別できない場合（そこでは、別のチャットのために
バッチを保持するものはありません）や、バッチが別のチャットからビルドされていない場合は、拒否されます。

### `aidlc engine bolt swarm-checkpoint` - approve a completed batch

swarm のバッチが落ち着いた後、エンジンは、別のバッチが始まる前に `swarm_checkpoint` を返すことが
あります。そのバッチ番号とユニットの一覧をそのまま使ってください。

```bash
aidlc engine bolt swarm-checkpoint --action status --batch <N> --units "<comma-separated Units>"
```

status が `ready: true` を報告した後でのみ、`swarm-checkpoint --action ask` を実行してください。
準備ができていないバッチは拒否されます。**Approve** / **Request Changes** を提示する前に、
現在のバッチ、正確なユニットの集合、フィンガープリント、
ユニットごとの `Command SHA-256` の集合について、1 回限りの質問を開きます。`checkpoint` と同じく、自身のセッションを見つけます。
`--session <id>` は上書きのためだけのものです。

```bash
aidlc engine bolt swarm-checkpoint --action ask --batch <N> --units "<Units>"
```
承認の質問では "Verified with `<full command>` (exit 0)" と表示してください。検証コマンドのツールの出力にある
完全な正規化済みの `command` を、省略せずにコードスパンにコピーし、
バッククォートはより長い区切りで保ってください。


このチェックポイントの質問に対する人の **Approve** / **Request Changes** の返信を、
そのチャットか新しいチャットで待ってください。独自の質問がない新しいチャットの
返信は、別のチャットが尋ねたチェックポイントの質問が最も新しい質問で、まだ開いている間は、
その質問のために保持されます。返信が許可するのは、一致するアクションだけです。
無関係な返信、エージェント自身の言葉、別の質問への返信では
許可されません。人が選んでいない `--user-input` を決して渡さないでください。利用者が選んだ
アクションだけを実行してください。

```bash
# Only after the human chose Approve:
aidlc engine bolt swarm-checkpoint --action approve --batch <N> --units "<Units>" --user-input 'Approve'
# Only after the human chose Request Changes and supplied feedback:
aidlc engine bolt swarm-checkpoint --action reject --batch <N> --units "<Units>" --user-input 'Request Changes' --reason '<human feedback>'
```

指摘のフラグ（`--fix-finding`、`--reject-finding`、`--reopen-finding`）は、
ユニットのチェックポイントと同じように機能します。バッチの却下では、それらが各ユニットの行に記録されます。

アクションは応答を消費します。変更されたチェックポイントには、新しい質問と
回答が必要です。ユニットのチェックポイントと同じく、`--message <id>` は、エンジンの別の質問が
画面に表示されている間に利用者が送った返信を記録します。`finalize` を再実行すると、どのセッションのものでも、
このインテントの開いているチェックポイントの質問と取得済みの応答がすべて取り下げられます。新たな検証とソースの
land の後、`ready: true` を確認してから改めて尋ねてください。`finalize` の前に取得した応答では、
新しい証拠を承認できません。自動の完了では、`human_required: false` の場合、`--user-input` を省略し、
`ask` も不要です。人による却下には、常にこの
準備済みの質問と回答の流れが必要です。準備ができているかどうかは、
完了したバッチの現在の証拠から判断されます。これには、各ユニットのネイティブの
`Command SHA-256` が、現在の
許可された Construction Verification Command と一致することも含まれます。バッチの承認も
そのダイジェストに結び付けられます。すでに承認されたバッチは、利用者が新しいコマンドを
承認しても承認を維持し、まだ承認されていないバッチは、そのコマンドで新たに検証する必要があり、
ダイジェストのない古いネイティブのレシートでは新たな検証が必要です。バッチの
クレームしたファイルや出力がチェックの後に変わった場合、strict の Guard Policy では
`changed_after_check: true` で準備ができていない状態となり、`ask` は利用者に
Request Changes だけを提示します。relaxed または off のもとでは、変更は維持され、
`CHANGE_ACCEPTED`（`swarm-batch`）として一度記録され、`ask` / `approve` は `notices` で
その行を返します。バッチ全体を作り直したり、合格をでっち上げたりするのではなく、
`errors` を解決してください。承認または却下の後には `next` を再実行してください。バッチの承認は、
ステージ全体の承認ではありません。後に続く完了専用のステージのディレクティブは、
別の本体、レビュアー、人への学びや承認の質問なしに、記録処理を片付けます。

Request Changes の後、`resume_existing: true` を持つ `invoke-swarm` ディレクティブは、
同じバッチと正確なユニットの集合を使います。却下によって以前の承認が無効になった場合は、
準備の前に、その修正について新たに Plan Approval を得てください。

```bash
aidlc engine swarm prepare --resume-existing --batch <N> --units "<exact emitted Units>"
```

worktree が残っている場合、ツールはそのソースを保持し、古いメタデータをアーカイブします。
ネイティブのソースの land によって worktree が削除された場合、ツールは land の証拠を検証した後、
すでに land された親のソースから新しい子を作成できます。どちらの経路でも、
却下の修正とその実際の承認の証拠が保持されます。同じインテント、対象、試行について承認が
あれば、セットアップの再試行に別の回答は必要ありません。`execution_allowed: true`（終了コード 0）を返す
`testing-posture verify` は、`ok: false` の場合でも、下げたフェンスのもとで承認後の続行を許可します。
この許可によって、古い試行の承認が復活することは決してありません。その証拠のない
子の欠落は拒否されます。マージされたすべての子が残っていると仮定したり、拒否された再開を
通常の prepare で置き換えたりしないでください。承認された開始点と異なるソースは、
作業を再開する前に調整し、承認を得る必要があります。

### Grouped Code Generation Plan Approval

swarm のバッチの複数のユニットで同時にプランの準備ができた場合、`next` はそれらについて
1 つの質問で尋ねます。各ユニットの要約とプランのパスに続いて、**Approve all**、
**Request Changes**、**I'll edit the files** が示されます。利用者は自分の言葉で答え、
エージェントは、利用者が選んだ選択肢を `aidlc engine log answer
--stage code-generation --checkpoint plan-approval --details "Approve Plan"`（または
`'Request Changes: <what they asked>'`、`"I'll edit the files"`）で記録します。一部のユニットについては `--units
"<unit>,<unit>"` を追加し、その後で残りを記録します。「すべて承認」は
すべてのユニットを承認し、ユニットを指定した変更（「billing を変更して:
Stripe を使って」）は、そのユニットだけをその言葉とともに差し戻し、残りを承認します。質問ボックスでの
"Approve all" の選択は、選んだ時点で記録されます。入力された
"Approve all" は、ほかの返信と同じくエージェントが記録します。どのユニットにも、
それぞれのプランに結び付いた個別の承認の記録が残ります。以前の記録済みバッチのコマンド
（`aidlc engine log decision|answer --stage code-generation --checkpoint
plan-approval --batch-file <manifest>`）は、更新前に始まったグループでの承認を
完了できるようにするためだけに残されています。エンジンの質問が開いている間は、
"Plan Approval is asked by the engine now. Run next, show the person
the question it returns, and end the turn."（Plan Approval は現在エンジンが尋ねます。next を実行し、返された質問を利用者に示して、ターンを終えてください。）として拒否されます。

承認されたユニットの一部が land された場合、`next` が保留中の集合を絞り込む中で、
残りの準備済みのワーカーは元の承認を維持します。`testing-posture verify` が `execution_allowed:
true`（終了コード 0）を報告した場合は、`ok: false` が編集された内容は承認されていないと正しく
報告している場合も含め、既存の worktree で続行してください。その続行について、`reason` は
新たな承認なしに続行するよう伝え、`approval_reason` には古くなった結び付きの詳しい理由が入りますが、
それによって実行が妨げられることはありません。既存のワーカーは、検証済みの親のインテントの
現在の plan-approval のフェンスを使い、後からの引き下げや引き上げも反映されます。
一部のバッチには、別の承認の回答も新たな `prepare` も必要ありません。これは、
チェックポイントの修正が準備された後にも当てはまります。中断された修正のセットアップは、
既存の現在の承認で再試行できます。準備に成功すると、以降の `next` のディレクティブから、
修正の準備を示すシグナルが取り除かれます。最初の
承認の後、同じ対象と試行に対するプラン、テストの手順、Testing Contract の編集は、
Guard Policy が `relaxed` または `off` であれば再承認なしに続行します。
`strict` のもとでは、編集されたプランについて再び尋ねられます。新しい試行には、引き続き独自の承認が必要です。元の
承認の証拠は維持し、編集された内容を承認済みとして説明しないでください。
人の Retry によってワーカーが明示的に破棄された場合、そのネイティブの破棄では、
再作成のために、コミットされた承認済みのベースラインを保持できます。置き換えたワーカーは、
バッチのほかのメンバーが続行中であるか、すでに land されている間も、同じ承認を維持できます。
ディレクトリの欠落や無関係な古い破棄の記録では、この回復は
許可されません。
質問がどのように提示されても、ユニットごとの承認は必須のままです。
[Construction の実行](../reference/03-orchestrator.md#construction-execution) を参照してください。

### `aidlc engine worktree restore` — recover files from a set-aside attempt

プロジェクトのメインのチェックアウトから実行します。

```bash
aidlc engine worktree restore --slug <slug> [--parked <stamp>] [--raw] [--repo <name|.>] [--intent <intent>] [--space <space>]
```

このコマンドは、`worktree discard` または承認済みの
`bolt abort --discard` によって保存されたファイルを回復します。`--parked` を指定しない場合は、
選択されたインテントの Bolt について最後に保存された `/head` を選びます。指定した場合は、その正確なスタンプを選びます。スタンプは UTC の
`YYYYMMDDTHHMMSSZ` で、衝突回避用の数値サフィックス `-N` が付くことがあります。最新の選択では
数値順に並べられます（`-10` は `-2` の後になります）。1 つのリポジトリにしか存在しない
正確なスタンプは、slug 全般の曖昧さを判定する前にそのリポジトリを選びます。
それでも選択が曖昧な場合は、`--repo <name>` で既存の兄弟 Git
リポジトリを、`--repo .` でプロジェクトルートを選びます。回復では、同じ slug の
`WORKTREE_CREATED` または `WORKTREE_DISCARDED` の監査の `Repo` フィールドに記録された
有効な Git リポジトリを、その兄弟の名前がシンボリックリンクであっても受け入れます。
フレームワークは、試行の worktree や退避を記録した場所そのものから回復してよいためです。
この受け入れは slug 単位です。ほかの slug の記録によって、この slug のリポジトリの集合が
広がることはありません。現在または過去のインテントのリポジトリ一覧に含まれているだけでは、
シンボリックリンクは受け入れられません。インテントの一覧の候補や、記録のない検出された兄弟は、
正規化したパスが正規化したワークスペースルートの直下にある、実在する直接の子ディレクトリで
なければなりません（`isWorkspaceRepoDir`）。任意のパスや、同じ slug の監査による
出どころを持たないシンボリックリンクの別名は拒否されます。restore と purge は、このセレクターを
現在のインテントのリポジトリ一覧とは独立して解決します。これにより、稼働中の worktree の
create/discard コマンドのセレクターが変わることはありません。ワークスペースの文脈を解決する
必要がある場合は `--intent` / `--space` を使ってください。どちらの回復コマンドも、選択や変更の前に、
未知のフラグと重複したフラグを拒否します。`--raw` は値を取らない restore 専用のフラグで、
purge では受け付けられません。

人が試行の回復を求めた場合、コンダクターはコマンド文字列ではなく、成功した
discard 付きの abort が保存した `restore_operation` を使います。この型付きの
`EngineInvocation` は、スタンプがわかっている場合、ルート `worktree` と引数
`["restore", "--slug", slug, "--parked", stamp, "--repo", repo ?? ".", "--intent", recordDirName, "--space", space]` を持ちます。
`aidlc engine worktree <args...>`（インストール済みの
`aidlc engine worktree` ルート）を呼び出し、列挙された各引数をそのまま
別々の argv 引数として渡してください。引数をつなげてシェルコマンドにしたり、slug だけの
選択を組み立て直したりしてはいけません。リポジトリのセレクターは常に含まれます。兄弟の
名前か、ルートの場合は `.` です。最後のセレクターは、アクティブなインテントが変わった後でも
所有するインテントを pin します。返された引数はすべて保持してください。

`restore_hint` は、その操作から `renderEngineInvocation` によって生成される、人に表示するための
任意のテキストで、ネイティブ/ソースの選択、ハーネスディレクトリの検証、シェルで安全な
引数のクォートが行われています。生成で例外が発生した場合（たとえば
ハーネスディレクトリが無効な場合）、ヒントは省略され、`restore_hint_error` に
理由が入ります。操作は引き続き利用できます。復元の提案は、生成されたヒントの有無ではなく、
`restore_operation` に基づいて行ってください。ヒントは
コンダクターの実行の入力ではなく、ヒントがないことは証拠のみの試行であることを意味しません。
これによって abort の引数、ガードの受け入れ、人の同意の要件が変わることはありません。
`worktree discard` は `parked_ref` と `parked_commit` を維持し、
`parked_stamp`、`parked_mode`（`snapshot`、`branch-tip`、`evidence-only` のいずれか）、
`parked_repo`（プロジェクトルートの場合は `null`、それ以外は兄弟の名前）を追加します。
`bolt abort` は `reason: "aborted"` を維持し、指定された `--reason` のテキストを
追加の `abort_reason` フィールドに返します。常に `parked_ref` を含み、
何も退避されなかった場合は `null` になります。`parked_ref` が null でない場合にのみ、`parked_stamp`、
`parked_mode`、`parked_repo` が追加されます。レビューの証拠だけが残っていた場合、discard は
`parked_mode: "evidence-only"` と `parked_commit: "-"` を報告します。abort は 4 つの
記述フィールドを維持しますが、`restore_operation`、`restore_hint`、
`restore_hint_error`、`parked_excludes` は省略します。

`snapshot` の場合、abort は
`parked_excludes: ["ignored files", "eol/text=auto normalization"]` を報告します。
`branch-tip` の場合は `["uncommitted files (no working tree existed)"]` を報告します。
コミット済みの作業しか保持できなかったためです。保存された名前空間はわかっているものの、
その discard の記述子が利用できない場合、フォールバックは、スタンプを厳密に解析できるときに限り
`parked_ref` から `parked_stamp` を導き、それ以外は `null` を報告します。
`parked_mode` と `parked_repo` は `null` に設定し、`restore_operation`、
`restore_hint`、`restore_hint_error`、`parked_excludes` は省略します。保存されたモードも
リポジトリもわからないためです。その代わりに、`recovery_hint` が人に doctor を実行して、
退避された試行とその正確な復元コマンドを一覧するよう求めます。このヒントは
単なる案内であり、実行可能な操作ではありません。
このフォールバックは、どのファイルが保存されたかを示すものではなく、復元を提案する根拠にも
なりません。名前空間が保存されなかった場合（`--discard` なしの場合を含む）、`parked_ref`
は `null` になり、`parked_stamp`、`parked_mode`、`parked_repo`、`restore_operation`、
`restore_hint`、`restore_hint_error`、`parked_excludes`、`recovery_hint` は含まれません。

restore は、ブランチ `restore/bolt-<id8>_<slug>-<stamp>` 上に
`.aidlc/restored/bolt-<id8>_<slug>-<stamp>` を作成します。稼働中の
`.aidlc/worktrees/bolt-<id8>_<slug>` のチェックアウトや `bolt-<id8>_<slug>` ブランチに触れることも、
中止されたライフサイクルを再開することも、レビューの権限を復活させることもありません。復元先のパスや
ブランチがすでに存在する場合は、上書きせずに拒否します。`/head` のない、証拠のみの
回復用 ref を選択した場合は、`--raw` を指定しても拒否されます。

```text
no restorable files were parked for <slug> <stamp>; only review evidence was kept
```

レガシーの復元は記録された名前を維持し、選択されたインテントの
正確な `WORKTREE_DISCARDED` の `Parked ref` による出どころを必要とします。
[Bolt の識別](../../core/knowledge/aidlc-shared/worktree-info-schema.md#bolt-identity) を参照してください。

| 選択 | 動作 |
|-----------|----------|
| 値なしの `--raw` | マーカーやレガシーの分類より優先し、保存された blob をバイト単位でそのまま書き込みます。`restore_mode` は `raw-requested` です |
| `/snapshot` マーカー | 保存された作業ツリーの blob をバイト単位でそのまま書き込みます。`restore_mode` は `snapshot` です |
| `/branch-tip` マーカー | フィルターやエンコーディング変換を含む、Git の通常のチェックアウトを使います。`restore_mode` は `branch-tip` です |
| マーカーなし（レガシー） | ツールが作成したスナップショットのコミットの識別情報を認識し、`legacy-snapshot` には生のバイトを、`legacy-branch-tip` には通常のチェックアウトを使います |

選択は表の順に行われます。生のバイトでの書き出しは、smudge/process
フィルターと `working-tree-encoding` を迂回します。以前の
`eol/text=auto` の正規化を元に戻したり、無視された未追跡ファイルを回復したりはしません。生のバイトで復元された
パスは、それ自身のフィルターのもとで変更ありと表示されることがあります。実行可能モードは
維持されます。シンボリックリンクは `core.symlinks` に従います。生のサブモジュールの gitlink は、
復元されたサブモジュールのチェックアウトではなく、空のディレクトリになります。必須のチェックアウト
フィルターが失敗すると、通常の復元は失敗します。`--raw` で再試行する前に、残っている
復元のチェックアウトとその復元ブランチを明示的に削除してください。失敗した生の
復元は、部分的なチェックアウトをその場に残し、そのパスを報告します。

成功時の JSON には、`restored: true`、`slug`、`parked_ref`、`worktree_path`、
`branch`、`reviewed_source_refs`、`raw_bytes`、`restore_mode` が含まれます。
`reviewed_source_refs` は、再び有効にされた ref ではなく、保持された回復の証拠を数えます。
`materialized` は `raw_bytes` が `true` の場合にのみ現れ、通常のファイルと
シンボリックリンクを数えます（サブモジュールの gitlink は除きます）。返された `worktree_path` を開いて、
必要なファイルを確認したりコピーしたりしてください。

Bun ベースのコピーインストールでは、`aidlc engine worktree` を
`bun .claude/tools/aidlc-worktree.ts` に置き換え、`.claude` をお使いのハーネスディレクトリに
読み替えてください。回復の手順は [ファイルを取り戻す](15-troubleshooting.md#a-bolt-attempt-was-set-aside-getting-the-files-back)
を、スナップショットの契約は [ステートマシン](../reference/12-state-machine.md)
を参照してください。

### `aidlc engine worktree purge` — remove recovery refs

```bash
aidlc engine worktree purge --slug <slug> [--parked <stamp> | --older-than <days>] [--repo <name|.>] [--intent <intent>] [--space <space>]
```

purge は、スナップショット/branch-tip のマーカーとレビュー済みのソースの ref を含む、
ローカルの回復用 ref を比較したうえで削除します。セレクターを指定しない場合は、選択されたインテントの
Bolt について保存されたすべてのスタンプを削除します。`--parked <stamp>` は正確なスタンプを 1 つ選びます。`--older-than <days>` は
小数を含む 0 以上の有限の日数を受け付け、そのしきい値より厳密に古いスタンプだけを
選びます。経過日数はスタンプの UTC の `YYYYMMDDTHHMMSSZ`
部分から計算され、衝突回避用のサフィックス `-N` は無視されます。コミットの日付は
影響しません。`--parked` と `--older-than` は併用できません。
共通の厳密なカレンダーパーサーは、存在しない日付や時刻を正規化せずに
拒否します。`--older-than` では、解析できないスタンプは削除されずに残り、
`skipped_unparseable` に表示されます。ちょうどしきい値の試行も
残ります。正確なスタンプの指定またはすべてのスタンプの purge では、解析できないスタンプも削除できます。

選択された試行のいずれかに復元されたチェックアウトがある間は、別の場所に移動された
チェックアウトも含め、purge は拒否します。先にそのチェックアウトを明示的に削除してください。稼働中の
Bolt のチェックアウトやブランチを削除することはありません。成功時の JSON は
`{purged: <number-of-refs>, slug, stamps: [...], skipped_unparseable: [...]}` です。
数は試行ではなく ref の数です。`skipped_unparseable` は常に含まれ、
`--older-than` が解析できないスタンプを飛ばした場合を除いて空です。
restore と purge は監査イベントを追加しません。

`/aidlc --doctor` と `aidlc doctor` は、保存された `/head` のエントリまたは実際のレビュー済みの
ソースの ref が存在する場合、通常の出力と詳細な出力の両方で、情報表示の **Parked attempts**
セクションを表示します。各エントリには、slug、正確なスタンプ、経過日数、
モード（`snapshot`、`branch-tip`、`legacy`、`evidence-only` のいずれか）、正規の復元先の
チェックアウトの有無、型付きの回復操作が含まれます。JSON では、すべてのエントリに
`purge_operation` があり、復元可能なエントリにのみ `restore_operation` があります。いずれも
ルート `worktree` を持つ `EngineInvocation` で、引数は `purge` または
`restore` で始まり、その後に `["--slug", slug, "--parked", stamp, "--repo", repo ?? ".", "--intent", recordDirName, "--space", space]` が続きます。
コンダクターは、各引数をそのまま argv としてそのルートを呼び出し、シェル向けに文字列を
つなげることはありません。任意の `restore_command` と `purge_command` は、人に表示する
ためだけの安全な表記です。生成で例外が発生した場合、対応する
コマンドは省略され、`restore_command_error` または `purge_command_error` が
理由を説明します。操作自体は残ります。証拠のみのエントリには、purge の操作と
そのコマンドまたはエラーのフィールドだけがあります。doctor は、前述した
slug 単位の回復用リポジトリの候補集合を同じように使います。移動されたチェックアウトは、doctor では
復元済みと表示されないことがありますが、purge は引き続きその Git の登録を確認します。
名前空間付きの試行は、インテントレジストリの UUID を通じて所有者を解決します。
レガシーの試行は、所有者の監査に、discard された正確な `Parked ref` があることを必要とします。
所有者が不明または曖昧な試行や、帰属のないレガシーの退避は省略されます。
doctor は同じ厳密なスタンプパーサーを使います。存在しない日付や時刻は、JSON では
`age_days: null` となり、人が読む出力では `unknown` と表示されます。
これらのエントリが警告や失敗を生むことはありません。ネイティブのコマンドを使わずに purge を実行する場合は、
restore について説明したコピーインストール用のプレフィックスを使ってください。

### `aidlc engine workspace codekb` - resolve the code knowledge directory

公開されている読み取り専用のクエリを使います。

```bash
aidlc engine workspace codekb --repo <repo>
```

アクティブなスペースの決定論的な
`aidlc/spaces/<space>/codekb/<repo>/` のパスを表示します。`--json` を付けると
`{space, repo, dir, record, intent, stage, staged}` が得られます。アクティブな記録、その
インテントの slug、リポジトリの Reverse Engineering のステージングフォルダーと、そこに
すでにあるファイルも含まれ（作業が選択されていない場合、`record` と `intent` は `null` です）、
パスはすべてプロジェクトからの相対パスです。このクエリは何も書き込まず、ディレクトリも作らず、監査
イベントも出しません。reverse-engineering ステージの本文も同じルートを呼び出すため、パスが
手作業で導かれることはありません。

### `aidlc-utility codekb-snapshot` - bind a scan to source and store generations

これは `/aidlc codekb-snapshot` コマンドではなく、**ユーティリティの直接呼び出し**です。

```bash
bun .claude/tools/aidlc-utility.ts codekb-snapshot \
  --repo <repo> --paths src/payments/,src/catalog/ --json
```

reverse-engineering のスキャンの直前に、このコマンドは共有の CodeKB の世代全体と、
スキャンが調べるパスに対するソースのフィンガープリントを取得します。ソースのトークンには、
利用できる場合は Git の作業ツリーのフィンガープリントを、Git の外ではバイト単位で正確なツリーによる
フォールバックを使います。要求された各パスの配下では、どちらも `.csproj`、
`.fsproj`、`.vbproj` ファイルの隣にある .NET の `bin/`、`obj/`、`out/` を除外し、
フォールバックはさらに `node_modules/` などの依存関係やキャッシュのディレクトリと、
`.DS_Store` などのツールの副産物のファイルも飛ばします。そのため、スキャン中の `dotnet build` で
無効になることはありません。`--paths` で指定したパスは常に読み込まれます。ワークスペースルートが
リポジトリのルートである場合は、`codekb-scope-diff`（後述）と同様に、どちらも AI-DLC 自身のファイルも
除外します。スペースとリポジトリ単位のロックにより、
2 つの値が並行する公開をまたぐことはありません。返される
`store_generation`、`source_fingerprint`、`paths` は
`codekb-publish` への入力です。`head_commit` は、ナレッジベースの Run Record に記される
コミットです（Git の外や最初のコミットの前は `unknown`）。

### `aidlc-utility codekb-publish` - guarded all-artifact publication

これは `/aidlc codekb-publish` コマンドではなく、**ユーティリティの直接呼び出し**です。

```bash
bun .claude/tools/aidlc-utility.ts codekb-publish \
  --repo <repo> \
  --staged <record>/.aidlc-engine/codekb-stage-<repo>/ \
  --paths src/payments/,src/catalog/ \
  --expect-store <generation> \
  --expect-source <fingerprint> \
  --json
```

ステージングされたディレクトリには、9 つの CodeKB 成果物がちょうど含まれていなければなりません。
公開では、同じスペースとリポジトリ単位のロックを取得し、2 つのスナップショットの値を再確認し、
タイムスタンプの最終的なスコープのフィンガープリントを検証したうえで、ロールバックとクラッシュからの
回復を備えた形で、候補全体を共有ストアに入れ替えます。並行する
CodeKB の公開があった場合は `CODEKB_STORE_CHANGED` を返し、何も公開せず、
後勝ちの上書きではなく、新たなマージのやり直しを求めます。strict な Guard Policy のもとで
ソースが変わっていた場合は `CODEKB_SOURCE_CHANGED`（または
`CODEKB_CANDIDATE_STALE`）を返し、新たにスキャンするまで何も公開しません。
`relaxed` または `off` のもとでは、スキャンは取得したとおりに公開され、コードを再度スキャン
するかどうかを尋ねる 1 行（`--json` では `change_notices`）が添えられます。

公開に成功すると、ユーティリティはステージングされたディレクトリを 1 回の操作で脇に名前変更し、
次に 9 つのファイルそれぞれを直前に公開したバイト列と照合してすぐに削除し、
最後に空になったディレクトリを再帰なしで削除します。JSON の結果は `"staged_removed": true` を報告します。
読み込んだ後に何かが変わっていた場合や削除に失敗した場合、ユーティリティは
削除したファイルを公開したバイト列から作り直してディレクトリを
元どおりに戻し（その間にステージングのパスが作り直されていた場合は、名前変更したコピーを
その隣に残し）、`"staged_removed": false` を報告し、残したディレクトリを
stderr に示します。

### `aidlc-utility codekb-scope-diff` - check the code knowledge base before a rerun

これは `/aidlc codekb-scope-diff` コマンドではなく、**ユーティリティの直接呼び出し**です。

```bash
bun .claude/tools/aidlc-utility.ts codekb-scope-diff --repo <repo>
bun .claude/tools/aidlc-utility.ts codekb-scope-diff --repo <repo> --compare <timestamp.md>
bun .claude/tools/aidlc-utility.ts codekb-scope-diff --repo <repo> --mint --paths src/payments/,src/billing/
bun .claude/tools/aidlc-utility.ts codekb-scope-diff --repo <repo> --check <timestamp.md>
```

reverse-engineering の再実行のガードです。codekb ストアはスペース単位で、
インテントをまたいで共有されます。完全な再スキャンはストアを置き換え、焦点を絞ったスキャンは
新しいナレッジを累積的にマージするため、ステージは先に次の確認を行います。

- **ステータスモード**（デフォルト）は、ストアの
  `reverse-engineering-timestamp.md` の Scope of Analysis ブロックを読み込み、分析済みのパスに
  対する内容のフィンガープリントを計算し直します。判定: `NO_STORE`（最初の
  スキャン）、`CURRENT`（分析済みのパスに変更なし。再利用しても安全）、`STALE`
  （分析済みのパスが変更された）、`UNVERIFIED`（フィンガープリントを計算できない。たとえば
  git の作業ツリーではない）、`UNKNOWN_SCOPE`（ストアがスコープの追跡より前のもの）、
  `SCANNED`（ステージのこの実行ですでにリポジトリをスキャンしてストアを公開済み。
  ステージが最後に開始された、ジャンプされた、または変更を求められて以降のリンクの連鎖全体が
  記録されている。再利用も再スキャンも不要）。
- **比較モード**（`--compare <incoming timestamp.md>`）は、入ってくる実行のスコープが
  ストアのスコープを包含しているかどうかに答えます。`COVERS`、または `NARROWER` と、
  深い網羅として主張されなくなった正確なパスとコンポーネントを返します。`COVERS` は
  焦点を絞ったマージの安全策で、マージされたスコープがストアの検証済みの網羅を維持していることを
  意味します。`kind: full` のスコープはリポジトリのルート（`./`）を含まなければならず、
  完全なストアを `NARROWER` の警告なしに包含できるのは、別の完全なスコープだけです。古いか
  未検証の、焦点を絞ったマージでは、以前の本文は維持され、検証できない分析済みのパスは
  `shallow.paths` に格下げされます。
- **発行モード**（`--mint --paths <a,b,...>`）は、アーキテクトが統合の時点でスコープの
  ブロックに貼り付けるフィンガープリントを表示します（git の作業ツリーの外やパス指定が
  無効な場合は `unknown`）。
- **チェックモード**（`--check <timestamp.md>`）は、ステージングされた候補など、公開用に書かれた
  タイムスタンプを読み込み、`VALID` とともにそのスコープのブロックが記録している内容と、
  フィンガープリントが現在のソースと一致するかどうか（`current`、`stale`、`unknown`）を、
  または `INVALID` とパーサーの理由を表示します。ストアを必要としないため、最初のスキャンでも
  公開前に候補を確認できます。

構造化された形式が必要な場合は `--json` を付けます。判定を出力に含めて常に終了コード 0 で終了します
（使用方法のエラーを除く）。何も書き込まず、監査イベントも記録しませんが、
1 つ例外があります。比較するファイルが、アクティブなインテントの記録の
`inception/reverse-engineering/` にある、比較対象のリポジトリについてのステージ自身の
`scope-draft-<repo>.md` である場合、比較は判定にかかわらずそれを削除し、
その旨を伝えます（"The scope draft has been removed."、または `--json` では
`"draft_removed": true`）。フィンガープリントは、
分析済みのパスに限定した一時的なインデックスに対する `git write-tree` で、
その隣に一時的なオブジェクトストアを置きます。リポジトリ自身の `.git` には何も書き込まないため、
ハーネスのサンドボックスが `.git` を読み取り専用にしている環境でも動作し、オブジェクトも残しません。
ワークスペースルートがリポジトリのルートである場合は、スキャンが決して読まない AI-DLC 自身の
ファイルを除外します。`aidlc/` ワークスペース、ハーネスの
ディレクトリ（`.claude/`、`.kiro/`、`.codex/`、`.cursor/`、`.opencode/`、
`.aidlc/`）、`.github/` と `.agents/` 配下の `aidlc` という名前のエージェント、フック、スキルと
生成されたステージランナー、インストールされたハーネスの投影が書き込むルートのすべてのファイル
（`.gitignore`、`AGENTS.md`、
`.mcp.json`、`.vscode/settings.json`、`opencode.json`、Cursor の
`install.ts` など）、そして `aidlc.settings.json` と `aidlc.settings.local.json` です。そのため、
AI-DLC の更新、設定の変更、インストールのコミットによってストアが
`CURRENT` でなくなることはなく、プロジェクト自身のファイルが変更された場合にのみ `STALE` になります。
codekb や状態の成果物が書き込まれても自身を無効にすることなく、ソースの作業ツリーの内容を
追跡します。履歴を書き換える rebase や squash にだまされることはなく、
編集を元に戻せば元のフィンガープリントに戻ります。

### `aidlc-utility detect` - read-only workspace scan

`bun .claude/tools/aidlc-utility.ts detect --json` は、ワークスペースのスキャン結果（プロジェクトの種類、言語、フレームワーク、ビルドシステム、宣言された git サブモジュールとその初期化状態を並べた `submodules` 配列）に加えて、解決されたスコープのディレクトリとスコープグリッドのパス、そして `proposalPath` を表示します。後者は、`validate-grid` が確認する前にコンポーザーがグリッドを書き込む、プロジェクトからの相対パスのファイル（`aidlc/spaces/<space>/intents/.aidlc-engine/composer-proposal.json`。git では無視されます）です。読み取りのみで、コンポーザーは現在のハーネスでスコープのデータがどこにあるかを知るためにこれを実行します。

### `aidlc engine workspace reclassify` - new project or existing code

`aidlc engine workspace reclassify --project-type <greenfield|brownfield> [--intent <slug>] [--space <name>] [--message <id>] [--then-rerun]` は、ワークフローの途中での `/aidlc --project-type` と、"this folder now has code"（このフォルダーにコードができた）という質問への回答の裏で動くコマンドです。フォルダーを再びスキャンし、種類を利用者が指定したものとして 1 回のロックされた書き込みで記録します。監査には先に `WORKSPACE_RECLASSIFIED` として記録されます。`next` が表示するステップは、種類を述べた利用者のメッセージの ID をエージェントが埋めるための `--message <id>` を、指定できるメッセージの ID と時刻とともに示します。エンジンは `config set` と同じようにそのメッセージを証明し、監査の行はそれを `Message Id` として、その言葉を `Person Reply` として記録します。言葉のファイルは書き込まれません。ワークフロー自体を動かすことはありません。現在のステージより前に Reverse Engineering を戻した場合は、次の `next` がそれを実行するやり直しのジャンプを示します。完了済みまたはアーカイブ済みの作業では、種類を記録し、プランはそのままにします。応答はワークフローを続行する `done` ディレクティブです（同じリクエストでさらに何かを求められた場合に使う `--then-rerun` では、同じ `next` をもう一度実行する `print` です）。利用者に伝えられる内容は、エージェントが話す次のステップとともに伝えられます。新しい種類と見つかったもの、コードができる前に実行されたステージとそのやり直し方、そして種類が変わった場合はその元に戻し方です。次にコードが文書化されることは、やり直しのジャンプが実行されるときに伝えられます（"Once you approve the code's documentation, we're back at Practices Discovery."）。そのため、同時に Reverse Engineering のスキップを求めた場合にこれが伝えられることはありません。

### `aidlc-workspace-sync` - clone and reconcile the declared repo set

これは `/aidlc workspace-sync` コマンドではなく、**ツールの直接呼び出し**です。
マルチリポジトリのワークスペースを、ワークスペースルートにある任意の `repos.json` マニフェストに
合わせて整合させます（
[リポジトリの集合の宣言](03-spaces-and-intents.md#declaring-the-repo-set-optional-manifest) を参照）。

```bash
aidlc system workspace-sync [--force]
```

整合処理はワークスペースのロックで直列化されます。このロックの稼働中の所有者が経過時間を理由に
回収されることはありません。その後、クローンと生成ファイルをステージングする前に、読み取り専用の
事前確認を実行します。生成された出力は、置き換えなしのリンクと、元に戻せる同一ファイルシステム内の
名前変更によってインストールされます。ステージング中に `.gitignore` や `aidlc.code-workspace` が変更された場合、
またはプランを読み込んだ後に `repos.json` が変更された場合、sync は古い状態を適用したり編集を
上書きしたりせずに中止します。正常に置き換えられた以前の生成ファイルは、確認用に、無視される
`.aidlc-workspace-sync-recovery-*` ディレクトリの下に残ります。中断された実行が残した
ステージング用の `.aidlc-workspace-sync-txn-*` フォルダーも無視され、削除してかまいません。このツールは、
`repos.json` で宣言されているがディスク上にないリポジトリをクローンし、ワークスペースの
`.gitignore` の管理ブロックを、リポジトリごとに 1 行の `/{name}/`（と、ツール自身の 2 つの
フォルダーのパターン）に書き換え、ルートと各子リポジトリを列挙した VSCode のマルチルートファイル
`aidlc.code-workspace` を書き込みます。Copilot のプロジェクトでは、このファイルに VS Code の
エージェントのリクエスト上限
`"settings": { "chat.agent.maxRequests": 200 }` も含まれます。これは設定がまだないファイルに
一度だけ追加されます。設定にすでにあるキーと値は維持されます（ファイルは
書き直されるため、その中のコメントは維持されません）。宣言された
`branch` は、新しいクローンでチェックアウトされます。すでにディスク上にあるリポジトリが
再クローンされたり切り替えられたりすることはありません。そこでの不一致は助言のままです。

孤立したチェックアウト（ディスク上にあるが `repos.json` にないもの）は実行をブロックし、
`--force` を渡し、かつツールがローカルにしかない状態がないことを証明できた場合にのみ、
アクティブな兄弟の集合から取り除かれます。この証明は、設定可能なステータスの
デフォルトより優先され、未追跡のファイルと無視されたファイルおよびディレクトリ（空の
ディレクトリを含む）、隠れたインデックスの状態、stash、ref と reflog、到達不能な Git
オブジェクト、リンクされた worktree、サブモジュール、LFS のオブジェクトストアを対象にします。キャッシュされた
リモート追跡 ref を信頼せずに実際の各リモートに問い合わせ、一致するオブジェクトグラフを
隔離されたプローブに取得するため、通知されているが提供できない
OID によって削除が許可されることはありません。ストレージやオブジェクトの alternates が
そのチェックアウトに依存しているローカルのリモートは、回復手段とはみなされません。

稼働中のリモートでの証明の後、チェックアウトはトランザクションの隔離領域に移され、
ローカルと稼働中のリモートでの完全な証明をもう一度受けます。隔離されたコピーは、再帰的に削除されずに、
無視される `.aidlc-workspace-sync-recovery-*` ディレクトリの下に保持されます。そのため、
すでにそのディレクトリを開いているプロセスが、証明と後始末の間に遅れて行った書き込みを
失うことはありません。保持されたチェックアウトと生成ファイルのバックアップを確認し、
不要になったら回復用ディレクトリを手動で削除してください。不確かな点があれば、手動での確認のために
ブロックします。終了コード: `0`
は完全に同期済み、`1` はブロックまたはエラー（稼働中のパスは変更なし）、`2` は同期済みだが
助言の警告が残っている（たとえば既存のチェックアウトのブランチの不一致）ことを示します。

マニフェストは任意で、ディスクの内容より優先されることはありません。インテントの作成では引き続き、
実際に存在する兄弟リポジトリが自動的に検出されるため、このツールは宣言された集合を再現して
整えるだけです。`--doctor` には、これに関する 3 つの助言の行があります（コミットされていない `aidlc/` の
記録、`repos.json` とディスク上の内容のずれ、古くなった管理対象の `.gitignore` ブロック）。ほかの
助言の行と同様に、doctor の終了コードを変えることはありません。

### Plugin state

`/aidlc plugin list` は、インストールされているプラグインの名前と、それぞれが有効かどうかを表示します。
`/aidlc plugin select [names]` が公開コマンドです。`select-plugins` はその
ユーティリティを直接呼び出す形で、`/aidlc select-plugins` コマンドではありません。
`bun .claude/tools/aidlc-utility.ts select-plugins` は、現在の選択
（`plugins` キーがない場合は `all enabled (no selection)`）と、既知の
プラグイン名を表示します。選択を設定するには、カンマ区切りの一覧を渡します。

```bash
bun .claude/tools/aidlc-utility.ts select-plugins test-pro
bun .claude/tools/aidlc-utility.ts select-plugins aidlc,test-pro
```

複数のハーネスがインストールされているプロジェクトでは、後述の `plugin sync` と同様に、`--harness-dir <dir>` で
変更するインストールを指定します（`aidlc engine plugin select aidlc --harness-dir .kiro`）。
指定しない場合、ネイティブの `aidlc` は、それを実行したチャットのツールのインストールを変更し、
ご自身のターミナルからは最初のもの（存在する場合は `.claude`）を変更します。

このコマンドは、名前を検証し、`.claude/tools/data/harness.json` を書き込み、新たに無効にしたプラグインがマージしたコントリビューションをコアのステージのソースから取り除き（構造的な追加は compose が書いたサイドカーによって、差し込まれた本文はその番兵マーカーによって取り除きます。再び有効にすると、次のセッション開始時に、Kiro CLI と opencode ではプラグインのフォルダーを指定して `aidlc engine plugin sync --plugin-root <folder>` を実行したときに復元されます）、無効なノードに `enabled:false` を付けてグラフ全体を再コンパイルし、ステージとスコープのランナーを整理/再生成し、生成されるスコープとステージの表（オーケストレータースキルの隣の `tables.md`）を、1 つのトランザクションで更新します。`aidlc` はコアです。これを省くと、常に有効な Initialization のステージを除くコアの機能が無効になります。アクティブなワークフローを立ち往生させる変更（そのスコープ、またはプランにある保留中の EXECUTE ステージを、新しい選択で無効になるプラグインが所有している場合）は、それぞれの依存関係を示して拒否されます。先にワークフローを完了するかアーカイブするか（一時停止中のワークフローも、再開時にはそのプラグインを必要とします）、プラグインを有効のままにしてください。ベースのプラグインを外し、その上に構築されたプラグインを残すと、そのプラグインは一時停止します。一覧は入力したとおりに書き込まれ、そのプラグインも無効にするコマンドとともにその旨を一度だけ 1 行で伝えます。ベースが無効の間、そのプラグインが追加するものは何も実行されず、`plugin list` には `paused` と表示されます。ベースを再び有効にすると、次のセッション開始時に（Kiro CLI と opencode では、プラグインのフォルダーを指定して `aidlc engine plugin sync --plugin-root <folder>` を実行したときに）元に戻ります。ベースを有効にする select は、どのプラグインが戻るかを一度だけ伝えます。

`/aidlc plugin sync`（`aidlc engine plugin sync`）は、ホストのプラグインの一覧から、インストール済みのプラグインの compose
フックを実行し、完了すると `plugin sync complete: <n>
plugin(s)` と表示します。繰り返し実行しても安全です。まずベースの
プラグインを compose し、次にその上に構築されたプラグインを compose します（プラグインはコアと、
マニフェストの `dependencies` で指定した最大 1 つのベースの上に構築されます）。ベースの上に
compose できないプラグイン（未インストール、プラグインのバージョン範囲外、別のプラグインの上に構築されている、
ベースが 2 つ指定されている）は保留されます。sync は完了の行の後に 1 行でその理由を一度だけ伝え、
ほかのすべてのプラグインを compose して終了コード 0 で終了し、`--json` では
`data.held` に列挙します。`aidlc engine plugin list` と doctor は、sync がそれを compose するまで
その行を表示し続けます。以前に compose された保留中のプラグインはそのまま残ります。2 回インストールされたプラグイン（再構築されたものの隣に古い `aidlc-<name>` のコピーがあるもの）は再構築されたほうから compose され、sync は実行のたびに古いコピーを削除するコマンドを示す 1 行を表示し、`--json` では `data.superseded` に列挙します。
セッション開始のフックが自身のプラグインだけを指定するホストでは、フックはまずそのプラグインの
インストール済みのベースを compose します。一覧に
無効なエントリがある場合や、プラグインのルートに `hooks/compose.ts` がない場合は、sync 全体を拒否して
問題のある箇所を示すため、中途半端に compose されることはありません。`--plugin-root <dir>`
は、`AIDLC_PLUGIN_ROOT` と同様に、compose するインストール済みのプラグインを指定します。Kiro IDE の
フォルダー投下のフックは、プラグイン自身のフォルダー `./.kiro/aidlc-plugins/<name>` を渡します。
`--harness-dir <dir>`
は、`AIDLC_HARNESS_DIR` と同様に、複数のハーネスがインストールされたプロジェクトで、compose 先の
ハーネスのフォルダーを指定します。Kiro CLI、Kiro IDE、Cursor、
Copilot、opencode はインストールされたプラグインの一覧を保持しないため、これらでプラグインのフォルダーを
指定せずに sync を実行しても何も compose されず、プロジェクトで compose されている
各プラグインについてのコマンドが示されます。

```text
$ aidlc engine plugin sync
aidlc engine plugin: Nothing synced: this tool keeps no list of your plugins, so name each one's folder: `aidlc engine plugin sync --plugin-root <test-pro's folder>`.
```

プロジェクトで別のハーネスが先に来る場合（たとえば `.kiro` の隣に `.claude` フォルダーがある場合）、
ネイティブの `aidlc` はそちらに compose してしまうため、コマンドは
ハーネスも指定します: `aidlc engine plugin sync --harness-dir .kiro --plugin-root <test-pro's folder>`。

そこで sync を促すすべての行（この行、変更したファイルのために拒否された sync、doctor の
compose 済みの機能と未コンパイルのステージについての修正）は、同じコマンドを示し、行がプラグインの
実際のフォルダーを知っている場合はそれを含めます。Claude Code がプラグインのレジストリ
（`~/.claude/plugins/installed_plugins.json`）を持つ前のネイティブの Claude Code のプロジェクトでは、
`host inventory unavailable; run sync through an installed plugin's host hook` と表示されます。
エンジンを再インストールまたはアップグレードした後は、毎回再実行してください。新しい
`dist/<harness>/` をコピーすると、出荷時のグラフとコアのステージのソースが元に戻るため、
以前に compose されたプラグインのグラフのエントリとコントリビューションのマージを
もう一度適用する必要があります。プラグインの SessionStart フックを持つホスト（Claude、Codex、Cursor、
Kiro IDE）は、次のセッション開始時に自動的に修復もされます。Kiro CLI では
明示的な sync が必要です。

`/aidlc plugin validate [path]` と
`/aidlc plugin build <harness> [outDir]` は、出荷されているスタンドアロンの
作成用ツールを最上位の CLI から利用できるようにします。検証のデフォルトは現在の
ディレクトリです。ビルドもプラグインのルートのデフォルトは現在のディレクトリです。ほかの場所から
呼び出す場合は `--plugin-root <path>` を渡してください。どちらも `--json` を受け付けます。
ベースの上に構築されたプラグインの検証には、そのベースのご自身のチェックアウトを `--with <base-root>` で
渡します。これにより、ベースのエージェントが主導するステージが、未確認として報告されずに
確認されます。

ターミナルから `aidlc plugin test [path] --install <project-root>`（コピーチャネルでは
`bun .claude/tools/aidlc.ts plugin test ...`）を実行すると、出荷されている
compose のテストが実行されます。`[path]`（デフォルトは現在のディレクトリ）のプラグインを、
`--install` で指定したプロジェクトの使い捨てのコピーに compose し、compose での欠落、
グラフの失敗、ファイルを変更する 2 回目の compose があった場合は終了コード
`1` で終了します。プロジェクト自体が変更されることはありません。そのプロジェクトに複数のハーネスがある場合は
`--harness <name>` を、ベースの上に構築されたプラグインには `--with <base-root>` を、
結果を 1 つのオブジェクトで得るには `--json` を渡してください。

### `aidlc-utility recompose` - in-flight plan flips

`aidlc engine recompose [--skip <slug,...>] [--add <slug,...>] [--sensors <on|off>] [--learnings <on|off>] [--summary-confirmation <on|off>] [--collaborators <on|off>] [--review <adversarial|advisory|none>] [--message <id>] [--reason <text>]` は、稼働中の状態ファイル上で、カーソルより先にある PENDING のステージのプランのサフィックスを切り替えます。2 つのフラグはどちらもカンマ区切りの一覧を受け付け、繰り返し指定できます。すべての指定が累積され、重複した slug は 1 回として数えられます。少なくとも 1 つの切り替えを指定し、一覧が空のフラグは省いてください。値のないフラグ、空の値、CSV の空の要素、未知のフラグ、余分な位置引数は使用方法のエラーとなり、切り替えは何も適用されません。設定のフラグは、切り替えと一緒に承認された設定を運びます。これらは同じ状態の書き込みで適用され、`CEREMONY_SET` または `REVIEW_CLASS_CHANGED` の行が `RECOMPOSED` の後に記録されるため、1 回の承認でプランが中途半端に変わった状態になることはありません。それらを承認した利用者のメッセージを `--message <id>` で指定すると、`config set` と同様に、エンジンがそのメッセージを証明した時点で設定は `set by you` と表示されます。設定だけを変える場合は `config set` を使います。`--reason <text>` は、プランが変わった理由を `RECOMPOSED` の行に記録します。ジャンプによってスキップされたステージがプランに戻されるとき、エンジンがこれを渡します。完了した作業（`Status: Completed`）では何も変更しません。拒否のメッセージは、最後のステージの後に求められたステージを実行するプランを示し、そのいずれかに切り替える（`/aidlc --scope <plan>`）と、そこで作業が再開されます。

```bash
# Equivalent ways to skip both stages after approval:
aidlc engine recompose --skip market-research --skip team-formation
aidlc engine recompose --skip=market-research,team-formation

# Add both pending stages back:
aidlc engine recompose --add market-research --add team-formation
```

`mvp` プランで実行中で Feasibility が進行中の作業では、Rough Mockups だけをスキップしようとすると
拒否されます。Refined Mockups がその出力を必要とするためです。両方をスキップすると通り、
両方を追加し直すと元に戻ります。

```text
$ aidlc engine recompose --skip rough-mockups
{"error":"Recompose rejected by the strict validator:\n  - Stage \"refined-mockups\" requires artifact \"wireframes\" whose producer(s) [rough-mockups] are not on the \"recomposed mvp\" path. Strict (recompose) mode rejects a starved required input.\n  - Stage \"refined-mockups\" requires artifact \"user-flow\" whose producer(s) [rough-mockups] are not on the \"recomposed mvp\" path. Strict (recompose) mode rejects a starved required input.\nTo make the change, also add a stage that produces what is missing, or also skip the stage that needs it."}
$ aidlc engine recompose --skip rough-mockups,refined-mockups
Recomposed: 2 skipped (rough-mockups, refined-mockups), 0 added (none)
Stages in scope: 20
Completed: 4/20
$ aidlc engine recompose --add rough-mockups,refined-mockups
Recomposed: 0 skipped (none), 2 added (rough-mockups, refined-mockups)
Stages in scope: 22
Completed: 4/22
```

監査ロックのもとで実行され、残りのステージから必須の入力を奪う切り替えを拒否します。ただし、`/aidlc --skip` で指定したスキップは例外で、その後、その入力を読むステージはあるものから実行されます。（ほかにも次のものを拒否します。完了済み/進行中のステージの切り替え、カーソルより後ろのステージの切り替え（例外として、Construction の開始前の Reverse Engineering は、`next` がまだ戻ってそれを実行する間はスキップでき、既存のコードでは元に戻せます。また、プランがスキップするステージのうち、作業がちょうど通過したばかりで、その後のステージがまだ何も始まっていないもの、たとえば Requirements Analysis が承認された直後の User Stories も例外で、元に戻すことができ、`next` は戻ってそれを実行します）、保護されたステージのルーティングの基準点である Construction の最初の EXECUTE ステージをどちらかの方向に動かす切り替え、Status が Running ではないワークフローに対する recompose、そして自律的な Construction のもとで、利用者のリクエストが記録されていない recompose、または `AIDLC_UNATTENDED=1` で実行された recompose です。この拒否は、Bolt ごとに承認のために停止する方式に切り替える `aidlc-bolt.ts set-autonomy --mode gated` を示します。）そのうえで、派生する状態フィールドを再構築し、`RECOMPOSED` を出力します。ステージの拒否はいずれも、代わりにできることを示します。そのステージへのジャンプやその先へのジャンプ、`/aidlc --stage <slug> --single` による単独での実行、欠けている入力に関係するステージの追加やスキップ、そのステージを実行またはスキップするスコープへの変更です。直接入力するのではなく、`/aidlc --skip <slug>`、`/aidlc --add <slug>`、スキップされたステージへのジャンプ、ワークフローの途中での `/aidlc compose` を通じて使われます。`/aidlc compose` の質問がまだ開いている間にそこで承認した変更については、出力の最後にエージェント向けの行がもう 1 行付きます。`next` を実行するよう伝えるもので、これにより作業は同じターンで続行されます。

この作業でスキップしたステージを、AI-DLC が示す取り消し方法で、または後からご自身の言葉で元に戻すことも拒否されません。そのステージは、スキップ前のプランでの扱いに基づいて判断されます。

その場でスキップされたステージ（スキップした時点で開始されていたため、スキップ済みと表示されるもの）は、`/aidlc --add <slug>` またはご自身の言葉で再び追加するとプランに戻り、次に実行されます。既存のコードでの Reverse Engineering は Construction が始まっていない間に限り（実行後、作業は元いたステージに戻ります）、それ以外のステージは作業がその次のステージに入ったばかりの間に限ります。作業がそれより先に進んでいる場合、追加は拒否され、ステージを単独で実行する方法 `/aidlc --stage <slug> --single` が示されます。

### `aidlc engine scope save` - keep a plan as a scope

`aidlc engine scope save --name <name> [--keywords <word,...>] [--intent <slug>] [--space <name>]` は、選択された作業の現在のプランを、再利用できるスコープとして保存します。プランが実行するステージ、深さ、Guard Policy、5 つのセレモニーの設定（センサー、学び、要約の確認、プランの承認、コラボレーター）、レビューのレベルを永続的な記録 `aidlc/scopes/<name>.md` に書き込み、その後コンパイルするため、このハーネスではすぐに `/aidlc --scope <name>` が使えます。ほかのハーネスでは次の `graph compile` で取り込まれます。実行中の作業はそのままです。ほかのプランがすでに持っているヒントは、求められたとおりに保存され、そのプランを示す 1 行が添えられます。「このプランを quick-fix として保存して」と伝えるとコンダクターがこれを実行します。compose のゲートで **Approve and save as scope** を選ぶと、作業の作成の一部として同じ保存が実行されます（`--save-as`）。キーワードも指定した場合は、作成の直後に実行されます。

たとえば、`mvp` プランで実行中の作業では次のようになります。

```text
$ aidlc engine scope save --name quick-fix
Saved as scope quick-fix (20 stages, sensors on, learnings on, summary confirmation on, reviews adversarial).
Next time: /aidlc --scope quick-fix "<what to build>"
```

名前は英小文字、数字、単独のハイフンで構成し、英字で始まる 40 文字以内で、既存のスコープの名前であってはなりません。`--keywords` を指定すると、リクエストの言葉からスコープを推測できるようになります。各キーワードは英字、数字、ハイフンからなる 1 語で、`validate-grid --keywords` と同様に、ほかのスコープが主張しているキーワードと照合されます。指定しない場合、スコープは `keywords: []` で保存され、名前でのみ解決されます。greenfield の実行での reverse-engineering のスキップは保存されないため、このスコープは既存のコードベースに対しては引き続きリバースエンジニアリングを行います。`SCOPE_SAVED` を出力します。サブエージェントは実行できません。

### Creating a workflow with its own stage changes (`--skip` / `--add`)

`next --scope <scope> --skip <slug,...> --add <slug,...> -- "<description>"` は、この作業に限ってステージを除いたり追加したりした `<scope>` 上のワークフローを作成します。コンダクターは独自に組み立てたプランのためにこれらを渡しますが、ご自身で入力することもできます。プランを指定しないリクエストと一緒に入力した場合、これらはプランの質問における利用者の指定として扱われます。Go ahead は提案されたプランをこれらを反映して作成し、一覧から選んだ別のプランも同様です（プランがすでに除外または実行しているステージは、そのプランでは対象外になります）。Tailor a plan はこれらをコンポーザーに渡すため、表示されるプランに反映されます。各 slug はステージを指定していなければなりません。initialization のステージ、両方の一覧にあるステージ、スコープがすでに行っている変更（スキップされるステージの `--skip`）は、何かが作成される前に拒否されます。作業の状態には `Plan: <name>` が記録され（`--plan-name <name>`。コンダクターはコンポーザーが提案した名前とともにこれを渡します。英小文字、数字、ハイフンで構成し、指定しない場合は `tailored plan`）、スコープファイルは書き込まれません。コンダクターが **Approve and save as scope** のために渡す `--save-as <name>` を指定すると、作成時に、[`scope save`](#aidlc-engine-scope-save-keep-a-plan-as-a-scope) が行う保存と同じように、プランをその名前のスコープとしても保存します。使用済みの名前やスコープ名にできない名前は、何かが作成される前に拒否されます。実行中のワークフローでは、`/aidlc --skip <slug,...>` と `/aidlc --add <slug,...>` が `recompose` を通じて残りのステージをすぐに変更します。承認の質問はなく（ステージを指定したのは利用者自身のため）、何が変わったかと元に戻す方法を 1 行で伝えます。同じコマンドで入力したスコープや設定が先に適用されます。指定したステージがすでに始まっている場合（進行中、変更を求めた後の修正中、またはそのステージ自身の承認の質問で待機中）は、ステージの種類にかかわらずその場でスキップされ、作業は続行します。それがプランの最後のステージだった場合は作業が終了します（元に戻すには、`/aidlc --stage <slug> --single` で単独で実行します）。開いているゲートで同じメッセージに入力した承認も記録されるため、両方が一度に行われます（複数のステージを示した質問では、"skip Functional Design and approve the rest" のように伝えると、スキップしなかったものがまとめて承認されます）。Construction がユニットを 1 つずつ実行している間は、ユニットが取り組んでいるステップが、言葉の指定に従って、そのユニットだけ、または以降のすべてのユニットについてスキップされます（ユニットを指定しないスキップは、以降のすべてのユニットが対象です）。1 つのユニットについて元に戻すには `/aidlc --stage <slug> --unit <unit>` を使います。コードのステップをスキップしたユニットにはチェックポイントがありません。すべてのユニットを自ら処理するチームのメインのチェックアウトも、スキップを同じように解釈します。ユニットの巡回の後のステップや、Build and Test などその後のステージは、すべてのユニットについてプランから外れます。作業がちょうど通過したばかりのステージを追加した場合（"Approve, and add User Stories next" は先に承認として記録されます）、その後のステージがまだ何も始まっていなければ、追加が先に来た場合と同じように元に戻されて次に実行されます。プランが受け入れられない切り替えは、代わりにできることを示して拒否されます。

スキップの後、その 1 行が示す取り消し方法は、変更で指定した最も早いステージを作業が通過するまで（追加したステージの場合は、そのステージが始まるまで）、またはプランでその後に何もない場合は作業が完了するまで有効で、どちらなのかはその行が伝えます。Construction がユニットを 1 つずつ実行している間は、すべてのユニットについてプランから外したユニットの巡回のステップは、巡回の後の最初のステージが始まるまで有効です。すべてのユニットが終わるまで作業は巡回にとどまるためです。作業がまだ実行していない Reverse Engineering は、Construction の最初のステージが始まるまで有効です。その後は、スキップされたステージを `/aidlc --stage <slug> --single` で単独で実行します。直前のステージの開いているゲートで入力したスキップでは、この方法だけが示されます。そのゲートを承認すると（同じ返信でそうすることが多いため）、作業はすぐにそのステージを通過するからです。

`--depth` と `--test-strategy` は、`minimal`、`standard`、`comprehensive` のいずれかだけを受け付けます。`next` は、コマンドを示す前にそれ以外の値を拒否します。

### `aidlc-graph ars` - deterministic ARS scoring

`bun .claude/tools/aidlc-graph.ts ars --iae <s> --csu <s> --ve <s> --r <s> --ua <s> [--completed <csv>] [--project-type <t>] [--method <codekb|fallback>]` は、適応型コンポーザーの Autonomy Risk Score の計算を行います。帯域ラベル付きの重み付き合成値、LOW/MED/HIGH の構成要素の帯域、出荷時のコストの事前分布に対するステージごとの期待値によるふるい分け、グリッドの差分数による最も近い既成のスコープ、そして markdown として事前に描画された 2 つのゲートの表です。重み、帯域の境界、ステージのコストの事前分布、EV のしきい値といったすべての定数は `tools/data/ars-priors.json` から読み込まれるため、同じ 5 つのスコアからは常に同じ数値が出力されます。コンポーザーは証拠から構成要素のスコアを付け、掛け算を自分で行う代わりにこの出力をコピーします。ファイルに名前のないプラグインのステージは、自身の `ars:` frontmatter ブロックからふるい分けられるか（両方がある場合はファイルが優先されます）、`no-prior` の行になります。各行の `priorSource` がどちらかを示します（`shipped`、`stage`、`null`）。`--completed`（カンマ区切りの slug）は、すでに EXECUTE を実行したステージを導出したグリッドに残します。`--project-type brownfield|greenfield` は、コンパイル済みの `condition:` によってもう一方の種類のプロジェクトに限定されているステージ（現時点では brownfield 専用の Reverse Engineering）を除外します。Reverse Engineering は CSU ではなくプロジェクトの種類によって決まります。既存のコードで実行され、`--project-type` がない場合も実行されます（新しいプロジェクトでは作成時に除外されます）。また、ステージ自体が、再スキャンの前に以前のスキャンが残したコードマップを確認します。`--method` は、コンポーザーの構造的な証拠の出どころを示します。独自に調整したプランを組み立てている間は 5 つのスコアと方法が保持され、プランの作成時に最初の監査エントリ（`WORKFLOW_STARTED`: `ARS Scores`、`ARS Method`）に書き込まれます。`codekb` または `fallback` 以外の値では終了コード 1 で終了します。JSON の結果は stdout に出力されます。範囲外のスコア、未知のステージの slug、事前分布のスキーマ違反では終了コード 1 で終了し、黙ってフォールバックすることはありません。合成値はゲートにいる人のための助言の指標であり、決定論的なルーティングがこれに基づくことはありません。

```bash
bun .claude/tools/aidlc-graph.ts ars --iae 0.55 --csu 0.75 --ve 0.65 --r 0.50 --ua 0.55
bun .claude/tools/aidlc-graph.ts ars --iae 0.30 --csu 0.80 --ve 0.40 --r 0.20 --ua 0.10 \
  --project-type greenfield --completed intent-capture,scope-definition
```

### `aidlc-graph validate-grid` - arbitrary-grid dependency check

`bun .claude/tools/aidlc-graph.ts validate-grid [--proposal <path>] [--strict] [--project-type <t>] [--keywords <csv>] [--guard-policy <strict|relaxed|off>] [--report] [--matched <stock-scope> | --custom]` は、任意の `{"<stage>": "EXECUTE"|"SKIP"}` 形式の JSON のグリッドを検証します。`--proposal` を指定しない場合は、コンポーザーの提案ファイル、つまり `detect --json` が表示する `proposalPath` を読み込みます。提案は、コンパイル済みのすべてのステージをちょうど 1 回ずつ指定していなければなりません。ステージの欠落、未知のステージ、無効なアクションはエラーです。寛容モードは `validate-scope` と同じ動作をします（経路外にある必須の生成元は助言扱い）。`--strict` はそれを厳しく拒否します（recompose での姿勢）。`--keywords` は、付与される各キーワードを既存のスコープがすでに持つキーワードと照合します。共有されているキーワードは、もう一方のプランを示す助言になり（`The hint "bug" is also bugfix's; the plan proposal lists it under both.`）、グリッドは引き続き検証を通ります（キーワードが付与される場合、コンポーザーはゲートの前にこれを実行し、`scope save --keywords` はヒントを保存して同じ行を表示します）。`--guard-policy`（または `stages` の隣の `guardPolicy` メンバー。廃止された `--change-control` フラグと `changeControl` メンバーも引き続き解決されます）は、コンポーザーが提案した Guard Policy の値を確認します。`strict`、`relaxed`、`off` 以外はエラーで、メモリ層の `Mode: strict` のもとでの `relaxed` または `off` の提案はそのファイルを示して拒否され、受け入れられた値は `guard_policy` と `change_control` の両方として返されます。`stages` の隣の `scopeSettings` メンバーは、コンポーザーの 6 つのスコープ設定を確認します。`sensors`、`learnings`、`summary_confirmation`、`plan_approval`、`collaborators`（それぞれ `on` または `off`）と `review_cap`（`adversarial`、`advisory`、`none` のいずれか）をちょうど指定していなければならず、未知のキー、欠けたキー、それ以外の語はエラーです。受け入れられた値は `scope_settings` として返され、`summary.off` にはそれらが無効にするものが列挙されます。`--matched <stock-scope>` または `--custom` は、front/report の提案についてのコンポーザーの経路を指定します。どちらも `scopeSettings` と Guard Policy を必要とします。どちらの経路もスコープファイルは書き込みません。`--matched` は、その既成のスコープと異なるグリッドと、そのデフォルトより低い Guard Policy を拒否します（より厳しい値は作成時に適用されます）。それ以外の設定は異なっていてもかまいませんが、プランの基になるスコープでプランの承認が有効な場合は、無効にすることはできません（無効にできるのは利用者だけで、ゲートでご自身の言葉で行います）。検証を通った実行は、`routing`、`matched_scope`、そして作成時にこの作業に適用される型付きの変更である `creation_settings`（たとえば `{"learnings": "off", "review": "adversarial"}`）を返します。`--custom` は、プランの基になる既成のスコープとして、Guard Policy のデフォルトが提案の値以下で（`strict` の場合はどれでもよい）、ゲートに表示されないものを何も追加しない（walking skeleton がなく、プランの深さ以外のテスト戦略がない）、最も近いスコープを選びます。`--project-type greenfield` では、条件を満たすものがあれば、`existing_code: true` と印の付いたスコープ（`bugfix`、`refactor`、`security-patch` は印付きで出荷されています）よりも新しい作業向けのスコープを優先します。検証を通った実行は、`routing`、`base_scope`、`plan_changes`（そのスコープのグリッドをプランに変える `skip` と `add` のステージの一覧）、そのスコープに対する `creation_settings`、そして提案の `depth` メンバーがそのスコープのものと異なる場合は `creation_depth` を返します。`depth` メンバー（`minimal`、`standard`、`comprehensive` のいずれか）を必須とし、どの既成のスコープのデフォルトもそれ以下にならない Guard Policy と、initialization のステージをスキップするプランを拒否します。front の compose の間、検証を通った `--matched` または `--custom` の実行は、compose のステップが開いたリクエストに対して、その `creation_depth`、`creation_settings`、そのスコープのデフォルトより上の Guard Policy とともにそのプランを保持するため、独自のプランを承認すると、ゲートに表示されたステージ、深さ、設定で作成されます。`--matched` なしのすべての実行は、独自のプランの出発点となる `guard_policy` と `scope_settings` である `custom_start` も返します。これは、プランの基になる既成のスコープにかかわらず、`classic` スコープのものです（`classic` が有効なスコープでない場合は省略されます）。すべての実行は `matched_start` も返します。これは、一致したプランが引き継ぐ既成のスコープ（`--matched` が指定したもの、なければ最も近いもの）の `depth`、`guard_policy`、`scope_settings` で、コンポーザーはこれらをスコープファイルではなくバリデーターから得ます。このマシンでキルスイッチ（`AIDLC_DISABLE_*`。設定されたもの、または記録されたもの）によって強制的に無効にされる `on` のセレモニーにも助言が付きます。スコープには `on` が保存されますが、そのセレモニーは実行されないためです。結果には `nearest_stock` も含まれます。グラフやプラグインが作成したすべての既成のスコープを、提案からのグリッドの距離の順に並べたもので（`{scope, diff, differs}` の昇順。コンポーザーが作成したスコープは除外）、コンポーザーの一致か独自かの判定は、LLM が数え直した値ではなくバリデーターの数値に基づきます。`--report` は、コードの調査結果のレポートのためのコンポーザーの実行です。この場合、`nearest_stock` は `bugfix` と `security-patch` だけを順位付けし、`--custom` はこの 2 つからベースを選び、それ以外のスコープを指定した `--matched` はエラーになります。そのため、グリッドがたまたま近いだけのより軽いスコープ（レビュアーもプランの承認もない `express` など）にレポートが割り当てられることはありません。利用者がレポートのスコープを自分で指定した場合、コンポーザーは `--report` なしで実行するため、その選択が尊重されます。

`summary` は、プランが実行するステージと承認ゲートを数えます。そのうち `shown` の数、つまり Initialization の後のステージの数が、利用者が読むすべての行で使われる数です。`--project-type greenfield` では、新しいプロジェクトでは作成時にスキップされる Reverse Engineering を除外するため、プランの提案に表示される数は、作成時に表示される数と一致します。検証を通った `--custom` の実行は、基になる既成のスコープが実行するステージをプランが外す場合（作成時にスキップとして記録されるステージ。新しいプロジェクトでは作成時にスキップされるため Reverse Engineering は除外）や、プラン自身の変更によって、残すステージがそのスコープでは外さない必須の入力を欠く場合に、`skipped_inputs` を追加します。スキップされる各ステージを名前で示し、それが作るものを使う残りのステージを添えます。使う残りのステージがなければ空です（`[{"stage": "Units Generation", "used_by": ["Contract Design", "Functional Design", "Code Generation"]}, {"stage": "Deployment Execution", "used_by": []}]`）。プランの提案はこれを一度だけ 1 行で伝えます。既成のスコープ自身の作られない入力は、助言にとどまります。

### `aidlc-sensor` — inspect and fire Sensors

センサーは、ステージの出力に対する `Write` や `Edit` のたびに実行される決定論的なチェックです（[ルールと学習ループ](09-rules-and-the-learning-loop.md) と、リファレンスの [センサーシステム](../reference/07-sensor-system.md) を参照）。PostToolUse フックが自動的に起動します。このツールを使うと、センサーの一覧表示、説明の表示、手動での起動ができます。

Sensors のセレモニーが `off` の場合、フックによる自動的な起動と、ブロッキングセンサーの
ゲートでのチェックは飛ばされます。フックはインストールされたままで、明示的な `fire`
はここで説明する診断を引き続き実行します。

| サブコマンド | 動作 |
|------------|--------------|
| `list` | フレームワークのすべてのセンサー（`id`、`kind`、`description`）をアルファベット順に表示します |
| `describe <id>` | 1 つのセンサーのマニフェスト全体（コマンド、デフォルトの重大度、`matches` の glob、タイムアウト）を表示します |
| `fire <id> --stage <slug> --output-path <path>` | ファイルに対してセンサーを実行し、`SENSOR_FIRED` の行と、それと対になる結果の行を出力します |

手動での起動では、`SENSOR_FIRED` の監査行を出力した後、終端の行をちょうど 1 つ出力します。`SENSOR_PASSED`、`SENSOR_FAILED`、`SENSOR_COULD_NOT_RUN`（チェックのスクリプトがクラッシュした、開始できなかった、または判定を出力しなかった）、`SENSOR_BUDGET_OVERRIDE` のいずれかで、その後に簡潔な JSON の判定の行が続きます。失敗すると、`<record>/.aidlc-engine/sensors/<stage>/`（インテントの記録ディレクトリ内）に詳細ファイルが書き込まれます。センサーの結果にかかわらず、fire コマンドは終了コード 0 で終了します。Sensors が `on` の場合、ゲートへの進入は別途 `blocking` の結び付けを強制し、検証済みの合格を必要とします。指摘事項、利用できないツール、実行できなかったチェック、ディスパッチャーのエラー、不正な形式の判定、タイムアウトはすべて進入を止めます。対話的な上書きは、別途記録される `Fix findings` / `Override blocking sensors` の決定で、その後に人による正確な回答と、`--override-blocking-sensors --user-input "Override blocking sensors"` を使った再試行が続きます。自律モードでは上書きできません。書き込み時に起動した結果は助言のままです。フレームワークに同梱される 6 つのセンサーは、`claim-sources`、`required-sections`、`upstream-coverage`、`traceability`、`linter`、`type-check` です。

```
bun .claude/tools/aidlc-sensor.ts list
bun .claude/tools/aidlc-sensor.ts describe required-sections
bun .claude/tools/aidlc-sensor.ts fire required-sections \
  --stage requirements-analysis \
  --output-path aidlc/spaces/default/intents/<YYMMDD>-<label>/inception/requirements-analysis/requirements.md
```

### `aidlc-learnings` — the learning-gate tool

これは、セクション 13 の学びのステップのうち、決定論的な部分です。ステージの完了メッセージの後、その承認の質問の前に、そのステージで利用者がエージェントを修正していた場合、オーケストレーターはこれを使って各修正についての質問を表示し、その後、残すと決めたもの（と、残すよう頼まれたメモ。こちらは質問なし）を永続化します。通常は直接呼び出すことはありません（オーケストレーターが自身の質問の前後で両方のステップを進めます）が、これが出力する監査行の意味がわかるようにここで説明しています。質問の内容は [ルールと学習ループ](09-rules-and-the-learning-loop.md) を参照してください。

Learnings のセレモニーが `off` の場合、ワークフローはこれらのステップを自動的に呼び出す代わりに、
日誌と学びのゲートを省略します。

| サブコマンド | 動作 |
|------------|--------------|
| `surface --slug <stage-slug> [--correction <your change> \| --correction-file <file>]...` | エージェントが利用者の言葉から読み取った修正ごとに、"You asked me to <it>. Keep that for next time?" という質問を `Yes, for this project`、`Yes, for the team`、`No` とともに表示し（修正がなければ質問もありません）、選択が書き込まれる `selections_file`（作業ごとに 1 つ、その記録の中）を示します。ステージの `memory.md` の日誌は数えられますが、提示されることはありません。読み取り専用です |
| `persist --slug <stage-slug> --selections-json <selections_file>` | 確認された学び（確認された学びはプラクティスです）を `aidlc/spaces/<active-space>/memory/project.md` / `team.md` に書き込み（センサーに結び付く学びの場合は、プロジェクト層のセンサーのひな形を作って結び付けます）、`RULE_LEARNED` / `SENSOR_PROPOSED` を出力します。選択の対象である作業について `surface` が示した `selections_file` だけを読み込み、それ以外のパスは拒否します |

確認された学びは、現在のワークフローの次の該当するステージ（ルールファイルは各ステージの配信の前に読み込み直されます）と、以降のすべてのワークフローからエージェントに届きます。新しいセンサーは、次のステージグラフのコンパイル時に結び付けられます。

`surface` は、マシンローカルの `runtime-graph.json` がコンパイル済みであればそこから日誌の場所を
特定し、そうでなければ同じパスを自分で導きます。
後者はワークフローの最初のゲートでの通常の状態であり、新しくクローンした直後の状態でもあります。その場合は、
グラフを再構築する `aidlc engine runtime compile` を示すメモを stderr に表示します。stdout の出力には
影響しません。

### `aidlc-runtime` — read the runtime graph

ランタイムグラフ（インテントの記録ディレクトリにある `runtime-graph.json`）は、このワークフローで実際に何が起きたかを記録するデータプレーンの記録です。どのステージが実行されたか、各 `memory.md` の日誌がどれだけ埋まったか、どのセンサーが起動し、それぞれ何を返したかが含まれます。構造的な `stage-graph.json` のランタイム版にあたります。フレームワークはステージの遷移のたびにこれを再コンパイルします。このツールを使うと、コンパイルを実行したり、1 つのステージの行を読んだりできます。

| サブコマンド | 動作 |
|------------|--------------|
| `compile` | `audit/` のシャードとステージごとの `memory.md` ファイルをたどり、`runtime-graph.json` を書き直します。遷移のたびにフックによって自動的に実行されます |
| `read <stage-slug>` | `runtime-graph.json` から 1 つのステージの行（タイムスタンプ、エージェント、メモリの内訳、センサーの起動、結果）を表示します |
| `summary [--json]` | グラフ全体に対する決定論的な集計を表示します。ステージ/フェーズの結果の集計、メモリのエントリ数、センサーの 5 状態の集計、取り込まれた学び、ワークフローの所要時間です。読み取り専用のセッションスキルが読むデータの出どころです |

```
bun .claude/tools/aidlc-runtime.ts read requirements-analysis
```

`runtime-graph.json` は gitignore の対象です。成果物の形式は [成果物リファレンス](14-artifacts-reference.md) を、スキーマ全体はリファレンスの [ランタイムグラフ](../reference/13-runtime-graph.md) の章を参照してください。

### `aidlc attest` — commit provenance

「このコミットの変更を所有するのはどのレビュー済みの作業単位か、そしてコミットされた内容はレビュアーが承認したものとまだ一致しているか」という問いに答えます。帰属は、コミットされた内容だけから導かれます。つまり、チェックアウトではなく git のツリーから読み出した、`audit/` のシャードにあるレビューの受領記録と、コミットされた `reviewed-source-*.tsv` の証拠ファイルです。そのため、手作業による通常の `git commit` でも、どのクローンでも、フックやコミットメッセージのトレーラー、push された ref がなくても動作し、同じコミットは常に同じように解決されます。

| サブコマンド | 動作 |
|------------|--------------|
| `resolve [<commit>]` | 読み取り専用です。コミットの第一親との差分（デフォルトは `HEAD`）をレビュー済みのユニットに帰属させ、変更された各パスを分類します。`verified`（コミットされた内容がレビュー済みの内容と等しい）、`drifted`（レビュー済みだがその後編集された）、`unattested`（どのユニットもクレームしていない）、`unverifiable`（証拠がない、改ざんされている、または gitignore されたローカルのスナップショットにしかない。安全側に失敗）、`indeterminate`（受領記録が曖昧。安全側に失敗）、`excluded`（フレームワークのシェル/記録のパス。範囲のベースのツリーですでに確立されたハーネスのシェルであり、変更によって持ち込まれたマニフェストや、チェックアウトにだけインストールされたシェルではないもの）です。JSON のレポートを stdout に出力します。`--commit <rev>` は位置引数のフラグ形式として受け付けられます |
| `resolve --diff <base>..<head>` | 任意の範囲に対して同じ分類を行います（`...` はマージベースを使い、マージリクエストの意味に一致します） |
| `resolve … --fail-on drifted,unattested,unverifiable,indeterminate` | いずれかのパスが指定したステータスのいずれかに一致すると、終了コード 3 で終了します。ゲートとして使う形式です。4 つのうち任意の部分集合を受け付けます。**未検証のパスを通すつもりでない限り、4 つすべてを指定してください**。`unverifiable` を省くと、レビュー済みの内容を何も確認できなかったパスが通ります |
| `resolve … --record-ref <ref>` | テスト対象のコミットではなく、`<ref>` のツリーから受領記録と証拠を読み込みます。変更が自身の承認を用意できないよう、変更が書き込めない ref（保護されたブランチ、記録専用の ref）を指定してください |
| `resolve … --require-trust <level>` | レポート自体の根拠が `informational` \| `reproducible` \| `independent` \| `signed` に達しない限り、終了コード 3 で終了します（`signed` は、判定の根拠となるすべての入力、つまり依拠する各受領記録の監査シャードと、それが選ぶ証拠ファイルが、署名付きのコミットで届いたことを意味します）。すべてのレポートには、どのレベルに達したかとその理由を示す `trust` オブジェクトが含まれます |
| `anchor [--commit <rev>]` | コミットがレビュー済みのクレームを land したことを記録する `SOURCE_COMMITTED` の監査行を追加します。補足情報にすぎません。`resolve` がアンカーを読むことはないため、アンカーのない手作業のコミットが不利になることはありません。デフォルトでは明示的な実行のみです。セッション開始時に最近のコミットもまとめて処理するには `AIDLC_SESSION_ANCHOR=1` を設定します |
| `anchor --reconcile [--max-commits <n>]` | 第一親の履歴（デフォルトは 100 コミット）をたどり、帰属できるコミットのアンカーを補完します。すでにアンカーのあるコミットとスウォームでマージされたコミットは飛ばし、帰属できないコミットは報告します |

```
# Gate a branch: three-dot (merge-base) range, all four failable statuses,
# receipts read from a ref the branch cannot write
bun .claude/tools/aidlc-attest.ts resolve --diff origin/main...HEAD \
  --record-ref origin/aidlc-records --require-trust independent \
  --fail-on drifted,unattested,unverifiable,indeterminate
```

このレシピの成否を分ける点が 3 つあります。`...`（ドット 3 つ）を使ってください。`origin/main..HEAD` は*先端同士*の差分を取るため、分岐点の後に `origin/main` に land されたものがこのブランチの変更として現れ、誤って失敗します。十分な履歴を取得してください。浅いチェックアウト（`actions/checkout` のデフォルトは深さ 1）には境界の親コミットがなく、`resolve` はツリー全体を黙って分類せずにエラーとして報告します。`fetch-depth: 0` を設定してください。そして、*報告*するのか*強制*するのかを意識して決めてください。`--record-ref` がない場合、自身の受領記録を書き込む変更は自分自身を検証できてしまいます。レポートはそのことを示しますが（`trust.level: reproducible`）、防ぐことはしません。2 つの信頼のフラグを外せば、正直な情報提供のレポートが得られます。

どちらの動詞も `--repo <name>`（マルチリポジトリのインテント）、`--space <name>`、`--intent <dir>` を受け付けます。各動詞は、もう一方の動詞のフラグを無視せずに拒否します。脅威モデル、信頼の段階、ステータスの意味については、リファレンスの [コミットの出どころ](../reference/20-commit-provenance.md) の章を参照してください。

### Session skills — report on a workflow

3 つの読み取り専用のスキルが、`aidlc engine runtime summary` の報告内容を読みやすい出力にまとめて示します。コマンドのように入力します。

| スキル | 動作 |
|-------|--------------|
| `/aidlc-session-cost` | 決定論的なコストの表示（所要時間、ステージの結果、メモリ、センサー、学び）。ターミナルのみ |
| `/aidlc-replay` | 非同期のレビュー向けの、読みやすいセッションの経過説明。ターミナルのみ |
| `/aidlc-outcomes-pack` | チーム向けの引き継ぎ文書。`OUTCOMES.md` を書き込みます |

3 つとも読み取り専用で（ステージを進めず、監査も出力しません）、すべての数値を `aidlc engine runtime summary --json` から得ます。お使いのハーネスがスキルをスラッシュコマンドとして提供しない場合（そのため `/aidlc-session-cost` が無効と報告される場合）は、そのコマンドを直接実行してください。同じコストの表示が得られます。詳しい手順は [セッション管理 § セッションスキル](11-session-management.md#session-skills) を参照してください。

### Issue report — when something went wrong

`/aidlc-issue-report`（Codex では `$aidlc-issue-report`）は、うまくいかなかった実行が残したものを、機密性のある詳細を取り除いたうえで集め、AI-DLC チームに送れるようにします。このプロジェクトについての AI-DLC の記録と AI ツール自身のログを読み込み、データ（`safe.json`）とその内容を説明する添え書きを `aidlc/diagnostics/issue-reports/<time>/` に書き込み、準備ができたと伝える前にそれらを確認します。何かを診断することはなく、自ら何かを送信することもありません。[スキル](17-skills.md) を参照してください。

---

## Environment Variables

AI-DLC は、以下のすべての変数を、ホストツールが AI-DLC を実行する環境から読み込みます。お使いのシェル、ホスト自身の
設定（`.claude/settings.json` の `env` ブロックなど）、またはエンジンが自身の子プロセスに渡す値です。
プロジェクトの `.env` ファイルは AI-DLC の設定ではありません。インストールされたエンジンはこれを読まないため、
クローンしたリポジトリがそのファイルを含めることで AI-DLC の動作を変えることはできません。

### `AWS_AIDLC_DEFAULT_SCOPE`

プロジェクトの暗黙のスコープをあらかじめ設定します。リゾルバーは、実際の環境変数（`.claude/settings.json` の `env` ブロックで指定された値を含む）、次に記録された `aidlc config flags --default-scope` の値、最後に `classic` の順に読み込みます。

**構文（`.claude/settings.json` 内）:**

```json
{
  "env": {
    "AWS_AIDLC_DEFAULT_SCOPE": "classic"
  }
}
```

**有効な値:** `enterprise`、`feature`、`mvp`、`poc`、`bugfix`、`refactor`、`infra`、`security-patch`、`classic`、`workshop`、`express`。

**優先順位:** 明示的な CLI フラグ > エージェントが提案し利用者が確認したプラン > 実際の `AWS_AIDLC_DEFAULT_SCOPE` 環境変数 > 記録されたデフォルトスコープのフラグ > `classic`。共有のデフォルトは `aidlc config flags --default-scope feature --project --yes` で記録し、このチェックアウトだけの場合は `--local` を使います。出荷時の設定の env エントリを含め、実際の環境の値はどちらの記録よりも優先されます。

**影響範囲:** ワークフローの初期化時にのみ適用されます。インテントの `aidlc-state.md` が存在した後は、状態ファイルが正となります。詳しい手順は [カスタマイズ § プロジェクトごとのデフォルトスコープ](13-customization.md#per-project-default-scope) を参照してください。

### Ceremony kill switches

`AIDLC_DISABLE_SENSORS`、`AIDLC_DISABLE_LEARNINGS`、
`AIDLC_DISABLE_SUMMARY_CONFIRMATION` は、それぞれちょうど `1` に設定すると、対応するセレモニーを強制的に `off`
にします。状態ファイルを書き換えることなく、インテントごとの設定とスコープの設定より優先されます。
優先順位、デフォルト、記録できる `aidlc config flags --bypass` の形式については、
[セレモニーの制御](#aidlc-sensors-learnings-summary-confirmation-ceremony-controls) を参照してください。

---

## Next Steps

- [スキルとランナーコマンド](17-skills.md) — 入力して使える `/aidlc-<scope>` と `/aidlc-<stage>` のランナー、および `--single` の動作
- [セッション管理](11-session-management.md): 再開、やり直し、ステージのジャンプの詳細
- [スコープ、深さ、テスト戦略](05-scopes-and-depth.md) — スコープの定義、ステージの対応、テスト戦略のレベル
- [トラブルシューティング](15-troubleshooting.md) — コマンドが期待どおりに動作しない場合
- [用語集](glossary.md) — コマンド、ユーティリティコマンド、スコープの定義
