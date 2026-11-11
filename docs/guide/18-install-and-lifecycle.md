# Install and Lifecycle

ネイティブのリリースチャネルは、`aidlc` コマンドと 1 つ以上のハーネスランタイムをインストールします。続いて `aidlc config` が、そのローカルランタイムからプロジェクトを作成または更新します。インストールされたコマンドと config の処理に Bun や Node.js は必要ありません。GitHub CLI（`gh`）は任意です。互換性のあるバージョンがあれば署名付きアテステーションの検証が追加されますが、存在しない場合や古いバージョンの場合でもインストールは妨げられません。

この章では、本リリースで利用できるネイティブのインストールライフサイクルを説明します。計画中の `aidlc setup` による体験、npm パッケージ、パッケージマネージャー用のフォーミュラはまだ提供されていません。手動コピーで利用する場合は Bun をインストールし、`aidlc-copy-runtime-X.Y.Z.tar.gz` からバージョン付きの Bun 起動型ランタイムを取り出します。ネイティブの `aidlc` コマンドは必要ありません。

## Install

リリースアセットの対象は次のとおりです。

- macOS x64 および arm64
- Linux x64 および arm64（glibc ビルドと musl ビルド）
- Windows x64

ネイティブインストールはユーザー単位です。Unix インストーラーは root を拒否し、`sudo` を必要としません。`USER` のないコンテナイメージ、CI のコンテナジョブ、セットアップスクリプトを root で実行するクラウドのエージェントサンドボックスなど、root が唯一のユーザーである環境では、`--allow-root` を渡す（または `AIDLC_ALLOW_ROOT=1` を設定する）と root 自身の `$HOME` 配下にインストールできます。このオプトインでも、`sudo` 配下（`SUDO_USER` が設定されている）の場合、`$HOME` が未設定か root の所有でない場合（シンボリックリンクのホームはたどります）、`AIDLC_BIN_DIR` または `AIDLC_INSTALL_ROOT` が root のホームの外を指している場合は拒否されます。そのため、通常のアカウントからの `sudo sh install.sh` は引き続き失敗し、root でのインストールがシステム全体に及ぶことはありません。`--allow-root` より前のリリースは root として自分自身をアンインストールできません。そのリリースをインストールする場合や、`aidlc use`、`update`、`rollback` で root として有効化する場合は警告付きで続行し、その警告で後から削除する方法（そのリリース自身の `aidlc update` を実行してから `aidlc uninstall --allow-root`）を示します。`aidlc uninstall` も同じ制限付きで同じ `--allow-root` オプトインを受け付けます。
Windows のインストールは PowerShell を実行しているアカウントを対象とします。通常の PowerShell ウィンドウから実行してください。UAC 配下で「管理者として実行」で開いたウィンドウでは、管理者としてインストールすると安全性が下がる（同じアカウントで動く別のプログラムが、昇格したインストーラーが実行するファイルに干渉できる）という警告が表示され、確認を求められます。`-Yes` を付けるとプロンプトなしで確認したことになります。`-Yes` のない非対話実行（`-Json` や `-Quiet` を含む）は、その案内を表示して停止します。`aidlc uninstall` も同じ警告を出しますが、何も尋ねません。ターミナルでは何かを削除する前に警告を表示し、`--yes` を付けた場合は結果に警告が含まれます。Windows Server の組み込み Administrator のように、UAC の昇格なしで完全な管理者トークンを持つセッションでは警告は表示されません。別のアカウントの資格情報で PowerShell を実行すると、そのアカウント向けにインストールされます。全ユーザー向けのモードはありません。

Alpine Linux 用の musl アセットは Bun 自身のランタイム要件に従います。Bun の musl ビルドは Node.js と同様に、システムの `libgcc` および `libstdc++` パッケージを必要とします。完全に静的な Bun の musl コンパイルターゲットは、現時点では利用可能なターゲットではなく、上流で追跡中の機能です。インストーラーまたはバイナリを実行する前に、前提パッケージをインストールしてください。

```sh
apk add libgcc libstdc++
```

この前提条件は x64 と arm64 の両方の Alpine システムに当てはまります。システムパッケージのインストールには管理者権限が必要な場合がありますが、AI-DLC のインストール自体は引き続き対象ユーザーとして実行します。インストーラーは対応するローダーの失敗を検出して上記のコマンドを表示しますが、`apk` を実行したりシステムパッケージをインストールしたりすることはありません。上流の Bun での追跡には `oven-sh/bun#15829` と `oven-sh/bun#29681` があります。

インストーラーには、`--harness` の `claude`、`codex`、`copilot`、`cursor`、`kiro`、`kiro-v2`、`opencode` 用のすべてのハーネスランタイムがまとめて含まれています。

```bash
tmp="$(mktemp -d)"
curl -fsSL \
  https://github.com/awslabs/aidlc-workflows/releases/latest/download/install.sh \
  -o "$tmp/install.sh"
sh "$tmp/install.sh"
rm -rf "$tmp"
```

### macOS and Linux

```bash
tmp="$(mktemp -d)"
curl -fsSL \
  https://github.com/awslabs/aidlc-workflows/releases/latest/download/install.sh \
  -o "$tmp/install.sh"
sh "$tmp/install.sh"
rm -rf "$tmp"
export PATH="$HOME/.local/bin:$PATH"
```

