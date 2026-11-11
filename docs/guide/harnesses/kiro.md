# Running AI-DLC on Kiro

このページでは、Kiro（`aidlc config --harness kiro`）について、Kiro IDE と、v3 エンジン上の
Kiro CLI（`kiro-cli --v3`）を対象に説明します。Kiro CLI (V2)（`aidlc config --harness
kiro-v2`、`kiro-cli --v2` で起動）については、ページ末尾に専用のセクションがあります:
[Kiro CLI (V2)](#kiro-cli-v2)。

フレームワークのハーネスの 1 つであるこの Kiro ランタイムは、同じ AI-DLC の
方法論を [Kiro](https://kiro.dev/) の中で実行します。Kiro IDE 1.x と Kiro CLI v3
は 1 つのエージェントハーネスを実行し、同じ `.kiro/` ツリーを読み込みます。1 つの決定論的なコア
（ツール、33 個のステージファイル、プロトコル、ナレッジ、センサー、スコープ、ルール）は
すべてのハーネスでバイト単位で共有されており、異なるのはシェル（スキル、エージェントの提供面、
フックの配線、有効化）だけです。

ハーネス固有のオンボーディングは `.kiro/steering/aidlc-onboarding.md` にあり、
その `inclusion: always` frontmatter によって自動的に読み込まれます。ルートの
`AGENTS.md` ブロックはハーネスに依存せず、インストール済みの他のハーネスと共有されます。
ただしエンジンのディレクトリは別々である必要があります（Kiro CLI (V2) と Kiro のディストリビューションは `.kiro/` を共有できません。
Kiro CLI (V2) を使っているプロジェクトでは、`aidlc config --harness kiro` が `.kiro/` をその場でこのディストリビューションに切り替えます。ターミナルでは
先に確認を求め、`--yes` を付けると切り替えます。ターミナルがない場合や、`--quiet` または `--json` の指定時は何も変更せず、
`--yes` を付けたコマンドを表示します）。
AI-DLC 2.11.3 以前と、2026-11-04 および 2026-11-05 の 2.11.4 プレビューでは、2 つの Kiro の名前の読み方が逆です: Kiro は `--harness kiro-ide`、Kiro CLI (V2) は `--harness kiro` であり、`kiro-v2` は認識されません。`--harness kiro-ide` は、それらのリリースでもこのリリースでも Kiro を指します
（[ルートの統合と所有権](../18-install-and-lifecycle.md#root-integrations-and-ownership)）。

> [!IMPORTANT]
> **Kiro 上の AI-DLC は Claude Opus 4.8 で実行してください。** コンダクターは
> ステージごとに複数ステップの手順を進めます。明確化のための質問、成果物の生成、
> レビュアーによる確認、学びの手順、そして承認ゲートです。Opus 4.8 は
> 手順全体に従い、すべてのゲートで正しく一時停止します。性能の低いモデルは
> 任意のステップ（レビュアーによる確認と学びの手順）を飛ばし、ゲートを
> 急いで通過することがあります。ワークフローを開始する前に、チャットモデルを
> **Claude Opus 4.8** に設定してください。

## Prerequisites

- サインイン済みの **Kiro IDE 1.1.70 以降**、またはサインイン済みの **Kiro CLI 2.24.1 以降**
  （`kiro-cli --version`）。これらは、このディストリビューションで動作確認済みの最も古いビルドです。
  `/aidlc --doctor` は、PATH 上の `kiro-cli` がそれより古い場合に警告します。
  Kiro IDE のバージョンは読み取れません。
- チャットモデルとして **Claude Opus 4.8** を選択していること（上の注記を参照）
- **bun** は、ソース／開発用の `dist/` プロジェクションを生成または実行する場合にのみ必要です。
  ネイティブインストールとバージョン付きのリリースランタイムは
  自己完結しています。

> [!TIP]
> ソースから生成した `dist/` をインストールする場合、IDE がフックとツールに渡す PATH 上に
> bun が必要です。ターミナルから開いた IDE はそのターミナルの PATH を渡すので、
> そこで `bun --version` が動作すれば十分です。Dock やデスクトップアイコンから
> 開いた場合はそうならないことがあります: フックが bun を見つけられない場合は、そのようなターミナルで
> `bun .kiro/tools/aidlc.ts doctor` を実行してください。その `Runtime hook PATH`
> の行に、追加すべきディレクトリと追加先が示されます。その後 IDE を再起動してください。それまでは、
> そのターミナルから IDE を開いてください。

## Install

### Native channel (recommended)

まだ信頼していないプロジェクトフォルダーで Kiro IDE 自身のターミナルからこれらのコマンドを実行すると、
Kiro はまずそのフォルダーを信頼するかどうかを尋ねます。**Trust
Folder & Continue** を選ぶのは、自分のプロジェクトか確認済みのプロジェクトの場合だけにしてください。
それ以外の場合は **Cancel** を選んでください（[初回の実行](#first-run) を参照）。

```bash
tmp="$(mktemp -d)"
curl -fsSL \
  https://github.com/awslabs/aidlc-workflows/releases/latest/download/install.sh \
  -o "$tmp/install.sh"
sh "$tmp/install.sh"
rm -rf "$tmp"
cd your-project
aidlc config --harness kiro
aidlc doctor
```

インストーラーは、リリースのメタデータ、実行ファイル、全ハーネス用ランタイムアーカイブを、公開されている SHA-256 チェックサムと照合して検証します。インストールされたランタイムに Bun、Node.js、Git は不要です。ハーネスの選択は `aidlc config` で行います。

Windows では、`install.ps1` をダウンロードし、その PowerShell ウィンドウで
`Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force` によってスクリプトを許可してから、
`& $installer` を実行してください。インストール中に Kiro が開いていた場合（Kiro 自身のターミナルで
インストーラーを実行した場合を含む）は、最初の `/aidlc` の前に Kiro を完全に終了して
開き直してください: Kiro は起動時の PATH を保持するため、再起動するまでフック、
エージェント、他のターミナルは `aidlc` を見つけられません。
**Developer: Reload Window** では不十分です。アカウントのスコープ、User PATH への自動登録、`-NoModifyPath` については
[Windows へのインストール](../18-install-and-lifecycle.md#windows-powershell)
を参照してください。
エアギャップ環境向けのパッケージでは、同じ
`Set-ExecutionPolicy` の行の後に、Unix では
`install.sh --from <release-directory> --offline`、Windows では
`& $installer -From <release-directory> -Offline` を使います。

`aidlc config` は、プロジェクトを開く前に Kiro のシェルを配置します。ネイティブの
`aidlc engine *` の信頼許可と、下記のコピーしたプロジェクト向けに挙げている読み取り専用コマンドおよび再有効化コマンドは、
コンダクター（`.kiro/agents/aidlc-aidlc.md`）の権限に含まれて提供されます。コンダクターが作業を渡すすべてのエージェントでは、
それらのコマンドは拒否されます。一部のコマンドは
コンダクターにも許可されていません: `aidlc engine config set *` は作業の設定を変更し、
`aidlc engine adapter *` は IDE 自身のフックが実行する入口であり、`$`、バッククォート、`>`、`<`、`&`、`@(`、`@{`、改行を含む AI-DLC
コマンドは、その 1 つのコマンド以上のものを実行、展開、リダイレクトできてしまうためです。エージェントがこれらのいずれかを実行するときは、Kiro IDE
が先に確認を求めます。以前のリリースでは、これを
`.vscode/settings.json` の `kiroAgent.trustedCommands` にもマージしていました。Kiro IDE 1.x はこのキーをもう
読まないため、このエントリは削除してかまいません。最初のワークフローの前に、
[初回の実行](#first-run) に従ってください: Kiro IDE で `your-project/` を開き、フォルダーを信頼して
ウィンドウを再読み込みし、チャットパネルのエージェントピッカーで **aidlc** を選び、
`/aidlc --doctor` を実行します。Kiro CLI では、代わりにプロジェクト内で `kiro-cli --v3` を実行します。
`aidlc` エージェントで起動します。初回は、Kiro CLI 2.33.0 以降がこのワークスペースを信頼するかどうかを
尋ねます: このフォルダーの内容を把握している場合は **Trust, continue** を選んでください。
Kiro CLI はそうしたときにだけフォルダーのフックとエージェントを読み込みます
（**Don't trust** を選ぶと終了します）。

### Versioned manual-copy alternative

[インストールとライフサイクル: コピーチャネル](../18-install-and-lifecycle.md#copy-channel)
の説明に従って、特定のリリースの `aidlc-copy-runtime-X.Y.Z.tar.gz` をダウンロードして展開し、
`RUNTIME_ROOT` を展開した `runtime/` ディレクトリに設定します。

```bash
mkdir -p your-project/.kiro your-project/aidlc
# Safe on fresh installs; required when upgrading from v2.5.56 or earlier.
for retired_hook in \
  audit-logger block mint runtime-compile stop sync-statusline \
  enforce-approval-gate plan-approval-guard review-freeze state-transition-guard \
  terminal-command-guard rebuild-stage-graph sync-workflow-state \
  after-shell write-audit-log terminal-command
do
  rm -f \
    "your-project/.kiro/hooks/aidlc-${retired_hook}.json" \
    "your-project/.kiro/hooks/aidlc-${retired_hook}.kiro.hook"
done
rm -f \
  your-project/.kiro/agents/aidlc.json \
  your-project/.kiro/agents/aidlc-*-agent.json \
  your-project/.kiro/hooks/aidlc-*.kiro.hook
# The copy replaces .kiro/settings/cli.json; keep the project's own first.
if [ -f your-project/.kiro/settings/cli.json ]; then
  cp your-project/.kiro/settings/cli.json your-project/cli.json.before-aidlc
fi
cp -R "$RUNTIME_ROOT/kiro-ide/.kiro/." your-project/.kiro/
cp -R "$RUNTIME_ROOT/kiro-ide/aidlc/." your-project/aidlc/     # the workspace shell (spaces/default/memory) — a sibling of .kiro/, not inside it
cd your-project && bun .kiro/tools/aidlc.ts config --from "$RUNTIME_ROOT" --harness kiro
```

最初の削除ループは、v2.5.57 でのフック名の移行です。2 つ目は、古いディストリビューションが提供していた
Kiro CLI 形式のエージェント JSON と、IDE 0.x の
`.kiro.hook` の登録を削除します。これらは Kiro IDE 1.x では決して実行されません。上書きコピーでは
廃止されたファイルを削除できません。このコピーは `.kiro/settings/cli.json` も置き換えます。
このファイルは Kiro CLI に v3 エンジンと `aidlc` エージェントを使うよう求めるものです（どの Kiro CLI リリースがそこからエンジンを読むかは
下のリストを参照）。このファイルは `.kiro/` の他の部分と同様に AI-DLC が
管理します: 提供されたものと異なるコピーがある場合、`aidlc config` はそれを上書きせずに競合として
停止し、`--force` を付けると置き換えます。独自の Kiro CLI 設定は、代わりにユーザーレベルの `~/.kiro/settings/cli.json`
に置いてください: Kiro CLI はそのファイルも読み込み、プロジェクトのファイルが優先されるのは
そのファイルが設定しているキーだけです。
プロジェクトが `.kiro/settings/cli.json` に設定を置いていた場合、上のブロックはそれを
`cli.json.before-aidlc` として保存するので、そちらに移すことができます。新規インストールでは、どちらの削除も
何もしません。このクリーンアップの後、
`cp -R <src>/. <dst>/` 形式は、`your-project/.kiro` がすでに存在するかどうかにかかわらず、ツリーの**内容**を
コピーします。単純な
`cp -r "$RUNTIME_ROOT/kiro-ide/.kiro" your-project/.kiro` では、既存の `.kiro/` の中に 2 つ目の `.kiro` が
入れ子になり、IDE は新しいファイルを認識しません。

`aidlc/` ディレクトリはワークスペースシェルです。エンジンが読み込む、事前にビルドされた
`aidlc/spaces/default/memory/` メソッドツリーが含まれています。これは `.kiro/` の**兄弟**
ディレクトリなので、別途コピーしてください（または
`$RUNTIME_ROOT/kiro-ide/` ツリー全体を一度にコピーしてください）。これがない場合、`/aidlc --doctor` は
"workspace shell ready" チェックで失敗します。

コピーしたプロジェクトでは、`aidlc` エージェントは AI-DLC のエンジンコマンド
（`bun .kiro/tools/aidlc.ts engine ...`）、ツールスクリプト、読み取り専用
コマンド（`doctor` と `--doctor`（`--verbose` の有無を問わない）、`version`、
`--version`、`status`、`--status`、`config --help`、`config --show` と
`config <section> --show`（`--json` の有無を問わない）、
`config <section> --help`）、およびチェックの再有効化
（`config flags --clear-bypass <switch> --yes`）を、カードなしで実行します。ネイティブインストールでも
同じコマンドを `aidlc ...` として、やはりカードなしで実行します。それ以外の
`config` の変更（引数なしの `config` やガイド付きセットアップを含む）、マシンの AI-DLC インストールを変更するコマンド（`use`、
`update`、`rollback`、`uninstall`、`system`）、および `$`、
バッククォート、`>`、`<`、`&`、`@(`、`@{`、改行を含む AI-DLC コマンドでは、先に Kiro のカードが表示されます。
AI-DLC に開始を依頼する作業や、プランに加えるよう依頼する変更に
これらの文字が含まれる場合、エージェントは代わりに
`aidlc/.aidlc-request-text/` に書き込むファイルを通じてあなたの言葉を渡すので、それらの言葉によってカードが
表示されることはありません。

バージョン付きのランタイムは、ネイティブの `aidlc` コマンドを使います。Bun 形式のソースプロジェクションが
必要なフレームワーク開発者は、リポジトリをクローンし、
`bun install --frozen-lockfile` と `bun scripts/package.ts` を実行してから、
代わりに Git の無視対象であるローカルの `dist/kiro-ide/` 出力を使ってください。

提供される `.gitignore` は、ワークスペースのコミット対象と無視対象の区分を定めています:
ユーザーごとのカーソル（`aidlc/active-space`、`aidlc/spaces/*/intents/active-intent`）
とマシンローカルのランタイム（`aidlc/.aidlc-clone-id`、`runtime-graph.json`、センサーの
キャッシュ、`spaces/*/knowledge/.sources.local.json`）は追跡対象外のままとし、
共有の記録（メソッドメモリ、状態、監査シャード、成果物）は
git で共有されます。最後の行であるコピー自身のセットアップは、それらの行と AI-DLC の
`AGENTS.md` の担当部分を既存のファイルの内容の後に追加するか、プロジェクトにファイルがなければ
作成します。これにより、最初のワークフローの前に AI-DLC のルールが揃います。

インストールで提供されるもの:

- `.kiro/skills/aidlc/SKILL.md`: aidlc エージェントに対する AI-DLC の指示です。
  エージェントは、隣にある `question-rendering.md` と `composer.md` とともに、
  すべてのチャットの最初の行からこれらを保持しているため、依頼に対応する前に
  ファイルを読むことはありません。
- `.kiro/agents/aidlc-aidlc.md`: ワークスペースのエージェントセレクターに表示される同じコンダクターです
  （`aidlc-*-agent.md` のスペシャリストより前に並ぶようにこの名前になっています: Kiro は
  エージェントをファイル名順に並べるため、ピッカーと
  `/` メニューで `aidlc` が最初に来ます）。`invoke_sub_agent`（Kiro IDE）または
  `orchestrate_subagent`（Kiro CLI）を通じて委任します。これらは、各ペルソナを
  それぞれのツールと拒否ルールのもとで実行するディスパッチツールです。
- `.kiro/agents/aidlc-*-agent.md`: 14 個の委任用ペルソナすべてで、
  `tools:` の許可と `permissions.rules` を持ちます。委任された呼び出しでは、Kiro は
  ペルソナの拒否ルールを適用しますが、コマンドはコンダクターの許可のもとで実行します。そのため
  各ペルソナは、委任先が実行してよい AI-DLC コマンドを除いて、その許可を拒否します。
  委任先は、コマンドをどのように引用したり空白を入れたりしても、ワークフローを進めること、ステージの状態を変更すること、
  インテントやスペースを切り替えたり作成したりすることはできません。Bolt の
  worktree を作成、マージ、破棄、復元するのは pipeline-deploy ペルソナだけです。
  agent-v1 の JSON は提供されません。これらのシェル
  ルールは、AI-DLC 自身のコマンドと `bun --version` を確認なしで
  実行します。プロジェクト独自のテストやビルドのコマンドは引き続き確認を求めます。
- `.kiro/settings/cli.json`: `"chat.agentEngine": "v3"` と
  `"chat.defaultAgent": "aidlc"` を設定します。Kiro CLI の v2 エンジンは
  `.kiro/hooks/` の登録をどれも実行せず、フックはそれを内部から検出できないため、
  Kiro CLI は `kiro-cli --v3` で起動してください。Kiro CLI 2.33.0 は、エンジンを
  コマンドラインか独自の Kiro CLI 設定からのみ受け取ります: プロジェクトの
  ファイルのエンジンの値は無視し、`kiro-cli settings --workspace
  chat.agentEngine` は拒否します。Kiro CLI 2.24.1 から 2.32.x は、このファイルからエンジンを読みます。
  Kiro IDE はこのファイルを読みません。`kiro-cli acp` は
  `chat.defaultAgent` を読みますが、エンジンの値は読まないため、ACP クライアントは
  `kiro-cli acp --agent-engine v3`
  で起動し、
  `initialize` リクエストで `clientCapabilities._meta.kiro.hooks` を `{ enabled: true, v2: true }` として
  宣言する必要があります。そうしないとセッションでフックが実行されません（
  [Kiro CLI のフックが動かない](../15-troubleshooting.md#kiro-cli-hooks-not-running) を参照）。
- `.kiro/steering/aidlc-active-memory.md`: 常に含まれるステアリングで、
  コンダクターと委任先エージェントのためにアクティブなスペースのメモリテキストを保持します。AI-DLC
  はチャットごとに `aidlc/spaces/<active-space>/memory/` からこれを書き出し、
  git の管理外にします。編集するのはメモリファイルであり、このファイルは決して編集しないでください。チャットは
  開始時のコピーを保持するため、メモリを編集した後は、新しいチャットを開始するまで
  ステージのルールが各ステップとともに届きます。
- `.kiro/steering/aidlc-onboarding.md`: 常に含まれる、ハーネスのセットアップとコマンドの説明です。
- `.kiro/hooks/aidlc-*.json`: Kiro の v2 フック形式のフレームワークフックです。
  どちらの環境もセッション開始時にこれらを登録し、Kiro IDE では
  Agent Hooks パネルに表示されます。そこで 1 つをオフにすると、そのファイルに `"enabled": false` が
  書き込まれ、その状態でコミットされたファイルは、そのようなパネルを持たない Kiro CLI でも
  オフになります。`/aidlc --doctor` はそれを報告し、次の AI-DLC のステップは変更ごとに 1 回それを伝えます。
  また `aidlc-record-human-turn` がオフの間は、承認時に、フォルダーを信頼するよう求めるのではなく、
  Kiro IDE では Agent Hooks でそれを再度オンにするよう、Kiro CLI ではそのファイルの
  `"enabled": false` を `true` に変更してから Kiro を終了し、フォルダーで `kiro-cli --v3` を再起動するよう案内します。
  ファイルはセッション開始時に読み込まれるためです。AI-DLC がフックを代わりに再度オンにすることはありません。
  IDE 0.x の `.kiro.hook` 形式はもう提供されていません: Kiro IDE 1.x はこれを決して
  実行しません。

## First run

Kiro IDE は、フォルダーを信頼してウィンドウを再読み込みした後にのみ、フォルダーのフックを実行し、
`aidlc` エージェントを読み込みます。信頼されていないフォルダーは Restricted Mode で開きます:
ウィンドウ上部にバナーが表示され、ステータスバーに "Restricted Mode" と表示されます。
それまでは AI-DLC のフックは実行されず、`aidlc` エージェントは
エージェントピッカーに表示されず、最初の承認の質問ではあなたの返答を認識できません。

信頼するのは内容を把握しているフォルダーだけにしてください: 自分のプロジェクトか、
確認済みのプロジェクトです。信頼すると、フォルダーの `.kiro` フックがあなたのマシン上でコマンドを実行できるようになるため、
他人のプロジェクトは確認が済むまで Restricted Mode のままにしてください。

Kiro はこの信頼をもっと早い段階で求めることがあります。フォルダーを信頼する前に、たとえばそこで
`aidlc config` を実行するために Kiro IDE のターミナルを開くと、Kiro はまず
"Do you trust the authors of the files in this folder?" と尋ねます。内容を把握しているフォルダーであれば、
**Trust Folder & Continue** を選んでください。これはステップ 1 と同じ方法でフォルダーを信頼します。
`aidlc config` が終了したら、ステップ 2 から続けてウィンドウを再読み込みしてください。
それ以外の場合は **Cancel** を選んでください。

1. Kiro IDE で `your-project/` を開きます。ウィンドウ上部に Restricted Mode のバナーが表示され、
   フォルダーの内容を把握している場合は、バナーの **Manage** を選び、
   開いた Workspace Trust ページで **Trust** を選びます。
2. コマンドパレット（Ctrl+Shift+P、macOS では Cmd+Shift+P）を開き、
   **Developer: Reload Window** を実行します。
3. チャットパネルのエージェントピッカーで **aidlc** エージェントを選びます（
   [Kiro IDE のチャットで AI-DLC を始める](#start-ai-dlc-in-a-kiro-ide-chat) を参照）。
4. チャットで `/aidlc --doctor` を実行してセットアップを確認し、
   `/aidlc <description>` でワークフローを開始します。

Kiro CLI では、`your-project/` で `kiro-cli --v3` を起動します。このワークスペースを信頼するかどうかを Kiro が尋ね、
フォルダーの内容を把握している場合は **Trust,
continue** を選びます。その後、ステップ 4 に進みます。

Kiro CLI 2.24.1 以降が PATH 上にあり、セットアップを Kiro IDE の外（そのターミナル以外）で
実行した場合、セットアップはこの Kiro CLI のステップを最初に、Kiro IDE の
ステップをその後に示します。doctor の "AIDLC hooks have not run in this project yet" に対する修正方法も
同様です。

`kiro-cli` が PATH 上にある場合、セットアップでは Kiro CLI のセッションモデルも提示され、
個人の Kiro 設定（`~/.kiro/settings/cli.json`）に保存されます。これは
Kiro CLI の設定です: Kiro IDE は自身のモデルピッカーで選んだモデルを使います。
この行では、セットアップはモデルだけを保存し、effort は決して保存しません。また Kiro auto より
推奨するモデルはありません: Enter を押すと Kiro auto のままになり、Kiro IDE 自身のターミナルでは
推奨セットアップは質問せず、Kiro auto のもとでは各ホストがどこでモデルを選ぶかを
示します（Customize ではステップ 2 で引き続き質問します）。
`aidlc config models --session-model <id>` で後から保存できます。

チャットメッセージを送信した後に doctor が "AIDLC hooks have not run in this project yet" と報告する場合は、
ステップ 1 から 3 を繰り返してください。
[トラブルシューティング: Kiro IDE のフックが動かない](../15-troubleshooting.md#kiro-ide-hooks-not-running) を参照してください。

## Usage

Claude Code ハーネスと同じです: `/aidlc <description>` で
ワークフローを開始し、`/aidlc --status` で現在位置を確認できます。`/aidlc --doctor`、`--stage`、
`--phase`、`--depth`、`--test-strategy`、`--config [section]` もすべて動作し、
ステージごと（`/aidlc-domain-design`）およびスコープごと（`/aidlc-feature`）のランナー
スキルもインストールされます。init コマンドはありません。提供されたシェルが
ワークスペースの土台を用意し、最初の `/aidlc` で AI-DLC が自動的に最初のインテントを作成します。

### Start AI-DLC in a Kiro IDE chat

Kiro IDE は、新しいチャットを毎回 `aidlc` ではなく、自身の **Default** エージェントで開始します。
`.kiro/settings/cli.json` の `"chat.defaultAgent": "aidlc"` の行は Kiro
CLI の設定です: これは `kiro-cli` のセッションでのみ `aidlc` をデフォルトのエージェントにするもので、
Kiro IDE はこのファイルを読みません。そのため、Kiro IDE で新しいチャットを開くたびに次のようにします:

1. チャットパネルのエージェントピッカーを開き、**aidlc** を選びます（そこにある
   AI-DLC のエージェントの最初のもので、説明は "AI-DLC. Choose this agent in
   the agent picker" で始まります）。この
   エージェントは、AI-DLC のコマンドを確認なしで実行させ、AI-DLC がスペシャリストのエージェントを
   呼び出せるようにします。また、チャット開始時点ですでに AI-DLC の指示を保持しているため、
   最初の行に対応する前にファイルを読むことはありません。
2. 依頼全体を入力してから Enter を押します。たとえば `/aidlc --doctor` や
   `/aidlc build a to-do app` です。`/aidlc` だけでも動作します: `/` メニューが
   開いている間は、Enter で行がメニューの最初の項目、つまり `/aidlc`
   自体に補完され、次の Enter で送信されます。
3. `/` メニューから `aidlc` を選んでも、チャットが使うエージェントは変わりません。
   チャットがまだ **Default** のままの場合、最初の依頼に対して AI-DLC が最初に行うのは
   自身のコマンドの実行であり、Kiro はその承認を求め、その後も
   実行するコマンドごとに確認を求めます。これを避けるには、先にエージェントピッカーで **aidlc** を選んでください（ステップ 1）。
   **Default**（または **Default v2**）の場合、`/aidlc` の依頼を入力すると、AI-DLC はチャットでこのことを 1 回伝えます。
   ただし、Kiro の承認カードのいずれかで AI-DLC のコマンドに対してすでに **Always allow** を
   選んでいる場合は伝えません。
   `/` メニューにある `aidlc-...-agent` の項目は、ワークフロー中に AI-DLC が呼び出す
   スペシャリストです。自分で開始する必要はありません。

`aidlc` エージェントを選択していれば、
"start an AI-DLC workflow for a to-do app" や "continue my AI-DLC workflow" のように、平易な言葉で依頼することもできます。
Kiro のウェルカムパネル（Spec、Plan、Bug Fix、Quick Spec）に並んでいるのは Kiro 自身の
ワークフローです。AI-DLC はそこには表示されないので、上記のようにチャットから開始してください。

### Supervised or Autopilot

チャットボックスの右下（**Settings > Agent > Agent Autonomy** にもあります）にある
**Autopilot** スイッチは、エージェントの操作について Kiro 自身がどのくらいの頻度で
承認を求めるかを設定します。ワークフローの途中を含め、いつでも変更できます。
どの AI-DLC チェックポイントで停止するかは変わりません。

- **Supervised**（スイッチオフ）: ファイルを変更した各ターンの後、Kiro は
  **Review changes** カードを表示し、**Accept** か **Reject** を選ぶまで待ちます。
  AI-DLC は質問、ステージのドキュメント、プランをファイルに保存するため、
  ワークフローでは AI-DLC 自身の質問に加えて、このカードが何度も表示されます。
- **Autopilot**（スイッチオン）: ファイルの変更はカードなしで反映されます。変更内容を確認するには、
  チャットで **View changes** を選びます。
- どちらのモードでも、`aidlc` エージェントがまだ許可していないコマンド（プロジェクトのテストコマンド、`git push`、
  再帰的な削除など）については、Kiro が **Allow** を求めます。
  あなた自身のコマンドには AI-DLC のルールが適用されないため、一度
  **Always allow** を選ぶと、Kiro はリダイレクトや連結を含めてそれを記憶します。
  AI-DLC 自身のコマンドは、リダイレクト、連結、置換（`>`、`&&`、`$(...)`）を含む場合に確認を求めるため、
  許可された AI-DLC コマンドを別のコマンドに拡張することはできません。また Kiro は、
  意図的に保留されている 2 つの AI-DLC コマンドの前にも確認を求めます（
  [Native channel](#native-channel-recommended) を参照）。
- Kiro CLI（v3 エンジン）では、並列ビルドのワーカーや
  レビュアーが確認を求めると、Kiro はそれらの要求を "tool approval pending" ボックスにまとめます。
  (f) **Approve all pending and auto-approve future subagent requests** を選ぶと、
  後から開始するレビュアーも含め、それらが求めるものを確認なしで実行できるようになります。
  ただし Kiro CLI 3.1.0 では、同じチャットの後の時点で、たとえば AI-DLC の質問に答えた後などに、
  そのボックスが再び表示されることがあります。再び表示された場合は、もう一度 (f) を選べます。
  メインチャットからの要求は引き続きこれまでどおり確認を求め、Kiro は (f) について設定を書き込みません。
- Kiro CLI 3.1.0（v3 エンジン）では、あなた自身のコマンドに対するメインチャットの承認カードに
  **Always allow** の項目がありません: 代わりに **Allow
  once** と **Create allow rule to approve future similar actions** があります。Kiro が
  最初に提示するルールは、そのコマンドに対する **exact match · session** です: これを選ぶと
  同じコマンドはカードなしで再実行できますが、別のテストファイルなど引数が異なるコマンドは
  そのルールの対象になりません。

どちらのモードでも、AI-DLC は各質問、すべての承認ゲート、そしてプラン承認がオンの場合の
Code Generation の **Approve Plan** で停止し、あなたが返答を入力するのを待ちます
（プラン承認は express と poc 以外のすべての提供スコープでデフォルトでオンであり、切り替えはあなたが決められます。
[プラン承認](../13-customization.md#plan-approval) を参照）。Kiro のスイッチで
これらを飛ばすことはできません。AI-DLC のフックはどちらのモードでも実行されます: そのうちの 1 つは、
プランが承認されるまで、またはプラン承認がオフの場合は AI-DLC がプランを確認なしでビルドすることを
記録するまで、コードの変更を拒否します。また AI-DLC は、質問に対してあなたが返答を入力した後にのみ、
回答や承認を受け付けます。定型的な承認を省略する AI-DLC の
設定は、Kiro の Autopilot ではなく、AI-DLC 自身の Construction での選択肢
**Continue automatically** です（
[最初のワークフロー](../02-your-first-workflow.md#construction-phase) を参照）。

たとえば最初のワークフローや、すべての変更が重要なコードなど、ファイルの変更をその都度すべて読みたい場合は
Supervised を使ってください。クリック数を大幅に減らしたい場合は
Autopilot を使ってください: その場合も AI-DLC のすべての
チェックポイントにはチャットで回答し、変更は後から確認できます。

### When the chat fills up

Kiro はチャットがどのくらい埋まっているかをあなたに表示しますが、作業中の AI-DLC にはその数値を渡しません。
Kiro IDE と Kiro CLI の v3 エンジンは、チャットが 80% に達すると自動的に要約し、
それ以降エージェントはその要約をもとに作業します。AI-DLC は
Kiro 自身のチャットの記録（`~/.kiro/sessions/`、読み取り専用）を読み、Kiro がチャットを要約した後の
最初の承認の質問またはユニットのチェックポイントで、そのことを 1 行で伝え、
一時停止して新しいチャットで続けることを提案します。
Kiro の自動コンパクションをオフにしている場合は、チャットが 80% に達した時点で同じ提案が出ます。
作業での最初のターンですでにチャットが半分以上埋まり、
それを埋めたモデルのまま実行されている場合、そのモデルで新しいチャットを始めても
ほぼ同じくらい埋まった状態で始まるため、その行は新しいチャットを提案する代わりにそのことを伝えます。
判断はあなたの回答次第です: 自動的に一時停止することはなく、その行はチャットごとに 1 回だけ伝えられます。

この行は、ステージの途中（いずれにせよ尋ねられる質問と一緒にのみ表示されます）、
このマシンに記録のないチャット、または Kiro のアップデートでチャットの記録方法が変わった場合には
表示されません。Kiro
IDE で 1 つのウィンドウに 2 つの AI-DLC チャットを開いている場合は、最後に開始したチャットについて
説明することがあります。同じことはいつでも手動で行えます:

1. ステージの境界で停止します: 承認すると同時に、同じ返答で今は止めたいと伝えるか、
   `/aidlc park` と入力します。すると `/aidlc --status` に
   `Paused at <stage> (you stopped it); pick it back up with /aidlc --resume` と表示されます。
2. 新しいチャットを開始します: Kiro IDE では、新しいチャットを開いてエージェントピッカーで
   **aidlc** を選びます（新しいチャットは Kiro の Default エージェントで始まります。
   [Kiro IDE のチャットで AI-DLC を始める](#start-ai-dlc-in-a-kiro-ide-chat) を参照）。Kiro
   CLI では `/chat new` と入力します。
3. `/aidlc --resume` と入力します。`/aidlc --status` に Paused と表示されなくなります。

以前のチャットで話し合ってもファイルに書き込まれなかった内容は引き継がれません。
そのため、ステージの終わりが最適なタイミングです（
[ワークフロー途中でのモデル変更](../11-session-management.md#changing-model-mid-workflow) を参照）。

## How hooks work on Kiro

Kiro IDE と Kiro CLI v3 は、`.kiro/hooks/` 配下の v2 フック JSON ファイル
（`{"version":"v1","hooks":[{name,trigger,matcher,action}]}`、PascalCase の
トリガー）を通じてフックを登録します（エージェント JSON の中に `hooks` ブロックを持つ Kiro CLI (V2)
ディストリビューションとは異なる仕組みです）。ネイティブのフックコマンドは
`aidlc engine adapter kiro-ide` を経由し、ソース／開発用のコピーは配置された
`aidlc-kiro-adapter.ts` シムを経由します。どちらも IDE のイベントを、
共有コアのフックが想定する形に正規化します。どの Kiro ツールが書き込み、シェル、委任、
読み取りに当たるかは、アダプターの隣にある 1 つの表 `aidlc-kiro-tool-names.ts` で定義されています。
アダプターはこれを読み、フックの登録は手書きの JSON で、そのマッチャーには
同じシェル、委任、監査対象の書き込みの名前が並んでいます。

Kiro IDE 1.x と Kiro CLI v3 は、フックのコンテキストを **stdin 上の JSON** として渡します（snake_case:
`{ session_id, tool_name, tool_input, tool_response }`。古い 0.12 ビルドでは代わりに
`USER_PROMPT` 環境変数に camelCase の同等の内容を設定し、
アダプターはどちらも受け付けます）。以前のビルドでは PostToolUse の書き込み／シェルツールの入力が
空だったため、入力に何もない場合、アダプターは結果のテキストから書き込まれたパスを復元し、
監査末尾のフック（`rebuild-stage-graph`、`sync-workflow-state`）は監査証跡から
実行されます。グラフ再構築の経路はシェルの結果とセッションの識別情報も保持するため、
成功した `intent-create` は呼び出し元のセッションに結び付きます: 新しい
イベントは正確な `session_id` を持ち、レガシーのチャネルは計測された
`VSCODE_IPC_HOOK`/`VSCODE_PID` 環境から安定したホストインスタンスの識別情報を導き、
SessionStart で保持します。新しい Stop も同様にイベント固有の `session_id` を優先し、
インテントが作成または選択された後に、同時に開いている別のチャットが
他のチャットの引き継ぎを消費してしまうのを防ぎます。レガシーの agentStop は保持された
識別情報にフォールバックします。その後の 1.x ビルドでは一部の PreToolUse と委任の入力が埋められるようになり、
アダプターはそれらのフィールドを保持します。Windows では、決定論的なユーティリティがこれらの
接点を使って IDE のシェル結果の転送を回避します: 送信された
プロンプトを公開するビルドでは UserPromptSubmit でユーティリティを実行し、1.0.242 のようなビルド
（プロンプトのフィールドが空）では、正確な `execute_pwsh` の PreToolUse コマンドにフォールバックします。
ユーティリティはターンごとに 1 回実行され、その UTF-8 テキストはターミナルの
プロトコル／制御バイトを除いて中継され、重複したシェル呼び出しは拒否されます。新しいチャットは
ターンと出力の状態を `session_id` ごとに保存し、セッションの識別情報がないコンテキストは
1 つのレガシー互換用バケットを使います。1.0 より前の camelCase のペイロードも
`toolArgs.command` を通じて同じフォールバックを取ります。生のプロンプトテキストは、より新しい世代向けの
互換形式にすぎません。

ペイロードの取得は、**ペイロードに依存するターゲットに限定**されています
（`audit-and-sensors`、`log-subagent`、
`plan-approval-guard`、`rebuild-stage-graph`、`review-freeze`、
`state-transition-guard`、およびそれらを実行する `guard-tool-call`、`person-message`、
`after-shell` のカード）、ターミナルコマンドの接点、さらに新しい `session_id` のための
`session-start` と `continue-workflow`、正確な承認の応答のための
`record-human-turn` です。空でない `USER_PROMPT`
は、0.12 ビルド（stdin を開くものの何も書き込まない）では直ちに消費されます。
それ以外の場合、アダプターは Kiro が閉じるまで 1.x の stdin チャネルを読むため、
負荷の高いマシンで遅れて渡されたメッセージも完全に記録されます。閉じないチャネルに対しては
60 秒の安全上の上限があります。何も届かなかった場合、あなたの
ターンは記録されず（何も入力しなかったとは記録されません）、エージェントには、
このチャットの最後のメッセージが AI-DLC の記録に届かなかったことを伝えるよう指示されます。そのため、
そのメッセージが質問への回答だった場合は、同じ質問が再度尋ねられます。それ以外のターゲットはどちらのチャネルにも触れず、
遅延ゼロの経路を維持します。

| フック | トリガー（マッチャー） | 目的 |
|------|-------------------|---------|
| `aidlc-session-start` | `SessionStart` | 新しいセッションが最初のプロンプトを受け取ったときに、ワークフロー再開のコンテキストを注入します（両方の環境。既存のセッションの再開では発火しません）。Kiro IDE 1.1.14 は新しいチャットで SessionStart フックを実行しないため、代わりに `aidlc-record-human-turn` がこの処理を行います |
| `aidlc-record-human-turn` | `UserPromptSubmit` | メッセージごとに 1 枚のカードです: まず直前の書き込みまたはコマンドが残した処理（下の「書き込みの後」と「コマンドの後」の 2 行）を完了し、次にすべてのプロンプトで人間のターンのイベントを記録し（人間の在席ゲート）、プロンプトのテキストが得られる場合は、コマンドを入力したチャットのために、モデルより前に status、doctor、help、ナビゲーション、その他のターミナルユーティリティを実行します。最後に確認したチャット以外のチャット、またはまだ開始していないチャットからのプロンプトは、`aidlc-session-start` と同様にまずそのチャットのセッションを開始し、チャットが `AIDLC Runtime Session:` の行または再開のコンテキストを受け取れるようにします。Kiro 自身が作るプロンプト（Workflows のステップの指示、ワークフロー終了の通知）はあなたのターンとして数えられません: `HOST_TURN` として記録され、セッションを開始せず、質問にも回答しません |
| `aidlc-continue-workflow` | `Stop` | 転送ループの監査（助言のみ。Stop トリガーは IDE 上でブロックできないため、強制はコンダクター自身の Stop プロトコルに依存します） |
| `aidlc-guard-tool-call` | `PreToolUse`（読み取り以外のすべてのツール） | 1 枚のカードです: まず直前の書き込みまたはコマンドが残した処理（下の「書き込みの後」と「コマンドの後」の行）を完了し、次に下の 4 つのチェックを、この順序で、同じ呼び出しに対して実行します。各チェックは横に示したツールに対してのみ実行されます。1 つが拒否した後もすべてのチェックが実行され、いずれかのチェックが拒否すると呼び出しは拒否され、それぞれの理由が 1 回ずつ示されます。読み取り（`read_file`、`list_directory`、検索）はカードなしで実行されます: 承認に回答することもワークスペースを変更することもできないためです。ヘルパーエージェントが回答を返す場合（`subagent_response`）も同様です: その回答に基づいて動作する呼び出しがチェックされます |
| `plan-approval-guard` | `aidlc-guard-tool-call` 内（読み取り以外のすべてのツール） | 引数がある場合は正確なターゲット分類によって、Code Generation のプラン承認を強制します。シェルツールは IDE の 3 つの名前 `execute_bash`、`execute_pwsh`（Windows）、`shell` のすべてで認識されます: それぞれ `Bash` として共有ガードに転送され、同じようにレガシーの復旧へ振り分けられます。アクティブなワークフローがない場合、シェル呼び出しは拒否されません。`execute_pwsh` は PowerShell として扱われるため、プランが承認待ちの間も、読み取り専用のコマンドレット（`Get-Content`、`Select-Object`、`ConvertFrom-Json` など）、`2>$null`、`aidlc.cmd` またはインストールされたエンジンのフルパスは実行できます。`Out-File`、`Set-Content`、`Add-Content`、`Tee-Object`、およびファイルへの `>` は実行できません。引数のないレガシーのペイロードでは、このフックに関する限り、計測された `fs_write`/`str_replace` によるプランの質問への書き込みだけが許可されます。`review-freeze` と `state-transition-guard` はそのような書き込みを拒否するため、実行されません。0.12 はその出力を破棄するため、PostToolUse は何も出力しません。呼び出し元の Code Generation の `next`／最後の `continue` ディレクティブが、保護された選択肢の権限を 1 つ持ちます。復旧にはまず平易な復旧の質問に対する利用者の同意が必要で、エージェントがそれを記録します。他の稼働中のウィンドウは復旧を開始できませんが、置き換えのウィンドウは、所有者の PID が終了するか IPC のみのエンドポイントが消えた後に復旧できます。引き継ぎでは、チャレンジを更新する前に古い応答の証拠を消去します。中断された書き込み前のウィンドウは、PostToolUse が実行されなくても復旧のラッチとして残ります。確定的な `toolSuccess:false` や認識された失敗の文言は、変更が起きなかったためラッチを解除しますが、結果が不明な場合はラッチされたままです。アダプターが所有する復旧は、再発行が成功した後、違反／ウィンドウの状態だけを消去し、人間への確認は保持します。`UserPromptSubmit` は正確な復旧／承認のラベルを送信できますが、それを表示したり転送したりはできません。不明な変更操作はフェイルクローズし、共有ファイルや監査に平文の秘密情報は残りません。 |
| `review-freeze` | `aidlc-guard-tool-call` 内（書き込みとシェル） | ゲートの前に、新しいターミナルレビューのレシートが対象としている間、ステージのレビュー済み出力に対する書き込みやシェルによる変更を拒否します。書き込みツールはターゲットパスとともに Write/Edit として、シェルツールは Bash として、チャットのセッションとともに共有フックに届きます。委任先エージェント自身の書き込みも同じように判定されます。`execute_pwsh` のコマンドは、ここでも `state-transition-guard` でも PowerShell として読み取られます（バックスラッシュのパス、`Set-Location`）。入力を読み取れない呼び出し（Kiro IDE 1.1.70 または Kiro CLI 2.24.1 より古いビルドは引数なしで送ることがあります）は、ワークフローの内外を問わず、また `AIDLC_DISABLE_REVIEW_FREEZE_HOOK=1` の場合も、フックの実行前に拒否され、サポートされるビルドを示す 1 行が表示されます。 |
| `state-transition-guard` | `aidlc-guard-tool-call` 内（書き込みとシェル） | AIDLC のフック、セッション制御、ランタイムの記録、監査証跡へのツール呼び出しによる書き込みを拒否します。状態ツール自身は、`aidlc-state.ts` のライフサイクル動詞の直接実行を拒否し、`aidlc-orchestrate.ts report` を案内します。Kiro IDE は呼び出しに委任先エージェントの名前を含めないため、委任先の呼び出しにはコンダクターのルールが適用されます。入力を読み取れない呼び出しは `review-freeze` と同じように拒否されます。 |
| `terminal-command-guard` | `aidlc-guard-tool-call` 内（`execute_bash\|execute_pwsh\|shell`） | プロンプトが空になる IDE バージョン向けのフォールバックです: 分類されたユーティリティを 1 回実行し、重複した Windows のシェル呼び出しを拒否します。チャットのターンでターミナルコマンドが実行された後は、そのチャットの同じターン内の他のシェル呼び出しも、何を実行するかにかかわらず拒否します（ペイロードが示すチャットについて）。これにより、`/aidlc --status` のターンで次のステージが渡されることはありません。Windows では、cmd.exe が分割してしまう値を持つ `aidlc` コマンドも拒否します（下の Windows の引用符の行を参照）。どのシェルでも、応答がファイルに出力される AI-DLC エンジンコマンド（`>`、`>>`、または `Out-File`、`Set-Content`、`Add-Content`、`Tee-Object`、`tee` へのパイプ）を拒否します。そのためエージェントはそれを単独で実行し、Kiro が完全な形で返す結果から応答を読み取ります。エージェントが応答をパイプしたりキャプチャしたりして結果から外すステップコマンド（`orchestrate next` または `continue`）（任意のプログラムへのパイプ、`$(...)`、`$x = ...`、破棄されるストリーム）は拒否されません: 次のカードは、チャットがそのステップのルールを渡されたことを忘れるため、エージェントの次のステップで再びルールが渡されます |
| 書き込みの後（`fs_write\|str_replace\|fs_append`） | カードなし: 次のカードが行う | 成果物の作成／更新をログに記録し、該当するセンサーを起動します。`aidlc-guard-tool-call` は、通過させた書き込みの対象ファイルを記録しておき、いずれにせよ実行される次のカード（次のガード、次のメッセージ、またはターンの終わり）が、ファイルが変更されていた場合に、書き込みのチャットとともに何よりも先にそれを記録します。ファイルを元のままにした書き込みは何も記録しません |
| `aidlc-log-subagent` | `PostToolUse`（`^(?!subagent_response$)(subagent_.+\|invoke_sub_agent\|orchestrate_subagent)$`） | 委任先の識別情報とともに `SUBAGENT_COMPLETED` を記録します。`orchestrate_subagent` のパイプラインでは、ステージごとに 1 行です。どの委任先の名前もアダプターに届くようにマッチャーは広くなっていますが、補助的な `subagent_response` シェルは除外しています。アダプターはこれを他のすべての入口でも除外します。各委任先が送られたという記録（`SUBAGENT_DISPATCHED`）は、ガードのカードが呼び出しを通過させた時点で書き込まれます |
| コマンドの後（`execute_bash\|execute_pwsh\|shell`） | カードなし: 次のカードが行う | 下の 2 つのフックを、この順序で、コマンドごとに 1 回、いずれにせよ実行される次のカードが、その中で何よりも先に実行します。コマンドが新しい作業を作成した場合は、再構築がその名前を受け取るため、それを実行したチャットがその作業に参加します |
| `rebuild-stage-graph` | コマンドの後 | ランタイムグラフを再コンパイルします（監査の末尾に基づいて実行を判断） |
| `sync-workflow-state` | コマンドの後 | 監査内の最新の `STAGE_STARTED` から `Current Stage` を前方向にのみ同期します（IDE は解析できるタスクのペイロードを提供しません）。その開始の後にユニットが再開された場合、`Current Stage` は再開によって設定された位置のままになります |

`aidlc-session-end` には**登録がありません**: Kiro の `Stop` トリガーは、両方の環境で、
会話の終了時ではなくアシスタントの各ターンの終わりに発火するため、
登録すると同じセッションのプロンプトの間に誤った `SESSION_ENDED` が追加されてしまいます。
Kiro が本当のセッション終了イベントを提供するまで、`SESSION_ENDED` は記録されません。

これらの登録のいずれかが実行されるたびに、Kiro は "Run Command Hook" カードを表示します: ファイルの変更、コマンド、または別のエージェントへの引き渡しの前に 1 枚、その後には表示されず、読み取りやヘルパーが回答を返す場合にも表示されず、送信するメッセージごとに 1 枚、各ターンの終わりに 1 枚（ターンの最後の書き込みやコマンドの処理も完了します）、ヘルパーが終了したときに 1 枚です。フックファイルが古いプロジェクトでは、`config` で更新されるまで、書き込み後のカード、コマンド後のカード、メッセージごとの 2 枚目のカードが引き続き表示されます。更新により `aidlc-write-audit-log.json`、`aidlc-after-shell.json`、`aidlc-terminal-command.json` が削除されます。

### Debugging hooks

フックが想定どおりに動作しない場合は、デバッグログをオンにしてください。各フックは
判断の経路（どのゲートを通ったか、解決されたパス、終了した理由）を
`<record>/.aidlc-engine/hooks-health/hook-debug.log` に追記します。これは**デフォルトでは
オフ**であり、通常の実行ではログは書き込まれず、オーバーヘッドもありません。有効にする方法は
2 つあり、どちらでも動作します:

- **ファイルシステムのマーカー（Kiro IDE で最も簡単）:** プロジェクトで `touch aidlc/.aidlc-hook-debug`
  （PowerShell では `New-Item -ItemType File aidlc/.aidlc-hook-debug`）
  を実行します。IDE を再起動しなくても次のフックの発火から有効になり、
  `rm aidlc/.aidlc-hook-debug` でオフに戻ります。
- **環境変数:** `export AIDLC_HOOK_DEBUG=1`。フックは IDE の起動時の
  環境を参照するため、IDE を終了してから、変数をエクスポートした
  ターミナルから開いてください。
  Windows では、代わりに PowerShell でユーザー変数として設定します。
  `[Environment]::SetEnvironmentVariable("AIDLC_HOOK_DEBUG", "1", "User")`
  を実行してから IDE を終了して開き直してください（オフにするには、`"1"` の代わりに `$null` を指定して
  もう一度実行します）。

## What's different on Kiro

| 領域 | Claude Code | Kiro IDE と Kiro CLI |
|------|-------------|----------|
| フックの登録 | `settings.json` の `hooks` ブロック | `.kiro/hooks/aidlc-*.json` の v2 フックファイル（Kiro IDE 1.x、Kiro CLI v3 エンジン） |
| ゲートと質問 | `AskUserQuestion` ウィジェット | 番号付きの散文の選択肢（番号で返答）。`[Answer]:` タグを持つ質問ファイルが引き続き信頼できる情報源です |
| ステータスライン | 現在のステージ + モデル + コンテキスト % | 利用できません: `/aidlc --status` と各ゲートの進捗表示を使ってください。チャットが埋まってきた場合は [チャットがいっぱいになったとき](#when-the-chat-fills-up) を参照してください |
| ディスパッチされるステージ（2.1 pipeline、2.2 subagent、2.4 mob、3.5 subagent） | `Task` ツール | `invoke_sub_agent`（Kiro IDE）または `orchestrate_subagent`（Kiro CLI）→ 14 個の Markdown ペルソナすべて。それぞれ自身の frontmatter にある `tools:` と拒否ルール、およびコンダクターのコマンド許可のもとで実行されます |
| Construction のスウォーム | 並列の `Task` を基本とし、任意で ultracode Workflow | サブエージェントへの展開のみ。`AIDLC_USE_SWARM=1` は何もしないことが通知されます |
| セッションの監査イベント | `SESSION_STARTED/RESUMED/ENDED`、`SESSION_COMPACTED` | 新しいセッションが最初のプロンプトを受け取ったときに `SESSION_STARTED`、Kiro IDE ではプロンプトが以前のチャットに戻ったときに `SESSION_RESUMED`（本当のセッション終了のトリガーがないため `SESSION_ENDED` はなく、コンパクション前のイベントもありません） |
| MCP サーバー | 5 つを提供（`.mcp.json`: `context7` + AWS の 4 サーバー） | 提供なし。AI-DLC 自身のチャット、14 個の委任用ペルソナ、プラグインが追加するエージェントは、ワークスペースとユーザーの `.kiro/settings/mcp.json` にあるすべてのサーバーを使え、各呼び出しで先に確認を求めます（Kiro CLI ではペルソナの 2 番目のステップから）。 |
| Windows での記録テキスト内の引用符 | 入力どおりに記録 | Kiro IDE はエージェントのコマンドを Windows PowerShell 5.1 で実行します。これは空の引数を落とし、値の中の二重引用符は `\"` と書かない限り削除します。エージェントは引用符をそのように書くため、回答やフィードバックは引用符を含めて記録されます。`&`、`<`、`>`、`^`、パイプ記号も含む値は、代わりに内側を単一引用符で書きます。`aidlc` コマンドは cmd.exe を通じて実行され、cmd.exe がこれらの文字を解釈してしまうためです。フックは、これらの文字が引用符の外で cmd.exe に届くコマンドを拒否します。それでも質問や回答が 2 つに分割されて届いた場合、AI-DLC は記録を拒否して渡し方を伝えるため、監査証跡にその一部だけが残ることはありません |
| ワークフローの途中でガード、要約確認、プラン承認をオフにする | 自分の言葉で依頼するとエージェントが設定コマンドを実行します。または `/aidlc config set summary-confirmation off` のようにチャットでスイッチを入力します | 同じです。ただし、フックに空のメッセージを渡す Kiro IDE のビルド（1.0.242 など）では設定コマンドも拒否されます。その場合、要約確認やプラン承認については、拒否メッセージが示すターミナルコマンドを実行すると、実行中の作業を含むすべての作業で直ちにオフになります（ネイティブインストールでは `aidlc config flags --bypass AIDLC_DISABLE_SUMMARY_CONFIRMATION --local --yes` または `--bypass AIDLC_DISABLE_PLAN_APPROVAL_GUARD`。`--clear-bypass` でオンに戻ります）。その他のガードについては、Kiro IDE をアップデートしてからスイッチを入力してください |

それ以外のすべて（ステートマシン、監査証跡、インテントごとの記録ディレクトリ
（`aidlc/spaces/<space>/intents/<YYMMDD>-<label>/`）配下の成果物、学びの
手順、センサー、スコープ、深度／テスト戦略）は同じように動作します。実際に
同一だからです: ネイティブインストールは `aidlc` を通じてディスパッチし、ソースのコピーは
`.kiro/tools/` の対応するツールを実行します。

プロジェクトの `aidlc/` ワークスペースはハーネスに依存しません。プロジェクトをハーネス間で移動すること
（または両方を並行して実行すること）はサポートされていますが、テストされていません。`/aidlc
--doctor` は、アクティブなワークフローがある状態で競合するハーネスのセットアップを検出すると警告します。

### A Unit's reviewer reads only that Unit

ユニットのレビュー中、レビュアーはそのユニットのファイルと、
渡された共有の契約だけを読むことになっています。Claude Code や他のハーネスでは
フックがそれを守らせますが、Kiro ではレビュアーの読み取りをフックで区別できないため、
代わりに Kiro がレビュアーにそれを守らせます。ユニットのレビューごとに、AI-DLC は
レビュアーのコピー `.kiro/agents/aidlc-scoped-review-<id>.md` を書き出します。その
権限は、他のすべてのユニットの `construction/` フォルダー（別の作業にある
同名のユニットを含む）での読み取りを拒否し、AI-DLC はそのコピーをディスパッチします。
Kiro はそのような読み取りを拒否し、それらのファイルを検索結果から除外します。`cat` のような
シェルによる読み取りは対象外で、先に確認を求めます。この
コピーは、ユニットの設計が統合ポイントとして名前を挙げているファイルであっても、他のユニットのファイルを開きません:
レビュアーは代わりにその点を指摘事項として記録します。この
コピーはレビューの実行中、Kiro のエージェント一覧と `/` メニューに表示されます。AI-DLC が
ディスパッチするので、選ぶ必要はありません。判定が記録されると削除されます。
判定がないままレビューが終わったコピー（隔離された `--single` 実行以外で、判定なしにユニットや
ステージを承認した場合）は後で削除されます:
その作業がフォルダーの現在の作業である間、AI-DLC はその作業の記録を読める状態になった時点で、
そこでチャットが開始されたとき、またはそこでレビューが求められたか記録されたときに削除します。
差し戻したレビューのコピーは、そのユニットまたはステージを承認するまで残ることがあります。
閉じたチャットが残したコピーで、その後何も行われていないものは残ります: AI-DLC は
それを別のチャットでまだ実行中のレビューと区別できないため、あなたが削除してかまいません。
AI-DLC の `.gitignore` ブロックにより、コピーはコミットに含まれません。書き出すべきコピーを AI-DLC が
書き出せない場合や、Kiro にコピーの名前のエージェントがない場合（古い Kiro のビルドでは
読み込まれないことがあります）、AI-DLC はコピーがない場合と同様にレビュアー自体をディスパッチし、
このレビューはチェックなしで実行されることを 1 行で伝えます。その場合 doctor は
フックの取りこぼしとして数えます: AI-DLC が書き出せなかったコピーは `reviewer-scope` の下で
原因を示し、Kiro が読み込まなかったコピーは `kiro-adapter` の下で数え、そのファイル
（`<record>/.aidlc-engine/hooks-health/kiro-adapter.drops`）がコピーの名前を示します。
Kiro CLI 2.31.0 と Kiro IDE 1.3.0 で計測し、ワークフロー全体については
Kiro CLI 2.31.1 で計測しました。並列にビルドされるユニットや、
レビュアーの読み取り範囲チェックがオフの場合はコピーはありません: enterprise 以外の
すべてのプランのデフォルトである Guard Policy `off` の場合、`guard.reviewer-scope off` の場合、
`AIDLC_DISABLE_REVIEWER_SCOPE_HOOK=1` の場合です。その場合、レビュアーは
指示によってのみそのユニットに限定されます。そのようなプランでコピーを使うには、チャットで
その作業についてレビュアーの読み取り範囲チェックをオンにするよう依頼してください（または
`/aidlc config set guard.reviewer-scope on` と入力します）。コピーは書き出された時点の
権限を保持するため、チェックをオフにした場合は次のユニットのレビューから適用されます。

### One agent on its own model and effort

Kiro では、エージェントにモデルや effort が設定されていない限り、すべての AI-DLC エージェントは
チャットのモデルと effort（Kiro IDE で選んだモデル、または Kiro CLI のセッションモデル）で実行されます。
Kiro は委任先エージェントを独自のモデルと effort で実行できます。
たとえば、レビュアーを別のモデルファミリーで実行できます:

```bash
aidlc config models --agent architecture-reviewer --model gpt-5.6-terra --effort high --project --yes
```

AI-DLC はモデルと effort を `aidlc.settings.json` に記録し、そのエージェントの
`.kiro/agents/aidlc-architecture-reviewer-agent.md` に
`model:`（引用符付きの文字列として）と `effortLevel:` を書き込みます。ユニットのレビュアーのコピーも
それらを保持します。Kiro は次にそのエージェントをディスパッチするときにそれらを読み込みます。すでに開いている
チャットでも同様です。モデルは、使用している Kiro が提供しているものである必要があります
（`kiro-cli chat --list-models`、または Kiro IDE のモデルピッカー）: 提供されていないモデルを指定すると、
そのエージェントのディスパッチが失敗することがあります。モデルが対応していない effort は
モデル自身のデフォルトで実行され、モデルを指定して effort を指定していないエージェントは、
チャットの effort ではなくそのモデルのデフォルトの effort で実行されます。`--model default` と
`--effort default` で設定を取り除けます。プリセットやグループの effort
（`--preset`、`--reviewing-effort`）は Kiro では引き続き何も変更せず、セットアップは
そこにプリセットを記録せず、コンダクターはチャットのモデルを使い続けます。

AI-DLC は、Claude Code の場合と同様に、提供するエージェントファイルのモデルと effort の行（`model:`、`effortLevel:`、および
他のツールが使う `effort:`、`variant:`、`tier:` のキー）を管理します:
`aidlc config` は設定を持つエージェントのそれらの行を書き込むため、手で変更するのではなく
`aidlc config models` で変更してください。`aidlc config models --check` は、手で追加したものも含め、
設定と一致しない `model:` または `effortLevel:`
の行を報告します。あなたが自分で追加したエージェント
ファイルには触れません。

エージェントの effort はすべてのツールで共有されますが、モデルはツールごとに記録されます。
そのため、別のツール向けにあるエージェントに記録された effort は Kiro にも適用されます。
`aidlc config` の実行で、Kiro のエージェントファイルにファイルの元の値と異なるモデルや effort の値が書き込まれる場合
（その実行自身が変更する値を除く）は、そのことを 1 行で伝え、
各値がどこで設定されているかを示す `aidlc config models --show --harness kiro`
を案内します。そのエージェントに対して `--effort default`（または `--model default`）を、
値があるファイルに応じて `--project`、`--local`、`--global`（マシンの設定）とともに、
プロジェクトに他のハーネスがある場合は `--harness kiro` も付けて指定すると、その値を取り除けます
（この方法で取り除いた effort は、すべてのツールから取り除かれます）。
`effortLevel` は Kiro のドキュメントには記載されていません。Kiro CLI 2.31.1 の
v3 エンジンと Kiro IDE 1.3.0 で計測しました。

### Kiro's Workflows feature stops AI-DLC's reviews and helpers

Kiro IDE の **Workflows** 機能（Settings > **Kiro Agent > Workflows:
Enabled**、または **Enable Workflows** コマンド）は、作業をバックグラウンドの
ワークフローとして実行します。これがオンの間、チャットは AI-DLC がレビューとヘルパーに使う
サブエージェントツールを失います: `aidlc` エージェントではそれらは実行されず、メインチャットが
その作業を自ら行います。Default エージェントではバックグラウンドで実行されますが、それらが終わる前に
チャットのターンが終了し、それらが実行するすべてのコマンドが **Approval
needed in agent-aidlc-...** パネルで確認を求め、Kiro が `/aidlc` の依頼を AI-DLC ではなく
自身のワークフローに渡してしまうことがあります。Workflows は、自分でオンにしない限りオフです。

このスイッチはすべてのプロジェクトに適用される Kiro の設定です: Kiro はこれを
ユーザー設定からのみ読み込むため、ワークスペースの `.vscode/settings.json` の値は効果がありません。
これがオンで、このコンピューターでまだ回答していない場合、AI-DLC は
オフにするかどうかを 1 回だけ尋ねます: `aidlc config` の実行中か、最初の Kiro
IDE のチャットでです。`aidlc config --yes` は推奨の回答を採用してオフにします。
はいと答えると、AI-DLC はその 1 つの設定だけを変更します。その後、開いているチャットに反映されるよう、
コマンドパレット（Ctrl+Shift+P）から **Developer: Reload
Window** を実行してください。いいえと答えるとオンのままになり、AI-DLC は再度尋ねません。

後で変更するには、エージェントに Kiro の Workflows をオフまたはオンにするよう依頼するか、次を実行します:

```bash
aidlc config trust --kiro-workflows off --yes   # or: on
```

Kiro 自身の **Disable Workflows** と **Enable Workflows** コマンドでも同じことができます。
`/aidlc --doctor` は、Workflows がオンの間、および Kiro の
設定ファイルに AI-DLC が読み飛ばせないエラーがある間、警告を表示します（Kiro はそのファイルから
Workflows をオンとして読み取る可能性があります。次のチャットで 1 回それが伝えられ、修正すべきは
そのファイルのエラーです）。

Kiro CLI には独自の Workflows スイッチがあります（`/settings` > Features >
**Workflows**、Kiro CLI 設定 `~/.kiro/settings/cli.json` の
`chat.enableWorkflows`）。これがオンの間、Kiro CLI は `aidlc` エージェントから両方のサブエージェント
ツールを取り除くため、Kiro CLI のチャットでは AI-DLC のレビューとヘルパーが実行されません。
"Workflows: sub-agent tool" 設定でもそれらは戻りません（Kiro CLI 2.31.1 と 2.32.0 で計測）。Kiro CLI のチャットは
このスイッチを自身の Kiro CLI 設定からのみ読み込むため、プロジェクトの
`.kiro/settings/cli.json` の値ではオフになりません。AI-DLC は同じ方法で、独自の回答として
これについて尋ねます: `aidlc config` の実行中か、最初の Kiro CLI のチャットで 1 回です。`aidlc config
--yes` は推奨の回答を採用してオフにします。AI-DLC は
Kiro CLI 自身の `kiro-cli settings` コマンドを通じてこれを変更します。その後 Kiro CLI を終了し、`kiro-cli
--v3` を再起動してください。後で変更するには、エージェントに依頼するか、次を実行します:

```bash
aidlc config trust --kiro-cli-workflows off --yes   # or: on
```

`/aidlc --doctor` は、これがオンの間、警告を表示します。

### Command Prompt as Kiro's terminal on Windows

Kiro IDE は、エージェントのコマンドをデフォルトのターミナルプロファイルで実行します。それが
**Command Prompt** の場合、Kiro は、コマンドが 1 秒以内に終了したときにだけ
その応答を AI-DLC に渡します（Command Prompt にはシェル統合がないため、
Kiro はコマンドを送信してから 1 秒後に結果を読み取り、すべての実行で終了コード
-1 を表示します）。ほとんどの AI-DLC のステップはそれより時間がかかります。そのため Command Prompt では、すべての
AI-DLC コマンドがまず `AI-DLC reply: aidlc/.aidlc-sessions/replies/<id>.txt` という 1 行を出力し、
応答をターミナルだけでなくそのファイルにも書き込みます。Kiro が
その行または応答の途中で結果を切り捨てた場合、エージェントはコマンドを再実行するのではなく、
そのファイルから続けます（最新の 20 件の応答が、git が無視するフォルダーに保存されます）。
長いコマンドが終了していない場合、エージェントはそのことと、
Kiro のターミナルパネルであなたの回答を待っている可能性があることを伝えます。次のターンになっても
応答が届かない場合、エージェントはそのことを伝え、そのコマンドが何を変更するはずだったかを確認します。
エージェントが PowerShell の流儀で単一引用符で囲んだ
値は、Command Prompt からは引用符が付いたままの複数の単語として届きます。AI-DLC は
それらを 1 つの値に結合し直します。それでもステップに余分なターンがかかるため、Kiro 自身も
PowerShell を推奨しています。

Kiro のターミナルが Command Prompt で、このコンピューターでまだ回答していない場合、
AI-DLC は PowerShell に設定するかどうかを 1 回だけ尋ねます: 
`aidlc config` の実行中か、最初の Kiro IDE のチャットでです。`aidlc config --yes` は
推奨の回答を採用して設定します。はいと答えると、AI-DLC はその 1 つの Kiro
設定（`terminal.integrated.defaultProfile.windows`、すべてのプロジェクトが対象）だけを変更します。
その後、チャットに反映されるよう Kiro を再起動してください。いいえと答えるとそのままになり、AI-DLC は
再度尋ねません。

後で設定するには、エージェントに依頼するか、次を実行します:

```bash
aidlc config trust --kiro-terminal powershell --yes
```

Kiro 自身の **Terminal: Select Default Profile** コマンドでも同じことができます。
`/aidlc --doctor` は、Kiro のターミナルが Command Prompt の間、警告を表示します。

ターミナルが Command Prompt の間、Kiro IDE のすべてのチャットには、そのターミナルの
動作と、依頼されたときに PowerShell を設定する方法が伝えられるため、どのチャットでも
"set Kiro's terminal to PowerShell" を実行できます。Command Prompt のままにした場合、
ステップの応答が初めて失われたときに、エージェントは何が起きたかと
PowerShell ならそれを避けられることを 1 回だけ伝えます。再度尋ねることはありません。

### PowerShell without Kiro's shell integration on Windows

Kiro の PowerShell ターミナルは、Kiro がターミナルの起動時に読み込むスクリプトである
Kiro のシェル統合を通じて、コマンドの終了を Kiro に伝えます。そのスクリプトが
読み込まれないと、ターミナルは Command Prompt と同じように動作します: すべてのコマンドが、成功した場合でも
終了コード -1 を表示し、Kiro はコマンドが約 1 秒以内に出力した内容だけを渡します。
PowerShell の実行ポリシーが原因の 1 つです:
`Restricted` または `AllSigned` は、署名のない Kiro のスクリプトをメッセージなしで停止します。
Kiro の設定 `terminal.integrated.shellIntegration.enabled` をオフにした場合も
同じ影響があります。

AI-DLC は自身のコマンドからこれを検知します。そのようなターミナルで、
`aidlc/` フォルダーがあるプロジェクトでは、エージェントが実行するすべての AI-DLC コマンドが、Command
Prompt の場合と同様に、まず `AI-DLC reply:` の行を出力して応答をそのファイルに保存します（パイプに送られる
出力にはその行は付きません）。結果がその行で止まった場合や、その後の応答が切り捨てられた場合、
エージェントはコマンドを再実行するのではなく、そのファイルから続けます。チャットで初めてこれが起きたとき、エージェントは
ターミナルがシェル統合なしで実行されていることを 1 行で伝え、その後依頼の処理を
続けます。Kiro のターミナルで実行した `aidlc doctor` は、
変更すべき内容とともに `Kiro terminal` の警告を表示します。チャットで入力した `/aidlc --doctor` は
ターミナルの外で実行されるため、これを検知できません。

確認するには、Kiro の PowerShell ターミナルで次を実行します（Windows PowerShell と
PowerShell 7 はそれぞれ別のポリシーを持ちます）:

```powershell
Get-ExecutionPolicy -List
$ExecutionContext.SessionState.LanguageMode
```

`MachinePolicy` または `UserPolicy` が設定されている場合、または言語モードが
`FullLanguage` でない場合（組織のアプリケーション制御）は、組織が
管理しています。IT チームに相談してください。`Process` が設定されている場合、Kiro は起動元から
それを引き継いでいます: 代わりにスタートメニューから Kiro を起動してください。有効なポリシーが `CurrentUser` または
`LocalMachine` に由来し、`Restricted` または `AllSigned` である場合は、自分のアカウントで Kiro のスクリプトを
読み込めるようにしてから、Kiro を再起動できます:

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

AI-DLC がポリシーを自ら変更することはありません: それはコンピューターのセキュリティ設定だからです。

### Command cards end with "dministrator: ...powershell.exe" on Windows

Windows では、Kiro IDE のコマンドカードが
`dministrator: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe` のような行で
1 回か 2 回終わることがあります。これはターミナルウィンドウのタイトルで、Kiro が出力を表示する際に
先頭が切れたものです。コマンドの動作や AI-DLC が記録する内容には
影響しません。

Kiro の設定に `"terminal.integrated.windowsUseConptyDll": false` を追加している場合は、
それを削除して Kiro を再起動してください: これにより Kiro がコマンドの出力を折り返すようになり、
Code Generation が停止することがあります。

### Kiro memories carry old AI-DLC advice

Kiro IDE は、プロジェクトの外、ホームフォルダー配下の `.kiro/memories/memories.db`
（Windows では `%USERPROFILE%\.kiro\memories\memories.db`）にメモリを保存し、
開くすべてのワークスペースにそれを読み込むことがあります。以前の AI-DLC の実行中にエージェントが保存したメモリには、
そのプロジェクトとバージョンにしか当てはまらず、他の場所では誤っているか安全でない
診断や回避策が含まれていることがあります: `record-human-turn` のようなフック
コマンドを手で実行すること、
`AIDLC_SKIP_HUMAN_PRESENCE_GUARD=1` を設定すること、要約確認をオフにすること、
Kiro IDE を "sessionless" なハーネスと呼ぶことなどです。AI-DLC はエージェントに、そのような助言を
決して保存せず、決してそれに従って行動しないよう指示していますが、それ以前に保存されたメモリは引き続き読み込まれます。

エージェントが "per my memory" と言ってこれらのいずれかを提案した場合や、すでに Kiro IDE の中にいるのに
"from Kiro IDE" で再開するよう伝えてきた場合は、次のようにします:

1. そのメモリを無視し、`/aidlc` が現在示している内容に従うよう伝えます。ガードと
   チェックポイントを切り替えるのはあなたです: エージェントがコマンドを示し、あなたがそれを入力します。
2. そのような助言を与えるメモリだけを削除します: AI-DLC の
   フックコマンドを手で実行すること、ガードを飛ばす変数を設定すること、
   チェックポイントをオフにすること、Kiro IDE を "sessionless" と呼ぶこと、すでに Kiro IDE の中にいるのに
   Kiro IDE から再開するよう伝えることです。それ以外は残してください。正確な
   Kiro IDE に関するメモ（たとえばステータスラインがないこと）、その他の AI-DLC に関するメモ、
   `AIDLC_HOOK_DEBUG` のようなデバッグのヒントも含みます（メモリの管理については
   Kiro のドキュメントを確認してください）。
3. 何かを削除する前に `memories.db` のコピーを保存し、ファイルを直接編集する場合は先に Kiro IDE を
   閉じてください: このファイルには AI-DLC のものだけでなく、Kiro のすべてのメモリが
   含まれています。

## For framework developers

`dist/kiro-ide` は、`core/` + `harness/kiro-ide/` から
`bun scripts/package.ts kiro-ide` によって**生成**されます（`{{HARNESS_DIR}}` トークンを
`.kiro` に置き換え、`rules/` → `steering/` に名前を変更した core のコピー）。出力は
Git の無視対象でローカルのみです。`bun scripts/package.ts --check` は、独立した 2 つの
一時ルートで 2 回ビルドし、CI の決定論性ガードとして結果をバイト単位で比較します。
手で作成された
Kiro IDE の提供面は `harness/kiro-ide/` にあります: オーケストレータースキル
（`skills/aidlc/`）、常に含まれるアクティブメモリのステアリング（`steering/`）、
コンダクターの Markdown（`agents/aidlc-aidlc.md`）、フックアダプターと v2 フック JSON
ファイル（`hooks/`）、ペルソナの委任先シェル拒否（`delegate-shell-deny.ts`）、
オンボーディングの埋め込み内容です。これら
（または `core/`）を編集し、生成された `dist/kiro-ide` は決して手で編集しないでください。

このハーネスは、Kiro CLI (V2) ハーネス（`harness/kiro/`）と次の 4 点で異なります:
`/aidlc` スキルと Markdown の `agents/aidlc-aidlc.md` がコンダクターの提供面であること
（`settings/cli.json` はそのエージェントを Kiro CLI のデフォルトにもします）。v2 フック
JSON ファイルを提供すること（Kiro CLI (V2) ハーネスはエージェント JSON の `hooks` ブロックに依存します）。常設のルールを
CLI エージェントのリソースではなく、常に含まれるステアリングを通じて事前に読み込むこと。共有の Kiro
プロジェクションがコアのペルソナから Claude 専用の `disallowedTools` キーを取り除くこと。そして
マニフェストが、両方のハーネスが追加する `tools:` の行に加えて、すべてのペルソナに
`permissions.rules` frontmatter を追加することです（ステージがインラインで読み込む各ペルソナは、それらを含まない形でも
`.kiro/aidlc-common/personas/<agent>.md` として提供されます。これはリードが自身のチャットで読むファイルです）。
Kiro CLI (V2) ハーネスの agent-v1 JSON は提供しません。
[新しいハーネスへの移植](../../harness-engineering/09-porting-to-a-new-harness.md) を参照してください。

## Kiro CLI (V2)

> [!NOTE]
> Kiro CLI 上の AI-DLC は **Claude Opus 4.8** で最もよく動作します。これには
> **有料の Kiro プラン**が必要です。性能の低いモデルでは、コンダクターがステージの任意の
> ステップ（レビュアーによる確認、学びの手順）を飛ばしたり、承認ゲートを急いで通過したりすることがあります。
> Kiro のディストリビューションについては
> [このページの前半](#running-ai-dlc-on-kiro) で説明しています。

> [!IMPORTANT]
> このセクションでは、Kiro のエンジンを固定しない Kiro CLI (V2) ハーネス（`aidlc config --harness
> kiro-v2`）について説明します。`kiro-cli --v2` で起動してください（Kiro CLI 2.21.4 以降。それより古い場合は
> [Kiro CLI (V2) prerequisites](#kiro-cli-v2-prerequisites) を参照）。
> 対話式の `aidlc config` では Kiro CLI (V2) として表示されます。Kiro
> CLI 2.24.1 以降を検出すると、Kiro より優先してこれをデフォルトにしますが、Kiro IDE
> 自身のターミナルでは Kiro が先頭になります。Kiro CLI 2.24.1 以降で Kiro の v3 エンジンを使う場合は、
> Kiro として表示される Kiro ハーネス（`--harness kiro`）を使ってください: これは
> Kiro IDE だけでなく Kiro CLI（`kiro-cli --v3` で起動）でも動作します。既存の
> Kiro CLI (V2) プロジェクトでは、`aidlc config --harness kiro` が `.kiro/` をその場でそれに切り替え、
> `aidlc/` は保持します（ターミナルでは先に確認を求め、`--yes` を付けると
> 切り替えます。ターミナルがない場合や、`--quiet` または `--json` の指定時は
> 何も変更せず、`--yes` を付けたコマンドを表示します）。単純な `aidlc config` による更新や
> `--harness kiro-v2` では Kiro CLI (V2) のままです。
> AI-DLC 2.11.3 以前と、2026-11-04 および
> 2026-11-05 の 2.11.4 プレビューでは、2 つの Kiro の名前の読み方が逆です: そこでは Kiro CLI (V2) が
> `--harness kiro`、Kiro が `--harness kiro-ide` であり、
> `kiro-v2` は認識されません。`--harness kiro-ide` は、それらのリリースでもこのリリースでも
> Kiro を指します。

フレームワークのハーネスの 1 つである Kiro CLI (V2) ランタイムは、同じ AI-DLC の
方法論を [Kiro CLI](https://kiro.dev/docs/cli/) 上で実行します。1 つの決定論的なコア
（ツール、33 個のステージファイル、プロトコル、ナレッジ、センサー、スコープ、ルール）
はすべてのハーネスでバイト単位で共有されており、異なるのはシェル（スキル、エージェントの
設定、フックの配線、有効化）だけです。

ハーネス固有のオンボーディングは `.kiro/steering/aidlc-onboarding.md` にあり、
コンダクターエージェントの `resources` を通じて読み込まれます。ルートの `AGENTS.md` ブロック
はハーネスに依存せず、インストール済みの他のハーネスと共有されます。ただしエンジンのディレクトリは
別々である必要があります（Kiro CLI (V2) と Kiro の行は 1 つの `.kiro/`
インストールを共有できません）。

### Kiro CLI (V2) prerequisites

- **Kiro CLI ≥ 2.6**（`kiro-cli --version`）、ログイン済み（`kiro-cli login`）。この
  セクションのコマンドは `--v2` で Kiro CLI を V2 エンジンで起動します。これには
  Kiro CLI 2.21.4 以降が必要です。それより古い場合は、
  `kiro-cli chat --help` の表示内容を確認してください:
  - `--agent-engine` がある場合（2.19.2 にはあります）、`--v2` の代わりに
    `chat --agent-engine v2` を使います: `kiro-cli --v2 --agent aidlc` は
    `kiro-cli chat --agent-engine v2 --agent aidlc` に、`kiro-cli chat --v2 …`
    は `kiro-cli chat --agent-engine v2 …` になります。
  - ない場合は、エンジンのフラグを付けずに `kiro-cli chat` で起動します（たとえば
    `kiro-cli chat --agent aidlc`）。
- **bun** は、ソース／開発用の `dist/` プロジェクションを生成または実行する場合にのみ必要です。
  ネイティブインストールとバージョン付きのリリースランタイムは
  自己完結しています。

### Kiro CLI (V2) install

#### Kiro CLI (V2) native channel (recommended)

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
```

インストーラーは、リリースのメタデータ、実行ファイル、全ハーネス用ランタイムアーカイブを、公開されている SHA-256 チェックサムと照合して検証します。インストールされたランタイムに Bun、Node.js、Git は不要です。ハーネスの選択は `aidlc config` で行います。

Windows では、`install.ps1` をダウンロードし、その PowerShell ウィンドウで
`Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force` によってスクリプトを許可してから、
`& $installer` を実行してください。アカウントのスコープ、User PATH への自動登録、`-NoModifyPath` については
[Windows へのインストール](../18-install-and-lifecycle.md#windows-powershell) を参照してください。
エアギャップ環境向けのパッケージでは、同じ
`Set-ExecutionPolicy` の行の後に、Unix では
`install.sh --from <release-directory> --offline`、Windows では
`& $installer -From <release-directory> -Offline` を使います。

`aidlc config` は、最初のチャットセッションの前に Kiro のシェルを配置します。その後、
プロジェクトのルートから Kiro を起動します:

```bash
kiro-cli --v2
```

ネイティブのプロジェクションは、`aidlc engine *` のエンジンコマンドと、正確に指定された
読み取り専用コマンドおよび再有効化コマンド（`aidlc doctor`、`aidlc config <section> --show --json`
と、下のセッション開始の項に挙げるその他のコマンド）を許可します。また
`chat.defaultAgent: "aidlc"` を設定した `.kiro/settings/cli.json` も提供するため、エージェントのフラグなしで `/aidlc` が
有効になります。最初のワークフローの前に、チャットで `/aidlc --doctor` を実行してください。

#### Kiro CLI (V2) versioned manual-copy alternative

[インストールとライフサイクル: コピーチャネル](../18-install-and-lifecycle.md#copy-channel)
の説明に従って、特定のリリースの `aidlc-copy-runtime-X.Y.Z.tar.gz` をダウンロードして展開し、
`RUNTIME_ROOT` を展開した `runtime/` ディレクトリに設定します。

```bash
mkdir -p your-project/.kiro your-project/aidlc
cp -R "$RUNTIME_ROOT/kiro/.kiro/." your-project/.kiro/
cp -R "$RUNTIME_ROOT/kiro/aidlc/." your-project/aidlc/    # the workspace shell (spaces/default/memory) — a sibling of .kiro/, not inside it
cd your-project && bun .kiro/tools/aidlc.ts config --from "$RUNTIME_ROOT" --harness kiro-v2
```

`aidlc/` ディレクトリはワークスペースシェルです。エンジンが読み込む、事前にビルドされた
`aidlc/spaces/default/memory/` メソッドツリーが含まれています。これは `.kiro/` の**兄弟**
ディレクトリなので、別途コピーしてください（または
`$RUNTIME_ROOT/kiro/` ツリー全体を一度にコピーしてください）。
これがない場合、`/aidlc --doctor` は "workspace shell ready" チェックで失敗します。

バージョン付きのランタイムは、ネイティブの `aidlc` コマンドを使います。Bun 形式のソースプロジェクションが
必要なフレームワーク開発者は、リポジトリをクローンし、
`bun install --frozen-lockfile` と `bun scripts/package.ts` を実行してから、
代わりに Git の無視対象であるローカルの `dist/kiro/` 出力を使ってください。

提供される `.gitignore` は、ワークスペースのコミット対象と無視対象の区分を定めています:
ユーザーごとのカーソル（`aidlc/active-space`、`aidlc/spaces/*/intents/active-intent`）
とマシンローカルのランタイム（`aidlc/.aidlc-clone-id`、`runtime-graph.json`、センサーの
キャッシュ、`spaces/*/knowledge/.sources.local.json`）は追跡対象外のままとし、
共有の記録（メソッドメモリ、状態、監査シャード、成果物）は
git で共有されます。最後の行であるコピー自身のセットアップは、それらの行と AI-DLC の
`AGENTS.md` の担当部分を既存のファイルの内容の後に追加するか、プロジェクトにファイルがなければ
作成します。これにより、最初のワークフローの前に AI-DLC のルールが揃います。

その後、プロジェクトでセッションを開始します:

```bash
cd your-project && kiro-cli --v2
```

インストールでは `chat.defaultAgent: "aidlc"` を設定した `.kiro/settings/cli.json` が提供されるため、
AI-DLC のコンダクターエージェントがデフォルトで有効になり、`/aidlc` がそのまま動作します。
**このワークスペースの設定は、あなたが設定しているかもしれないグローバルのデフォルトエージェントより
優先されます**。自分のデフォルトを使いたい場合は、その設定を削除し、代わりに
`kiro-cli --v2 --agent aidlc` を使ってください。

Kiro CLI は、`api.subagentTimeout` 秒の間進捗を示さないヘルパー（レビュアー、ペルソナ、ユニットのビルダー）を
キャンセルします。デフォルトは 3600 で、AI-DLC は
これを設定しません。開始しないヘルパーがステージを 1 時間止めてしまう場合は、
自身の Kiro CLI 設定（`~/.kiro/settings/cli.json`。`900`
は 15 分）で値を下げられます。出力は進捗として数えられますが、無音は数えられません: それより長い間
何も出力しないコマンドを実行しているヘルパーもキャンセルされ、そのコマンドは
単独で実行され続けます。そのため、期限は最も長く無音になるコマンド
（デプロイや、最後にしか出力しないテスト実行など）より長くしてください。

提供されるエージェントはどれもモデルを固定していません: 固定された ID は、
利用者の Kiro インストールでそのモデルが有効な場合にしか解決されないためです。チャットと、コンダクターが
ディスパッチする 14 個のペルソナのそれぞれは、セッションモデル、つまり
Kiro 設定に保存されたモデル（何も保存していない場合は Kiro auto）で開始します。`/model`
は現在のチャットだけを変更するため、コンダクターはそれに従いますが、
ペルソナは引き続き保存されたモデルで開始します。ペルソナも切り替えるには、
`/model set-current-as-default` で切り替えを保存するか、
`aidlc config models --session-model <id>` でモデルを選んでください。

#### Kiro CLI (V2) session model and effort

Kiro CLI は各 AI-DLC セッションを 1 つのモデルで実行し、エージェントごとの effort を設定する
仕組みを持たないため、AI-DLC はセッションのモデルと effort を、プロジェクトではなく**個人の**
Kiro 設定（`~/.kiro/settings/cli.json`。Kiro 自身の
`/model set-current-as-default` が書き込むファイル）に保存します。モデルの一覧は
Kiro のアカウントごとに異なり、チームメイトのアカウントにないモデルをプロジェクトで指定すると、
そのメンバーが送るすべてのプロンプトが失敗します。またプロジェクトの `chat.modelDefaults` は、
プロジェクト内では個人のマップ全体を置き換えてしまうため、提供される `cli.json` には含まれていません。

初回セットアップのステップ 2 "Session model" では、Kiro のアカウントで利用できるモデルが
Kiro の順序で、各モデルのクレジット倍率と `preview` または
`internal` のタグとともに一覧表示されます。Kiro auto（Kiro 自身のデフォルトで、Kiro がタスクごとに
モデルを選ぶ）のもとでは、AI-DLC はモデルを選ぶことを推奨します。そうすることで effort のプリセットが
そのモデルに適用されます。プリセットは、対話式セッション全体に 1 つの effort を設定します
（Kiro CLI 2.28.0 は、保存された effort を
`kiro-cli chat --v2 --no-interactive` の実行には適用しません。その場合は `--effort` を渡してください）:

| プリセット | セッションの effort |
|--------|----------------|
| `minimal` | `low` |
| `balanced` | `medium` |
| `thorough` | `xhigh`（extra-high） |

そのレベルを持たないモデルには 1 つ下のレベルが適用され、effort の設定を持たない
モデルではモデルだけが保持されます。推奨のデフォルトは、そのモデルについて effort をまだ保存していない場合にのみ
effort を設定します: すでに保存した effort は
そのまま残り、プリセットと異なる場合はセットアップがそのことを 1 行で伝えます。自分で
プリセットを選ぶ（ステップ 3 または `aidlc config models` で）か、
`aidlc config models` でセッションモデルを選ぶと、プリセットの effort が設定されます。エージェントやグループの設定では
変更されません。`aidlc config models` では後から同じ
選択肢（"1 session model, 2 preset"）が提示され、
`aidlc config models --session-model <id>` はアカウントの一覧にあるモデルを
確認なしで保存します。保存したモデルがアカウントで提供されなくなると、すべての
プロンプトが失敗するため、セットアップはそれを保持せずに別のモデルを尋ねます。`--dry-run` は
個人の Kiro 設定の変更も表示し、何も書き込みません。Kiro が書き込みを拒否した場合、
AI-DLC は何が保存されたかを正確に伝え、`config models` は終了コード 5（対応が必要）で終了します。
`aidlc doctor` は現在の設定を確認します: モデルが引き続き提供されていること、保存された
effort がそのモデルで利用できるものであること（プリセットが記録され、Kiro がモデルの
effort レベルを一覧表示する場合）、プロジェクトのファイルがそれを上書きしていないことです。保存した
effort がプリセットと異なる場合、それはあなたの選択です: doctor はプリセットのレベルと
それを適用するコマンドを示しますが、警告は出しません。以前のリリースでセットアップした
プロジェクトを更新すると、AI-DLC の古い effort のマップ（extra-high の `claude-opus-4.8`）が
`.kiro/settings/cli.json` から削除され、そのことが伝えられます。セッションモデルを選ぶには
`aidlc config models` を実行してください。
1 つのセッションだけ上書きするには、チャットで `/effort <level>` を使うか、`kiro-cli chat --v2
--effort <level>` を使います（low|medium|high|xhigh|max。古い Kiro CLI については [Kiro CLI (V2) prerequisites](#kiro-cli-v2-prerequisites) を参照）。Kiro IDE は `cli.json` を読みません。

### Kiro CLI (V2) refresh and version skew

`aidlc update` はマシンのランタイムを更新しますが、プロジェクトのファイルは変更しません。
`aidlc doctor` は、プロジェクトのスタンプが選択されたエンジンと異なる場合に報告します。
ワークフローの合間に、次のコマンドで更新内容をプレビューしてから適用します:

```bash
aidlc config --dry-run
aidlc config
```

config はユーザーが所有する内容を保持し、フレームワークのファイルへのローカルの編集を
競合として報告します。ワークフローが開いている間の更新も実行され、開いている作業はそのまま続くことが
伝えられます。アップグレードとロールバックはプロジェクトを変更しないため、ワークフロー中でも
安全です。

### Kiro CLI (V2) usage

プロジェクトで `kiro-cli --v2` を起動し、
`/aidlc <description>` でコンダクターを呼び出します。`/aidlc --status` で現在位置を確認でき、
`/aidlc --config [section]` でセッション内でプロジェクトの設定変更をまとめて行えます。
`/aidlc --doctor`、`--stage`、`--phase`、`--depth`、`--test-strategy` もすべて動作します。ワークスペースの
移動には `/aidlc intent [name]`、`/aidlc space [name]`、
`/aidlc space-create <name>` を使います。ステージごと（`/aidlc-domain-design`）および
スコープごと（`/aidlc-feature`）のランナースキルもインストールされます。

status、doctor、help、version、ワークスペース移動のコマンドは、モデルがそれらをワークフローの作業に
変えてしまう前に、Kiro のフックによってディスパッチされます。その子プロセスの
出力は UTF-8 としてデコードされ、ターミナルのプロトコル／制御バイトはその
プレーンテキストの中継境界でのみ除去されます。通常の Unicode、パス、タブ、改行、
エスケープのように見えるリテラルのテキストは変更されません。

**セッションはプロジェクトのルートから開始してください。** ネイティブインストールでは、
インストールされた `aidlc engine ...` コマンドと、次に挙げる読み取り専用コマンドおよび再有効化コマンドを、
記載どおりの形の `aidlc ...` として事前承認します。
ソース／開発用のコピーが事前承認するのは、
プロジェクト相対で実行される AI-DLC 自身のワークフローコマンドだけです: エンジンコマンド
（`bun .kiro/tools/aidlc.ts engine ...`）、読み取り専用の `doctor`、`version`、
`--version`、`--doctor`（`--verbose` の有無を問わない doctor）、`status`、
`--status`、`config --help`、`config --show` と
`config <section> --show`（`--json` の有無を問わない）、
`config <section> --help`、
`config flags --clear-bypass <switch> --yes` によるチェックの再有効化、そして AI-DLC のツールスクリプト
（`bun .kiro/tools/aidlc-<tool>.ts`）です。それ以外の `config` の変更（チェックを
オフにすることを含む）、マシンの AI-DLC インストールを変更するコマンド（`use`、`update`、`rollback`、`uninstall`、
`system`）とその背後にあるツールスクリプト（`aidlc-doctor.ts`、`aidlc-init.ts`、
`aidlc-lifecycle.ts`、`aidlc-machine-config.ts`）、絶対パス、
`KIRO_PROJECT_DIR` の展開、そして事前承認されていないコマンドを追加する行
（前に置いた `cd`、または別のコマンドへのパイプや `&&`）では、引き続き確認を
求めます。

**引き続き確認を求めるもの。** developer エージェントと `aidlc` エージェントは、
プロジェクトのファイル、つまりプロジェクト内でトップレベルの名前がドットで始まらないパスに、
確認なしで書き込みます。並列ビルドでは、ワーカーはユニットの
チェックアウト（`.aidlc/worktrees/<unit>/`）に同じように書き込みます: そこでのコードと記録は
確認なしで反映され、チェックアウト自身の `.git`、`.gitignore`
などのドットで始まる項目では確認を求めます。Kiro CLI の v2 エンジンでは、チェックアウトのドット
フォルダー（その `.kiro/`、`.github/`）も確認なしで反映され、ユニットが取り込まれるまで
そのチェックアウト内にとどまります。`.kiro/`（`aidlc` エージェント自身の
`.kiro/sensors/` ファイルを除く）、`.git/`、その他の
トップレベルのドット項目への書き込みや、プロジェクト外へのあらゆる書き込みでは、引き続き確認を求めます。
プロジェクト独自のテストやビルドのコマンド（`bun test`、`npm test`、`pytest` など）、
`git push`、再帰的な削除も引き続き確認を求めるため、実行前に 1 つずつ
確認できます。並列ビルドでは、ワーカーとレビュアーは、あなたが承認したテストコマンドを
AI-DLC 自身の `swarm check` を通じて実行するため、確認は求められません。
並列ビルドのワーカーやレビュアーが確認を求めると、Kiro はそれらの要求を
"tool approval pending" ボックスにまとめます。そこで (f) **Approve all pending and
auto-approve future subagent requests** を選ぶと、後から開始するレビュアーも含め、
それらが求めるものを確認なしで実行できるようになります。ただし Kiro CLI
3.1.0 では、同じチャットの後の時点で、たとえば AI-DLC の質問に答えた後などに、
そのボックスが再び表示されることがあります。再び表示された場合は、もう一度 (f) を
選べます。メインチャットからの要求は引き続きこれまでどおり確認を求め、Kiro は (f) について
設定を書き込みません。

**承認者のいないセッションは、確認を求めずに停止します。** 事前承認されたもの以外は、
対話式の回答が必要です。`kiro-cli chat
--v2 --no-interactive` では尋ねる相手がいないため、Kiro はコマンドを
`non-interactive mode (no user to approve)` として即座に拒否します。ACP 経由では、クライアントが
`session/request_permission` に応答する必要があります。これらの要求を無視するクライアントは、
権限エラーとまったく同じように見えます。`--trust-all-tools` は、`git push` や再帰的な削除も含め、
すべてのコマンドを確認なしで実行します。シェルへの全面的なアクセスが許容される
使い捨てのサンドボックス内でのみ使ってください。

**どの Kiro CLI エンジンがフックを実行するか。** このディストリビューションは、
`.kiro/agents/aidlc.json` にフックを登録します。Kiro CLI は、`aidlc` エージェントが有効な間、
ACP 経由を含め、v2 エンジンでそのブロックを実行します: `kiro-cli acp` は
`kiro-cli --v2` と同じように `chat.defaultAgent` からエージェントを選びます。Kiro CLI の
v3 エンジンは、AI-DLC が提供したままのファイルを実行しません。Kiro が
v3 向けにアップグレードした後はそのファイルを実行し、v2 エンジンもアップグレードされたファイルを引き続き実行します。

Kiro CLI 2.37 以降、Kiro は v3 で起動したとき（`kiro-cli --v3`、または V3 を選択済みであれば
単純な `kiro-cli`）に、自動的にそのアップグレードを行います。ただし、自身の Kiro CLI 設定で
`chat.enableAutoAgentUpgrade` が `false` の場合は行いません。
Kiro は `.kiro/agents/` 内のすべてのエージェント JSON ファイルを書き換え（隣にある `.md` のプロンプトは
そのまま）、それぞれの元のファイルを隣に
`<name>.json.bak` として保存します（すでにある場合は `.json.bak.1` 以降）。そのため
`git status` には、そこにある各エージェント JSON ファイルが変更され、隣に新しいバックアップがあると表示されます。
これらは AI-DLC が書き込んだファイルではありません。
Kiro CLI 3.1.0 では、Kiro がフォルダーを信頼するかどうかをまだ尋ねている間にファイルが変更されており、
信頼しないと回答してもファイルは変更されたままでした。
Kiro は画面上でそのことについて何も伝えませんでした。その後は、v3 でもフックが実行されます。
`aidlc doctor --verbose` は、それらのエージェント JSON ファイルをあなたの変更ではなく Kiro によるアップグレードとして数え、
更新は、
[更新時の安全性](../18-install-and-lifecycle.md#refresh-safety) に別の記載がある場合を除き、
それらを Kiro が残したままの状態に保ちます。アップグレードを元に戻すには、まず `kiro-cli settings
chat.enableAutoAgentUpgrade false` を実行してください。そうしないと、Kiro は次に v3 で
起動したときに再びファイルをアップグレードします。次に、各エージェント JSON ファイルの最新のバックアップ（最も大きい
`.json.bak.N`、他にない場合は `.json.bak`）の名前を `.json` に戻し、
`kiro-cli --v2` で Kiro CLI を起動します: v3 は、アップグレードしていないファイルのフックを
一切実行しません。

その設定が `false` の間、Kiro はファイルをアップグレードしません。また
Kiro CLI 3.1.0 では、`kiro-cli acp --agent-engine v3` は何もアップグレードしませんでした。
ファイルがアップグレードされていない状態の v3 のセッションでは、これらのフックは一切実行されず、
別のエージェントに切り替えたセッションでも同様です。フックがないと `HUMAN_TURN` のレシートが
記録されないため、すべての承認と確認が拒否されます。書き込みイベントも
記録されないため、レビューも拒否されます。

Kiro 自身が作るプロンプト（たとえば、新しい
kiro-cli がサブエージェントの終了後に送る `[SYSTEM] Sub-agent synthesis:` メッセージ）は、あなたのターンとして数えられません: 
`HOST_TURN` として記録され、質問には回答しません。あなた自身のメッセージは、その文言にかかわらず数えられます。Kiro の
チャットの記録がどちらなのかを示しており、不明なものはあなたのメッセージとして扱われます。
最初のワークフローステージの後、`/aidlc --doctor` はこれを "Hooks have
never executed" と報告します。それより前は、doctor は AI-DLC のフックが
このプロジェクトでまだ実行されていないことを警告し、同じ手順を示します。別のエージェントを選んでいる場合は、
`/agent` と入力して `aidlc` を選び、同じチャットで続けてください。`aidlc` が
その一覧にない場合、Kiro はファイルがアップグレードされていない状態で v3 で動作しています（Kiro は起動画面に `agent
"aidlc" needs upgrading for this agent engine` と表示します）。この場合、
v3 で再起動するだけでは解決しません。AI-DLC の案内には次の方法が示されるので、
いずれかを選んでください:

- `/upgrade-agent` と入力してワークスペースのエージェントを選び、Kiro を終了して、
  このフォルダーで `kiro-cli --v3` を再起動します。入力したチャット自体は Kiro の
  デフォルトのエージェントのままです。
- Kiro CLI の v3 エンジン向けに作られた [Kiro](#running-ai-dlc-on-kiro) ディストリビューションに
  切り替えます: `aidlc config --harness kiro` は `.kiro/` をその場でそれに切り替え、
  `aidlc/` は保持します（ターミナルでは先に確認を求め、`--yes` を付けると切り替えます。
  ターミナルがない場合や、`--quiet` または `--json` の指定時は何も変更せず、
  `--yes` を付けたコマンドを表示します）。その後 `kiro-cli --v3` を起動します（フォルダーを信頼するかどうかを
  尋ねられ、内容を把握している場合は **Trust, continue** を選びます）。
- Kiro を終了し、このフォルダーで `kiro-cli --v2 --agent aidlc` を起動します。

ACP クライアントは `kiro-cli acp --agent-engine v2` で起動します。Kiro に切り替えた後は、
[Kiro's install](#install) の `.kiro/settings/cli.json` の項目に示されている方法で起動します。
[Kiro CLI のフックが動かない](../15-troubleshooting.md#kiro-cli-hooks-not-running)
を参照してください。

#### When a Kiro CLI (V2) chat fills up

Kiro はチャットがどのくらい埋まっているかをあなたに表示しますが、作業中の AI-DLC にはその数値を渡しません。
そのため AI-DLC は、Kiro が各ターンの終わりにチャットについて保存する数値
（`~/.kiro/sessions/cli/<chat id>.json`、読み取り専用）を読みます。それが 80% 以上になると、
次の承認の質問またはユニットのチェックポイントに、チャットがどのくらい埋まっているかを伝える 1 行が付き、
一時停止して新しいチャットで続けることが提案されます。
作業での最初のターンですでにチャットが半分以上埋まり、
それを埋めたモデルのまま実行されている場合、そのモデルで新しいチャットを始めても
ほぼ同じくらい埋まった状態で始まるため、その行は新しいチャットを提案する代わりにそのことを伝えます。判断はあなたの回答次第です:
自動的に一時停止することはなく、その行はチャットごとに 1 回だけ伝えられます。

この行は、ステージの途中（いずれにせよ尋ねられる質問と一緒にのみ表示されます）、
Kiro がチャットの最初のターンを保存する前、このマシンに記録のないチャット、
または Kiro のアップデートで数値の保存方法が変わった場合には表示されません。
Kiro 自身の数値が上がってきたら、同じことを手動で行えます:

1. ステージの境界で停止します: 承認すると同時に、同じ返答で今は止めたいと伝えるか、
   `/aidlc park` と入力します。すると `/aidlc --status` に
   `Paused at <stage> (you stopped it); pick it back up with /aidlc --resume` と表示されます。
2. `/chat new` と入力して新しいチャットを開きます。Kiro は以前のチャットを保持しており、`/chat resume`
   で戻れます。現在のチャットをリセットしてしまう `/clear` ではなく、こちらを使ってください。
3. `/aidlc --resume` と入力します。`/aidlc --status` に Paused と表示されなくなります。

以前のチャットで話し合ってもファイルに書き込まれなかった内容は引き継がれません。
そのため、ステージの終わりが最適なタイミングです（
[ワークフロー途中でのモデル変更](../11-session-management.md#changing-model-mid-workflow) を参照）。

### What's different on Kiro CLI (V2)

| 領域 | Claude Code | Kiro CLI (V2) |
|------|-------------|----------|
| ゲートと質問 | `AskUserQuestion` ウィジェット | 番号付きの散文の選択肢（番号で返答）。`[Answer]:` タグを持つ質問ファイルが引き続き信頼できる情報源です |
| ステータスライン | 現在のステージ + モデル + コンテキスト % | 利用できません: `/aidlc --status` と各ゲートの進捗表示を使ってください。チャットが埋まってきた場合は [When a Kiro CLI (V2) chat fills up](#when-a-kiro-cli-v2-chat-fills-up) を参照してください |
| ディスパッチされるステージ（2.1 pipeline、2.2 subagent、2.4 mob、3.5 subagent） | `Task` ツール | Kiro の `subagent` ツール → エージェントの設定（14 個のペルソナすべてに設定が提供されます） |
| Construction のスウォーム | 並列の `Task` を基本とし、任意で ultracode Workflow | サブエージェントへの展開のみ。`AIDLC_USE_SWARM=1` は何もしないことが通知されます |
| セッションの監査イベント | `SESSION_STARTED/RESUMED/ENDED`、`SESSION_COMPACTED` | `SESSION_STARTED` のみ（Kiro にはセッション終了／コンパクション前のフックがありません） |
| 転送ループの強制（Stop フック） | 対話式 + ヘッドレス | 対話式セッションのみ。`--no-interactive` の実行では Stop フックによるブロックが守られません |
| 権限 | `settings.json` の許可リスト | ソースから生成したプロジェクション: AI-DLC 自身のプロジェクト相対のワークフローコマンド（エンジン、読み取り専用のディスパッチャーコマンド、AI-DLC のツールスクリプト）。ネイティブおよびバージョン付きのリリースランタイム: `aidlc engine *`。その他のシェルコマンドは確認を求めます。 |
| ウェルカムメッセージ | セッション開始時に `settings.json` の `companyAnnouncements` から表示 | なし。Kiro にはウェルカム表示に相当する機能がなく、セッション開始フックは再開のコンテキストだけを注入します |
| MCP サーバー | 5 つを提供（`.mcp.json`: `context7` + AWS の 4 サーバー） | 同じ 5 つを `.kiro/settings/mcp.json` で提供し、すべてデフォルトで無効です。有効にするには、サーバーごとに `"disabled": false` に切り替えます。Kiro は設定された HTTP ヘッダーの値を、環境変数のプレースホルダーを展開せずにそのまま送るため、Kiro では Context7 はキーなしで使います。AI-DLC 自身のチャットと 14 個の委任用ペルソナは、最後に config を実行した時点でこのファイルと `~/.kiro/settings/mcp.json` にあるすべてのサーバーを使え、各呼び出しで先に確認を求めます。後から追加したサーバーは、config を再実行した後に使えるようになります。Kiro CLI の v3 エンジンでは、ペルソナは config を実行しなくても、2 番目のステップからそれらのファイルにあるすべてのサーバーを使えます。 |

それ以外のすべて（ステートマシン、監査証跡、インテントの記録ディレクトリ
（`aidlc/spaces/<space>/intents/<YYMMDD>-<label>/`）配下の成果物、学びの
手順、センサー、スコープ、深度／テスト戦略）は同じように動作します。実際に
同一だからです: ネイティブインストールは `aidlc` を通じてディスパッチし、ソースのコピーは
`.kiro/tools/` の対応するツールを実行します。

プロジェクトの `aidlc/` ワークスペースはハーネスに依存しません。プロジェクトをハーネス間で移動すること
（または両方を並行して実行すること）はサポートされていますが、テストされていません。`/aidlc
--doctor` は、アクティブなワークフローがある状態で競合するハーネスのセットアップを検出すると警告します。

### Kiro CLI (V2) for framework developers

`dist/kiro` は、`core/` + `harness/kiro/` から
`bun scripts/package.ts kiro` によって**生成**されます（`{{HARNESS_DIR}}` トークンを
`.kiro` に置き換え、`rules/` → `steering/` に名前を変更した core のコピー）。出力は
Git の無視対象でローカルのみです。`bun scripts/package.ts --check` は、独立した 2 つの
一時ルートで 2 回ビルドし、CI の決定論性ガードとして結果をバイト単位で比較します。
手で作成された Kiro の提供面は `harness/kiro/` にあります: オーケストレータースキル
（`skills/aidlc/`）、エージェントの JSON（`agents/`）、フックアダプター
（`hooks/aidlc-kiro-adapter.ts`）、`settings/cli.json`、`settings/mcp.json`、`onboarding.fills.ts` です。
これら（または `core/`）を編集し、生成された `dist/kiro` は決して手で編集しないでください。
[新しいハーネスへの移植](../../harness-engineering/09-porting-to-a-new-harness.md) を参照してください。
ビルドは、それらのエージェント JSON のすべてのフックコマンドに `; exit $LASTEXITCODE` を付加します
（マニフェストの `hookCommandSuffix`）: Kiro CLI は Windows では
PowerShell で、それ以外では bash でフックを起動し、この形にするとどちらでもフックの終了コード
2 が Kiro に返されるため、Windows でも拒否によってツール呼び出しが停止します。

Claude 向けの対応するテストと並んで、ライブの TUI ジャーニーテストがあります:
`tests/e2e/t-tui-kiro-intent-capture.serial.test.ts` は、提供されたツリーに対して
キー入力で `kiro-cli chat` を操作します（番号付きの散文のゲートには
"1" = 推奨の選択肢で回答し、ディスク上の状態で終了を判定します）。
`AIDLC_KIRO_TUI_LIVE=1` で有効にします。tmux、`kiro-cli`、または
ログイン済みの Kiro セッションがない場合は、理由を示してスキップします。

## Next steps

インストールと有効化は済みましたか？ 方法論はどのハーネスでも同じです。
ハーネスに依存しない各章に進んでください:

- [最初のワークフロー](../02-your-first-workflow.md): 注釈付きのエンドツーエンドの実行例です。
- [フェーズとステージ](../04-phases-and-stages.md): 5 つのフェーズと 33 のステージです。
- [スコープ、深さ、テスト戦略](../05-scopes-and-depth.md): 実行の規模を適切に決める方法です。
- [用語集](../glossary.md): すべての用語の定義です。

その他のハーネス: [Codex CLI での AI-DLC](codex-cli.md) · [Cursor での AI-DLC](cursor.md) · [ハーネスファミリーの索引](README.md)。
