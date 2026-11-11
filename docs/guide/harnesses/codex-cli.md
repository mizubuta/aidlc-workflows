# AI-DLC on Codex CLI

Codex ランタイムは、OpenAI **Codex CLI** ハーネス向けの、フレームワークのハーネス配布物の
1 つです。決定論的なコアは 1 つ、ハーネスは多数。エンジン、ステートマシン、監査ログ、グラフ、
スウォームのレフェリー、学びのゲートは、すべての配布物でバイト単位で同一であり、
異なるのはシェルだけです。ソース/開発用のツリーは、`bun scripts/package.ts codex` によって
`core/` + `harness/codex/` から、無視対象のローカル `dist/codex/` に**生成**されます。
決して手で編集しないでください。

プロジェクトの `.codex/config.toml` は、
[Codex configuration reference](https://developers.openai.com/codex/config-reference/)
に記載されているとおり、`developer_instructions` に Codex 用のオンボーディングを設定します。
信頼済みプロジェクトの Codex セッションは、ファイルを読まずにこのオンボーディングを受け取ります。
`.codex/onboarding.md` は、同じ内容を人間が読める写しとして保持します。
ハーネス中立のルート `AGENTS.md` ブロックはその写しを一覧に載せ、エンジンディレクトリが
異なる他のインストール済みハーネスとも共有されます。

新しい Codex セッションで、このハーネスの AI-DLC コマンドを尋ねてみてください。
答えは `.codex/onboarding.md` を読まずに `$aidlc` と `.agents/skills/` を挙げるはずです。
`$aidlc --doctor` は読み取り可能な写しを検証します。

### Root instruction overrides

Codex は各ディレクトリで、`AGENTS.md` より先に `AGENTS.override.md` を確認し、
最初に存在したファイルを選択します（
[Codex のファイル選択](https://github.com/openai/codex/blob/rust-v0.163.0/codex-rs/core/src/agents_md.rs)
を参照）。空または空白のみのオーバーライドは何の指示も提供せず、`AGENTS.md` に
フォールバックすることもありません。`AGENTS.override.md` という名前のディレクトリは選択されませんが、
指示ファイルへのシンボリックリンクは選択されます。

Doctor は Codex が選択するルートファイルを確認し、オーバーライドが優先される場合はその名前を示します。
AI-DLC ブロックが欠けているか変更されている場合は、助言的な警告を出します。
また、Codex がオーバーライドを読み込む場合でも、config や refresh を止めてしまう
`AGENTS.md` 内の破損したブロックを報告します。Doctor はどちらのファイルも編集しません。

`AGENTS.override.md` はあなたのものです。AI-DLC はこれを決して書き込まず、config と
セッション開始時の処理は AI-DLC のブロックを `AGENTS.md` に保ちます。そのブロックを
オーバーライドにコピーするかどうかは任意です。Codex はいずれにしても `.codex/config.toml` から
AI-DLC 自身の指示を受け取るからです。Codex にブロックも読ませたい場合は、
`AGENTS.md` の `<!-- BEGIN AI-DLC:agents -->` through `<!-- END AI-DLC:agents -->`（開始マーカーから終了マーカーまで）のブロックを、
自分のテキストを残したままオーバーライドにコピーし、アップデートでブロックが変わったら
もう一度コピーしてください。ブロックが破損または重複している場合は、さらにコピーを追加するのではなく、
そのブロックと壊れたマーカー行を置き換えてください。

## Prerequisites

- **Codex CLI >= 0.145.0** - それより前のリリースでは、ターン途中の自動コンパクションの後に
  compact ソースの `SessionStart` が遅延するため、ワークフローのミッションが復元されないまま
  モデルの継続が 1 回実行されることがあります。0.139.0 より前のリリースでは、さらに
  サブエージェントのロール帰属と、ハイフン付きのエージェント TOML の解決が信頼できません。
  `/aidlc --doctor` がバージョン指定について助言します。`codex --version` で確認してください。
- **bun** は、ソース/開発用の `dist/` 投影を生成または実行する場合にのみ必要です。
  ネイティブインストールとバージョン付きリリースのランタイムは、
  自己完結しています。
- **対象プロジェクトの Git リポジトリ** — Codex はプロジェクトの
  `.codex/hooks.json` を Git リポジトリ内でのみ検出します。ネイティブインストーラーと
  AI-DLC ランタイム自体は Git に依存しません。
- **モデルプロバイダー** — 同梱のプロジェクト `config.toml` はプロバイダーを選択しません。
  Codex は、プロバイダー、認証情報、モデル、コンテキストウィンドウ、推論努力度を
  `~/.codex/config.toml` から引き継ぎます。エージェントのロールは選択されたモデルを引き継ぎ、
  balanced のレビュアーは medium の推論努力度の上限だけを保持します。
  モデルプロバイダーはそのユーザーレベルのファイルで設定してください。Codex は
  プロジェクトレベルの `model_provider` と `model_providers` を無視します。その他の設定
  （`model` など）は、信頼済みプロジェクトの `.codex/config.toml` にあればユーザー設定より
  優先されるため、プロジェクトレベルのモデルキーは、意図して共有するオーバーライドの場合にのみ
  追加してください。

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
codex
```

インストーラーは、リリースのメタデータ、実行ファイル、全ハーネス対応のランタイムアーカイブを、公開されている SHA-256 チェックサムと照合して検証します。インストールされたランタイムには Bun、Node.js、Git は不要です。ハーネスの選択は `aidlc config` で行います。

Windows では、`install.ps1` をダウンロードし、その PowerShell ウィンドウで
`Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force` によりスクリプトを許可してから、
`& $installer` を実行します。アカウントのスコープ、User PATH への自動登録、`-NoModifyPath` については
[Windows へのインストール](../18-install-and-lifecycle.md#windows-powershell)
を参照してください。
エアギャップ環境向けのパッケージでは、同じ
`Set-ExecutionPolicy` の行を実行した後、Unix では
`install.sh --from <release-directory> --offline` を、Windows では
`& $installer -From <release-directory> -Offline` を使います。

`aidlc config` は Codex シェルを投影し、`.gitignore` と `AGENTS.md` の AI-DLC ブロックを
マージし、`.codex/config.toml`、フック、権限ルール、対応する `.codex/trust-seed.toml` を
書き込みます。それらのフックが実行される前に、Codex ではプロジェクト固有のフック信頼操作が
1 回必要です:

- `codex` を起動し、フックのダイアログで **Trust all and continue** を選びます（ダイアログを
  通り過ぎてしまった場合は、Codex で `/hooks` と入力し、`t` を押してすべてを信頼してから
  Esc を押します。これは同じチャットの次のメッセージから有効になります）。または
- `.codex/trust-seed.toml` の `<PROJECT_DIR>` をプロジェクトの絶対パスに置き換え、
  その `[hooks.state]` の一式をすべて `$CODEX_HOME/config.toml` にマージします。
  そのフックのパスに既存の一式がある場合は、重複した TOML テーブルを追加するのではなく
  置き換えてください。

生成された `.codex/config.toml` はプロジェクトスコープのままにしてください。
`~/.codex/config.toml` にマージしてはいけません。`developer_instructions` がこのプロジェクトの
AI-DLC オンボーディングを運んでいるからです。プロバイダーとモデルの設定はユーザー設定に置きます。
その後、Codex で `$aidlc --doctor` を実行してください。

生成される `sandbox_mode = "workspace-write"` はトップレベルの TOML 設定であり、
`[shell_environment_policy]` のメンバーではありません。AI-DLC はこれを
`developer_instructions` と並ぶ自身の項目の 1 つとして管理します。プロバイダーの回答では変更されず、
refresh ではあなたが設定した値が保たれ（リリースが別の値を同梱した場合は注記が出ます）、
キーが削除されていれば同梱の値が戻されます。ファイルの残りの部分はあなたのものです。
現在のプロバイダーを選択すると、帰属が明らかな旧来の Bedrock の既定値だけが削除されます。
サンドボックスのポリシーは変わりません。

### Versioned manual-copy alternative

[インストールとライフサイクル: コピーチャネル](../18-install-and-lifecycle.md#copy-channel)
の説明に従って特定リリースの `aidlc-copy-runtime-X.Y.Z.tar.gz` をダウンロードして展開し、
`RUNTIME_ROOT` を展開した `runtime/` ディレクトリに設定します。

1. 配布物をプロジェクトにコピーします（プロジェクトは **git
   リポジトリ**でなければなりません — Codex はプロジェクトの `.codex/hooks.json` を
   Git リポジトリ内でのみ検出します）:

   ```bash
   cp -r "$RUNTIME_ROOT/codex/.codex/"  your-project/.codex/
   cp -r "$RUNTIME_ROOT/codex/.agents/" your-project/.agents/
   cp -r "$RUNTIME_ROOT/codex/aidlc/"   your-project/aidlc/      # the workspace shell (spaces/default/memory) — a sibling of .codex/, not inside it
   ```

   `aidlc/` ディレクトリはワークスペースシェルです。エンジンが読む、事前ビルド済みの
   `aidlc/spaces/default/memory/` メソッドツリーを同梱しています。これは `.codex/` の
   **兄弟**ディレクトリなので、別途コピーしてください（または
   `$RUNTIME_ROOT/codex/` ツリー全体を一度にコピーします）。これがないと、`$aidlc --doctor` の
   "workspace shell ready" チェックが失敗します。

2. プロジェクトから、コピーに含まれるセットアップを 1 回実行します:

   ```bash
   bun .codex/tools/aidlc.ts config --from "$RUNTIME_ROOT" --harness codex
   ```

   これにより、最初のワークフローの前に、`AGENTS.md` と `.gitignore` の既存の内容の後ろに
   AI-DLC の行が追加されます（ファイルがなければ作成されます）。各インテントの
   `audit/` 配下にあるクローンごとの監査シャードは意図的にコミットされます
   （各クローンは自分の `<host>-<clone>.md` に書き込むので、同時に追記しても
   git の競合は起きません）。一方、ユーザーごとのカーソルとマシンローカルの
   ランタイム状態は無視対象のままです。

3. プロジェクトを信頼し、フックの信頼情報を事前に登録します。Codex は信頼されていない
   フックを決して実行しません（`--dangerously-bypass-hook-trust` フラグでも実行されません）。
   対話型の TUI セッションを 1 回実行してフックのダイアログで "Trust all and
   continue" を選ぶか、AI-DLC のソースチェックアウトから決定論的に事前登録します。
   固定バージョンの開発依存関係を一度インストールしてから、
   エントリを生成します:

   ```bash
   bun install --frozen-lockfile
   bun scripts/package.ts codex trust --project "/abs/path/to/your project"
   ```

   このコマンドは、`$CODEX_HOME/config.toml` にそのまま貼り付けられる
   `[hooks.state]` エントリを出力します
   （各ハッシュは、Codex のハッシュ方式に従ってフックのイベント、マッチャー、コマンド、
   タイムアウトを対象とし、パスは対象にしません。そのため、出力されるエントリは
   コピーした `hooks.json` に対して正確です）。コマンドは出力全体を
   TOML としてシリアライズするので、引用符付きのパス、空白、Windows のバックスラッシュは
   保持されます。フックマニフェストが `<project>/.codex/hooks.json` にない場合は、
   その正確なパスを明示的に渡します:

   ```bash
   bun scripts/package.ts codex trust \
     --project "/abs/path/to/your project" \
     --hooks-json "/abs/custom path/hooks.json"
   ```

   シェルではどちらの引数も引用符で囲んでください。`--hooks-json` は Codex の信頼 ID として
   そのまま使われます。エントリを生成した後で正規化したり置き換えたりしないでください。
   コマンドの標準出力をすべてユーザー設定に貼り付けます。同じ `hooks.json` のパスの
   エントリがすでにある場合は、その一式全体を置き換えてください。
   重複した TOML テーブルは設定全体を無効にするので、2 つ目のコピーを追記してはいけません。

   AI-DLC のアップグレードで `.codex/hooks.json` が変わったときは、新しいマッチャーを追加する
   アップグレードも含めて、この信頼コマンドを再実行してください。新しい Codex セッションを
   開く前に古いテーブルを置き換えます。そうしないと、Codex は起動時に変更されたフックについて
   再度尋ねます（Codex 0.160.0 で確認）。

4. `your-project/` に戻り（ステップ 3 は AI-DLC のソースチェックアウトで実行しました）、
   同梱の設定を信頼済みプロジェクトの `.codex/config.toml` に置いたままにします。
   `~/.codex/config.toml` にマージしてはいけません。その `developer_instructions` が
   このプロジェクトの AI-DLC オンボーディングを運んでいるからです。次のコマンドで検証します:

   ```bash
   cd your-project
   bun .codex/tools/aidlc-utility.ts doctor
   ```

バージョン付きランタイムは、ネイティブの `aidlc` コマンドを使います。Bun 形式の投影が必要な
フレームワーク開発者は、リポジトリをクローンし、
`bun install --frozen-lockfile` と `bun scripts/package.ts` を実行してから、
無視対象のローカル `dist/codex/` の出力を使えます。ソースチェックアウトの信頼情報ジェネレーターは
Bun 形式のフックコマンドに固有のものであり、ネイティブランタイムでは
使われません。

## Refresh and version skew

`aidlc update` はマシンのランタイムを更新しますが、プロジェクトは書き換えません。
`aidlc doctor` は、プロジェクトのランタイムスタンプを選択されたエンジンと比較します。
ワークフローとワークフローの間に、プロジェクトの refresh をプレビューしてから適用します:

```bash
aidlc config --dry-run
aidlc config
```

config はユーザーが所有する内容を保持し、フレームワークのファイルへのローカルな編集は
競合として報告します。ワークフローが開いている間に refresh すると、refresh は完了し、
開いている作業はそのまま続くと伝えます。アップグレードとロールバックはプロジェクトのファイルに
触れないため、ワークフロー中でも安全です。refresh によって Codex のフック ID が変わることがあるので、
Codex が求めたら、config の後で新しい信頼ダイアログを承認するか、対応する trust-seed の
エントリを置き換えてください。

## Use

オーケストレーターは `$aidlc`（または `/skills` → aidlc）の後にスコープまたは説明を続けて
呼び出します。コマンドは Claude ハーネスと同じです（`$aidlc --status`、
`$aidlc --config [section]`、`$aidlc --help` など）。ステージランナーは明示的な呼び出し専用です:
`$aidlc-domain-design`、`$aidlc-bugfix` など（37 個のランナーの説明でインデックスが
汚れないよう、暗黙のスキルマッチングから除外されています）。

## Harness differences vs Claude Code

- **ゲート**は、同梱の設定フラグで有効になっている場合は `request_user_input` ツールで
  レンダリングされ、そうでない場合は番号付きの散文にフォールバックします（番号または
  自由記述で答えます）。いずれの場合もゲートのセマンティクスはエンジンにあります。Codex は
  そのピッカーをまだ開発中としているため、同梱の
  `.codex/config.toml` はそれを有効にし、それに関する Codex の起動時の警告を
  無効にします。このプロジェクトで作業している間は、他の開発中の機能に関する警告も
  表示されなくなります。1 つのセッションで番号付き散文のゲートを使うには、
  `codex -c features.default_mode_request_user_input=false` で Codex を起動します。
- **ステータスライン（プロジェクトの選択）**: ワークフローの位置は
  `update_plan` ツール（`task-progress` ステータスライン項目）と
  `$aidlc --status` で示されます。AI-DLC のステータスラインは、誰かが選ぶまでオフです。セットアップ時に
  一度尋ねられ、`$aidlc --statusline on|off`（または `aidlc config --statusline
  on|off`）で、共有の `.codex/config.toml` にある AI-DLC の `[tui]` `status_line` 行を
  追加または削除します。これはプロジェクトで作業する全員に適用されます。Codex には
  プロジェクト内の個人別設定ファイルがないため、代わりに各自が Codex の `/statusline` で
  自分のステータスラインにタスクの進捗を追加できます。
- **時間切れになる質問ボックス**: Codex は、回答がないまま約 2 分経つと質問ボックスを
  閉じます。勝手に回答されることはありません。戻ってくると、
  同じ質問がチャットで待っています。
- **サンドボックス下の Git**: `workspace-write` は設計上、サンドボックス内で `.git` を
  読み取り専用に保ちます。対話セッションは自動的に権限を昇格し、同梱の
  `.codex/rules/default.rules` は `git worktree`/`commit`/`add` を事前に許可しています。
  ヘッドレス実行（CI）では
  `writable_roots = ["<main repo>/.git"]` が必要です — テンプレートは同梱の
  `config.toml` にあります（リンクされたワークツリーは `<main>/.git/worktrees/*` に解決されるため、
  メインリポジトリの `.git` でなければなりません）。
- **サンドボックス下のネットワーク**: 同梱の `.codex/config.toml` は
  `[sandbox_workspace_write]` の下で `network_access = true` を設定しているため、このプロジェクトで
  Codex がサンドボックス内で実行するコマンドは、あなた自身の Codex の既定値にかかわらず
  ネットワークに到達できます（パッケージのインストールや依存関係の取得のため）。
  このプロジェクトで閉じたままにするには、そこで `network_access = false` を設定します。
  refresh は設定した値を保持します。1 つのセッションだけなら、
  `codex -c sandbox_workspace_write.network_access=false` で Codex を起動します。
- **スウォームの下限 = 開発者サブエージェント**: 生成された Construction のユニットごとに
  `aidlc-developer-agent` サブエージェントが 1 つずつ並行して動き、それぞれがそのユニットの
  Bolt 用の隔離されたワークツリーで作業し、同じ決定論的なレフェリーが付きます。サブエージェントは
  あなたのサンドボックス内で、あなたの承認設定に従って実行されます。あなたが承認したテスト
  コマンドは AI-DLC 自身の `swarm check` を通じて実行され、同梱のルールがそれを許可しているので、
  確認なしにサンドボックスの外で実行されます。ここでは `AIDLC_USE_SWARM=1` に対応する Workflow ツールがなく、
  明示的に機能低下します（`SWARM_DEGRADED` が監査ログに記録されます）。
- **メインチャットで入力したものだけがあなたの返答として扱われます。** Codex は
  サブエージェントへのブリーフと、エージェントがそれに送るすべてのフォローアップを、あなたの
  メッセージと同じプロンプトフックを通じて送り、自身のレビュアー
  （`/review`、自動レビュー）についても同様にします。それらにはサブエージェントの ID か、
  自身のスレッドのトランスクリプトが付いています。AI-DLC は
  それらを決してあなたのターンとして数えません。承認を満たすことはなく、あなたの回答や
  変更要求として読まれることもありません。サブエージェント自身のスレッド（`/subagents`）に
  切り替えてそこで入力した場合、そのメッセージはワーカー向けなので、
  メインチャットの質問への回答にもなりません。
- **セッションのライフサイクル**: Codex には SessionEnd イベントがありません。閉じられなかったセッションは、
  次のセッション開始時に、推定の `SESSION_ENDED` 監査行として整合されます。コンパクション後、
  Codex は `source=compact` 付きの SessionStart を発行します。このサポートされたイベントにより、
  コンパクション後の最初の継続の前にワークフローのミッションが再注入されます。この即時の処理が、
  AI-DLC が Codex >= 0.145.0 を必要とする理由です。
- **成果物の監査の忠実度**: ヘッドレスの `codex exec` 実行では、モデルがシェルのヒアドキュメントで
  ファイルを書くことが多く、これは `apply_patch` フックのマッチャーを迂回するため、
  `ARTIFACT_*` 行がまばらになることがあります。対話型の TUI セッション（システムプロンプトが
  `apply_patch` を必須としている）が、忠実度の高い監査モードです。
- **AIDLC のルールレイヤー**は、ワークスペースルートの `aidlc/spaces/<active-space>/memory/` にあります（手で編集できる唯一のソースで、どのハーネスでも同一）。Codex にはそれらを常時取り込む仕組みがありません。エンジンがアクティブなスペースのファイルを読み、各ステップにそのルールを渡します（`config.toml` は `AIDLC_RULES_DIR` を空で同梱しており、エンジンはこれを「なし」と解釈します。そのため 2.11.0 からコピーされたプロジェクトは自身のツールで refresh されます。以前のリリースの行が指定しているスペースは、次の `aidlc config` で空になります）。Codex ネイティブの `.codex/rules/` ディレクトリは Starlark の権限ルールを保持するもので、AIDLC のメソッドとは別物です。
- **ウェルカムメッセージなし**: Claude ハーネスは、セッション開始時に `settings.json` の
  `companyAnnouncements` から Phases/Stages/Scopes のオンボーディングバナーをレンダリングします。
  Codex には同等の仕組みがありません。セッション開始の処理はワークフローのコンテキストだけを注入します。
- **MCP サーバー**: Codex は `config.toml`（プロジェクトの `.codex/config.toml` または `~/.codex/config.toml`）の
  `[mcp_servers.<name>]` テーブルから MCP の定義を読み込みます。
  必要なサーバーはそこに追加してください。同梱の設定は**何も**宣言していません（
  Claude ハーネスは `.mcp.json` で 5 つを同梱しますが、Codex は既定で 0 個です）。

## Regenerating

```bash
bun scripts/package.ts codex          # regenerate dist/codex from core/ + harness/codex/
bun scripts/package.ts --check        # build twice and byte-compare (every harness)
```

コアの `.ts` ファイルは、`core/tools/` と `core/hooks/` にあるソースとバイト単位で同一です
（`tests/unit/t150-codex-packaging.test.ts` で固定されています）。散文には、パッケージャーが
`.codex` に置換する `{{HARNESS_DIR}}` トークンが含まれます（加えて
`rules/` → `aidlc-rules/` の名前変更）。これが許可された唯一の変換の種類です。ライブの
エンドツーエンドのジャーニーは `tests/e2e/t-exec-codex-status.serial.test.ts` です（ゲート:
`AIDLC_CODEX_EXEC_LIVE=1`）。

## Next steps

インストールと信頼の設定は済みましたか？ 方法論はどのハーネスでも同じです。ハーネス中立の
章に進みましょう:

- [最初のワークフロー](../02-your-first-workflow.md) — 注釈付きのエンドツーエンドの実行例。
- [フェーズとステージ](../04-phases-and-stages.md) — 5 つのフェーズと 33 のステージ。
- [スコープ、深さ、テスト戦略](../05-scopes-and-depth.md) — 実行の規模を適切に決める。
- [用語集](../glossary.md) — すべての用語の定義。

その他のハーネス: [Kiro で AI-DLC を実行する](kiro.md) · [Cursor での AI-DLC](cursor.md) · [ハーネスファミリーの索引](README.md)。