オンラインで実行するには `curl` または `wget` が必要で、どの実行でも `sha256sum` または `shasum` が必要です。GitHub CLI は任意で、リリースに必要なアテステーション用フラグをサポートしている場合にのみ使われます。
デフォルトでは、バージョンを `${XDG_DATA_HOME:-$HOME/.local/share}/aidlc/versions/` 配下にインストールし、有効なバージョンを実行する小さなランチャーである `$HOME/.local/bin/aidlc` を書き込みます。AI-DLC 自身のプラグインカタログはバージョンと並んで `${XDG_DATA_HOME:-$HOME/.local/share}/aidlc/plugins/` に置かれ、インストーラーがそのフルパスを表示します（[プラグイン](#plugins)を参照）。

インストーラーは、`--profile <absolute-path-under-$HOME>` が明示されない限りシェルの起動ファイルを編集しません。このオプションを指定すると、`BEGIN AI-DLC:PATH` ブロックを 1 つトランザクションとして書き込みまたは更新し、ファイルの残りの部分は保持します。プロファイルを AI-DLC のインストールルートやコマンドルートの中に置くことはできません。既存のマーカーは一意で、行全体と完全に一致し、begin が end より前に来る順序である必要があります。マーカーの配置が不正な場合は、プロファイルを変更せずに拒否します。

### Windows PowerShell

```powershell
$download = Join-Path $env:TEMP "aidlc-install-$PID"
New-Item -ItemType Directory -Force $download | Out-Null
$installer = Join-Path $download install.ps1
Invoke-WebRequest `
  -Uri https://github.com/awslabs/aidlc-workflows/releases/latest/download/install.ps1 `
  -OutFile $installer
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
& $installer
Remove-Item -Recurse -Force $download
```

`install.ps1` は署名されておらず、Windows PowerShell は実行ポリシーが許可する場合にのみスクリプトファイルを実行します。Windows 10 および 11 の既定値は `Restricted` で、どのスクリプトファイルも実行しません。`Set-ExecutionPolicy` の行は、この PowerShell ウィンドウと、そこから起動した PowerShell で、ウィンドウを閉じるまでスクリプトの実行を許可します。保存されている実行ポリシーは変更されません。グループポリシーによる実行ポリシー（`Get-ExecutionPolicy -List` の `MachinePolicy` または `UserPolicy` の行。両方が設定されている場合は `MachinePolicy` が有効）は、この行より優先されます。そのポリシーが `Bypass` でない限り、この行はエラーを表示します。Windows PowerShell ではそのエラー名は `ExecutionPolicyOverride` で、英語ではポリシーが `updated … successfully, but the setting is overridden`（正常に更新されたが設定が上書きされている）と表示されます。`Bypass`、`RemoteSigned`、`Unrestricted` の下ではインストーラーはそれでも実行されます。`AllSigned` の下ではデジタル署名がないとしてインストーラーを拒否し、`Restricted` の下ではスクリプトの実行が無効になっているとして拒否します。インストール方法を管理者に問い合わせてください。

Windows ではバージョンを `%LOCALAPPDATA%\aidlc\versions\` 配下にインストールし、`aidlc` コマンドを `%LOCALAPPDATA%\aidlc\bin\` に置きます。そこには、有効なバージョンを起動する AI-DLC バイナリのコピーである `aidlc.exe` と、その隣の `aidlc.cmd` シムがあります。PowerShell、CMD、Git Bash はいずれも、素の `aidlc` に対して `aidlc.exe` を実行します。これはシェルから渡された引数をそのまま有効なバージョンに渡すため、間に cmd.exe が入って引数内の `&`、`^`、`|`、`<`、`>`、`%NAME%` を解釈することはありません。AI-DLC 自身のプラグインカタログは `%LOCALAPPDATA%\aidlc\plugins\` にあります。検証とインストールが成功すると、インストーラーはその bin ディレクトリを現在のアカウントの永続的なユーザー PATH に登録します。既存のエントリは保持し、再実行しても重複させません。また、現在の PowerShell プロセスも更新し、環境の変更を Windows に通知するので、その後に起動したプログラムは `aidlc` を見つけられます。すでに開いていたターミナルアプリや IDE（Kiro、VS Code、Cursor）は起動時の PATH を保持し続けます。そこで `aidlc` や AI-DLC のフックを使う前に、完全に終了してから開き直してください。その中で新しいタブを開いたり Reload Window を実行したりするだけでは不十分です。
マシン PATH の変更や PowerShell プロファイルの編集は行いません。

成功時の人間向け出力は、`Installed`、バージョン、アカウント、インストールされたコマンドのパスで始まります。次に、プロジェクトのディレクトリで `aidlc config` を実行してください。インストーラーが PATH エントリを追加した場合は、すでに開いていたターミナルアプリや IDE を終了して開き直すようにも案内します。

インストーラーは、ユーザー PATH エントリを追加した場合にのみ、インストールルート配下に `windows-path.json` を書き込みます。この所有記録は `-NoModifyPath` を含む再実行後も残るため、後でアンインストール時にエントリを削除できます。すでに存在していたエントリは所有を主張しません。`aidlc uninstall` と `aidlc uninstall --purge` はどちらも、記録されたエントリを削除し、以前から存在したエントリや後から行われた無関係な PATH の変更は保持します。

永続的な PATH で別の `aidlc` コマンドが優先される場合、結果にそのコマンドを示し、インストールしたコマンドのフルパスを表示します。その PATH の競合を解消するか、インストールしたコマンドを直接呼び出してください。PATH の登録に失敗した場合、インストーラーはファイルはインストールされたが PATH の設定がまだ必要であることを、復旧手順と終了コード 1 とともに報告します。

**永続的な PATH と現在のプロセスの PATH の両方の変更**をスキップするには、上記の `& $installer` を `& $installer -NoModifyPath` に置き換えます。インストーラーは、PATH 登録なしで実行するための直接のコマンドを表示します。既定の場所の場合は次のとおりです。

```powershell
& "$env:LOCALAPPDATA\aidlc\bin\aidlc.exe" config
```

インストール先を変更した場合は、表示されたコマンドパスを使ってください。PATH の自動登録を有効にするには、`-NoModifyPath` なしでインストーラーを再実行します。このスイッチは、以前の登録を元に戻したり、その所有記録を消したりはしません。

PowerShell インストーラーのパラメーターは、`-Version`、`-From`、`-Offline`、`-ReleaseBaseUrl`、`-CaBundle`、`-NoModifyPath`、`-Yes`、`-Quiet`、`-Json`、`-NoColor` のように、ネイティブの名前を使います。

バージョン付きのリリース URL からダウンロードしたインストーラーは、プレビューを含め、既定でそのリリースそのものを選びます。`latest/download` のインストーラーは引き続き最新の安定版リリースを選びます。明示的な `--version` / `-Version` の指定はパッケージに組み込まれた既定値より優先され、`--from` / `-From` はローカルのリリースマニフェストからバージョンを読み取ります。

### Automation

インストール時にハーネスについて質問することはありません。人間による実行でも非対話実行でも、同じバイナリとすべてのハーネスランタイムをインストールします。

一時的または隔離された Windows インストールでは、`-NoModifyPath` を渡し、報告されたコマンドパスを直接呼び出してください。そうすれば、一時的な bin ディレクトリがユーザー PATH や現在のプロセスに入りません。

PowerShell の `-Json` は、`schemaVersion: 1`、`ok`、`code`、`status`、`message` を含む結果を 1 つ出力します。ファイルのインストール後、`data` には次の内容が含まれます。

| フィールド | 意味 |
|-------|---------|
| `installed` | `true`。後続の PATH ステップが失敗した場合も含む |
| `ready` | 推奨コマンドがすぐに使えるかどうか。PATH の競合や失敗の場合は `false` |
| `version`, `account`, `installRoot`, `command` | インストールしたバージョン、Windows アカウント、インストールルート、コマンドのフルパス |
| `path.scope` | `"user"` |
| `path.status` | `"updated"`、`"unchanged"`、`"skipped"`、`"conflict"`、または `"failed"` |
| `path.changed` | この実行で永続的なユーザー PATH を変更したかどうか |
| `path.owned` | この実行でインストーラーの所有を確認したかどうか。`-NoModifyPath` により以前の記録が評価されないままの場合は `null` |
| `nextSteps` | プロジェクトの設定や PATH の復旧を含む手順の配列 |

登録の成功、または一致する PATH がすでに存在する場合は、通常 `status: "ok"` と終了コード 0 になります。Windows が他のアプリケーションに通知できない場合、結果は `status: "warning"` と `data.ready: true` になり、条件付きでサインアウトの手順が示されます。永続的なコマンドの競合は `status: "warning"`、終了コード 0、`data.ready: false` になります。自動化では終了コードだけでなく準備状態も確認してください。`-NoModifyPath` の場合は `status: "ok"`、`path.status: "skipped"`、`data.ready: true` となり、`nextSteps` に直接のコマンドが入ります。インストール後に PATH の処理が失敗した場合は `status: "failed"`、終了コード 1、`data.installed: true`、`data.ready: false` になります。

### Installer Options

| Unix | PowerShell | 意味 |
|------|------------|---------|
| `--version <version>` | `-Version <version>` | 最新版ではなく特定のリリースをインストールする。安定版の `x.y.z` またはプレビューの `x.y.z-preview.YYYYMMDD.N` の ID |
| `--from <dir>` | `-From <dir>` | フラットなリリースセットをローカルから読み込み、オフラインモードを暗黙的に有効にする |
| `--offline` | `-Offline` | ネットワークアクセスを禁止する。`--from` / `-From` が必要 |
| `--release-base-url <url>` | `-ReleaseBaseUrl <url>` | 互換性のあるリリースミラーを使う |
| `--ca-bundle <absolute-path>` | `-CaBundle <absolute-path>` | カスタムの CA バンドルを使う |
| `--profile <absolute-path>` | 利用不可 | Unix の PATH ブロックをトランザクションとして追加する |
| 利用不可 | `-NoModifyPath` | 永続的なユーザー PATH と現在のプロセスの PATH の変更をスキップし、直接のコマンドを表示する |
| `--allow-root` (or `AIDLC_ALLOW_ROOT=1`) | 利用不可 | root が唯一のユーザーである環境（コンテナ、CI ジョブ、エージェントサンドボックス）で root としてインストールする。`sudo` 配下、`$HOME` が未設定か root の所有でない場合、`AIDLC_BIN_DIR` または `AIDLC_INSTALL_ROOT` が root のホームの外にある場合は引き続き拒否する。`--allow-root` より前のリリースは root として自分自身をアンインストールできないため、警告付きでインストールする |
| `--yes` | `-Yes` | 自動化モード。整合性チェックは回避しない。Windows では、bin ディレクトリにある AI-DLC が書き込んでいない `aidlc` も置き換え、そのファイルをバックアップとして残す |
| `--quiet` | `-Quiet` | 進捗表示を抑え、結果を 1 行で出力する |
| `--json` | `-Json` | 進捗表示を抑え、スキーマバージョン付きの JSON 結果を 1 つ出力する |
| `--no-color` | `-NoColor` | カラー出力を無効にする |
| `--help` | 非公開 | Unix インストーラーの使い方を表示する |

`AIDLC_RELEASE_BASE_URL` と `AIDLC_CA_BUNDLE` はインストーラーの既定値を与えます。明示的なオプションが優先されます。`AIDLC_RELEASE_REPOSITORY` は、既定のダウンロードに使う GitHub リポジトリと、来歴（provenance）検証で信頼するリポジトリの両方を選びます。既定値は `awslabs/aidlc-workflows` です。
`AIDLC_RELEASE_WORKFLOW` は信頼する署名者ワークフローを上書きします。既定では、インストーラーは安定版に `<AIDLC_RELEASE_REPOSITORY>/.github/workflows/release.yml` を、プレビュー版に `<AIDLC_RELEASE_REPOSITORY>/.github/workflows/preview-release.yml` を選びます。ワークフローのパスが異なるフォークやミラーでは、そのリリースベース URL とあわせて上書きを明示的に設定してください。ダウンロード URL だけを変えても来歴の信頼の起点は変わりません。`AIDLC_GH_BIN` は、両方のインストーラーで使う GitHub CLI の実行ファイルを明示的に選びます。その実行ファイルが存在しないか、`--signer-workflow`、`--source-ref`、`--source-digest` に対応していない場合、来歴の検証はスキップされますが、チェックサムの検証は引き続き必須です。

フォークでのリリースに GitHub App や追加のリポジトリは必要ありません。タグのワークフローは、有効期間の短い `GITHUB_TOKEN` を使って同じリポジトリに公開します。最後のジョブは `release` 環境を使うため、公開前にレビュアーの承認を必須にできます。

`AIDLC_INSTALL_ROOT` と `AIDLC_BIN_DIR` は、マシンとコマンドの配置場所を上書きします。Unix ではこれらのパスは絶対パスである必要があります。Windows では、`-NoModifyPath` が設定されていない限り、選択した bin ディレクトリが現在のアカウントのユーザー PATH に登録されます。PowerShell インストーラーは `AIDLC_OFFLINE=1` にも従います。Unix インストーラーでは明示的に `--offline` または `--from` と書く必要があります。

### Release Authentication

インストーラーは次の処理を行います。

1. `version.json`、`checksums.txt`、`aidlc-release.intoto.jsonl` をダウンロードまたは読み込む。
2. 互換性のある GitHub CLI が利用できる場合、`github.com` 上のリポジトリと署名者ワークフローに対して `checksums.txt` のアテステーションを検証する。GitHub CLI の既定ホストが GitHub Enterprise のホストであっても同様。
3. `version.json` の SHA-256 を検証し、そのバージョン ID とソースの識別情報を読み取り、明示的に指定したバージョンと一致しない場合はリリースバイナリをダウンロードまたは実行する前に拒否する。
4. `sourceRef` が、安定版では `refs/tags/v<version>`、プレビューでは `refs/heads/main` と等しいことを要求し、来歴の検証が利用できる場合は、その ref と認証済みの `sourceDigest` に対してアテステーションを再検証する。
5. 選択したバイナリとハーネスのアーカイブを、SHA-256 と宣言されたバイト長で検証する。
6. 検証済みのバイナリにリリースを検証させ、トランザクションとしてインストールさせる。

実行前にブートストラップスクリプト自体を認証するには、最新の GitHub CLI を使ってください。`gh` の既定が GitHub Enterprise のホストになっている場合でも、`github.com/` というリポジトリの接頭辞と `--hostname github.com` によって、これらのコマンドは github.com に対して実行されます。

```bash
tmp="$(mktemp -d)"
tag="$(gh release view --repo github.com/awslabs/aidlc-workflows --json tagName --jq .tagName)"
gh release download "$tag" --repo github.com/awslabs/aidlc-workflows --dir "$tmp" \
  --pattern install.sh --pattern aidlc-release.intoto.jsonl
gh attestation verify "$tmp/install.sh" \
  --bundle "$tmp/aidlc-release.intoto.jsonl" \
  --repo awslabs/aidlc-workflows \
  --signer-workflow awslabs/aidlc-workflows/.github/workflows/release.yml \
  --hostname github.com \
  --source-ref "refs/tags/$tag"
sh "$tmp/install.sh" --version "${tag#v}"
rm -rf "$tmp"
```

メタデータは 1 MiB、個々のリリースアセットは 1 GiB に制限されています。アセット名にパスを含めることはできません。アーカイブの展開では、リンク、特殊ファイル、パストラバーサル、絶対パス、重複エントリ、過大な展開を拒否します。

リリースワークフローは候補を一度だけ組み立てます。ステージングと Unix/Windows のライフサイクルジョブは、署名の権限なしで `checksums.txt` を検証し、そのバイト列をテストします。実際のアテステーションはそれらのテストの後に作成されるため、これらのジョブはジョブローカルの検証用フィクスチャを追加します。このフィクスチャがアップロードされることはありません。
`publish` は候補を再検証してアテストし、`aidlc-release.intoto.jsonl` をエクスポートし、インベントリ全体を検証して、ワークフローの成果物を 1 つアップロードします。`release` はタグとチェックサムを再確認し、`GITHUB_TOKEN` を使ってこのリポジトリに GitHub Release を作成し、ローカルとリモートのアセットインベントリを比較します。バンドルは `version.json` と `checksums.txt` の外にあります。これらのファイルはインストール可能な成果物を対象とし、バンドルはそれ自体が Sigstore の信頼チャネルです。
オンラインの転送では TLS を強制し、すべてのインストールで SHA-256 を強制します。互換性のある GitHub CLI のバージョンでは、署名付きの来歴検証が追加されます。OS のコード署名と公証はリリースに含まれません。[サプライチェーンセキュリティ](../reference/19-supply-chain-security.md)を参照してください。

インストーラーは、所有者が混在する既存のコマンドを拒否します。また、既存の Homebrew や Nix のコマンドは置き換えずに譲ります。このプロジェクトはまだそれらのパッケージマネージャーのチャネルを提供していません。所有しているマネージャーを使うか、空の `AIDLC_BIN_DIR` を明示的に選んでください。

## Configure or Refresh a Project

ハーネスを開く前に config を実行してください。[はじめに](01-getting-started.md#2-set-up-your-project)で、初回セットアップの各画面を紹介しています。

```bash
cd your-project
aidlc config --harness claude --dry-run
aidlc config
aidlc doctor
```

`aidlc config` はトランザクションとして動作し、このマシンにすでにあるリリースを使って処理します。プロジェクトが必要とするリリースをダウンロードするのは、プロンプトで同意した場合か `--download` を渡した場合だけです。選択したハーネスのツリー、`aidlc/` ワークスペースの外枠、ルートの統合、プロジェクション（projection）のスタンプ、所有権のベースラインを作成します。ワークフローのインテントは作成しません。

すべてのハーネスランタイムがインストールされているため、ターミナル外で初回セットアップやプレビューを行う場合は `--harness <name>` でハーネスを指定します。ターミナルでは、config は検出した 1 つのハーネスを使うか、どれを使うかを尋ねます。
プロジェクトに複数のハーネスがある場合、そのうちの 1 つをセットアップ、プレビュー、更新するコマンドでは `--harness <name>` でハーネスを指定します。唯一の例外は、他に変更するものがない状態で `aidlc config flags --bypass` または `--clear-bypass` によってバイパスを記録またはクリアする場合です。バイパスは 1 つのハーネスではなくプロジェクトに属するためです。どのハーネスが共存できるか、また同梱の `.gitignore` エントリがどのように統合されるかについては、[ルートの統合と所有権](#root-integrations-and-ownership)を参照してください。

あるリリースから 1 つのハーネスを書き込んだ実行では、続けて、別のリリースのままになっているインストール済みの他のハーネスをすべて示し、作業が開いているかどうかにかかわらず、書き込んだばかりのリリースにそろえるためのコマンドを 1 つ示します。

```
  Kiro CLI (.kiro) is still on 2.9.0. To bring it to 2.10.0: `aidlc config --harness kiro-v2`.
```

`--from` に渡したファイルにそのハーネスが含まれている場合、コマンドはそれを再利用します。それ以外の場合、コピーしたプロジェクトでは、まずそのリリースのコピーランタイムファイルを入手するよう案内します。すでにインストールされているコピーしたハーネスに `--download` を使っても、すでに持っているリリースを取得するだけだからです。ネイティブのプロジェクトでは、そのリリースをインストールする `aidlc config --pin` を示します。
マシンに登録されているチェックアウトの git worktree は同じプロジェクトとして扱われます。その worktree のピン留めされたコマンドは、マシンにそのリリースがあれば worktree 自身の `.aidlc-version` が示すリリースを実行し、それを登録します。そのため、ホストがセッションごとに作る worktree は、ブランチがチェックアウトと異なるリリースにピン留めしていても、再び `config --pin` を求めることはありません。メインのチェックアウト自体や別のクローンでは、これまでどおり `config --pin` によってのみ変更されたピンを登録します。

スキャフォールドまたは更新が成功した後、config は低コストのインストール結果チェックを実行します。確認するのは、非対話のフック用 PATH、ホストの信頼ファイル、記録済みのプロバイダーアクションだけで、ハーネスの CLI を起動したりプロバイダーに接続したりはしません。トランザクションは引き続き終了コード 0 で終わります。非 TTY の人間向け出力では、未対応の項目すべてと、それに対応する `aidlc config runtime`、`aidlc config trust`、`aidlc config providers --check` のフォローアップを正確に示します。2 つの項目には対応する AI-DLC コマンドがないため、その行では利用者自身が行う手順を示します。Codex 自身のフックの信頼（Codex で `/hooks` と入力し、`t` を押してすべてを信頼し、Esc を押す）と、Kiro IDE でエラーのある設定ファイル（示されたファイルのエラーを修正する。その `command` である `config trust --check` は結果を確認するだけ）です。JSON には `data.outstandingActions` が含まれ、このような項目にはその `step` も含まれます。quiet 出力は、問題がなければ 1 行のままで、フォローアップが必要な場合は未対応アクションの行を 1 行追加し、さらにコミット済みの記録を隠している ignore ルールごとに `Warning:` の行を 1 行追加します。

人間が使う TTY では、引数なしの初回実行（フラグなしの `aidlc config`。`--harness`、`--dry-run`、`--yes`、`--json` などのフラグを付けるとウィザードはスキップされる）は、質問ではなく検出から始まります。検出対象は、`PATH` 上にインストールされたハーネスの CLI、プロジェクトの状態、ローカルの AWS 資格情報とリージョン、非対話のフックランタイムです。ハーネスが 1 つ検出された場合、ウィザードはその名前を示し、推奨の既定値、6 ステップのカスタマイズ、何も書き込まずに終了、の 3 つの選択肢を提示します。複数のハーネスが検出された場合は、まず番号付きのハーネス選択が表示されます。ハーネスが検出されなかった場合は、既定値なしで完全な選択肢が表示されます。2.24.1 以降の Kiro CLI は、両方の Kiro ハーネスとして数えられます。選択肢には Kiro CLI (V2)（`--harness kiro-v2`）が表示されます。これはエンジンをピン留めせず、Kiro CLI の v2 エンジン向けに作られています。また Kiro（`--harness kiro`）も表示され、こちらは Kiro IDE と v3 エンジン上の Kiro CLI 向けです。この 2 つのうち Kiro CLI (V2) が先に表示されるため、より前に検出されたハーネスが一覧の先頭にある場合や、ハーネスのエディター自身のターミナルでセットアップを実行していてそのハーネスが先頭に来る場合を除き、Enter を押すと Kiro CLI (V2) が選ばれます。

セットアップの選択肢を示す前に、ウィザードは一時的なファイルとディレクトリを使ってプロジェクトのファイルシステムを調べ、その後それらを削除します。トランザクションのロック、排他的なファイル作成、通常ファイルの `fsync`、可変ファイルへの追記と読み戻し、ディスクリプターとパスの同一性、リネームによるファイル置換、ディレクトリのリネーム、Unix の `chmod`、ランタイムのワークフローロックの連携を確認します。ディレクトリの `fsync` がサポートされていなくても許容します。これらのチェックは利用できない操作を検出するためのもので、成功してもアトミック性やクラッシュ時の永続性を保証するものではありません。調査に失敗すると、ストレージの対処方法を示し、何も書き込まずにセットアップを停止します（削除できなかった調査用ファイルがある場合はその名前を示します）。[config がストレージエラーで失敗する](15-troubleshooting.md#config-fails-with-a-storage-error)を参照してください。

推奨の既定値では、ハーネスの現在のモデルプロバイダーを維持します。カスタマイズでは、Harness、Model provider、Model effort preset、Plugins、MCP servers、モデルプリセットの設定レイヤーを順に設定します。プロバイダーのステップでは、1 番目に現在のプロバイダーの維持、2 番目に Amazon Bedrock を提示します。Kiro CLI (V2) の行では、ステップ 2 は代わりに Session model になります。Kiro アカウントのモデルを一覧表示し、選んだモデルをプリセットのセッション effort とともに、AI-DLC のすべてのステップの後、最後に個人の Kiro 設定に保存します。Kiro が auto の場合、推奨の既定値ではそのモデルについての質問を 1 つだけ行います。[Kiro CLI (V2) のセッションモデルと effort](harnesses/kiro.md#kiro-cli-v2-session-model-and-effort)を参照してください。
Kiro の行については[初回実行](harnesses/kiro.md#first-run)を参照してください。
完全な選択肢以外の番号付きプロンプトにはすべて角括弧付きの既定値があり、無効な入力ではその場で再度尋ね、各回答は復唱されます。回答の確認表では、Enter で適用、ステップ番号で編集ができます。この最終ゲートより前にファイルが書き込まれることはありません。適用後は、プロジェクトのファイルとモデルプリセットの設定レイヤーを示す進行形の受領表示が出て、続いて本当に対応が必要なアクションが示され、最後にハーネスの起動コマンドと最初のワークフローコマンドが正確に表示されます。

ステップ 3 では 4 つ目の選択肢 `unchanged` も提示されます。これはプリセットを記録せず、既存のモデル設定を維持します。モデルのポリシーがないプロジェクトでは同梱の既定値を使います。推奨の既定値は引き続き `balanced` ですが、Kiro IDE、Cursor、GitHub Copilot は例外です。これらではすべてのエージェントがセッションのモデルと effort を使うため、セットアップは `unchanged` を推奨して既定値とし、再実行時のマップでは Models をホストのセッション名とともに `[ok]` と表示します。

既存プロジェクトでの再実行では、Harnesses、Models、Runtime、Flags、Project、Providers、Trust、Workspace の 8 行のマップを表示します。各行は小文字の `[ok]` または `[needs]` です。既定が yes のゲートが 1 つあり、Models、Runtime、Providers、Trust の指摘だけを順に確認します。Workspace は報告するだけで、順に確認はしません。`aidlc/spaces/default/memory/` の外枠が欠けている場合は、質問ではなく `aidlc config --harness <name>` による明示的な更新で修復されます。外枠が不完全な間はゲート自体を提示しません。各セクションが存在しないディレクトリで失敗するか、何もしない回答では何も再構築されないためです。その場合、最後の一覧は再構築コマンドから始まります。Bun 起動型のプロジェクション（`aidlc-copy-runtime-X.Y.Z.tar.gz` の `runtime/<name>/` ルートや、チェックアウトの `dist/<name>/` ツリーからコピーしたもの）では、更新元となるインストール済みランタイムがないため、そのコマンドには `--download` も付きます。これはプロジェクトのリリース用のコピーランタイムを取得して検証します。ネイティブインストールではこのフラグなしでインストール済みのランタイムから更新します。`aidlc/` ルートが存在しない場合は 1 回だけ数えられます。Trust セクション自身の `workspace-root-missing` の問題は Workspace の行にまとめられます。Providers の行は、独自にモデルアクセスを提供する Kiro CLI と Kiro IDE、および回答がない場合にセッション自身のモデルアクセスを意味する GitHub Copilot と Cursor では、記録された回答がなくても `[ok]` と表示されます（例: `model access comes with your GitHub
Copilot session`）。独自の Amazon Bedrock を使う場合は、`aidlc config providers` がそこに記録します（Cursor では IDE だけが Bedrock のキーを受け付けます）。GitHub Copilot、Cursor、Kiro IDE では Models の行が `[ok]` と表示され、ホストの名前を示します（例: `every agent uses your GitHub
Copilot session's model and effort`）。これらのホストではプリセットは何も変えない（Kiro では 1 つのエージェント自身のモデルと effort だけが適用される）ため、尋ねるべきポリシーがなく、記録済みのポリシーはそこでは適用されないと示されます。[モデルと effort の選び方](#choosing-a-model-and-effort)を参照してください。
Runtime では、すぐに行うべきアクションを先頭に示し、診断には `aidlc config runtime --show` を案内します。最後の一覧は、ラベルとコマンドを対応させた簡潔なリストです。セクション名を指定したコマンド、非 TTY の実行、`--dry-run`、`--json`、`--quiet` では決定的な出力を維持し、対話型のウィザードは表示しません。

### Config Options

| オプション | 意味 |
|--------|---------|
| `--project-dir <path>` | 現在のディレクトリではなく、このプロジェクトを対象にする |
| `--harness <name>` | インストール済みのハーネスランタイムを選ぶ |
| `--from <dir-or-tgz>` | インストール済みランタイムの代わりにローカルのリリースファイルを使う。`aidlc-copy-runtime-X.Y.Z.tar.gz`（隣にある `.sha256` で確認する）、それを展開した `runtime/` フォルダー、または 1 つのプロジェクションのディレクトリかアーカイブ |
| `--download` | このマシンにプロジェクトが必要とするリリースがない場合に取得・検証してからコマンドを完了する。ドライランのプラントークンを適用する際にも再度必要 |
| `--mcp defaults\|none` | Claude 用の任意の同梱 MCP エントリを追加または省略する |
| `--dry-run` | 対象ディレクトリを作成せず、バイトも変更せずに、完全なプランを計算する |
| `--plan-token <token>` | JSON のドライランで承認されたプランそのものだけを適用する |
| `--show` | 何も変更せずに設定を表示する。セクションを指定しない場合はすべてのセクションを順に表示し（`aidlc config --show`。`--json` ではセクションをキーとする 1 つのオブジェクトを出力し、各値はそのセクションの `--show --json` の出力と同じ）、セクションを指定した場合はそのセクションだけを表示する |
| `--force` | ポリシーが許す範囲で、ローカルで変更されたフレームワーク所有のファイルと管理ブロックを置き換える |
| `--yes` | 認識されない対象ディレクトリやセクションの変更を確認する。MCP への同意を意味したり、セクションの回答を選んだりはしない |
| `--json` | 件数、アクション、`data.notes`、`data.planToken` を含む結果オブジェクトを 1 つ出力する |
| `--quiet` | 要約または対処方法を 1 行で出力する |
| `--no-color` | カラー出力を無効にする |

`--project-dir`、`--json`、`--quiet`、`--no-color`、`--yes`、`--offline`、`--verbose` は AI-DLC 自身のオプションです。`aidlc config --project-dir /work/shop` や `aidlc doctor --json` のように、実行するコマンドと一緒に指定してください。あなた自身の言葉がたまたまこれらのどれかに似ていても、それはあなたの言葉として扱われます。そのため `/aidlc add a --project-dir flag to the CLI` は、入力したすべての語をそのまま保ったまま、その作業を依頼したことになります。

### Model Policy

`aidlc config models` は、選択した設定レイヤー（`--project` の場合は `aidlc.settings.json`）にモデルのポリシーを記録し、通常の config のプラン、確認、トランザクションを通じて適用します。モデルプロバイダーに接続することはありません。Kiro CLI では、セッションモデルを選ぶと、アカウントのモデル一覧とそのモデルの effort レベルを Kiro CLI に問い合わせます。`--yes` を付けた場合に問い合わせるのは `--session-model` だけです。

公開されているグループは次のとおりです。

| グループ | エージェント | 同梱のティア |
|-------|--------|--------------|
| Deciding | 設計、実装、プロダクト、セキュリティ、品質の 9 エージェント | judgment |
| Reviewing | プロダクトリードとアーキテクチャレビュアー | balanced |
| Writing up | デリバリー、パイプラインとデプロイ、運用 | templated |

ポリシーはエージェントごとに次の順序で解決されます。

1. エージェントごとの例外
2. グループのダイヤル（直接設定、またはプリセット経由）
3. 同梱のティアの既定値
4. セッションからの継承

ピン留めは双方向に効きます。ピン留めされたエージェントは、後でセッションがより大きなモデルに移っても、ピン留めされたままです。フレームワークが自らエージェントをセッションより上に引き上げることはありません。ポリシーが記録されていない場合、Deciding と Writing up は継承します。一段下げた設定を同梱しているのは、計測に基づく reviewing ティアのベースラインだけです。初回実行ウィザードの既定の選択肢は `balanced` プリセットを記録し、3 つのグループすべてを medium の effort に設定します。Kiro IDE、Cursor、GitHub Copilot ではプリセットを記録しません。

```bash
aidlc config models --show
aidlc config models --reviewing-effort xhigh --project --yes
aidlc config models --agent architect --effort xhigh --model provider/raw-id --project --yes
aidlc config models --check
aidlc config models --reset --project --yes
aidlc config models --keep --yes
aidlc config models --session-model claude-opus-4.8   # Kiro CLI: your personal session model
```

変更の最後には、何が変わったか、元に戻すコマンド、誰がそれを反映するか（さらに未完了のセットアップ手順があればそれも）が示されます。`--json` と `--quiet` の出力は変わりません。

Claude Code、Codex、opencode、Kiro CLI では、ポリシーが記録されていない間、回答するまで更新のたびにセットアップがモデルについて尋ねます（Kiro IDE、Cursor、GitHub Copilot では尋ねません）。モデルを現状のままにすることも回答の 1 つです。モデルの質問で Enter、初回セットアップで `unchanged`、または `--keep` を選ぶと、プロバイダーの回答と並んで `<harness>/tools/data/harness.json` に記録され、セットアップは尋ねなくなります。`--reset` で消去されます。

`--show --json` は、すべてのエージェントの実効モデル、effort、その出所を出力します。`--check` は CI 向けの逆方向のチェックで、記録されたポリシーがハーネスの設定面に完全に反映されていない場合に非ゼロで終了します。プリセットが何も変えない Kiro IDE、Cursor、GitHub Copilot では、書き込めない記録済みの effort は `aidlc doctor` と同様にセッションに委ねられ、チェックの失敗にはなりません。セッションがすべてのエージェントのモデルと effort を決める Cursor と GitHub Copilot では、問題がない場合のチェックは、記録済みのポリシーが反映されたとは言わず、`aidlc doctor` と同様にその旨を伝えます。

モデルとフラグのポリシーは、次の階層を通じて葉ごとに解決されます。

1. ティア表とプリセット表にある同梱の既定値
2. マシンの `${AIDLC_INSTALL_ROOT:-~/.local/share/aidlc}/aidlc.settings.json`
3. プロジェクトの `aidlc.settings.json`
4. 個人の `aidlc.settings.local.json`
5. 環境変数

バイパスは上書きではなく加算されます。3 つのファイルのいずれかに記録されていれば、そのスイッチは有効です。

プロジェクトのファイルは、コミットされるチームのポリシーです。ローカルのファイルは個人用です。AI-DLC が管理する `.gitignore` ブロックにこのファイルが含まれており、`.gitignore` がそれより古いインストールでは、このファイルを作成する config コマンドが代わりにクローン自身の `.git/info/exclude` に追加します。どちらのファイルもチームのコードとしては扱われません。変更には `--project`、`--local`、`--global` のうちちょうど 1 つが必要です。ただしバイパスは例外で、レイヤーなしの `--bypass` はローカルのファイルに記録し、レイヤーなしの `--clear-bypass` は自分のターミナルで入力した場合、それを記録しているすべてのファイルからクリアします（チャットからエージェントが実行した場合はローカルのファイルだけをクリアし、チェックを無効にしたままのプロジェクトまたはマシンのファイルを示します）。対話型ウィザードはレイヤーを尋ね、リポジトリ内ではプロジェクトのポリシーを推奨します。認識されたプロジェクトの外ではマシンのレイヤーだけが有効なので、`--global` が推定されます。`--show` は各実効値に、それを決めた出所のラベルを付けます。

3 つのファイルはすべて 1 つの厳密なスキーマを使います。未知のキーは安全側に倒して失敗し、`offline` や `release-base-url` などの更新・リリース用のキーはマシン専用です。エディターからは生成された `<harness>/tools/data/aidlc-settings.schema.json` を参照できます。既定値の設定ファイルは書き込まれません。

変更不可で effort のみを指定するプリセットが 3 つ同梱されています。

| プリセット | Deciding | Reviewing | Writing up |
|--------|----------|-----------|------------|
| `thorough` | セッションの effort | `xhigh` | セッションの effort |
| `balanced`（ウィザードの既定値） | `medium` | `medium` | `medium` |
| `minimal` | `medium` | `medium` | `low` |

プリセットがモデル ID を設定することはありません。明示的なグループのダイヤルとエージェントごとの例外で、プリセットの effort を上書きできます。各セッションを 1 つのモデルで実行する Kiro CLI では、各プリセットは代わりに 1 つのセッション effort になります。`minimal` は low、`balanced` は medium、`thorough` は xhigh、またはそのモデルの 1 段下のレベルです（[Kiro CLI (V2) のセッションモデルと effort](harnesses/kiro.md#kiro-cli-v2-session-model-and-effort)）。

アップグレード時、`preset: balanced` または `preset: minimal` を記録したインストールは、次にプロジェクションが再生成されたときにこれらの effort を反映します。`aidlc update` の後は、各プロジェクトで `aidlc config --yes` を実行して記録済みのポリシーを再適用するか（開いている作業はそのまま続きます）、`aidlc config models --preset balanced --project --yes`（必要に応じて `minimal` に置き換える）で明示的に選んでください。update が変更するのはマシンのランタイムだけです。doctor と `aidlc config models --check` は問題を報告しますが、変更は適用しません。モデルのポリシーが記録されていないインストールは同梱のティアの既定値を維持し、影響を受けません。

プリセットまたは既存のプロファイルから、プロジェクトのプロファイルを派生させます。

```bash
aidlc config models --from thorough --reviewing-effort medium \
  --save-as my-profile --project --yes
```

プリセットとプロファイルに含まれるのはグループの effort だけです。生のモデル ID を指定できるのはエージェントごとの例外だけです。`--yes` は変更を確認しますが、ポリシーを選ぶことはありません。決め手となるフラグがない場合、TTY ではモデルポリシーのウィザードが開き、非 TTY の実行は使い方の案内を表示して失敗します。

ハーネスには、読み取れる設定だけが渡されます。Codex は `max` の effort を `xhigh` に切り下げます。opencode は `xhigh` を `high` に切り下げます。Kiro CLI では、プリセットは対話セッション全体に 1 つの effort を設定し、セッションモデルとともに個人の Kiro 設定に保存されます（[Kiro CLI (V2) のセッションモデルと effort](harnesses/kiro.md#kiro-cli-v2-session-model-and-effort)）。明示的なグループのダイヤルに対応する Kiro の設定面はありません。エージェントごとのモデルの例外は、その effort をプロジェクトの `chat.modelDefaults` を通じて渡し、そのプロジェクトでは個人の effort マップを置き換えます。Kiro CLI でモデルなしに設定した effort は他のツールのために保持され、コマンドはその旨を 1 行で伝えます。
Kiro IDE、Cursor、GitHub Copilot にはプリセットやグループの effort を反映する設定面がないため、セットアップはそこではプリセットを記録しません。Kiro では、`aidlc config models --agent` で引き続き 1 つのエージェント自身のモデルと effort を設定できます（[1 つのエージェントを独自のモデルと effort で動かす](harnesses/kiro.md#one-agent-on-its-own-model-and-effort)）。Cursor と Copilot では、それでも設定したモデルや effort は、それを適用するツールを使うチームメイトのために保持されますが、効果のないキーは書き込まず、このツールは独自のモデルピッカーで選んだモデルを使う旨をコマンドが 1 行で伝えます。この 3 つでは、すべてのエージェントがセッションのモデルと effort を使います（Kiro では、そのエージェントにモデルや effort が設定されていない限り）。セットアップのチェックと `aidlc doctor` はポリシーを尋ねる代わりにその旨を伝え、doctor が警告するのは、そのハーネス名を指定して記録されたエージェントのモデルについてだけです。

モデルのポリシーはエージェント単位です。ステージのファイルがモデルや effort のキーを持つことはなく、ステージの重要度は引き続きスコープが管理します。

### Choosing a Model and Effort

AI-DLC は高性能な推論モデルで最もよく機能します。推奨モデルは Claude Opus 4.8 です。モデルと effort を設定する場所はハーネスによって異なります。

- **GitHub Copilot、Cursor、Kiro IDE:** ホスト側で、セッション全体に対して設定します。
  すべてのエージェントは `/aidlc` を実行したチャットのモデルと effort を使い、AI-DLC が記録する内容でそれが変わることはありません。例外は Kiro で、`aidlc config models --agent` で 1 つのエージェント自身のモデルと effort を設定できます（[1 つのエージェントを独自のモデルと effort で動かす](harnesses/kiro.md#one-agent-on-its-own-model-and-effort)）。`model:` 行を手で書くのではなく、こちらで設定してください。手書きの行は `aidlc config models --check` がドリフトとして報告します。Copilot では、エージェントの `.md` ファイル内の `model:` によるピン留めは、CLI と VS Code でモデル名の読み方が異なるため移植性がありません。
- **Claude Code、Codex CLI、opencode:** セッションのモデルと effort が、コンダクターと継承するすべてのエージェントを動かします。`aidlc config models` でエージェントの effort（`balanced` プリセットでは medium に設定）とエージェントごとの例外を設定できます。
- **Kiro CLI:** セッションモデルとその 1 つの effort です。`aidlc config models` はそれらを個人の Kiro 設定に保存します（プリセットが effort を設定します。`minimal` は low、`balanced` は medium、`thorough` は extra-high）。Kiro の中では `/model` と `/effort` で変更できます。`/model` による切り替えがコンダクターのディスパッチするエージェントに届くのは、`/model set-current-as-default` で保存した後だけです。推奨のセットアップでは、そのモデル用にすでに保存してある effort を維持します。Kiro IDE の行では、セットアップは Kiro CLI のセッションモデルだけを保存します（[初回実行](harnesses/kiro.md#first-run)）。[Kiro CLI (V2) のセッションモデルと effort](harnesses/kiro.md#kiro-cli-v2-session-model-and-effort)を参照してください。

組織で Opus がなく、Claude Sonnet のような中位のモデルしか提供されていない場合は、次のようにします。

- **セッションは medium の effort で始めます。** ワークフローは多くのターンを実行し、effort を上げるとモデルはそのすべてのターンで長く考えるため、high や最大の effort では実行全体が大幅に遅く高コストになります。medium は、エージェントの effort を設定できるハーネスで `balanced` プリセットが与える値です。
- **effort を上げるのは、それが必要なステージだけにします。** たとえば難しいユニットの Code Generation で上げ、その後また下げます。GitHub Copilot、Cursor、Kiro IDE、Kiro CLI では、それはセッションの effort です（Kiro IDE では、developer エージェントに effort が設定されていない限り）。Claude Code、Codex CLI、opencode では、Code Generation は developer エージェントで実行され、このエージェントはプリセットや effort が記録されていない間だけセッションに従います。それ以外の場合、effort は記録された値です。上げるには、たとえば `aidlc config models --agent developer --effort high --project --yes` のように記録します。これはワークフローが開いている間でも機能します。config が developer エージェントのファイルを書き換え、ハーネスは次にそのエージェントを起動するときにそれを使います（Claude Code ではエージェントの次回起動時。すでに実行中のステップは開始時の設定を維持します）。コマンドは元に戻すためのコマンドを表示します（1 つのエージェントの設定の場合は `--effort default` で、その設定だけを削除します）。
- **モデルや effort の変更は、ステージの間に新しいチャットで行います。** [ワークフロー途中でのモデル変更](11-session-management.md#changing-model-mid-workflow)を参照してください。

ワークショップを運営する場合は、[ファシリテーターガイド](facilitator-guide.md)で準備状況のチェックと各実行を小さく保つ方法を説明しています。

### In-session alias

`/aidlc --config [section]` は、これと同じ config セクションに対する会話型のエイリアスです。コンダクターは尋ねる前に現在の JSON の状態を読み、希望する変更だけを集め、受け入れられた各セクションを、厳密に 1 つの `aidlc config <section> <explicit value flags> --yes` コマンドで反映します。セクションを変更しない場合はコマンドを実行しません。反映または辞退の後、エイリアスは停止します。ワークフローの作業を前進させたり再開したりすることはありません。

### Runtime Diagnostics

`aidlc config runtime` は、プロジェクトのフックが実際に使う環境を確認します。Linux では、`getconf PATH` に加えて、`/etc/environment` の `PATH` 行、`/etc/login.defs` の `ENV_PATH`、`environment.d` から非対話のベースラインを導き出します。macOS では `getconf PATH` に加えて `/etc/paths` と `/etc/paths.d` を使います。Windows では、シェルのプロファイルを読み込まずにユーザー PATH とマシン PATH を読み取ります。その後、インストールされたフックのバイト列が必要とするコマンド（コピー版プロジェクションでは `bun`、ネイティブプロジェクションでは `aidlc`）を解決し、選択したハーネスの CLI を確認します。

```bash
aidlc config runtime --show
aidlc config runtime --check
aidlc config runtime --record-paths --yes
aidlc config runtime --reset --yes
```

`--record-paths` は、解決した結果を `harness.json` に記録します。フックのコマンドは書き換えません。ホストの許可ルールと Codex のフックの信頼は、素の `bun` または `aidlc` コマンドの接頭辞に結び付いているため、それを絶対パスに置き換えると既存の信頼の取り決めが無効になります。コマンドが対話時にしか使えない場合や存在しない場合、このセクションは代わりにプラットフォームごとの PATH の手順を示します。

`/aidlc --doctor` は、同じ調査結果を `Runtime hook PATH` 行として表示します。コマンドが現在のシェルの PATH にしかないものの、このプロジェクトのフックが発火している場合（`.aidlc-engine/hooks-health/` 配下に、直近 10 分以内の古くないハートビートがあり、それがその後終了していない起動によるもの）、この行は合格となり、フックが最後に発火した時刻を示します。ハーネスはその PATH をフックに渡しているとみなせるためです。それ以外の場合は警告し、コマンドが見つかったディレクトリを示したうえで、ターミナルから起動したハーネスでは変更が不要であること、`.bashrc` や `.zshrc` を編集してもこのチェックは変わらないことを伝えます。

ハーネスの CLI のチェックでは、対応するハーネスに応じて `claude`、`kiro-cli >= 2.6.0`、`codex >= 0.145.0`、`opencode` が必要です。Copilot CLI と Cursor の `agent` CLI は、それらのインストールが VS Code や IDE からしか使われない場合があるため、参考扱いです。
Kiro のディストリビューション（`--harness kiro`）では別の CLI は不要です。`kiro-cli` はそこでは任意で、ターミナルから AI-DLC を実行する場合にのみ必要であり、存在する場合は 2.24.1 に対して確認されます。

### Provider Diagnostics

`aidlc config providers` は、このプロジェクトのインストールに対するプロバイダーの回答を記録します。ハーネスですでに設定されているプロバイダーを維持するのが既定の回答です。Amazon Bedrock は明示的なオプトインです。Kiro CLI と Kiro IDE では、モデルアクセスが Kiro に含まれているため尋ねません。手動で設定したプロバイダー向けに `other` も引き続き利用でき、確認を促すリマインダーが付きます。

```bash
aidlc config providers --provider current --yes
aidlc config providers --provider amazon-bedrock \
  --region us-east-1 --profile default --yes
aidlc config providers --show --json
aidlc config providers --check
aidlc config providers --mark-done bedrock-model-access --yes
aidlc config providers --reset --yes
```

`--region`、`--profile`、`--opencode-default` は、記録済みまたは選択したプロバイダーが `amazon-bedrock` の場合にのみ適用されます。

資格情報の検出はオフラインでのみ行います。AWS の環境変数、`~/.aws/config`、`~/.aws/credentials`、ロールやコンテナの資格情報の変数、AWS SSO のキャッシュを調べます。STS、Bedrock、モデルのエンドポイント、その他のネットワークサービスを呼び出すことはありません。

記録された Bedrock の回答は、通常のステージング付き config トランザクションを通じて適用されます。

| ハーネス | 記録された回答の適用 |
|---------|-----------------------------|
| Claude Code | Bedrock を有効にし、自分専用の `.claude/settings.local.json` に `AWS_REGION` と任意の `AWS_PROFILE` を書き込む（git はこのファイルを無視し、プロファイルは `harness.json` に記録されない）。また `.mcp.json` 内の AWS MCP の URL と `AWS_REGION` メタデータも同じリージョンに合わせる |
| Codex CLI | 選択を記録し、プロバイダー、資格情報、モデルを `~/.codex/config.toml` に保持するよう利用者に案内する |
| Kiro CLI | プロバイダーの回答なし。モデルアクセスは Kiro に含まれる |
| Kiro IDE | プロバイダーの回答なし。モデルアクセスは Kiro に含まれる |
| opencode | `provider.amazon-bedrock.options.region/profile` を `opencode.json` に書き込むことを提案する。`--opencode-default yes|no` がその回答を記録する |
| GitHub Copilot | 手動の BYOK 環境設定を確認したことを記録する |
| Cursor | 手動のプロバイダーとモデルピッカーの設定を確認したことを記録する |

Bedrock のモデルアクセスと IAM 権限の検証は、オフラインでは自動化できません。そのため記録には名前付きの保留アクションが含まれます。`--show` はそれらを一覧表示し、`--check` は保留中の間は非ゼロのままで、`--mark-done <id>` は完了を記録します。Codex のプロバイダー設定は、実効的なユーザー設定や別の資格情報チャネルをオフラインでは解決できないため、完了後も明示的に自己申告のままです。`--check` は設定を検証済みとは言わず、その警告付きで成功を返します。
`--provider current` はハーネスで設定済みのプロバイダーを維持し、Claude、Codex、opencode のプロジェクトファイルから、AI-DLC が同梱の既定値または以前の記録によるものと判断できる Bedrock の値だけを削除します。カスタマイズされた Claude のモデルエイリアスは保持されます。カスタマイズされた従来の Codex の Bedrock ブロックも保持されます。そのブロックがまだそのプロバイダーを指している場合、`--check` は設定に問題がないとは言わず、警告を報告します。`--provider other` は手動で設定した Bedrock 以外のプロバイダーを記録し、`--acknowledge` が渡されるまでその設定を保留中として報告します。`--reset` は、Claude、Codex、opencode のプロジェクトファイルから、AI-DLC の従来の Bedrock の既定値そのもの、または以前に記録された回答のために書き込まれた値を削除します。由来を確認できない値は保持されます。Claude Code では、`--provider current`、`--provider other`、`--reset` は `.claude/settings.local.json` から Bedrock のスイッチとリージョンも取り除きます。そこにある `AWS_PROFILE` は残り、コマンドはその旨を伝えます。

Claude Code では、Bedrock の回答は各個人のものです。チームメイトのクローンにはプロジェクトの記録は含まれますが、あなたの設定は含まれないため、`aidlc config providers
--check` と doctor は、Bedrock が記録されているものの自分には有効になっていないことを一度だけ警告します。以前のリリースでセットアップしたプロジェクトは、誰かが改めて `aidlc config providers` に回答するまで、共有の `.claude/settings.json` に回答を保持します。回答した時点でその人自身のファイルに移され、その旨が伝えられます。更新やアップデートではそのまま残ります。

質問は目の前のハーネスに合わせた文言になっているため、各インストールでは、そのハーネスに実際に存在する 2 つの選択肢が提示されます。

| ハーネス | `amazon-bedrock` が記録する内容 |
|---------|--------------------------|
| Claude Code | 自分専用の `settings.local.json` に AWS のリージョンとプロファイル、存在する場合は `.mcp.json` に AWS MCP のリージョン |
| Codex CLI | プロジェクトの記録に AWS のリージョンとプロファイルを記録し、その後 `$CODEX_HOME/config.toml` でのユーザーレベルのプロバイダー設定を案内する |
| OpenCode | AWS のリージョンとプロファイルを記録し、それを `opencode.json` に書き込むことを提案する |
| GitHub Copilot | Copilot の BYOK プロバイダー変数を自分で設定したこと |
| Cursor | Cursor でプロバイダーを自分で設定したこと |


Kiro CLI と Kiro IDE は独自のモデルアクセスを提供するため、AI-DLC はそれらのモデルプロバイダーを設定しません。`aidlc config providers` はモデルアクセスが Kiro に含まれていることを伝え、何も尋ねません。`kiro-cli` を読み取れない場合の Kiro IDE での初回実行ウィザードも同様です。一方、Kiro CLI、および読み取り可能な `kiro-cli` がある Kiro IDE では、ウィザードのステップ 2 で代わりに Kiro CLI のセッションモデルを選びます。プロバイダーのフラグは拒否され、従来の記録があっても Providers の行は `[ok]` と表示されます。
以前のビルドが残した記録は `aidlc config providers --reset --yes` で消去できます。以前のビルドの `builtin` の記録は引き続き読み込まれ、保留アクションなしのハーネス管理として扱われます。従来の Kiro の Bedrock の記録も、保留アクションを含めて無視され、そこから何かが書き込まれることはありません。`.kiro/settings/mcp.json` 内の `aws-mcp` のリージョンは通常の MCP 設定であり、モデルプロバイダーの回答ではありません。以前のビルドの Bedrock の回答によるものであれ、あなたが設定したものであれ、そのファイルにあるリージョンは更新をまたいで保持され、`--reset` はそのファイルに触れません。

他のすべてのハーネスでは、現在のプロバイダーを維持するか、Amazon Bedrock にオプトインするかを尋ねます。AWS の資格情報が検出された場合も含め、現在のプロバイダーを維持するのが既定です。Copilot と Cursor は独自の BYOK やプロバイダー設定を通じて Bedrock に接続します。AI-DLC はそれを実行するのではなく、保留アクションとして追跡します。

Kiro では、`--check` は回答が不要であることを伝え、従来の記録があってもゼロで終了します。回答のない GitHub Copilot と Cursor では、モデルアクセスがセッションに含まれているため、`--check` と `doctor` は回答が不要であることを伝え、`--check` は独自の Amazon Bedrock アクセスを記録するためのコマンドを示します。それ以外の未記録のセクションでは、検証済みの回答を報告する代わりにその状態を示し、同梱のフォールバックのバイト列が引き続き有効であるため、やはりゼロで終了します。

これらのハーネスでは、`keep current` が 1 番目の回答で既定値です。`amazon-bedrock` は 2 番目の回答です。記録済みの回答のままセクションに入り直すと、その回答は保持されます。Bedrock は明示的に置き換えない限りリージョンとプロファイルを維持し、`other` は手動設定の保留アクションを維持します。保留アクションは引き続き `--mark-done` で完了できます。

### Trust Diagnostics

`aidlc config trust` は、ホストネイティブの信頼を読み取って検証します。信頼のシード、許可ルール、IDE の設定を再生成することはありません。プロジェクト外への書き込みは、Kiro IDE と Kiro CLI の Workflows スイッチ、および Windows での Kiro IDE の既定のターミナルだけで、それも要求した場合に限ります。

```bash
aidlc config trust --show
aidlc config trust --check
aidlc config trust --acknowledge --yes
aidlc config trust --reset --yes
aidlc config trust --kiro-workflows off --yes   # Kiro IDE; or: on
aidlc config trust --kiro-cli-workflows off --yes   # Kiro CLI; or: on
aidlc config trust --kiro-terminal powershell --yes   # Kiro IDE on Windows
```

Kiro IDE では、`--show` と `--check` は Kiro の Workflows スイッチも読み取ります（Kiro のユーザー設定の `kiroAgent.workflows.enabled` で、すべてのプロジェクトに適用されます。これが有効な間は AI-DLC のレビューやヘルパーが動作しません）。`--kiro-workflows off|on` はそのキー 1 つだけを変更し、ファイルの他のバイトはすべて保持します。Workflows が有効で、このコンピューターでまだ回答していない場合、セットアップは一度だけ尋ね、`aidlc config --yes` はそれを無効にします。回答は AI-DLC のインストールフォルダー内の `kiro-ide-workflows` に保存されるため、他のプロジェクトで再び尋ねられることはありません。その設定ファイルに AI-DLC が読み進められないエラーがある場合、Kiro は Workflows を有効として読む可能性があります。そのため、`--show` は読み取れなかったことを伝え、`--check` とセットアップはファイルが再び読めるようになるまで（以前の回答にかかわらず）警告し、次の Kiro IDE のチャットで一度だけ通知されます。そのファイルのエラーを修正してください。同じ Kiro の行で、`--show` と `--check` は Kiro CLI 自身の Workflows スイッチ（Kiro CLI の設定の `chat.enableWorkflows`。`kiro-cli
settings` を通じて読み取る）も読み取ります。これは Kiro CLI のチャットで同じツールを使えなくします。`--kiro-cli-workflows off|on` は Kiro CLI を通じてこれを設定し、その回答は Kiro IDE とは別に `kiro-cli-workflows` に保存されます。Kiro CLI が応答しない場合は表示されません。[Kiro の Workflows 機能](harnesses/kiro.md#kiros-workflows-feature-stops-ai-dlcs-reviews-and-helpers)を参照してください。

Windows では、Kiro IDE はエージェントのコマンドを既定のターミナルプロファイル（Kiro のユーザー設定の `terminal.integrated.defaultProfile.windows`）で実行します。それがコマンドプロンプト（Kiro はコマンドが 1 秒以内に終了した場合にしか応答を表示しないため、AI-DLC のほとんどの応答が空で返ってくる）で、このコンピューターでまだ回答していない場合、セットアップは PowerShell に設定するかどうかを一度だけ尋ねます。`aidlc config --yes` はそれを設定し、`--kiro-terminal powershell` は単独でそれを設定します（すでに PowerShell の場合はその旨を伝えます）。いずれもそのキー 1 つだけを変更してファイルの他のバイトはすべて保持し、回答は AI-DLC のインストールフォルダー内の `kiro-ide-terminal` に保存されます。[Kiro のターミナルとしてのコマンドプロンプト](harnesses/kiro.md#command-prompt-as-kiros-terminal-on-windows)を参照してください。

Codex では、`$CODEX_HOME/config.toml` にプロジェクト固有の信頼シードのエントリ一式がすべて揃っていることがチェックの条件です。サポートされている対処方法は 2 つで、TUI で `Trust all and continue` を一度実行するか、`<PROJECT_DIR>` を置き換えてシード全体をマージするかです。それまでは Codex のフックは 1 つも発火しません。`--dangerously-bypass-hook-trust` を使っても信頼されていないフックは発火せず、2 つ目のシード一式を追記すると TOML として不正になります。

Kiro のディストリビューションでは、信頼はコンダクターの `permissions`（`.kiro/agents/aidlc-aidlc.md`）に同梱されているため、チェックで追加するものはありません。Kiro IDE 1.x は `.vscode/settings.json` の `kiroAgent.trustedCommands` を読み取らなくなりました。`--show` は、選択したハーネスの信頼ファイルと許可リストのファイルを一覧表示します。

Copilot では、チェックは Copilot CLI の `trustedFolders`（`COPILOT_HOME` 配下の `config.json`。未設定なら Windows では `%USERPROFILE%\.copilot`、それ以外では `~/.copilot`）を読み取り、プロジェクトが含まれていない場合に警告します。プロジェクトより上位のフォルダーも対象として数えます。VS Code はその一覧を読みません。VS Code のフックには信頼されたワークスペースと、Chat: Use Hooks 設定の有効化が必要ですが、AI-DLC からはそれらを確認できないため、Trust の行でそれらを示します。CLI がプロジェクトを信頼していない場合、`aidlc config trust`（およびセットアップの確認手順）がその方法を伝えます。プロジェクトで一度 `copilot` を実行し、"Yes, and remember this folder for future sessions"（はい、今後のセッションのためにこのフォルダーを記憶する）を選んでください。フォルダーを信頼するとそのコードの実行を許すことになるため、AI-DLC が CLI の `config.json` を自ら編集することはありません。

信頼のチェックでは、コピーによるインストールで漏れがちなプロジェクト内の兄弟ディレクトリも確認します。すべてのハーネスの `aidlc/`、Codex の `.agents/`、opencode と Copilot の `.aidlc/` エンジンです。

doctor は、インストールされた指示ファイルを config の所有権ベースラインと照らして分類します。管理ブロックが損なわれていなければ `block present, user
content preserved` と報告します。ブロックやファイルがない場合は `aidlc config` を実行するよう伝えます。手で変更された管理ブロックや、フレームワーク所有のファイル全体が変更されている場合は競合として報告します。
オンボーディングの `# AI-DLC` タイトルの代わりに書かれた行は競合にはなりません。それはプロジェクト自身のものです（後述）。複数のハーネスのツリーがある場合、この行は呼び出したハーネスに従います。

### Project Flags

`aidlc config flags` は、既定のスコープ、スウォームモード、フックのデバッグ、センサーのタイムアウト、質問の保持期間、明示的なガードのバイパスやセレモニーの停止スイッチについて、プロジェクトの回答を記録します。

```bash
aidlc config flags --default-scope <installed-scope> \
  --swarm on --hook-debug off --sensor-timeout-ms 90000 \
  --question-retention-days 30 --project --yes
aidlc config flags --bypass AIDLC_SKIP_ARTIFACT_GUARD --local --yes
aidlc config flags --show
aidlc config flags --check
aidlc config flags --reset --project --yes
```

実際の環境変数が常に優先されます。既存のツールとフックは、まず環境を読み、変数がない場合にローカル、プロジェクト、マシンの設定を解決します。これにより、CI や一時的なシェルの export をスクリプトから扱いやすく保ちます。

`--question-retention-days <days|unlimited>` は、回答されなかった質問について、AI-DLC がその依頼の写しをどれだけの期間保持するかを制御します。正の整数が記録されていない限り、既定値は無制限です。値を設定すると、その日数より古い写しは、次に AI-DLC が作業を行ったときに削除され（status、help、その他の問い合わせでは何も削除されません）、期限切れの質問に回答しても拒否されます。`unlimited` を渡すと、他のフラグを変えずに、選択した設定レイヤーからその値を削除します。
`AIDLC_QUESTION_RETENTION_DAYS` が対応する環境変数での上書きです。

既定のスコープ名は、インストールされたスコープファイルから読み取ります。このセクションは組み込みのスコープ名で分岐しないため、スコープの名前変更やプラグインのスコープもデータとして扱われます。Claude Code では、config は `.claude/settings.json` 内のステージングされた `AWS_AIDLC_DEFAULT_SCOPE` の値も書き換えます。そうしないと、同梱のセッション環境が優先度の低い記録を覆い隠してしまうためです。

記録できるバイパスには、文書化された復旧用とセレモニー用のスイッチが含まれます。

- `AIDLC_SKIP_ARTIFACT_GUARD`
- `AIDLC_SKIP_HUMAN_PRESENCE_GUARD`
- `AIDLC_SKIP_REVISION_BACKSTOP`
- `AIDLC_SKIP_SUMMARY_CONFIRMATION_GUARD`
- `AIDLC_DISABLE_ENSEMBLE_EVIDENCE`
- `AIDLC_DISABLE_PLAN_APPROVAL_GUARD`
- `AIDLC_DISABLE_REVIEWER_SCOPE_HOOK`
- `AIDLC_DISABLE_REVIEW_FREEZE_HOOK`
- `AIDLC_DISABLE_USAGE_TRACKING`
- `AIDLC_DISABLE_SENSORS` — センサーの実行とセンサーゲートのチェックを無効にする
- `AIDLC_DISABLE_LEARNINGS` — ステージの学びの手順を無効にする
- `AIDLC_DISABLE_SUMMARY_CONFIRMATION` — 独立したサマリー確認のチェックポイントを無効にする（ステージの承認は無効にしない）

ウィザードがバイパスを提示することはありません。バイパスには明示的な `--bypass <name>` が必要です。`--show` は有効なバイパスをすべて、ガードが弱まる影響とともに表示します。
記録またはクリアした場合は、何が変わったかと元に戻すコマンド、切り替えたチェックについての行（さらに未完了のセットアップ手順があればそれも）だけを表示します。`--json` と `--quiet` の出力は変わりません。
使用状況の追跡、センサー、学び以外のバイパスはすべて利用者からチェックを奪うため、いずれかが有効な間、AI-DLC はその旨を 1 行で伝えます。伝える場所は、次のステップ、すべてのチャットの開始時（セッション開始時のコンテキストを表示しない opencode を除く）、`--show`、doctor の Flags 行です。その行では、チェックの名前、いつから、どのように設定されたか、チェックを元に戻す `--clear-bypass` コマンドを示します（[CLI コマンド](12-cli-commands.md)の「Environment kill switches」を参照）。

このうち 4 つは、マシン全体でフェンスを無効にします。問題がマシン全体ではなく 1 つの作業にある場合は、`/aidlc config set guard.<fence> off` を使うと、その作業に限って 1 つのフェンスを下げ、それを記録し、次の作業では元に戻します。[ガードポリシー](13-customization.md#guard-policy)と[5 つのフェンス](13-customization.md#the-five-fences)を参照してください。

### Project Choices

`aidlc config project` は、インストールするプラグインの選択、MCP への同意、シェル補完についての回答を記録します。

```bash
aidlc config project --plugins aidlc,test-pro --mcp none \
  --completions zsh --yes
aidlc config project --show --json
aidlc config project --check
aidlc config project --reset --yes
```

コピーチャネルのプロジェクションでは、`config project` はプラグイン、MCP、補完の選択を、プロジェクトがすでに持っているリリースのプロジェクト自身のファイルから適用するため、ダウンロードは不要です。MCP を再び有効にする場合は、ハーネスのフォルダーが保持している同梱のサーバー一覧を読み取ります。リリースが必要になるのは、プロジェクトが別のリリースにピン留めされている場合か、その一覧がハーネスのフォルダーにない場合だけです。その場合、ターミナルでは確認を求め、スクリプトでは `--download` を付けます。
`--from <path>` を使うと、代わりにダウンロード済みのファイル（`aidlc-copy-runtime-X.Y.Z.tar.gz`、それを展開した `runtime/` フォルダー、または 1 つのハーネスルート）を使います。自分で `.mcp.json` に追加したサーバーが記録されたり削除されたりすることはありません。

プラグイン名は、インストールされたグラフ、スコープ、プラグインのサイドカーから検出されます。ハードコードはされていません。選択には引き続き `harness.json` の既存のトップレベルの `plugins` 配列を使うため、グラフとランナーの再生成は、プラグインの合成と同じ選択の仕組みを使います。ワークフローが開いている間のプロジェクトの変更は、他の更新と同様に実行され（「Refresh Safety」を参照）、何が変わったかと、1 つのコマンドで表せる場合は以前の選択に戻すコマンドを伝えます。開いている作業が必要とするプラグインを無効にすると、その作業の名前が示されます。その作業はプラグインが再び有効になると続行します。同じコマンドに `--dry-run` を付けると、プロジェクトや設定ファイルを変更せずにプランをプレビューできます。

MCP への同意は引き続き `defaults` または `none` です。以前の同意がない非対話のプロジェクト変更では `none` を記録します。ただし、`.mcp.json` に同梱どおりの同梱サーバーがすでにある場合は、`defaults` を記録してそこにあるサーバーを維持し、何も追加しません。`--yes` は変更を確認するだけで、MCP のエントリを追加することはありません。

Claude Code では、`.mcp.json` が同意によって管理される設定面です。`--check` は `defaults` と `none` の両方を検証し、その後の通常の config の更新で回答が再適用されます。Kiro CLI は常に `.kiro/settings/mcp.json` を同梱しています。`defaults` はそのファイルと 5 つの同梱サーバーで満たされ、`none` は案内のみの希望でありフレームワーク所有のファイルを削除しません。現在の Codex、opencode、Copilot、Kiro IDE、Cursor のディストリビューションは MCP の設定面を同梱していないため、記録された回答は参考情報であり、`--check` が恒常的に失敗することはありません。`--show` は、実際の MCP ファイルが存在する場合は常にその名前を示します。

補完は案内のみで、マシン上のファイルは書き込みません。ネイティブインストールでは次のようなコマンドを表示します。

```bash
eval "$(aidlc system completions bash)"
```

コピーチャネルのインストールでは、対応する Bun の呼び出しを表示します。例:

```bash
eval "$(bun .claude/tools/aidlc.ts system completions bash)"
```

Fish では `... completions fish | source`、PowerShell では `... completions powershell | Out-String | Invoke-Expression` を使います。

既存のプロジェクトのスタンプがある場合、更新時のハーネスはそれで決まります。新しいプロジェクトでは、対話時はハーネスを尋ね、非対話の実行では `--harness` が必要です。
`.aidlc-version` が存在する場合、config はそのバージョンと完全に一致し、プロジェクトのハーネスにも一致するソースを必要とします。

config は、`.git`、`package.json`、`Cargo.toml`、`go.mod`、`pyproject.toml` のいずれかを含むディレクトリ、またはすでに AI-DLC のハーネスツリーがセットアップされているディレクトリを認識します。それ以外の場合、対話モードでは確認を求め、非対話モードでは `--project-dir` が必要です。

Claude 用の任意の MCP 統合は、TTY がない場合は既定で `none` になります。人間が使う TTY では、以前の選択がない場合に確認を求めます。`--yes` と `--json` は MCP への同意を与えません。確実な自動化では、`--project-dir`、`--harness`、`--mcp defaults|none` を明示的に指定します。JSON は出力を制御するだけで、それだけで TTY のプロンプトを無効にはしません。

スクリプトで厳密に承認するには、次のようにします。

```bash
token=$(
  aidlc config --project-dir "$PWD" --harness claude --mcp none \
    --dry-run --json | jq -r .data.planToken
)
aidlc config --project-dir "$PWD" --harness claude --mcp none \
  --plan-token "$token" --json
```

両方の呼び出しで同じソースと動作のオプションを使ってください。プレビューの後にソースのバイト列、オプション、プロジェクトの状態が変わるとトークンが変わり、適用は安全側に倒して失敗します。

### Refresh Safety

作業が開いている間に実行した `aidlc config` は、保留中のワークフローを含め、拒否されずに実行されます。設定の変更（`config models`、`flags`、`runtime`、`providers`、`trust`）はプロジェクト自身のファイルを読み、リリースを取り込みません。それぞれ、何が変わったかと、1 つのコマンドで以前の値に戻せる場合はそのコマンドを表示します。モデルやフラグの変更では、それを反映する開いているワークフローも示します。バイパス、フックのデバッグ、センサーのタイムアウト、質問の保持期間は再起動なしですぐに適用されます。モデルとスウォームは次のステップから適用されます（すでに実行中のステップは開始時の設定を維持します）。ただし Kiro CLI では、実行中のセッションは開始時の設定を維持するため、モデルや effort の変更は次の Kiro CLI セッションから適用されます。
既定のスコープは新しい作業にのみ適用され、保存したモデルのプロファイルは `--from` で読み込むまで何も変えません。runtime、providers、trust の回答ではワークフローの行は表示されません。また、コマンドを実行する環境でその変数が設定されている間に、他のどの設定ファイルにも記録されていないバイパスを記録またはクリアするコマンドも、他に何を変更するかにかかわらず、ワークフローの行を表示しません。開いている作業は、それが動いているエディターや CLI の環境を読み取りますが、コマンドからはその環境が見えないためです。

リリースファイルを取り込む更新（素の `aidlc config`、`--from`、`--download`、`config project`、またはピン留めされたプロジェクトが最初に必要とするアップデート）も実行され、その旨が伝えられます。

```
  Updated. Your open work (default/add-login) carries on.
```

新しいリリースが、すでに完了したステージを異なる方法で行うようになっていても、あなたの側で何も変わっていなければ、作業はそのまま有効で、やり直しを求められることはありません。そのステージが以前は不要だったドキュメントを必要とするようになった場合は、次のステップで一度だけ伝えられます。例: "AI-DLC now does Functional Design differently from when yours ran; your
work stands, and `/aidlc --stage functional-design` goes back to it (the stages
after it reopen too)."（AI-DLC は、あなたの実行時とは異なる方法で Functional Design を行うようになりました。作業はそのまま有効で、上記のコマンドでそのステージに戻れます。その後のステージも再び開かれます。）それ以外の場合は伝えられず、`/aidlc --status` にその行が表示されます。あなた自身の作業に対する変更では、通常どおりの質問が行われます。

ファイルが別のリリースから来た場合は、次の行で戻し方が示されます。

```
  To go back: `aidlc config --pin 2.9.0 --yes` (this pins the version for everyone on the project; `aidlc config --unpin` removes the pin).
```

すでにピン留めされているプロジェクトでは、`--pin` の部分だけが示されます。コピーしたプロジェクトでは、代わりに以前のリリースのファイルが示されます。`aidlc-copy-runtime-2.9.0.tar.gz` とその `.sha256` を 1 つのフォルダーに入れてから、`bun .claude/tools/aidlc.ts config --from <that file> --yes` を実行してください。開いている作業と並べて追加したハーネスは、指定したファイルから取り込まれ、その行では追加したフォルダーが示されます（`Added .codex. Your open work (...) carries on.`）。ハーネスを削除するコマンドはないため、元に戻す行はありません。

更新では次のものを保持します。

- 同梱のプロジェクションに含まれない、すべてのワークスペースの記録、監査ログのシャード、ナレッジ、その他のプロジェクトファイル
- 既存の `aidlc/active-space` とスペースのメモリファイル（プロジェクト所有のシード）
- 可変の `tools/data/harness.json` 内の、識別情報以外のすべての兄弟キー（プラグインの選択や将来のポリシー記録を含む）
- プラグインが合成したファイルと記録されたステージへの寄与。そのうえで、グラフ、ランナー、スコープ、コンパイル済みの表を再生成する
- 上流で作成されたオーケストレーターの散文。保持されたプロジェクトの構成から、コンパイル済みのステージとスコープの領域を再構築する
- AI-DLC が管理するフォルダー内に、ホストのツールが自分用にインストールするもの。たとえば opencode が初回起動時に `.opencode/` 配下に書き込む `package.json`、`.gitignore`、`node_modules/`（リンクを含む）。config がこれらをコピー、所有、削除することはない
- Kiro CLI が 3.0 エンジン用にアップグレードした `.kiro/agents` 内のエージェントファイル（Kiro は置き換えた各ファイルをその隣に `.json.bak`、次に `.json.bak.1` というように残します）。更新ではそれらを Kiro が残したまま保持し、その旨を 1 行で伝え、リリースの残りを取り込みます。`aidlc config --harness kiro-v2
  --force` を使うとリリースのバージョンが採用され、Kiro は次に 3.0 エンジンで起動したときに再びそれらをアップグレードします。リリースがそのようなファイルの許可内容を変更する場合（Kiro のアップグレード元のファイルにあった allow エントリを削除するか、deny エントリを追加する場合）、更新は次の条件をすべて満たすときに限り、リリースのバージョンを自動的に元に戻します。Kiro が次の 3.0 起動時に自動的に再アップグレードすること（自動アップグレードが有効、つまり自分の Kiro 設定で `chat.enableAutoAgentUpgrade` が true であること。Kiro の初回 3.0 起動時の **Enable auto-upgrade** で保存されます）、そのファイルが AI-DLC がそこに書き込んだものであること、Kiro が書き込んで以降誰も Kiro のコピーを変更していないこと（Kiro がファイルに書き込む `kiroUpgrade` スタンプがまだ一致していること。Kiro がスタンプしていないコピーの場合は、`tools`、`hooks`、`permissions` 以外のすべてのフィールドがまだファイル本来のものであること）です。それ以外の場合は Kiro のコピーを保持し、そのファイルが古いリリースの許可内容をまだ許可していること、および `--force` でリリースのバージョンを採用できることを 1 行で伝えます。リリースがそのようなファイルを同梱しなくなり、Kiro のアップグレード元のファイルが AI-DLC がそこに書き込んだものでスタンプもまだ一致している場合、更新は、リリースが同梱しなくなった他のファイルと同様にそのファイルを削除し、Kiro によるバックアップは残し、その旨を 1 行で伝えます

`aidlc/` 配下では、インストールと更新はそれらのシードだけをコピーします。クローンの識別情報、セッション、エンジンの健全性、その他のマシンごとの状態は、インストール済みのランタイムからコピーされることも、インストールのベースラインに記録されることもありません。

ローカルで変更されたフレームワーク所有のファイルは、以前のベースラインと照らして競合となります。`--force` は、手書きのオーケストレーターの散文へのローカルな編集も含め、それらのファイルを更新後の候補で置き換えます。無関係なプロジェクトの内容の所有を主張することはありません。改行コードの違いだけではローカルな変更とはみなしません。Git はチェックアウト時に改行コードを書き換える（Git for Windows は既定で LF のファイルを CRLF としてチェックアウトする）ため、設定済みプロジェクトのクローン、ブランチの切り替え、stash pop、新しい worktree は競合なしで更新されます。`aidlc doctor` は、更新が拒否するファイルを `AI-DLC files` の行で示します。

1 行だけ例外があります。Claude Code の `.claude/CLAUDE.md` と、Copilot の `AGENTS.md` のうち AI-DLC の部分は、`# AI-DLC` というタイトルで始まります（以前のリリースではそこに `# Project Name <!-- Replace with your project name -->` を同梱していました）。変更がタイトルの位置に書かれた行だけの場合、更新はその行を残して残りを更新し、競合にはなりません。それ以外の変更は上記のとおり競合になります。プロジェクトの名前、メモ、ルールは、すべてのツールが読み取る `aidlc/spaces/<space>/memory/project.md` に書いてください。

新しいリリースが同梱しなくなった、変更されていないフレームワークのファイルは削除され、config はそれぞれの名前を示します。更新では `Removed N files that are no longer
part of AI-DLC <version>:` とその一覧を表示し、`--dry-run` は同じ一覧を `Will remove` として表示します。1 つのフォルダー内の複数のファイルは 1 行で表示されます。削除されたファイルをすべて git が追跡している場合、config はファイルを 1 つ戻すための `git restore <path>` も示します。その後の更新では、戻したファイルには触れません。JSON では、これらのアクションに `detail: "no longer shipped"` が付きます。

`.claude/settings.json` はプロジェクトに属します。AI-DLC はファイル全体を所有するのではなく、エントリを提供します。リリースによる更新（プロバイダー、スコープ、モデルの回答に伴うものを含む）は、所有権の競合なしにそれらのエントリをマージし、`--force` でもこれは変わりません。

Claude では、更新は AI-DLC のフック登録を同梱どおりのイベント、マッチャー、コマンドに戻し、その後にあなた自身のフックグループを追加します。同梱の `permissions.allow` エントリを先頭に置き、あなたが追加した allow エントリ、`deny`、`ask`、その他の許可関連のキーを保持します。廃止された同梱の allow エントリが自動的に削除されることはありません。チームがそこに設定したステータスラインは保持されます。AI-DLC 自身の `statusLine` はこのファイルに同梱されなくなりました。更新ではそれを取り除き、それを持っていた各クローンでは、その人自身の `.claude/settings.local.json` に保持します（その人自身のステータスラインを覆う場合は一度だけ伝えます）。
それ以外の場合は、誰かが `--statusline on` で有効にしたときにだけ表示されます。
`companyAnnouncements` は、存在しない場合か、同梱のベースラインと一致したままの場合に更新され、それ以外の場合はあなたの値が保持されます。どちらかのカスタムキーを削除して更新すると、同梱の値が採用されます。環境やその他のトップレベルの設定（`disableAllHooks` を含む）は、記録されたプロバイダーやプロジェクトの回答に由来する値を除き、あなたのものとして残ります。

`.codex/config.toml` もプロジェクトに属します。AI-DLC は設定をキー単位で提供します。`developer_instructions`、`sandbox_mode`、`suppress_unstable_features_warning`、`tool_output_token_limit`、および `[shell_environment_policy]`、`[sandbox_workspace_write]`、`[agents]`、`[features]`、`[tools]` の各テーブルで同梱しているキーです。AI-DLC の `[tui]` の `status_line` はプロジェクトの選択（`--statusline on|off`）であり、更新でそれを同梱、廃止、変更することはありません。プロバイダー、スコープ、モデルの回答に伴うものを含め、更新はそれらのキーだけを変更し、他のバイトはすべて保持します。保持されるのは、あなた自身のキー（AI-DLC のテーブル内のものも含む）、あなた自身のテーブル（`[agents.<role>]` や `[mcp_servers.<name>]` など）、コメント、順序、表記です。誰も変更していない AI-DLC の値はリリースの値を採用します。あなたが変更した値はあなたのものとして残り、リリースが異なる値を同梱している場合は注記が付きます（キーを削除して更新すると採用されます）。削除された AI-DLC のキーは注記付きで復元されます。以前のリリースが同梱していた `AIDLC_RULES_DIR` の値は、切り替えによってどのスペースに残っていても空にされ、そのテーブル内のあなた自身のキーは残ります。そこにあるあなた自身のフォルダーもあなたのものとして残ります。すでに独自の `.codex/config.toml` を持っているプロジェクトでは、初回インストール時にそれを保持し、AI-DLC は自分の設定を追加します。更新で安全にマージできないファイル（解析できない、または AI-DLC のテーブル名をテーブルの配列など別の用途に使っている）は引き続き競合として報告されます。その場合、`--force` は無関係なプロジェクト所有のフィールドを保持しつつ AI-DLC のテーブルを復元します。明示的な `--from` を指定すると、プロジェクトのコピーではなくそのソースが選ばれます。

`.claude/settings.json` や `.codex/config.toml` の AI-DLC のエントリが復元または追加された場合、また、このリリースが異なる値を同梱しているのにそのいずれかに対するあなた自身の値（カスタムの Claude のアナウンス、または Codex のキー）が保持された場合、人間向けの出力には `Note:` が表示されます。JSON 出力では同じメッセージが `data.notes` に含まれます。これらを復元するには、引数なしの対話型セットアップではなく、`aidlc config --harness claude` または `aidlc config --harness codex` を使ってください。
コピーチャネルのプロジェクトでは、さらに `--from <the runtime/<harness> root you copied
from>` も渡します。

`opencode.json` はチームに属します。config は AI-DLC のエントリをそこに追加し、他のキー、値、コメント、行はすべて保持します。プロバイダーの回答で編集されるのは、AI-DLC が書き込んだエントリだけです。

### Root Integrations and Ownership

| 設定面 | ハーネス | ポリシー |
|---------|-----------|--------|
| `.gitignore` | すべて | インストールされたハーネスの同梱エントリを合わせたものを含む、マーク付きの AI-DLC ブロックを 1 つ所有する。その外側のバイトはすべて保持する |
| `.mcp.json` / `mcpServers` | Claude | 同意済みでベースラインが所有するエントリだけを追加・削除する。ユーザーのキーと上書きは保持する |
| `.kiro/settings/mcp.json` / `mcpServers` | Kiro CLI | `.mcp.json` と同様にサーバー単位で扱う。変更していない AI-DLC のサーバーだけリリースに追従し、追加したサーバーや有効にした同梱サーバーは保持する。セットアップ時にプロジェクトがすでに持っていたファイルは拒否せずにマージする |
| `AGENTS.md` | Kiro CLI, Kiro IDE, Codex, Cursor, OpenCode, Copilot | マーク付きのブロックを 1 つ。ハーネスに依存せず共有される（`shared: "identical"`）。ただし Copilot は例外で、そのブロックには `@` によるインポートが含まれる。プロジェクトの指示は保持する |
| `opencode.json` | OpenCode | `json-entries`: AI-DLC のエントリ（`$schema`、その `skills.paths` と `instructions` の文字列、その `permission` ルール）を存在しない場合にだけ追加する。チーム自身のコマンドに対するルール（`rm -r...` と `git push` の前の確認）はマップの先頭に置き、その後のチームのルールが引き続き判定できるようにする。チームのモデル、プロバイダー、独自の指示とルール、コメント、レイアウトは保持する。AI-DLC が書き込んだものを記録し、まだその値を保っているエントリだけを追従または廃止する。以前のリリースで AI-DLC がファイル全体を書き込んだものは引き継ぐ。コピーランタイムにはこのファイルは含まれず、そのセットアップ（またはセットアップが実行されなかった場合は最初のセッション）で AI-DLC の部分を追加する |
| `.vscode/settings.json` | Copilot | `jsonc-settings`: プロジェクトが `chat.agent.maxRequests` を設定していない場合にだけ（200 で）追加する。他者が設定した値、他のキー、コメントは決して変更しない。AI-DLC が追加したものだけを記録し、廃止時には AI-DLC が書き込んだ値を保っている間だけ削除する。一度追加した後、チームが保持しているファイルからそのキーを取り除いた場合は再追加しない。コピーランタイムにはこのファイルは含まれない |

**1 つのプロジェクトに複数のハーネスを置く場合。** ハーネスは、エンジンのディレクトリが異なり、排他的な管理ブロックを共有しない場合に共存できます。`AGENTS.md` は Kiro CLI、Kiro IDE、Codex、Cursor、OpenCode の間で中立かつバイト単位で同一（`shared: "identical"`）なので、エンジンのディレクトリが異なるものどうしであれば共存できます。Codex 固有のオンボーディングは、プロジェクトが信頼されている場合にプロジェクトの `.codex/config.toml` の `developer_instructions` を通じて注入され、`.codex/onboarding.md` がその読める写しです。
Claude Code は他のどのハーネスとも共存できます。Copilot の `AGENTS.md` は排他的なままです。そのブロックを同梱する別のハーネスと組み合わせると、どちらを先にインストールしたかにかかわらず `cannot coexist in one project` で拒否されます。
Kiro CLI と Kiro IDE は引き続き `.kiro/` を共有し、OpenCode と Copilot は `.aidlc/` を共有するため、これらの組み合わせは共存できません。Kiro CLI と Kiro IDE は代わりに互いを置き換えることができます。どちらか一方があるプロジェクトで `aidlc config --harness kiro`（または `--harness kiro-v2`）を実行すると、`.kiro/` がその場でもう一方に切り替わります。素の `aidlc config` による更新では、プロジェクトにある方の Kiro ハーネスが維持されます。Kiro CLI (V2) のプロジェクトでは、`--harness kiro` で Kiro に切り替わります（ターミナルでは先に確認を求めます。`--yes` を付けると一度だけその旨を伝えて切り替えます。ターミナルがない場合や `--quiet` または `--json` の場合は何も変更せず、`--yes` 付きの切り替えと `--harness kiro-v2` による更新の両方のコマンドを表示します。ドライランの `--plan-token` は計画どおりに切り替えを適用します）。そのため、Kiro CLI (V2) のプロジェクトを更新するスクリプトでは `--harness kiro-v2` を渡します。AI-DLC 2.11.3 以前、および 2026-11-04 と 2026-11-05 の 2.11.4 プレビューでは、2 つの名前を逆に読みます（Kiro CLI (V2) が `--harness kiro`、Kiro が `--harness kiro-ide` で、`kiro-v2` は未知の名前）。`--harness kiro-ide` は、それらのリリースでも本リリースでも Kiro を指します。そのうち 2.11.1 以降のリリースで、コピーしたプロジェクト自身のツールが更新をより新しいファイル自身の config に引き渡す場合（[コピーチャネル](#copy-channel)）、`--harness kiro` は Kiro CLI (V2) として届きます。そのため Kiro のプロジェクトでは、切り替えはそのツールの名前で同じように確認されます。ターミナルでは先に確認を求めます。`--yes` を付けると一度だけその旨を伝えて切り替えます。ターミナルがない場合や `--quiet` または `--json` の場合は何も変更せず、実行したツール向けに、`--yes` 付きの切り替えと `--harness kiro-ide` による更新の両方のコマンドを表示します。ドライランの `--plan-token` は計画どおりに切り替えを適用します。2.11.0 以前のコピーしたツールは更新を引き渡しません。`--harness kiro` を自分で Kiro CLI (V2) として読み、こうした確認は行いません。
切り替えは、インストールされている行の所有権ベースラインから計画される更新です。その行だけが同梱していたファイルを削除し、`aidlc/` は保持し、置き換えまたは削除することになるローカルで変更されたファイルは競合として報告し、他の更新と同様に作業が開いている間でも実行されます。
Kiro CLI が 3.0 エンジン用にアップグレードした `.kiro/agents` 内のエージェントファイルは、Kiro のアップグレード元のファイルが AI-DLC が書き込んだもので、その後誰も Kiro のコピーを変更していない（`kiroUpgrade` スタンプがまだ一致している）場合、ローカルな変更とはみなされません。Kiro への切り替えではそれを他のファイルと一緒に削除し、Kiro の `.json.bak` バックアップは残し、その旨を 1 行で伝えます。Kiro がスタンプしていないコピーは競合のままです。
Kiro への切り替えでは、AI-DLC が所有していない `.kiro/hooks/*.json` ファイルをすべて示します。Kiro はそれらを v3 エンジン（`kiro-cli --v3`）と Kiro IDE で実行します。そのようなファイルがある場合、切り替えはそのファイル一式をあなたが承認した場合にのみ適用されます。ターミナルでプロンプトに回答するか、`--dry-run` で切り替えを実行してファイルを確認し、そのドライランが表示する `--plan-token` で適用してください。確認後に AI-DLC が所有していないフックファイルが追加、削除、名前変更、変更されると、切り替えは停止します。切り替えがトランザクションのロックを取得する前や、ファイルをコミットしている最中に現れたものも含まれ、その場合は切り替えをロールバックします。削除されたプラグイン自身の登録（切り替えはこれを認識し、名前を示しません）が、そのプラグインのものとして認識されなくなる変更でも同様です（削除によって Kiro が実行するものが減る場合は切り替えを止めません）。`.kiro/hooks` がリンクやファイルである場合（その配下を読む前に判定）、または AI-DLC が所有していないフックのエントリがリンクや通常ファイル以外のものである場合、切り替えはそれをたどらずに拒否します。切り替えにはそのベースライン（`.kiro/tools/data/aidlc-manifest.json`）が必要です。ベースラインがない場合は、まず `aidlc config --harness <installed>` でインストールされている行を更新してください。AI-DLC が同梱するファイルだけをそこに記録するようになる前に記録されたベースラインも同様で、更新によって最新の状態になります。切り替えが壊れたベースラインを変更することはありません。表示された手順にフルパスが示されるので、自分でそれを別の場所に移してから、その更新を実行してください。信頼の確認をクリアする切り替えでは、それを再び記録するための `config trust --harness <row>` コマンドが示されます。`--from` で渡したリリースに `--harness` がない場合、行が切り替わることはありません。OpenCode と Copilot はこの方法では切り替えられません。
ルートのブロックが共有されていない古いインストール済みハーネスについては、`predates shared onboarding` エラーが、まず `aidlc config --harness <name>` で更新するよう提案します。これは古い兄弟ハーネス向けのヒントであり、更新すれば共存できるという約束ではありません。更新後も拒否される場合、その兄弟のブロックは排他的です。Copilot のブロックは更新後も排他的なままです。
`AGENTS.md` を共有と宣言しなくなった更新元は、別のインストール済みハーネスがそれを共有している間は拒否されます: `refusing to refresh <harness> from a release whose AGENTS.md is not shared`。
ブロックを共有と宣言しているリリースを使ってください。`--force` でもこのガードは回避できません。
別のリリースの兄弟ハーネスが所有する共有の `AGENTS.md` ブロックは、延期された更新ではなく競合です。エラーは更新の順序を示します。選択したハーネスを兄弟と同じリリースから更新するか、先に兄弟を選択したリリースから更新してください。
Cursor の手動コピー用インストーラーは単一ハーネス用です（専用の `AIDLC CURSOR` マーカーを使い、`aidlc config` の所有権ベースラインはありません）。複数ハーネスのプロジェクトでは、代わりに `aidlc config --harness cursor` で Cursor を追加する必要があります。

`.gitignore` は `shared: "union"` を宣言しているため、`aidlc config` はインストールされたすべてのハーネスの同梱エントリを組み合わせたブロックを 1 つ書き込みます。追加のエントリは `# <harness> harness` の下に表示されます。
ハーネスを追加すると、兄弟ハーネスの同梱ブロックの写しが利用できる場合、その兄弟が所有する未変更のブロックと結合されます（`merge (combined with <harness>)`）。その写しを持たない古いインストールでは、更新されるまで所有権を保持します。各ハーネスは、次に config を呼び出したときに同じ結合済みブロックのハッシュを記録します。
複数のハーネスがある場合、すべての `aidlc config` の呼び出しで `--harness <name>` が必要です。ただし、バイパスだけを記録またはクリアする場合は例外です。

`aidlc doctor` は、各ハーネスツリーが `tools/data/aidlc-stamp.json` に記録しているリリースを比較します。異なる場合は `Harness trees on
different releases` と警告し、各ツリーのリリースを示し、他のツリーをそろえるためのコマンドを示します。ピン留めされたプロジェクトでは、config がすべてのツリーをピンに合わせて更新するのと同様に、ツリーをピンにそろえます。コピーしたプロジェクトでは、それは `bun <harness-dir>/tools/aidlc.ts config --harness <name> --download` です。ピンがない場合、ネイティブではエンジンのリリースにないツリーごとに `aidlc config --harness <name>` を実行します。コピーしたプロジェクトでは各ツリーが自分のリリースを実行するため、他のツリーは最も新しいツリーのリリースから、その `aidlc-copy-runtime-<version>.tar.gz` を `--from` で渡して更新します。まだどの config の実行も記録していないツリーは、まず自分のリリースで `--download` による更新を 1 回行います。開いている作業は更新をまたいで続くため、完了を待つ必要はありません。

AI-DLC の `.gitignore` の行は、AI-DLC 自身のエントリだけです。以前のリリースでは、そのブロックの先頭に汎用のテンプレート（ログ、`node_modules`、`dist`、エディターのファイル）も置いていました。アップグレード後の最初の更新では、それらの行を AI-DLC の部分より上の、あなたの部分に残し、一度だけその旨を伝えます。そのため、それらが無視していたものが git から見えるようになることはありません。config が一度も実行されていないコピーでは、最初のチャットの開始時または最初に作業を作成したときに、コピー自身の `tools/data/root-blocks/` から同じ AI-DLC ブロックと `AGENTS.md` のブロックが追加されます。その後 config は、リリースが同梱したものと完全に一致するブロックを自分のものとして扱います。`opencode.json` 内の opencode のエントリも、同じタイミングで同じフォルダーから追加されます。

過去に同梱したプロジェクションに含まれていた既知のマークなしファイルや JSON のエントリは、記録されている SHA-256 の署名と完全に一致する場合にのみ引き継がれます。それ以外のマークなしの内容は、AI-DLC について何が書かれていてもチームのものです。config はそれを先頭部分として保持し、新しい管理ブロックを後ろに追加します。名前の変更や削除は不要です。有効な UTF-8 でないルートファイル（`.gitignore`、`AGENTS.md`、その他 AI-DLC がマージするもの）は手を付けずに残され、config が安全にマージできるようにするには文字コードの変換が必要です。

Git リポジトリ内では、config は `--dry-run` の間も含め、ユーザーが所有するルールがコミット済みのワークフローの記録を隠していないかも確認します。`aidlc/` のようなルールは記録を隠します。config はそれでも完了しますが、新しい記録がチームメイトに届かなくなるため、最後にそのルールのファイル、行、隠れている記録のパス（`memory/**`、`codekb/**`、`intents.json`、`aidlc-state.md`、`audit/*.md`）を示す注記を出します。git がすでに追跡しているファイルは引き続きコミットされます。初回セットアップと `--quiet` の出力でも同じ指摘が表示されます。ルールはあなたのものなので、config がそれを書き換えたり拒否したりすることはありません。意図した隠し方でなければ、ルールの範囲を狭めてください。Git が利用できない場合やプロジェクトが Git リポジトリでない場合、このチェックはスキップされます。

`/aidlc --doctor` は、コミットされていない記録のチェックと並んでこのチェックを繰り返します。その **Workspace record visibility** の助言は、config の成功後に ignore ルールが追加された場合に、同じルールと隠れているパスを示します。この警告は doctor の終了コードを変えず、記録が隠れていない場合や Git がプロジェクトを確認できない場合は表示されません。

`--force` は、変更された、ベースラインが所有する管理ブロックや管理対象のハーネスファイルを置き換えることができます。`opencode.json` 内のチーム自身のエントリを含め、ユーザーが所有する JSON の値を上書きすることはできません。不正な JSON、不正または重複したマーカー、通常ファイルでない対象、整合性を証明できない廃止済みの所有コンテンツは、確定的な競合です。

計画されたすべてのパスには、アクションが 1 つ割り当てられます。

| アクション | 意味 |
|--------|---------|
| `create` | 存在しないフレームワークのパスを追加する |
| `update` | フレームワーク所有のバイト列を更新する |
| `merge` | 管理ブロック、JSON のマップ、JSON の配列を調整する |
| `preserve` | 現在のバイト列やプロジェクト所有のバイト列を保持する |
| `remove` | 以前ベースラインが所有し、上流で廃止された内容を削除する |
| `conflict` | 所有権や整合性を証明できないため拒否する |

config が成功すると、ホストごとの次のステップが表示されます。

| ハーネス | 次のステップ |
|---------|-----------|
| Claude Code | このプロジェクトで Claude Code を開き（このフォルダーですでに開いている場合は終了して起動し直す）、`/aidlc --doctor` を実行する |
| Kiro CLI | `kiro-cli --v2` を実行し（Kiro CLI 2.21.4 以降。それより古い場合は [Kiro CLI (V2) の前提条件](harnesses/kiro.md#kiro-cli-v2-prerequisites)を参照）、続いて `/aidlc --doctor` を実行する |
| Kiro IDE | このプロジェクトを Kiro IDE で開く。ウィンドウ上部に Restricted Mode のバナーが表示され、このフォルダーの中身を把握している場合は、バナーの Manage を選び、続いて Trust を選ぶ。コマンドパレット（Ctrl+Shift+P、macOS では Cmd+Shift+P）から `Developer: Reload Window` を実行し、チャットパネルのエージェントピッカーで aidlc エージェントを選んでから `/aidlc --doctor` を実行する（Kiro CLI では、代わりにプロジェクトで `kiro-cli --v3` を起動する。このワークスペースを信頼するかを Kiro が尋ね、このフォルダーの中身を把握している場合は "Trust, continue" を選ぶ。その後 `/aidlc --doctor` を実行する） |
| Codex CLI | `codex` を実行し（フックについて尋ねられたら Trust all and continue を選ぶ）、続いて `$aidlc --doctor` を実行する |
| OpenCode | `opencode` を実行し、続いて `/aidlc --doctor` を実行する |

Kiro の行（Kiro IDE と v3 上の Kiro CLI）では、Kiro CLI 2.24.1 以降が PATH にあり、config を Kiro IDE の外（そのターミナル以外）で実行した場合、次のステップでは Kiro CLI が先に示されます。このプロジェクトで `kiro-cli --v3` を起動してから `/aidlc --doctor` を実行し、その後に Kiro IDE の手順が続きます。

## Update and Version Selection

| Command | Public options and behavior |
|---------|-----------------------------|
| `aidlc update` | Install the newest release of the machine's channel with the complete all-harness runtime, then atomically activate. Accepts `--version <version>`, `--channel <stable\|preview>`, `--from <release-dir>`, `--release-base-url <url>`, `--release-api-url <url>`, `--ca-bundle <path>`, `--offline`, and `--dry-run`. |
| `aidlc update --check` | Refresh update metadata for the channel without installing. Returns 5 when the channel has a release newer than the running one, 0 when current (a running release newer than the channel's newest is current), 3 when unavailable/offline, and 1 when checks are disabled. |
| `aidlc use <version>` | Install the exact stable or preview version when it is not retained, then make it machine-active without changing project files. |
| `aidlc version` | Print the machine-active release as `aidlc X.Y.Z (runtime X.Y.Z)`. It runs on that release even in a pinned project, and there, run at a terminal, it also names the pin on stderr (`aidlc: this project is pinned to aidlc 2.10.0, which runs its workflow (machine active: 2.9.0)`); redirected or captured, it prints only that line. `--json` adds `pinnedVersion`. The stdout line is the same in every folder, because an update checks the release it activates against it. |
| `aidlc config --channel [stable\|preview]` | Set the machine release channel, or print it when no value is given. Set from a release of the other channel, it says what the next `aidlc update` does: `aidlc update moves to <version> now` when that channel has a newer release, otherwise that it waits for one and how to go now. |
| `aidlc config --pin <version>` | Install and validate the exact version when needed, then atomically write `.aidlc-version`, record its machine-local resolved target, and register the project pin without changing the machine-active pointer. |
| `aidlc config --unpin` | Remove `.aidlc-version`, its machine-local resolved target, and its registry entry. |
| `aidlc config --statusline <on\|off>` | Show AI-DLC's progress in the status line, or take it out, and say so in one line. Claude Code: your own `.claude/settings.local.json`. Codex: the shared `.codex/config.toml`, for everyone in the project. `--harness <claude\|codex>` picks one when both are installed. |
| `aidlc config ... --download` | Let any config command fetch the release the project needs when this machine lacks it: the pinned release, otherwise the one its files already have. Natively it installs and registers that release, as `--pin` does; on a copied project it downloads `aidlc-copy-runtime-X.Y.Z.tar.gz`. It verifies the checksum, and the release attestation when `gh` is installed, then finishes the command. `--release-base-url` and `--ca-bundle` choose a mirror. At a terminal config asks instead; scripts need the flag. |

Human lifecycle output states each completed fact. Update reports the
old-to-new version check, verified download, atomic switch, retained prior
version, any pruned unprotected releases, and the project-refresh courtesy.
A no-op says `You're on the latest version of aidlc (<version>).`; `--dry-run`
says `Would update aidlc from <old> to <new>.`, or
`You're on the latest version of aidlc (<version>); nothing to update.` when
nothing would change. On the preview channel the update lines say
`preview releases` and `latest preview version`. A plain `aidlc update` never
installs a release older than the one running: on a machine that follows stable
and runs a newer preview, it changes nothing and says `You're on <preview>, newer
than the latest stable <x.y.z>, so there's nothing to update.`, then how to go
back to stable (`aidlc update --channel stable`) and how to keep getting
previews (`aidlc config --channel preview`); once a newer stable ships, it moves
to it. An update that moves onto the other channel for one run (`--channel`,
`--version` or `--from`) says that the machine follows its channel, with the
same two ways on. `aidlc use`
distinguishes `Now using` from `Already using`, and uninstall states exactly
which machine state was removed or kept. JSON and quiet messages retain their
stable machine contracts; update JSON carries `channel` and, on a switch,
`channelSwitch`.

Update downloads and fully validates a candidate before changing the active
pointer. Failed updates automatically restore the prior consistent
installation. A successful update retains the prior active version and every
registered project pin, then prunes older unprotected versions automatically.
There is no separate rollback or retained-version management command; to go
back, see [Go Back to an Earlier Version](#go-back-to-an-earlier-version).

## Go Back to an Earlier Version

To put this machine back on an earlier release, run `aidlc use <version>`, for
example `aidlc use 2.10.0`. It downloads that release if the machine does not
have it, then says `Now using aidlc 2.10.0 (was 2.11.0; retained locally, no
project changes).` Every project without a pin now runs on 2.10.0; run
`aidlc config` in a project to bring its files to 2.10.0 as well.
`aidlc use 2.11.0` goes forward again.

To put one project back for everyone on it, pin it, then refresh it:

```bash
aidlc config --pin 2.10.0 --yes
aidlc config --yes
git add .aidlc-version
```

The pin switches the project to 2.10.0; the refresh brings its files to that
release and lists any file 2.10.0 does not ship. `aidlc config --unpin` makes
the project follow the machine again ([Project Pins and CI](#project-pins-and-ci)).

Releases before 2.8.0 have no native release, so `aidlc use` and
`aidlc config --pin` cannot select them; they stop with `No published native
release is available yet`. A copied project goes back with the earlier
release's `aidlc-copy-runtime-X.Y.Z.tar.gz` (published from 2.9.0), as
[Refresh Safety](#refresh-safety) shows.

## Release Channels

`main` is the shared development branch. The **stable** channel publishes a
selected commit as a GitHub release tagged `vX.Y.Z`. The **preview** channel
lets users try changes from `main` before the next stable release as a GitHub
prerelease that is never marked "latest". Source versions and changelog entries
are updated during release preparation.

Scheduled and manual runs share one serialized publication queue. A run skips
publication when the source is unchanged since the latest published preview, or
when a newer preview already contains it (an older run, retried or queued); its
checks and nightly tests still run, and no preview is published. When `main`
advances more than once on the same UTC date, each changed source can publish a
new preview with the next build counter.

A preview still publishes when its nightly tests fail. Its release notes then
open with a warning and end with a Full Suite failure report (failed jobs and
any failing tests), so check it before relying on a preview.

A preview id is `<x.y.(z+1)>-preview.<YYYYMMDD>.<N>`: the next patch after the
source tree's current stable version, the UTC build date chosen during
planning, and a retry counter (`1` initially). The workflow calculates the
preview version without editing the source version. Drafts and tags left by
failed attempts reserve ids. A retry or another changed source on the same date
advances `N` past those occupied ids.

Stable ids stay exactly `x.y.z`, and nothing else is accepted anywhere a
version appears (installer flags, `use`, pins, `.aidlc-version`, retained
version directories). Ids order numerically on `x.y.z`; at an equal base the
stable release sorts above every preview built from it, and previews order by
build date then counter. Inside a preview artifact `aidlc version`,
`version.json`, and doctor bundles all report the preview id; the source tree is
never modified.

```bash
aidlc config --channel preview   # follow the preview stream
aidlc update                     # newest published preview
aidlc update --check             # 5 when a newer preview exists
aidlc config --channel stable    # back to the stable stream
aidlc update --channel stable    # newest stable now
```

The channel is machine-local: `aidlc config --channel` writes a `channel`
marker beside the update cache and `pins.json` under the install root
(`stable` when the marker is absent), `aidlc uninstall` preserves it with the
other machine settings, and `--purge` removes it. `aidlc update --channel <c>`
overrides the marker for one run; `--version` and `--from` select an exact
release regardless of channel. Stable discovery is unchanged (the
`latest/download` redirect). Preview discovery lists the releases of the
repository behind the release base URL through the GitHub API, keeps the
newest published prerelease by preview version id, and installs it through
the exact-version path. Drafts and tags without a published release are ignored.
For `github.com` base URLs the API endpoint is derived; for any other host set
`--release-api-url <url>` or `AIDLC_RELEASE_API_URL`.
An API failure, a rate limit, or a repository with no published preview is
reported as unavailable (exit 3); the client never falls back to the stable
release. The update cache records the channel it was refreshed for, so a cached
preview result never answers a stable check or the reverse.

Switching back is `aidlc config --channel stable`. A plain `aidlc update` then
waits for a stable release newer than the preview you are running; to go back
now, run `aidlc update --channel stable`, which installs the newest stable even
when its id sorts below the preview. Preview retention is a bounded window on top
of the protection every release has (active, rollback, in use, pinned): after
an update the two newest complete previews stay, and every older preview
without its own protection is pruned; stable retention is unchanged.

Version pruning also uses recorded file lists and empty-directory cleanup.
If a selected version contains unowned or changed paths, pruning is refused
and the files are kept for review.

Project pins keep overriding the machine channel: `aidlc config --pin <id>` and
`.aidlc-version` accept preview ids, and a pinned project dispatches to that
exact retained version whatever the machine follows.

Previews are `main` as it stands, including project state-schema changes. A
preview that raises the state schema writes project state a stable build does
not understand, and the code refuses to open state newer than the build; that
project cannot be walked back to stable until a stable release ships the same
schema. Use the preview channel on projects you can recreate, or pin them.

## Project Pins and CI

```bash
aidlc config --pin 2.10.0
git add .aidlc-version
```

`aidlc config --pin <version>` installs and validates the version if needed,
writes `.aidlc-version`, records the absolute binary target under the
gitignored `aidlc/.aidlc-sessions/` runtime directory, and registers the real
project path in machine-local `pins.json`.
Registry reads canonicalize filesystem aliases (including macOS `/var` and
`/private/var`) and JSON output reports the canonical project path. Equivalent
keys with the same version collapse; conflicting equivalents fail closed until
`config --pin` or `config --unpin` reconciles every alias for that project.

Commit only `.aidlc-version`. The stable `aidlc` launcher starts the
integrity-checked active binary, whose dispatcher validates the complete pinned
binary and runtime before selecting it. A missing, malformed, tampered, or
unavailable target fails closed with `aidlc config --pin <version>` remediation, and
`aidlc doctor` reports the same condition. Machine lifecycle commands use the
active binary; `doctor`, `config`, and `use` are never trapped behind a broken
pin. When a teammate commits a pin to a release this machine lacks, a config
command on the project names it, and `--download` (or a yes at the terminal)
installs and registers it as `config --pin` would, then finishes the command.
An installed pinned release is used without asking, and files behind it are
updated first.

A pin switches the engine that serves the project at once, but the project's
own files stay at the version they were last refreshed to until the next
`aidlc config`. `config --pin` and `config --unpin` are done as asked, also
while work is open; when the project's files are on another version than the
one it now follows (or on a release from before versions were recorded), the
reply ends with "Run `aidlc config` to finish updating this project."

A fresh clone or CI runner installs the committed version before config:

```bash
version=$(cat .aidlc-version)
tag="v$version"
tmp="$(mktemp -d)"
gh release download "$tag" --repo github.com/awslabs/aidlc-workflows --dir "$tmp" \
  --pattern install.sh --pattern aidlc-release.intoto.jsonl
gh attestation verify "$tmp/install.sh" \
  --bundle "$tmp/aidlc-release.intoto.jsonl" \
  --repo awslabs/aidlc-workflows \
  --signer-workflow awslabs/aidlc-workflows/.github/workflows/release.yml \
  --hostname github.com \
  --source-ref "refs/tags/$tag"
sh "$tmp/install.sh" --version "$version" --quiet --yes
rm -rf "$tmp"
aidlc config --pin "$version" --project-dir "$PWD" --quiet
aidlc config --project-dir "$PWD" --harness claude --mcp none --quiet
aidlc doctor --project-dir "$PWD" --quiet
```

A CI container job or agent sandbox that runs as root, with root as its only
user, adds `--allow-root` to the installer line. The pinned version must be a
release that ships `--allow-root`: the recipe runs that tag's own `install.sh`,
and an older one rejects the flag as an unknown argument.

```bash
sh "$tmp/install.sh" --version "$version" --quiet --yes --allow-root
```

## Harness Selection

Harness selection belongs to `aidlc config --harness <name>`. Machine-level
harness management is not a public command.

## Offline Packages

The release asset set is the offline package. It includes
`aidlc-release.intoto.jsonl` alongside the binaries, runtime, installers,
`version.json`, and `checksums.txt`. Download one complete release on a
connected machine and transfer that directory unchanged. Local installation
fails closed if the bundle is missing or does not authenticate
`checksums.txt`:

```bash
gh release download v2.5.45 --repo github.com/awslabs/aidlc-workflows --dir ./aidlc-offline
```

Install on the disconnected machine:

```bash
bash ./aidlc-offline/install.sh \
  --from ./aidlc-offline --offline
```

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
& .\aidlc-offline\install.ps1 `
  -From .\aidlc-offline -Offline
```

For native commands, `--offline`, `AIDLC_OFFLINE=1`, or global `offline=true`
prevents release sockets. A network operation without `--from` then fails
before mutation. Config, doctor, version, and uninstall are local regardless.

## Mirrors, Proxies, CAs, and Update Settings

Release settings resolve in explicit option, environment, machine-config,
default order:

| Setting | Environment | Machine config |
|---------|-------------|----------------|
| Offline | `AIDLC_OFFLINE=1` (`0` explicitly enables network) | `aidlc system config global set offline on` |
| Mirror | `AIDLC_RELEASE_BASE_URL` | `aidlc system config global set release-base-url <url>` |
| Preview releases API | `AIDLC_RELEASE_API_URL` (or `aidlc update --release-api-url <url>`) | derived from the mirror for `github.com`; not a machine config key |
| CA bundle | `AIDLC_CA_BUNDLE` | `aidlc system config global set ca-bundle <absolute-path>` |

Manage the four machine keys:

```bash
aidlc system config global list
aidlc system config global get update-check
aidlc system config global set update-check off
aidlc system config global set offline on
aidlc system config global set release-base-url https://mirror.example/releases
aidlc system config global set ca-bundle /absolute/path/corporate-ca.pem
aidlc system config global clear ca-bundle
```

The keys are `update-check`, `offline`, `release-base-url`, and `ca-bundle`.
Boolean values accept `true|false`, `on|off`, `1|0`, or `yes|no`.
`aidlc config <get|set|clear|list> ... --global` is equivalent.

Mirror base URLs must use HTTPS, except loopback HTTP for local testing, and
cannot contain credentials, a query, or a fragment. The native lifecycle
client follows at most five redirects; redirected URLs may contain a query but
still cannot contain credentials or a fragment. Its errors redact URL
credentials, queries, and fragments.

The native release client honors `HTTPS_PROXY` / `https_proxy` and
`NO_PROXY` / `no_proxy`; proxy URLs must use HTTP or HTTPS. It does not read
`HTTP_PROXY`. The bootstrap scripts delegate proxy behavior to `curl`,
`wget`, or `Invoke-WebRequest`. On Windows, a custom CA bundle requires
`curl.exe`.

Bare help and management listings never refresh the network. They may display
a valid cached update notice. Interactive human `aidlc doctor`, and
`aidlc doctor --export` with or without a terminal, may refresh stale or
absent metadata within 750 ms. Other non-TTY, `--json`, and `--quiet` doctor
runs are cache-only unless `--check-updates` is explicit. A doctor update row
read from a check older than 24 hours names the version as the latest known
one and says how long ago it was checked. On a machine that has never checked,
a refresh that fails inside that budget leaves the row passing as not checked
yet, naming `aidlc update --check`; only an explicit `--check-updates` reports
the failed check.
`doctor --check-updates` and `update --check` use a five-minute metadata
backstop. Preview discovery and metadata downloads share the same deadline;
looking up the preview release does not restart the timeout.
Update checks accept version identifiers of at most 84 characters, and each numeric component
must be a safe integer (at most 9,007,199,254,740,991). Invalid identifiers in
release metadata or an existing cache are rejected before displaying notices.
The cache expires after 24 hours; a failed refresh or metadata older
than the installed binary in the same channel does not replace a valid cache.
A successful refresh can correct a previously cached future version: the cache
is advisory and does not establish a trusted minimum version.
`update-check=off` disables even explicit refreshes
but does not prevent an explicit `aidlc update`.
Update checks (the doctor refresh and `aidlc update --check`) download
`version.json` and `checksums.txt`, verify the manifest checksum, and neither
download nor verify `aidlc-release.intoto.jsonl`. This checks integrity against
the supplied checksums; it does not authenticate the release's origin.
Every install path (`aidlc update`, `aidlc use`, `aidlc config --pin`, and
`--from`) requires the provenance bundle and verifies checksums before
activation. When a compatible GitHub CLI is available, installation also
verifies the signed provenance and rejects a failed verification.

## Plugins

AI-DLC's own plugins (test-pro and those that follow) share one `aidlc-plugins`
catalogue, which a native install keeps at one path across releases:
`${XDG_DATA_HOME:-$HOME/.local/share}/aidlc/plugins/` on Linux and macOS,
`%LOCALAPPDATA%\aidlc\plugins\` on Windows. The installer prints the full path
after it installs, and `aidlc doctor --verbose` names it under Machine. Add it
once to your tool:

```bash
/plugin marketplace add <path>          # Claude Code
codex plugin marketplace add <path>     # Codex CLI
```

Install, `aidlc update`, `aidlc use <version>` and `aidlc system rollback`
refresh the catalogue from the release they make active, writing it whole and then swapping
it in, so the tool never reads half of one. After an update, the tool's own
marketplace update (`/plugin marketplace update aidlc-plugins` in Claude Code)
reads the new plugin versions from the same path. If the folder cannot be
replaced, the command still succeeds, the folder keeps the earlier catalogue,
and one line says why and what to do: on Windows, when a program may hold a
file in it, close Claude Code and Codex and run `aidlc use <version>`; for any
other cause, such as a full disk, fix that and run it. A file you put in the
folder yourself is never replaced; the line names it instead. A machine that updated to this release
with an earlier release's `aidlc update` gets the folder at its next `aidlc
use <version>` or update. The source checkout's `dist/plugins/` and the copy
runtime's `plugins/` are the same catalogue for the copy channel.

`aidlc doctor` reports installed-versus-composed plugin state. Plugin changes
are project configuration and converge through `aidlc config`; there is no
separate public plugin command.

## Output, Automation, and Exit Codes

The public commands support human, `--quiet`, and `--json` output where
declared by the route registry. `--json` emits a schema-versioned result
with `ok`, `code`, `status`, `message`, and command-specific `data` when
available. `--quiet` emits one success line or remediation line. Download
progress appears only in human mode.

The native diagnostic form is
`aidlc doctor [--project-dir <path>] [--verbose] [--json|--quiet]
[--check-updates] [--release-base-url <url>] [--ca-bundle <path>]
[--offline]`. `--export` writes a redacted diagnostic bundle, with
`--output <directory>` overriding its default project location; export output
is additional to the selected live-report mode. Human output groups Machine,
Project, and Framework integrity checks. Every section keeps warning/failure
rows visible and collapses healthy rows by default; `--verbose` expands every
check. Warnings are advisory and exit 0; any failed check exits 1.

`--no-color` and `NO_COLOR` disable ANSI output. `--project-dir <path>` selects
project context without changing the shell directory. Destructive operations
such as `uninstall` ask nothing on a TTY: they print what they remove and
keep, then do it. Without a TTY they require `--yes`. `--yes` never bypasses
ownership, integrity, active-workflow, or release-authentication refusals, with
one exception on Windows: an `aidlc` in the bin directory that AI-DLC did not
write, such as a hand-made Git Bash forwarder. `aidlc use`, `aidlc update` and
the installer ask once at a terminal whether to replace it; `--yes` answers
yes, and the file is kept beside it as `aidlc.bak-<time>`.

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Operational failure |
| 2 | Usage or invalid machine configuration |
| 3 | Required network result or retained runtime unavailable |
| 4 | Integrity or ownership refusal |
| 5 | Check completed and action is required, such as an available update |

## Help and Completions

`aidlc --help` prints exactly the six public commands. Each public command also
has side-effect-free command help: `aidlc <command> --help` (or `-h`) for
`config`, `doctor`, `version`, `update`, `use`, and `uninstall`. Command help
names every option that command accepts, with a one-line meaning for each,
grouped so download and mirror options and the options that only shape output
read separately from the rest; `use` also describes the version it takes. The
intent is that you can run a command correctly from its help alone. Config-level
help names all six policy sections; `aidlc config <section> --help` keeps the
section-specific help. `aidlc help --all` reveals the hidden `engine` and
`system` namespaces and points to
`aidlc engine --help` / `aidlc system --help` for their full inventories.
The installer places Bash, Zsh, Fish, and PowerShell files under the per-user AI-DLC data root's
`completions/` directory, generated from the public route registry; there is
no public completion-generation verb.

## Transactions and Recovery

Project and machine mutations stage on the destination filesystem, validate
the candidate, and commit through rename boundaries. The filesystem remains
responsible for coherent file identity and append behavior, atomic file
replacement and directory rename, exclusive creation, and meaningful regular-file
`fsync`. AI-DLC adds no copy-and-delete fallback for rename. Concurrent changes
detected against planned state abort instead of overwriting new bytes.
Abandoned owner-private staging is swept only after lock and ownership checks.
Unsupported directory `fsync` is tolerated, so a successful command alone
cannot promise metadata durability after a crash.

Transaction locking uses one owner-stamped directory, `.aidlc-transaction.lock`
in the project, made with an exclusive create on every storage; it also keeps
out a live file lock left by an older release. A dedicated owner-stamped gate in
the local temporary directory serializes transactions and ownership changes
among commands that share it. Commands with different temporary directories,
WSL beside Windows on one folder, or two computers on a shared folder are kept
apart by the project lock itself: a lock stamped by another computer is never
cleared from this one, and the command names the folder to delete once nothing
is running there. AI-DLC makes no hard links: a document copy or a generated
file is put at its name with an exclusive create and a rename, and never
replaces a file another command put there first. This is not distributed
locking across independent mounts that do not share one view of the folder,
even when they access the same bucket.

The first-run probe is the capability check, and transactions do not repeat
it. It does not replace validation or real lock acquisition, and passing calls
cannot prove atomic rename or crash durability. A transaction whose commit
rename the storage does not support stops with the same storage line and fix.
Compatibility depends on the driver and its actual semantics:

| Storage | Compatibility boundary |
| --- | --- |
| Local ext4 or XFS, including EC2 EBS volumes | Suitable project storage for the required filesystem operations. |
| Storage without hard links, such as exFAT, network shares, or WSL `/mnt/c` | Config and transactions run as on local storage when the remaining requirements hold; AI-DLC makes no hard links. |
| Mountpoint for Amazon S3 | Full workflows remain incompatible: directory rename and general mutable-file updates are unavailable. S3 Express single-file rename does not remove those limits. See [Mountpoint filesystem semantics](https://github.com/awslabs/mountpoint-s3/blob/5e400f788f8cbca028f3314d84ef2df2c7fcf536/doc/SEMANTICS.md). |
| s3fs-fuse | Rename uses copy then delete and is not atomic. Passing probes does not establish support for general crash-safe AI-DLC transactions. See [s3fs limitations](https://github.com/s3fs-fuse/s3fs-fuse/blob/fc5778fe83b533a9beed9383f3ce99de76207ef9/README.md#L153-L163). |

Upstream semantics and simulated failures are not live validation of a
particular S3 mount. Use compatible local/EBS project storage when a mount
cannot meet these requirements; see [filesystem troubleshooting](15-troubleshooting.md#config-fails-with-a-storage-error).
For foreign or incomplete lock owners, follow [Transaction lock ownership](15-troubleshooting.md#transaction-lock-ownership)
before any manual recovery.

If rollback of an interrupted commit cannot be completed safely, evidence is
retained in a named `.aidlc-recovery-*` quarantine under the machine install
root or project root. `aidlc doctor` reports it. Recover any needed files,
ensure no AI-DLC mutation is running, then remove only the listed directory
manually. Automatic staging cleanup never deletes quarantines.

Windows uninstall uses a recoverable continuation because a running executable
cannot remove its own command shim. A later command resumes a valid pending
continuation before doing other work, unless one is still running. A worker
that stops without recording a result is resumed at most three times.

When cleanup fails, it records the step that failed and the reason, and
`aidlc doctor` reports both. A failed cleanup is never relaunched by other
commands, which keep working, but machine changes stay blocked until the
uninstall finishes. Resolve the reported problem and run `aidlc uninstall`
again (with `--purge` if the original used it):

- If nothing was removed yet, the failed plan is discarded and uninstall plans
  again from the files on disk, so a file edited after confirmation is kept.
- If removal had begun, the same plan resumes. Files edited since are kept.

The `aidlc` command itself (`aidlc.cmd` and `aidlc.exe` in bin, the shim, the
active-version pointers, and the active `aidlc.exe`) is removed last, after every other file and the
User PATH entry, so a failed cleanup still leaves a command to retry it. Because
the PATH entry may already be gone, run that command by its full path, which
`aidlc doctor` prints (by default
`& "$env:LOCALAPPDATA\aidlc\bin\aidlc.exe" uninstall`, or the recorded `aidlc.cmd`
when `aidlc.exe` and `aidlc.cmd` in bin are not both as AI-DLC installed them). If the failure
happened while removing those last files and the command no longer runs, run
the installer again: it retries the pending cleanup first, so you may need to
run it twice.

## Copy Channel

The supported manual-copy payload is the versioned `aidlc-copy-runtime-X.Y.Z.tar.gz`
release asset. Download one exact release, extract it, and copy the complete
`runtime/<harness>/` root so the harness tree, `aidlc/` workspace shell, and
any project-root files the harness needs stay together. The copy runtime leaves
out files a team's editor owns, such as Copilot's `.vscode/settings.json`, so
copying never replaces them; the [Copilot guide](harnesses/copilot.md#vs-code-request-cap)
names the one setting to add yourself. It leaves out your `.gitignore` and
`AGENTS.md` too: AI-DLC adds its own lines to them, after everything already
there, or creates them when the project has none. opencode's `opencode.json`
is left out the same way: the copy's setup adds AI-DLC's entries to your file
and keeps everything else in it, or writes the file when there is none. Claude Code's `.mcp.json` is
left out as well, so a copy starts with no MCP servers, as `aidlc config` does
by default; to turn the shipped servers on, run
`bun .claude/tools/aidlc.ts config project --harness claude --mcp defaults --yes`. It also leaves out the team's memory
files (`aidlc/spaces/default/memory/team.md`, where Practices Discovery records
the practices you affirmed, and `project.md`, where your project rules and
learnings go) and your chosen space (`aidlc/active-space`). Copying a newer
release, or a second harness, over a project therefore keeps them. In a fresh
copy, AI-DLC creates the two memory files from its bundled copy the first time
you start work. Use **Bun 1.4.4 or newer**, which includes the Brotli decoder fix
for CVE-2026-111497; the native `aidlc` executable is not required. Updating
a copied runtime does not update the Bun installation that runs it.
Native installations instead need an AI-DLC release built with the fixed Bun:
updating a separate Bun installation does not replace the runtime embedded in
the `aidlc` executable. Markdown analysis (summary confirmation,
Plan Approval tags, and the claim-sources sensor) uses Bun's built-in
`Bun.markdown` renderer and follows the installed Bun's rendering:

```bash
tag=vX.Y.Z
tmp="$(mktemp -d)"
runtime_asset="aidlc-copy-runtime-${tag#v}.tar.gz"
runtime_checksum="${runtime_asset}.sha256"
source_repo="${AIDLC_RELEASE_REPOSITORY:-awslabs/aidlc-workflows}"
release_workflow="${AIDLC_RELEASE_WORKFLOW:-$source_repo/.github/workflows/release.yml}"
gh release download "$tag" --repo "github.com/$source_repo" --dir "$tmp" \
  --pattern "$runtime_asset" \
  --pattern "$runtime_checksum" \
  --pattern aidlc-release.intoto.jsonl
gh attestation verify "$tmp/$runtime_asset" \
  --bundle "$tmp/aidlc-release.intoto.jsonl" \
  --repo "$source_repo" \
  --signer-workflow "$release_workflow" \
  --hostname github.com \
  --source-ref "refs/tags/$tag"
(cd "$tmp" && sha256sum -c "$runtime_checksum")
tar -xzf "$tmp/$runtime_asset" -C "$tmp"
RUNTIME_ROOT="$tmp/runtime"
cp -R "$RUNTIME_ROOT/claude/." your-project/
cd your-project && bun .claude/tools/aidlc.ts config --from "$RUNTIME_ROOT" --harness claude
```

The last line is the copy's own setup, run once from the extracted runtime: it
adds AI-DLC's lines to your `.gitignore` and `AGENTS.md` before the first chat
and checks the rest of the setup. Without it, AI-DLC adds them when the first
chat starts, or at the latest when you start work.

Later, a copied project fetches releases itself. When a config command needs
files the project does not have (a teammate's newer pin, a harness you add,
or restored files), it asks at a terminal, or accepts `--download` in a
script, to download that exact `aidlc-copy-runtime-X.Y.Z.tar.gz`, verify its
`.sha256` and, when `gh` is installed, its release attestation, and finish the
command. Without network access, the error's `offline:` line names the file
to fetch elsewhere and pass with `--from`.

When a refresh (`config` with `--from` or `--download`) brings in files from a
newer release than the project's own tool, that release's own config runs the
command as you typed it, and a `Note:` line names the release whose tool ran
it. A same or older release is refreshed by the project's own tool. Either
way the project's own tool reads the command first, so its `--harness`
names are the ones that tool's release reads (for Kiro, see
[Root Integrations and Ownership](#root-integrations-and-ownership)).

The archive is assembled from the freshly regenerated Bun projections under
`dist/`. Its generated hooks and tools invoke the included TypeScript through
Bun. The native installers and lifecycle commands instead consume
`aidlc-runtime-X.Y.Z.tar.gz`, assembled from `dist-release/`; users do
not normally download that archive directly.

The copy archive stays outside `version.json` and `checksums.txt` so existing
2.8.x native clients can continue to parse release metadata and self-update.
Its versioned `.sha256` sidecar authenticates the bytes directly, and the
release provenance covers both files.

On a copy install, the AI-DLC files in your project are code you run. Its hooks
run them through Bun, and on every harness except GitHub Copilot the settings it
ships also pre-approve the agent's calls to them; each harness guide describes
how its pre-approval behaves. Trusting the project folder therefore means
trusting those files: anyone who can change the project can change what those
hooks and pre-approved commands run.

When native executables are permitted, prefer `aidlc config`. It installs the
native runtime transactionally and records ownership for later refreshes.

Framework developers may instead clone the source, install dependencies, and
materialize ignored local outputs:

```bash
bun install --frozen-lockfile
bun scripts/package.ts
```

That creates the Bun-invoking `dist/<harness>/`, native `dist-release/<harness>/`,
and plugin projections locally. Neither generated root is committed. Direct
`bun .../tools/*.ts` calls remain source/development and debugging mechanisms,
not a second native lifecycle interface.

## Remove AI-DLC from a Project

No command removes AI-DLC from one project. `aidlc uninstall` removes the
machine install, for every project on this machine, and leaves project files
alone. The steps below take a project back
to how it was before AI-DLC was added to it. They keep your own harness files and
settings, which share folders and files with AI-DLC (a `.kiro/` you had before,
your own `.claude/settings.json` entries, your part of `AGENTS.md`). Close the
harness in the project first.

### 1. Keep your records

If the project is pinned (it has a `.aidlc-version` file), unpin it first,
while AI-DLC is still installed: `aidlc config --unpin`, or in a copied project
`bun <harness folder>/tools/aidlc.ts config --unpin`. That also clears the
machine's record of the pin, which deleting the file does not.

On Claude Code, turn AI-DLC's status line off too, while AI-DLC is still
installed: `aidlc config --statusline off`, or in a copied project
`bun .claude/tools/aidlc.ts config --statusline off`. When AI-DLC's status line
replaced one of your own, that is the only step that puts yours back: AI-DLC
keeps it in `aidlc.settings.local.json`, which you delete below.

`aidlc/` holds your work: each intent's state, artifacts and audit trail, and
the space's memory and knowledge. Archive it before you delete it. In a POSIX
shell:

```bash
tar -czf aidlc-records.tar.gz aidlc && rm -rf aidlc
```

In PowerShell (Windows 10 and later include `tar`):

```powershell
tar -czf aidlc-records.tar.gz aidlc; if ($?) { Remove-Item -Recurse -Force aidlc }
```

Delete these too if they are there: `aidlc.settings.json`,
`aidlc.settings.local.json`, and the `aidlc.settings.local.json` line config
may have added to `.git/info/exclude`.

### 2. Delete AI-DLC's own files

`aidlc config` records what it installed in
`<harness folder>/tools/data/aidlc-manifest.json`: every file with its SHA-256,
and the shared files it added its part to. This script reads that record and
deletes only the files it lists whose content still matches, counting a file
whose only difference is its line endings (a CRLF checkout) as unchanged. It
keeps and lists a file you changed, never deletes a file the record does not
list apart from the few AI-DLC generates while it runs, and removes only the
folders it emptied. Save it as `remove-aidlc.cjs` in the folder that holds the
project, next to the project folder rather than inside it:

```js
// Usage, from the project root: node <path to>/remove-aidlc.cjs <harness folder>   (Bun works too)
const fs = require("node:fs");
const path = require("node:path");
const { createHash } = require("node:crypto");
const dir = process.argv[2];
const manifestPath = path.join(dir, "tools", "data", "aidlc-manifest.json");
const manifest = JSON.parse(fs.readFileSync(manifestPath, "utf8"));
if (manifest.shippedOnly !== true) {
  console.log(`${manifestPath} predates the shipped-files-only record. Refresh it first with aidlc config --harness <harness>, then run this again.`);
  process.exit(1);
}
const sha = (bytes) => "sha256:" + createHash("sha256").update(bytes).digest("hex");
// A file whose only difference is its line endings (a CRLF checkout) is unchanged.
const unchanged = (file, hash) => {
  const bytes = fs.readFileSync(file);
  const lf = bytes.toString("latin1").replaceAll("\r\n", "\n");
  return [bytes, Buffer.from(lf, "latin1"), Buffer.from(lf.replaceAll("\n", "\r\n"), "latin1")].some((form) => sha(form) === hash);
};
const roots = manifest.rootContributions ?? {};
const shared = new Set([...Object.keys(manifest.entries ?? {}), ...Object.keys(roots)]);
const owned = Object.entries(manifest.files).filter(([file]) => !shared.has(file));
for (const [file, part] of Object.entries(roots)) if (part.policy === "whole-file") owned.push([file, part.hash]);
const removed = [];
for (const [file, hash] of owned) {
  if (!fs.existsSync(file)) continue;
  if (unchanged(file, hash)) { fs.rmSync(file); removed.push(file); }
  else console.log(`kept, changed since install: ${file}`);
}
// Files AI-DLC generates while it runs, which the record does not list.
for (const file of ["harness.json", "scope-grid.json", "stage-graph.json"].map((name) => path.join(dir, "tools", "data", name))
  .concat(path.join(dir, "steering", "aidlc-active-memory.md"), manifestPath)) {
  if (fs.existsSync(file)) { fs.rmSync(file); removed.push(file); }
}
// Remove only the folders this emptied, never one that holds anything else.
for (const file of removed) {
  for (let folder = path.dirname(file); folder !== "." && folder !== path.dirname(folder); folder = path.dirname(folder)) {
    try { fs.rmdirSync(folder); } catch { break; }
  }
}
for (const file of shared) if (fs.existsSync(file)) console.log(`edit by hand: ${file}`);
```

Run it from the project root with Node.js or Bun, once for each harness folder
in the project. The same commands work in a POSIX shell and in PowerShell:

| Harness | Run |
|---------|-----|
| Claude Code | `node ../remove-aidlc.cjs .claude` |
| Kiro CLI, Kiro IDE | `node ../remove-aidlc.cjs .kiro` |
| Codex CLI | `node ../remove-aidlc.cjs .codex` |
| Cursor | `node ../remove-aidlc.cjs .cursor` |
| OpenCode, GitHub Copilot | `node ../remove-aidlc.cjs .aidlc` |

If it says the record predates the shipped-files-only record, an earlier
release wrote it: refresh with `aidlc config --harness <name>` first (in a
copied project, `bun <harness folder>/tools/aidlc.ts config --harness <name>`),
then run the script again. Review any file it kept: it is an AI-DLC file you edited, so
keep it only if you still want your edit.

### 3. Take AI-DLC's part out of shared files

The script lists each shared file still holding AI-DLC's part. If git has the
project from before AI-DLC was added, restoring them from that commit is the
quickest way, for example
`git checkout <commit> -- .gitignore AGENTS.md .claude/settings.json`. That also
undoes any change you made to them since, so check `git diff --cached` before
you commit. Otherwise remove AI-DLC's part by hand. Remove a value only while it
still holds what AI-DLC wrote; a value your team changed is yours, so keep it:

| File | Remove |
|------|--------|
| `.gitignore` | The lines from `# BEGIN AI-DLC:gitignore` to `# END AI-DLC:gitignore`, and the blank line above them. Several harnesses share this one block. |
| `AGENTS.md` | The lines from `<!-- BEGIN AI-DLC:agents -->` to `<!-- END AI-DLC:agents -->`, and the blank line above them. |
| `.claude/settings.json` | `companyAnnouncements`, unless your team set its own. `statusLine` when its command runs `aidlc`. Every hook group whose command runs `aidlc` (`aidlc engine hook ...`, or `bun "$CLAUDE_PROJECT_DIR/.claude/tools/aidlc.ts" engine hook ...` in a copied project). The `permissions.allow` entries that name `aidlc`, and `Edit(/**)`, `Task` and `WebSearch` unless you added them. `env` keys that start with `AIDLC_` or `AWS_AIDLC_`, and any provider keys you chose in `aidlc config providers`. |
| `.claude/settings.local.json` | A `statusLine` whose command runs `aidlc`; AI-DLC's status line lands here when it is turned on. Step 1's `--statusline off` takes it out. |
| `.mcp.json` (Claude Code), `.kiro/settings/mcp.json` (Kiro CLI) | The MCP servers AI-DLC ships (`context7`, `aws-mcp`, `aws-pricing`, `aws-iac`, `aws-serverless`), unless you set one up yourself. Kiro CLI gets them by default, turned off; Claude Code gets them when you turned them on, by answering yes when `aidlc config` asked at a terminal or with `config project --mcp defaults`. |
| `.codex/config.toml` | `developer_instructions`, `sandbox_mode`, `suppress_unstable_features_warning` and `tool_output_token_limit`; AI-DLC's keys in `[shell_environment_policy]` (`set`, holding `AIDLC_RULES_DIR`), `[sandbox_workspace_write]` (`network_access`), `[agents]` (`max_depth`), `[features]` (`default_mode_request_user_input`), `[tools]` (`experimental_request_user_input`) and `[tui]` (`status_line`); the `#` comments directly above them; and any of those tables left empty. Keep keys you set yourself, also inside those tables. |
| `opencode.json` (or `opencode.jsonc`) | `$schema` unless it is yours; `.aidlc/skills` in `skills.paths`; the `instructions` entries under `.aidlc/` and `aidlc/`; and AI-DLC's `permission` rules: the ones naming `.aidlc/`, `"aidlc engine *": "allow"` under `bash` (a native install), its asks before `rm -r*` and `git push *`, and the `"*"` rules earlier releases added under `edit` and `bash`, unless you wrote the same rule yourself. |
| `.vscode/settings.json` (Copilot) | `chat.agent.maxRequests` (`200`) and `chat.useHooks` (`true`), unless you set them yourself. |

Delete a file when nothing of yours is left in it: AI-DLC created it. Cursor's
root `install.ts` and the other whole files AI-DLC wrote at the project root are
deleted by the script in step 2. Codex keeps its trust for the project's hooks
outside the project, in `$CODEX_HOME/config.toml` (`~/.codex/config.toml` by
default); remove the project's entries there if you want Codex to forget them.

### 4. Check

`git status` shows AI-DLC's files deleted and your shared files back to what they
held before. Searching each shared file from step 3 for `AI-DLC` and `aidlc`
should find nothing of AI-DLC's left:

```bash
grep -n "AI-DLC\|aidlc" .gitignore AGENTS.md
```

```powershell
Select-String -Path .gitignore, AGENTS.md -Pattern 'AI-DLC|aidlc'
```

A project copied from `aidlc-copy-runtime-X.Y.Z.tar.gz` gets the manifest when
its setup command runs (see [Copy Channel](#copy-channel)). If that command never
ran, there is no manifest: delete the files that are byte-identical to the
`runtime/<harness>/` folder you copied, then follow steps 3 and 4. Copying also
replaces files of the same name that the project already had (such as your own
`.claude/settings.json`); restore those from git or your backup.

## Uninstall

`aidlc uninstall` removes the machine install. To remove AI-DLC from one
project, see [Remove AI-DLC from a Project](#remove-ai-dlc-from-a-project).

```bash
aidlc uninstall
aidlc uninstall --purge --yes
```

Uninstall uses an explicit list of installer-owned files and checks their
contents before deleting them. It does not recursively remove installation or
version directories. Directories are removed only when empty; project trees,
unlisted files, changed files, and linked targets are preserved. The result
lists, once, each path it left because AI-DLC did not install it or it changed
after install.

New installations record a full per-version `installed-files.json` inventory,
whose hash is stored in `version.json`. Older installations use their verified
runtime inventory where available, plus the plugin folders their release
unpacked beside it, which carry the marker the release build wrote; other files
without ownership evidence are kept. The plugin catalogue's own record lists
the files its last refresh wrote: those go, and anything else you put in that
folder is kept. A shell completion counts as AI-DLC's when
it is exactly what an AI-DLC release renders, so the completions an earlier
release wrote while it updated to this one are removed, and an edited one is
kept.
Without `--purge`, machine config, update cache, pin registrations, and the
default harness are also preserved. `--purge` selects those known machine
records for removal; it does not broaden deletion to unrelated files.

On Windows, both forms remove the User PATH entry recorded in the install
root's `windows-path.json`. Entries that existed before installation and
unrelated changes made afterward are preserved. An install without an
ownership record leaves User PATH alone. `-NoModifyPath` on a later installer
run preserves an earlier record, so that entry is still removed on uninstall.

At a terminal, uninstall says what it removes and keeps, then proceeds; without
a terminal it needs `--yes`. It refuses filesystem, home, shared-system, and
project roots, as well as root-owned, package-manager-owned, or
mixed-ownership commands. A root install made with `--allow-root` is removed
with `aidlc uninstall --allow-root` (or `AIDLC_ALLOW_ROOT=1`); the opt-in is
refused under `sudo` and when `$HOME` is not owned by root. On Windows, a bound file list and expected checksums
are recorded before cleanup is scheduled. The worker rechecks paths and hashes,
refuses reparse points, and deletes files individually after the running command
exits; uninstall is done when the `aidlc` command is no longer found. An
interrupted continuation can resume only with its validated file plan.
Older journals without such a plan are refused and left for inspection. See
[Transactions and Recovery](#transactions-and-recovery) for how a failed
cleanup is reported and retried.
