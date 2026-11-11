# Getting Started

このページでは、何もない状態から最初のステージの承認までを案内します: AI-DLC をインストールし、
`aidlc config` でプロジェクトをセットアップし、`aidlc doctor` で確認し、その後 Claude Code で
`/aidlc` を実行して最初の承認まで進みます。以下の画面はすべて、新しいホームフォルダーでの
実際の実行から取ったもので、省略した箇所はそう明記しています。
サンプルでは、`X.Y.Z` はインストールしたリリースを、`/home/you` はホーム
フォルダーを表し、Claude Code の描画文字（枠、矢印、チェックマーク）は
プレーンテキストで示すか省略しています。他のツールではいくつかの手順が異なり、それらは
[他のツールの場合](#on-the-other-tools)に記載しています。

## Before you start

- **インストール済みでサインイン済みのハーネス。** このページでは Claude Code を使います。3 つの
  ツールについて、AI-DLC は動作する最も古いバージョンを確認し、お使いのバージョンが古い場合は
  `aidlc doctor` がそれを知らせます:

  | ツール | 最も古いバージョン |
  | --- | --- |
  | Codex CLI | 0.145.0 |
  | GitHub Copilot CLI | 1.0.74（VS Code のみのインストールでは CLI が含まれない場合があります） |
  | Kiro CLI | Kiro CLI (V2) は 2.6.0、Kiro は 2.24.1 |

- **Codex CLI と Cursor には Git。** Codex CLI はプロジェクトのフックを
  Git リポジトリ内でしか見つけられず、Cursor はリポジトリ外ではフックを省略することがあるため、新しいフォルダーでは
  `aidlc config` の前に `git init` を実行してください。
- **それ以外は不要です。** ネイティブインストーラーが AI-DLC の実行に必要なものをすべて用意するため、
  Bun や Node.js は必要ありません。

## 1. Install AI-DLC

macOS、Linux、または WSL:

```bash
curl -fsSL https://github.com/awslabs/aidlc-workflows/releases/latest/download/install.sh | sh
```

Windows PowerShell:

```powershell
irm https://github.com/awslabs/aidlc-workflows/releases/latest/download/install.ps1 | iex
```

インストーラーは、AI-DLC に必要なものをすべてセットアップします: ネイティブの `aidlc` コマンドと
すべてのハーネスランタイムです。Bun や Node.js をインストールする必要はありません。ネイティブ
インストールでは、フックとコマンドは `aidlc` を通じて実行され、`bun` は呼び出されません。
Windows では、現在のアカウント向けにインストールし、bin ディレクトリを永続的なユーザー PATH に
登録し、現在の PowerShell プロセスを更新します。通常の PowerShell ウィンドウから実行してください。
"Run as administrator" で開いたウィンドウでは警告と確認が表示されます。管理者としてのインストールは安全性が低いためです。すでに開いていたターミナルアプリや IDE（Kiro、VS Code、Cursor）は起動時の PATH を保持し続けるため、そこで `aidlc` を使う前に一度終了して開き直してください。その中で新しいタブを開くだけでは不十分です。
両方の PATH 変更を省略し、代わりに表示された直接のコマンドを呼び出すには、[`-NoModifyPath`](18-install-and-lifecycle.md#windows-powershell) を使います。macOS、
Linux、WSL では、`aidlc` が見つからない場合はインストーラーの PATH に関する指示に従ってください。

macOS と Linux では、`~/.local/bin` がすでに PATH にある場合、
次のように終了します:

```text
PASS installed AI-DLC X.Y.Z with all harness runtimes
Plugin marketplace for Claude Code and Codex: /home/you/.local/share/aidlc/plugins (add it once; updates keep it current)
Next: aidlc config
```

PATH にない場合は次のようになります:

```text
PASS installed AI-DLC X.Y.Z with all harness runtimes
Plugin marketplace for Claude Code and Codex: /home/you/.local/share/aidlc/plugins (add it once; updates keep it current)
Add AI-DLC to PATH for this shell:
  export PATH="/home/you/.local/bin:$PATH"
Then run: aidlc config
```

表示された `export` の行を実行し（保持するにはシェルの起動ファイルに追加します）、
コマンドを確認します:

```text
$ aidlc version
aidlc X.Y.Z (runtime X.Y.Z)
```

これらの行の前に、ダウンロードではリリースファイルごとに `Downloaded <file>` の行が 1 行
表示され、新しい GitHub CLI がないマシンでは
`WARN GitHub CLI attestation verification is unavailable; continuing with SHA-256 release checksums.` も表示されます。
SHA-256 のチェックは常に実行され、GitHub CLI は署名付きの来歴チェックを追加します
（[リリースの認証](18-install-and-lifecycle.md#release-authentication)）。
Windows では、インストーラー自身の結果は `Installed AI-DLC X.Y.Z` で始まり、
以降の画面のパスは `.claude\` のように Windows の区切り文字を使います。
プラグインマーケットプレイスの行は AI-DLC 独自のプラグイン用です: そのパスを
Claude Code または Codex で一度追加するか（[プラグイン](18-install-and-lifecycle.md#plugins)）、
使わない場合は省略してください。

プロジェクトのファイルを自分で管理したい、またはネイティブの実行ファイルを
インストールできない場合は？ Bun が必要になるのは手動コピーの方法だけです:
[Bun](https://bun.sh/) をインストールし、
[リリース](https://github.com/awslabs/aidlc-workflows/releases/latest)から `aidlc-copy-runtime-X.Y.Z.tar.gz` をダウンロードし、
`runtime/<harness>/` ディレクトリ全体をプロジェクトにコピーし、[コピーチャネル](18-install-and-lifecycle.md#copy-channel)に示すとおり、
そのセットアップを一度実行します。
コピーによって `.gitignore` や `AGENTS.md` が置き換えられることはありません。AI-DLC は独自の
行をそれらに追加します。手動コピーの方法では、ネイティブの `aidlc`
コマンドは必要ありません。

## 2. Set up your project

プロジェクトのフォルダーで `aidlc config` を実行します。AI-DLC は Git のルートを上位に
探しに行かないため、このフォルダーが `aidlc/` の置き場所になります。複数のリポジトリがある場合は、
[`aidlc/` の置き場所](03-spaces-and-intents.md#where-aidlc-lives)を参照してください。この実行では
新しい空のフォルダーを使います:

```text
$ mkdir inventory-api && cd inventory-api && git init -q
$ aidlc config

  AI-DLC setup - first run in this project.

  Checked your machine and this repo:

    Harness    Claude Code detected  (2.1.296 on your PATH)
    Project    git repo, no AI-DLC files yet
    AWS        credentials not found
    Runtime    aidlc on this shell's PATH only: start Claude Code from a terminal

  Set up AI-DLC for Claude Code with recommended defaults?

    1. Yes, use recommended defaults   MCP servers on, all plugins, current model provider
                                       preserved
                                       Records balanced (default): medium project agent effort for
                                       deciding, reviewing, and writing up; your session
                                       (conductor) effort stays unchanged.
    2. No, customize step by step      harness, provider, preset, plugins, MCP, record layer
    3. Exit, nothing written

  Choice [1]:
```

回答するまでは何も書き込まれません。確認した内容は次のとおりです:

- **Harness**: PATH 上で見つかったツールで、セットアップの対象になります。
  複数ある場合は、まずどれを使うかを尋ねます。1 つもない場合は、
  [ハーネスのインストール前にセットアップする](#set-up-before-you-install-a-harness)を参照してください。
- **Project**: ここが Git リポジトリかどうか、また AI-DLC がすでに
  セットアップされているかどうか。セットアップ済みのプロジェクトでは、`aidlc config` は
  これらの質問をする代わりに、現在の内容を確認します
  （[プロジェクトの構成または更新](18-install-and-lifecycle.md#configure-or-refresh-a-project)）。
- **AWS**: AWS の認証情報が見つかったかどうか。これはモデルプロバイダーとして
  Amazon Bedrock を選ぶ場合にのみ関係します。
- **Runtime**: AI-DLC のフックが `aidlc` コマンドを見つけられるかどうか。ターミナルから
  起動した Claude Code はそのターミナルの PATH をフックに引き継ぐため、シェルの PATH に `aidlc`
  があれば十分で、この行はターミナルから起動するよう案内します。
  ここで検出された PATH の修正はセットアップの最後に表示されます。代わりにデスクトップの
  アイコンから Claude Code を起動する場合は、手順 4 で `aidlc doctor` が追加すべき内容を
  知らせます。

Enter キーを押すと推奨のデフォルトが使われます。記録される内容は次のとおりです:

- **MCP servers on**: 5 つの[付属サーバー](#mcp-servers-optional)を含む
  `.mcp.json`。
- **All plugins**: インストール済みのすべてのプラグインが有効になります。
- **Current model provider preserved**: AI-DLC はプロバイダーの設定を書き込まないため、
  Claude Code はすでに使っているプロバイダーとログインを維持します。
- **balanced プリセット**: `aidlc.settings.json` に、AI-DLC 自身の
  エージェントが使う effort を記録します。あなた自身のチャットは、そのモデルと effort を維持します
  （[モデルと effort の選択](18-install-and-lifecycle.md#choosing-a-model-and-effort)）。

続いて、Claude Code のステータスラインについてもう 1 つ質問します:

```text
  Show AI-DLC's progress in your status line? It takes the place of your own while you work in this
  project.

    1. Yes  (recommended)
    2. No, keep mine

  Choice [1]:

  AI-DLC's status line shows in this project; /aidlc --statusline off turns it off.

  Writing project files ... done  (.claude/ and aidlc/, 316 files)
  Recording model preset ... done  (aidlc.settings.json in this project)
  Setup complete. Start your first workflow:

    claude                         open Claude Code in this repo
    /aidlc "what you want built"  describe your first intent
```

この回答は `.claude/settings.local.json` と
`aidlc.settings.local.json` に保存され、これらはあなた専用です（どちらも git で無視されます）。

### Customize instead

選択肢 2 では 6 つの手順を順に進みます。Enter キーを押すとそれぞれ `[default]` が使われます。この実行では、
プロバイダーで Amazon Bedrock を選んだ以外はすべてデフォルトのままにしました:

```text
  Customize setup - 6 steps, Enter accepts the [default].

  Step 1 of 6 - Harness
  Detected on this machine: Claude Code.

    1. Claude Code     (detected, default)
    2. Codex CLI
    3. GitHub Copilot
    4. Cursor
    5. Kiro CLI (V2)
    6. Kiro            Kiro IDE and Kiro CLI on its v3 engine
    7. opencode
  Harness [1]:
  Using Claude Code.

  Step 2 of 6 - Model provider
  No AWS credentials were detected.
    1. keep current     inherit the provider already configured in the harness (default)
    2. amazon-bedrock   write the AWS region and profile to settings.local.json, just for you
  Provider [1]: 2
  AWS region [us-east-1]:
  AWS profile [default credential chain]:
  Using amazon-bedrock in us-east-1 with the default credential chain.

  Step 3 of 6 - Model effort preset
    1. balanced    medium effort for deciding, reviewing, and writing up (recommended, default)
    2. thorough    session effort for deciding and writing up, extra-high reviewing
    3. minimal     medium deciding and reviewing, low writing up
    4. unchanged   records no preset and keeps existing settings; new projects use shipped defaults
                   where agents inherit your session's model and effort
  Preset [1]:
  Using the balanced preset.

  Step 4 of 6 - Plugins
    1. all installed   (default)
    2. none optional   core AI-DLC only
    3. choose          comma-separated installed names
  Plugins [1]:
  Using all installed plugins.

  Step 5 of 6 - MCP servers
    1. on
    2. off
  MCP [1]:
  MCP servers on.

  Step 6 of 6 - Where to record the model preset
    1. this project, committed     aidlc.settings.json - shared with your team  (default)
    2. this project, just for you  aidlc.settings.local.json - gitignored
    3. this machine                every project you set up here
  Preset in [1]:
  Recording the model preset in this project, committed.

  Your choices - Enter to apply, or a number to change:
    1. Harness      Claude Code
    2. Provider     amazon-bedrock, us-east-1, default credential chain
    3. Preset       balanced
    4. Plugins      all installed
    5. MCP          on
    6. Preset in    this project, committed
  Apply? [Y/n]:
```

各手順が書き込む内容:

| 手順 | 書き込む内容 |
| --- | --- |
| 1. Harness | どのツールのファイルをセットアップするか（Claude Code の場合は `.claude/`）。 |
| 2. Model provider | `keep current` は何も書き込みません。`amazon-bedrock` は `CLAUDE_CODE_USE_BEDROCK` と `AWS_REGION`（およびプロファイルを指定した場合はそれも）を、あなた専用の `.claude/settings.local.json`（git は無視します）の `env` ブロックに追加し、共有の `.claude/settings.json` には決して書き込みません。プロジェクトに記録される回答にはプロファイルは含まれません。 |
| 3. Model effort preset | 手順 6 で選んだファイルにプリセット名を記録します。`unchanged` は何も記録しません。 |
| 4. Plugins | インストール済みのどのプラグインを有効にするか。 |
| 5. MCP servers | 付属サーバーを含む `.mcp.json`、またはなし。 |
| 6. Where to record the preset | `aidlc.settings.json`（コミットする）、`aidlc.settings.local.json`（git で無視される）、またはすべてのプロジェクトに適用されるこのマシンの設定。 |

`Apply?` では、手順の番号を入力するとその回答を変更でき、`n` を入力すると何も書き込まずに
終了します。ステータスラインの質問の後、Bedrock ではあなたにしかできない手順が
1 つ残ります:

```text
  One thing needs you - it can't be done automatically:

    Verify Amazon Bedrock model access in the recorded region and confirm the AWS principal has
    bedrock:InvokeModel permission.
    fix: aidlc config providers --check
```

### Without questions

スクリプト、CI、またはプロンプトのないターミナルでは、ウィザードは表示されません。すべてのハーネス
ランタイムがインストールされているため、使うハーネスを指定します。`--yes` は確認を行いますが、MCP
サーバーは有効にしません（有効にするには `--mcp defaults` を追加します）:

```text
$ aidlc config --harness claude --mcp defaults --yes
configured /home/you/ci-demo for Claude Code X.Y.Z; next: open Claude Code in this project (if it is already open in this folder, exit it and start it again) and run `/aidlc --doctor`
```

Kiro IDE では、`--yes` は、すべてのプロジェクトに対する Kiro の設定についてセットアップが
尋ねる質問にも推奨の回答を使います: Kiro IDE と Kiro CLI で Kiro の
Workflows 機能をオフにし（オンのままだと AI-DLC の
レビューとヘルパーが実行されません）、Windows で Command Prompt が Kiro のターミナルになっている場合は
PowerShell に設定します。Workflows を再びオンにするには、
`aidlc config trust --harness kiro --kiro-workflows on --yes`（Kiro IDE）または
`aidlc config trust --harness kiro --kiro-cli-workflows on --yes`（Kiro CLI）を実行します。

`--harness` を指定しないと、何も書き込む前に停止します:

```text
$ aidlc config < /dev/null
error: multiple harnesses are installed; pass --harness <claude|codex|copilot|cursor|kiro-v2|kiro|opencode>
fix: aidlc config --harness <name>
```

config が何を書き込むかを事前に確認するには、プレビューします。
ターミナルではハーネスと MCP の選択を尋ねます。それ以外の環境ではハーネスを指定します:

```text
$ aidlc config --harness claude --dry-run
config plan for /home/you/ci-demo: create=330 update=0 merge=0 preserve=1 remove=0 conflict=0
```

質問なしのセットアップではモデルプリセットが記録されないため、AI-DLC のエージェントは
付属のデフォルトを使います。すべての質問に対応するフラグがあります:
[構成オプション](18-install-and-lifecycle.md#config-options)と `aidlc config <section> --help` を参照してください。

### Set up before you install a harness

サポートされているツールが PATH 上にない場合、ウィザードはまず、どれをセットアップするかを
デフォルトなしで尋ねます:

```text
  No supported harness CLI was detected. Choose one to configure:

    1. Claude Code
    2. Codex CLI
    3. GitHub Copilot
    4. Cursor
    5. Kiro CLI (V2)
    6. Kiro            Kiro IDE and Kiro CLI on its v3 engine
    7. opencode
  Harness [1-7]:

  That's not one of the choices - enter 1, 2, ... or 7.
  Harness [1-7]: 1
  Using Claude Code.
```

残りは同じで、`detected` の代わりに `Claude Code selected` と表示されます。

## 3. What config left in your project

```text
$ git status --short
?? .claude/
?? .gitignore
?? .mcp.json
?? aidlc.settings.json
?? aidlc/
```

| パス | 内容 | コミットするか |
| --- | --- | --- |
| `.claude/` | Claude Code 用の AI-DLC のファイル: `/aidlc` スキル、エージェント、ステージ、フック、ツール、`settings.json`（フック、権限、ウェルカムメッセージ） | はい |
| `aidlc/` | AI-DLC の記録: `aidlc/spaces/default/memory/` にルール（フレームワークのデフォルトと、チームおよびプロジェクト独自のもの）があり、各作業の記録もここに作られます | はい |
| `.gitignore` | AI-DLC 独自のマーク付きブロック。既存のファイルがあればそれに追加されます | はい |
| `.mcp.json` | MCP サーバー（選択した場合） | はい |
| `aidlc.settings.json` | モデルプリセット | はい |
| `aidlc.settings.local.json`, `.claude/settings.local.json` | あなた自身の選択（ここではステータスライン） | いいえ、git で無視されます |

チームが同じセットアップを使えるよう、最初の 5 つをコミットします:
`git add .claude aidlc .gitignore .mcp.json aidlc.settings.json && git commit -m "Set up AI-DLC"`。
インストール済みのリリースからファイルを更新するには、いつでも `aidlc config` を再実行してください。
編集した内容は保持されます
（[プロジェクトの構成または更新](18-install-and-lifecycle.md#configure-or-refresh-a-project)）。

## 4. Check it

ここでは fix の行を 1 つ短縮しています:

```text
$ aidlc doctor
AI-DLC doctor

Machine
  warn  Runtime hook PATH: aidlc is on this shell's PATH (/home/you/.local/bin/aidlc) but not on the system-wide PATH
        fix: Nothing needs changing when you start Claude Code from a terminal: it hands that terminal's PATH to AI-DLC's hooks. If you start it from a desktop icon, the dock, or a service and AI-DLC's hooks do not run, add /home/you/.local/bin to the PATH line in /etc/environment, ENV_PATH in /etc/login.defs, or a PATH= line in ~/.config/environment.d/*.conf, then restart Claude Code. Editing .bashrc or .zshrc does not change this check.
  ok    9 checks passed

Project (.claude, Claude Code)
  warn  AIDLC hooks have not run in this project yet
        fix: This is expected before your first Claude Code chat in this folder. ...
  warn  Workspace records: 8 uncommitted change(s) under aidlc/ (advisory - the shared records travel by git; commit & push so teammates get them: `git add aidlc/ && git commit && git push`)
        fix: add --verbose to see the details, correct the named condition, then run doctor again
  ok    49 checks passed

Framework integrity
  ok    all 12 checks passed

0 problems, 3 warnings.
Warnings are advisory - if everything works, ignore them.
Add --verbose to see every check.
```

新しいプロジェクトでは、これら 3 つの警告は想定どおりです: 1 つ目はデスクトップのアイコンから
Claude Code を起動する場合にのみ関係し、2 つ目は最初のチャットの後に、
3 つ目はコミットの後に消えます。`fail` の行は実際の問題であり、
その修正方法が併記されます。Claude Code の中では、`/aidlc --doctor` で同じチェックを実行できます。

## 5. Your first run in Claude Code

プロジェクトで Claude Code を起動します:

```text
$ claude
```

新しいフォルダーで初めて起動すると、Claude Code はそのフォルダーを信頼するかどうかを尋ね、
AI-DLC の設定によって確認なしで実行できるコマンドを一覧表示します:

```text
 Accessing workspace:

 /home/you/inventory-api

 Quick safety check: Is this a project you created or one you trust? (Like your own code, a
 well-known open source project, or work from your team). If not, take a moment to review what's in
 this folder first.

 Claude Code'll be able to read, edit, and execute files here.

 ! This folder pre-approves 45 tool permissions in .claude/settings.json:
   WebSearch, Edit(/**), Bash(aidlc engine *), Bash(aidlc doctor), Bash(aidlc doctor --verbose),
 Bash(aidlc version), Bash(aidlc --doctor), Bash(aidlc --doctor --verbose), and 37 more
 These will apply without asking. Only proceed if you trust this configuration.

 > No, exit
   Yes, I trust this folder
```

自分のプロジェクトであれば、**Yes, I trust this folder** を選びます。MCP サーバーが有効な場合、
Claude Code は続いて `.mcp.json` にある 5 つのサーバーのうちどれを有効にするかを尋ねます。
認証情報を持っているものを有効にするか、Esc キーを押してすべてを省略してください。AI-DLC は
どちらでも動作します。

起動時に、Claude Code は AI-DLC のウェルカムメッセージを表示します。その AI-DLC の見出しの下には
"Run `/aidlc` to start or resume a workflow. Describe what to build and
AI-DLC proposes a plan for you to confirm."（ワークフローの開始または再開の方法と、作りたいものを説明すれば AI-DLC が確認用の計画を提案すること）と表示され、続いてフェーズ、ステージ、
スコープ（あらかじめ用意された計画）と、それぞれが実行するステージが一覧表示されます。
最初の起動では、Claude Code が代わりに独自の通知を表示し、
`1 more notice hidden` と表示することがあります。その場合、ウェルカムメッセージは次回以降の起動時に表示されます。

次に、作りたいものを説明します:

```text
> /aidlc Build a REST API for inventory management
```

エージェントはあなたの言葉を読み取り、計画、つまりこの種の作業のためにあらかじめ用意された
ステージのセットを提案します。ここでは `mvp` が選ばれました。あなたの言葉によっては別のものになる場合があります:

```text
 Plan
 I'd run the "mvp" plan for: "Build a REST API for inventory management" - 19 stages, 19 approval
 gates, 5 stages repeat per unit of work in Construction. Do you want me to go ahead with it, use
 a different plan, or tailor one to this task?
 > 1. Go ahead with the "mvp" plan
      19 stages: skips operations and ships the core API.
   2. Use a different plan
      Pick another ready-made plan (e.g. classic, feature, express, poc).
   3. Tailor a plan to this task
      Have a plan composed specifically for this work.
   4. Type something.
   5. Chat about this
```

Go ahead を選ぶと作業が始まります。**Use a different plan** を選ぶと、他の計画がその
長さとともに一覧表示され、どれかを選ぶか名前を入力します:

```text
 Plan
 Which plan would you like for the inventory REST API?
 > 1. classic (14 stages)
      V1-style ceremony through Inception and Construction - the implicit default.
   2. feature (29 stages)
      Full lifecycle for new features, practical depth.
   3. express (6 stages)
      Lightest run: requirements to deploy, no design pass, no reviewers.
   4. poc (4 stages)
      Prove feasibility fast.
   5. Type something.
   6. Chat about this
```

この実行ではそこで "mvp after all" と入力し、`mvp` のまま進めました。
[ワークフロープロファイル](workflow-profiles.md)で各計画を説明しています。**Tailor** を選ぶと、
エージェントがあなたの言葉から独自の計画を提案します。

どの質問にも **Type something**（自分で回答を入力する）と **Chat
about this** が用意されています。以下の画面では、ステージの最初の質問を除き、
この 2 つを省略しています。

続いて AI-DLC はこの作業の記録を作成し、最初のステージである
Intent Capture を開始します。最初の質問は、ステージの質問に
どのように回答したいかです:

```text
 Answer mode
 I've created 8 questions at
 aidlc/spaces/default/intents/261013-inventory-rest-api/ideation/intent-capture/intent-capture-questions.md.
 How would you like to answer them?
 > 1. Guide me
      Walk through each question interactively here
   2. I'll edit the file
      Fill in the answers in the file directly
   3. Chat
      Discuss freely; I'll extract the decisions from the conversation
```

選ぶのは一度だけです。以降のステージでは同じ回答が使われ、そのことが 1 行で示されます
（"Answering the way you chose earlier: Guide me. Say if you'd rather edit the
file or chat."、つまり以前に選んだ方法で回答すること、ファイルの編集やチャットを望む場合は伝えればよいこと）。**Guide me** を選ぶと、質問が数個ずつ表示され、それぞれに
選択肢と、本文の前に通常それに答える人が示されます（この
画面は AI-DLC がそのラベルを表示するようになる前に取得したもので、ラベルの位置を示すために
`[Product owner]` を書き加えています）。最上部の行に質問が一覧表示されます。最後の質問の後、
**Submit** で回答を確認用に表示し、**Submit answers** で送信します:

```text
 < [ ] Q1 Problem  [ ] Q2 Users  [ ] Q3 Success  [ ] Q4 Trigger  Submit >
 Q1. [Product owner] What business problem should the inventory API solve first? (If it's not
 defined yet, pick "Type something" and say so.)
 > 1. Stock counts drift
      A. We don't know accurately what stock we have (counts drift, spreadsheets go stale)
   2. Stock-outs / overstock
      B. Levels aren't visible in time to reorder
   3. Shared source of truth
      C. Other systems (shop, ordering, warehouse tools) need one reliable source of stock data
   4. Multi-location tracking
      D. Tracking stock across several locations is manual and error-prone
   5. Type something.
   6. Chat about this
```

矢印キーと Enter キーで選択肢を選ぶか、**Type something** を選んで
自分の言葉で回答します。"Select all that apply" と書かれた質問では
複数を選べます。すべての質問に回答すると、エージェントは何かを書き込む前に
回答を読み上げて一度確認を求めます:

```text
 Confirm
 - Problem: other systems need one shared, reliable source of stock data (Q1)
 - Callers: internal staff tools and an online store / ordering system (Q2)
 - Success: at least one real consumer system integrated and using the API in daily work (Q3)
 - Trigger: a new business or product line that needs inventory tracking from day one (Q4)
 - Stakeholders: just you; you decide scope and priority; no progress reporting needed (Q5-Q7)
 - Out of the first version: supplier purchasing / automatic reorder, and any user interface (API
 only) (Q8)

 Does this all look correct before I generate the artifact?
 > 1. Looks correct
      Generate the artifact from these answers
   2. Request changes
      Revise one or more answers before generation
```

続いて、ステージのドキュメントを書き込みます。それらが、回答で確定していない前提に
基づいている場合は、前提のまま残すか質問に変えるかを尋ねます。
レビュアーエージェントが主要なドキュメントを読みます。途中でエージェントの誤りを
訂正していた場合は、ここでその訂正を次回のために残すかどうかを尋ねます。
この実行では何も訂正しなかったため、次は承認です。

続いて最初の承認です（省略: ソースチェックの行、レビューファイルの
パス、3 つの指摘のうち 2 つを省いています）:

```text
Intent Capture & Framing is ready for your review. It produced intent-statement.md,
stakeholder-map.md and intent-capture-questions.md (8 questions, all answered), in
aidlc/spaces/default/intents/261013-inventory-rest-api/ideation/intent-capture/.

Stage: Intent Capture & Framing
Review outcome: Concerns remain for your decision.

| ID   | Severity | Where                                      | Status |
|------|----------|--------------------------------------------|--------|
| R-01 | Minor    | intent-statement.md > Initial Scope Signal | New    |
| R-02 | Minor    | intent-statement.md > Initiative Trigger   | New    |
| R-03 | Minor    | intent-statement.md > Success Metrics      | New    |

> R-03 Finding: The table carries a single, binary success metric ("at least one real consumer
  system integrated and uses the API in daily work") with no accuracy, timeliness, or
  adoption-breadth dimension, ...
> R-03 Required action: Human should confirm at the gate that a single integration-adoption
  metric is sufficient, or ask a follow-up to add a data-accuracy or reliability metric before
  inception begins.

Decision options:
- Approve - continue.
- Request Changes - return to the listed artifacts so the required actions can be addressed.

Review: aidlc/spaces/default/intents/261013-inventory-rest-api/ideation/intent-capture/

The two wording points (R-01, R-02) are things I added that you didn't confirm, and they're easy
to remove. R-03 is your call: one success measure is enough, or you may want to add one for stock
accuracy. After you approve, the next step is Feasibility & Constraints.

Approve accepts the open review findings R-01 (...), R-02 (...) and R-03 (The table carries a
single, binary success metric ("at least one real...) as risk; you can also approve and ask for
one to be fixed, or reject one as not applying, saying why.

 Approval
 Intent Capture & Framing complete. How would you like to proceed?
 > 1. Approve
      Continue to Feasibility & Constraints
   2. Request Changes
      Provide revision feedback
```

ドキュメントは示されたフォルダーにあります。判断する前に開いて確認してください。
**Approve** は先に進み、未解決の指摘を受け入れます。**Request Changes** は
ステージを差し戻して指摘に対処させます（何を変更するか、または
追加の質問を望むことを伝えます）。この実行では承認しました:

```text
Progress: 1 of 19 stages done. Next: Feasibility & Constraints.
Now working on Feasibility & Constraints, wearing the architect hat.
```

この作業の記録には、次のものが含まれるようになりました（主要なファイルのみに省略）:

```text
aidlc/spaces/default/intents/261013-inventory-rest-api/
  aidlc-state.md                 where the work stands
  audit/                         every question, answer and approval, as it happened
  project-description.json       your request, word for word
  ideation/intent-capture/       intent-statement.md, stakeholder-map.md,
                                 intent-capture-questions.md, reviews/review-01.md
  ideation/feasibility/          the next stage, under way
```

ここから先、各ステージは同じリズムで進みます: 質問、ドキュメント、レビュー、
そしてあなたの承認です。[最初のワークフロー](02-your-first-workflow.md)ではそのリズムの各部分を
説明しています。[実例集](16-worked-examples.md)では実行全体を
追っています。

## On the other tools

手順はどこでも同じですが、次の点が異なります:

| ハーネス | config の値 | 開き方 | 呼び出し |
| --- | --- | --- | --- |
| Claude Code | `claude` | `claude` | `/aidlc` |
| Kiro (Kiro IDE, Kiro CLI on v3) | `kiro` | プロジェクトを開き、チャットパネルのエージェントピッカーで **aidlc** を選ぶか、プロジェクトで `kiro-cli --v3` を実行します（フォルダーを信頼するかを Kiro が尋ね、その中身を把握している場合は **Trust, continue** を選びます） | `/aidlc` |
| Kiro CLI (V2) | `kiro-v2` | `kiro-cli --v2`（Kiro CLI 2.21.4 以降。それより古い場合は [Kiro CLI (V2) の前提条件](harnesses/kiro.md#kiro-cli-v2-prerequisites)を参照） | `/aidlc` |
| Codex CLI | `codex` | `codex` | `$aidlc` |
| Cursor | `cursor` | Cursor を開くか `agent` を実行します | `/aidlc` |
| opencode | `opencode` | `opencode` | `/aidlc` |
| GitHub Copilot CLI / VS Code | `copilot` | Copilot CLI または VS Code | `/aidlc` |

config の後、ツールごとに config が示す 1 つの手順を完了します:

| ハーネス | 一般的な操作 |
| --- | --- |
| Claude Code | このフォルダーで Claude Code がすでに開いている場合は、終了して再起動します |
| Kiro (Kiro IDE, Kiro CLI on v3) | 構成したプロジェクトを開き、チャットパネルのエージェントピッカーで **aidlc** を選ぶか、プロジェクトで `kiro-cli --v3` を起動します（フォルダーを信頼するかを Kiro が尋ね、その中身を把握している場合は **Trust, continue** を選びます） |
| Kiro CLI (V2) | `kiro-cli --v2` を起動します（Kiro CLI 2.21.4 以降。それより古い場合は [Kiro CLI (V2) の前提条件](harnesses/kiro.md#kiro-cli-v2-prerequisites)を参照）。プロジェクトが AI-DLC のエージェントを選択します |
| Codex CLI | フックの信頼確認を承認するか、生成された信頼シードを適用します |
| Cursor | 構成したプロジェクトを開くか、`agent` を実行します |
| opencode | プロジェクトで `opencode` を起動します |
| GitHub Copilot | プロジェクトのフォルダーを信頼します |

ウェルカムメッセージは Claude Code だけのものです。AI-DLC は Claude Code では独自のステータスラインを
描画します。Codex では、セットアップ時に Codex の組み込みステータスラインを提案し、
AI-DLC が最新に保つタスクの進捗を表示します。Kiro CLI 2.24.1 以降で Kiro の v3 エンジンを使う場合は、
`kiro` を使い、`kiro-cli --v3` を起動します（フォルダーを信頼するかを尋ねられ、
その中身を把握している場合は **Trust, continue** を選びます）。`kiro-v2` は Kiro CLI (V2) です
（既存の Kiro CLI (V2) プロジェクトを切り替える方法は [Kiro CLI (V2)](harnesses/kiro.md#kiro-cli-v2) を
参照してください）。

Kiro IDE では、**aidlc** エージェントは、フォルダーを信頼してウィンドウを再読み込みした後でなければ
エージェントピッカーに表示されません: Restricted Mode のバナーが表示されている場合は、
バナーの **Manage**、続いて **Trust** を選び、**Developer: Reload Window** を実行します
（[初回の実行](harnesses/kiro.md#first-run)を参照）。まだ信頼していないフォルダーで Kiro IDE 自身の
ターミナルを使うと、Kiro はまずそのフォルダーを信頼するかを尋ねます。**Trust Folder & Continue** を選ぶのは、自分のプロジェクトか
確認済みのプロジェクトの場合だけにしてください。信頼すると、そのフォルダーの `.kiro` のフックが
あなたのマシン上でコマンドを実行できるようになるためです。

残りの内容は、[他のハーネスでの実行](harnesses/README.md)にある各ツールの章で
説明しています。

## AWS Bedrock Setup

Claude Code、Codex、opencode のディストリビューションは、ユーザーがすでに構成している
プロバイダーを維持します。Amazon Bedrock は明示的に選ぶオプションです。

### Provider-neutral default

AI-DLC は、付属の Claude Code、Codex、opencode のプロジェクト構成でモデルプロバイダーを
選択しません。ハーネスは、ユーザーがすでに構成しているプロバイダー、
認証、モデル、コンテキストの設定を維持します。
balanced のレビュアーエージェントは推論の effort に上限を設けることがありますが、
プロバイダー固有のモデルには固定しません。

### Configure Bedrock (optional)

上記のカスタマイズしたセットアップの手順 2 で `amazon-bedrock` を選ぶか、後で
`aidlc config providers` で選びます。その後:

1. Amazon Bedrock のモデルカタログで、構成した Anthropic のモデルへのアクセスを
   有効にします。
2. 通常の SDK の認証情報チェーンを通じて AWS の認証情報を用意します。たとえば
   `aws configure` や `aws sso login --profile <profile>` を使います。
3. それらのモデルが利用可能なリージョンを使います。
4. `aidlc config providers --check` を実行して、まだ保留中の項目を確認します。

認証情報や個人的な上書き設定は、共有の
`.claude/settings.json` に入れないでください。`.claude/settings.local.json` または
標準の AWS 認証情報ファイルに入れます。

Claude Code ですでに選択しているプロバイダーを維持するには、`aidlc config providers` で
`keep current` を選びます（または `--provider current` を渡します）。Claude Code がサポートする別のプロバイダーを
明示的に記録するには、`aidlc config providers
--provider other --yes` を実行します。これにより、AI-DLC が所有していた古い Bedrock の上書き設定が
共有のプロジェクト設定から削除され、手動のセットアップ手順が保留のまま残ります。
そのプロバイダーの認証フローを完了してから、`aidlc config providers
--acknowledge --yes` を実行してその手順を完了済みにします。
[Claude Code の認証ガイド](https://code.claude.com/docs/en/authentication)を参照してください。

IAM の詳細、モデルへのアクセス、SSO、リージョンに関するトラブルシューティングについては、
[Claude Code on Amazon Bedrock](https://community.aws/content/2tXkZKrZzlrlu0KfH8gST5Dkppq/claude-code-on-amazon-bedrock-quick-setup-guide)
と [Amazon Bedrock のドキュメント](https://docs.aws.amazon.com/bedrock/)を参照してください。

## MCP Servers (optional)

推奨のデフォルトでは、Claude Code のプロジェクトで付属の MCP サーバーが有効になります。
質問なしのセットアップでは `--mcp defaults` でそれらを追加します:

```bash
aidlc config --harness claude --mcp defaults
```

省略するには `--mcp none` を使います。コピーチャネルでのインストールは MCP サーバーなしで始まります。追加するには
`bun .claude/tools/aidlc.ts config project --harness claude --mcp defaults --yes` を実行します。
デフォルトのセットは次のとおりです:

| サーバー | 提供するもの | 認証情報 |
| --- | --- | --- |
| `context7` | ライブラリと SDK のドキュメント | `CONTEXT7_API_KEY` |
| `aws-mcp` | AWS API へのアクセス | AWS の認証情報チェーン |
| `aws-pricing` | AWS の料金の照会 | AWS の認証情報チェーン |
| `aws-iac` | Infrastructure as Code のツール | AWS の認証情報チェーン |
| `aws-serverless` | サーバーレス開発のツール | AWS の認証情報チェーン |

4 つの AWS サーバーには `uvx` が必要で、標準の AWS 認証情報チェーンを使います。

Claude のセッション内のすべてのエージェントは、利用可能な MCP サーバーを継承します。認証情報が
ないとそのサーバーは利用できなくなりますが、ワークフローはブロックされません。コミットする `.mcp.json` に
シークレットを決して入れないでください。

## Updating

`aidlc update` はマシンのランタイムを更新します。構成済みのプロジェクトは
書き換えません。各プロジェクトを更新してください。進行中の作業はそのまま続けられます:

```bash
aidlc update
cd /path/to/your-project
aidlc doctor
aidlc config
```

config はプロジェクトが所有する内容を保持し、ワークフローが進行中でも更新は
実行され、その作業を続けられるかどうかが表示されます。プラグインを使うプロジェクトでは、
エンジンの更新後に `/aidlc plugin sync` を実行してください。

バージョンの選択、プロジェクトでのバージョン固定、オフラインインストール、ミラー、カスタム CA、
リリースの認証、自動化、アンインストールについては、
[インストールとライフサイクル](18-install-and-lifecycle.md)を参照してください。

## Troubleshooting

まずは次を実行します:

```bash
aidlc doctor
```

その後、フック、プロバイダーへのアクセス、承認ゲート、古い状態、診断については
[トラブルシューティング](15-troubleshooting.md)を参照してください。ハーネス固有のセットアップの問題は、
対応する[ハーネスガイド](harnesses/README.md)で扱います。

## Next Steps

- [オンボーディング: ガイド付きの最初の 1 週間](onboarding.md) - 初めてのチーム向けのメンタルモデルと、ガイド付きの 5 回の実行
- [ワークフロープロファイル](workflow-profiles.md) - 適切なワークフローを選ぶ
- [最初のワークフロー](02-your-first-workflow.md) - 実行の各部分の意味
- [スペースとインテント](03-spaces-and-intents.md) - プロジェクトの状態を理解する
- [インタラクションモード](07-interaction-modes.md) - 質問とゲートの扱い方
- [インストールとライフサイクル](18-install-and-lifecycle.md) - ネイティブランタイムを管理する
