# Troubleshooting

この章では、よくある問題とその解決策を症状別に整理して説明します。

> **ハーネスに関する注記。** 以下の症状と対処は **Claude Code** を前提に書かれています（フック
> のファイル名、`settings.json` のブロック、コンパクションの挙動）。決定的なコア
> — 状態、監査、エンジン — はどのハーネスでも同じように動作しますが、
> シェルレベルの部分は異なります。他のハーネスはフックと設定をそれぞれ独自の方法で接続します
> （[Running on other harnesses](harnesses/README.md) を参照）。対処の説明に
> `.claude/` のパスや Claude 固有の仕組みが出てくる場合、相当するものはお使いのハーネスの
> 設定ディレクトリにあります。

---

## Quick Fix Table

| 症状 | 手早い対処 |
|---------|-----------|
| 監査エントリが記録されない | `aidlc doctor` を実行します。コピーインストールの場合は、`bun` がフックの PATH 上にあることも確認してください |
| Windows: 監査やその他の記録で `—` が `â€”` と、`→` が `â†’` と表示される | ファイルは UTF-8 ですが、Windows PowerShell 5.1 が ANSI として読み込んでいます。`Get-Content -Encoding UTF8` で読み込むか、PowerShell 7 を使ってください。その読み込み結果から書き直したファイルには誤った文字が残るため、記録は `Copy-Item` またはエクスプローラーでコピーし、実行内容の共有には `/aidlc --doctor --export` を使ってください |
| ファイルを手で編集した。エージェントはそれを受け入れるか？ | はい。編集してから **done** と伝えてください（またはそのまま続けてください）。完了済みのステージへの編集はそのまま使われますが、再承認はされません。承認を受けたい場合は `/aidlc --stage <name>` で戻ってください。そのステージとそれ以降のすべてのステージが再び開かれます。各ケースについては [ファイルを自分で編集する](07-interaction-modes.md#editing-files-yourself) を参照してください |
| Claude のフックがポリシーで制限されている | Claude Code の管理者に、マネージド設定の `allowManagedHooksOnly` を解除するよう依頼してください。プロジェクト設定ではこれを上書きできません |
| Cursor: 承認が一切記録されない | プロジェクトが git リポジトリ内にない場合は、そこで `git init` を実行し、Cursor を完全に再起動してフォルダを信頼してください（[Cursor project outside a git repository](#cursor-project-outside-a-git-repository) を参照） |
| Kiro IDE または Kiro CLI v3: `deny fs_read matching ".kiro/"` | `/aidlc --doctor` を実行し、表示された ignore ファイルから `.kiro/` のルールを削除してください（[Kiro Read Denials](#kiro-read-denials) を参照） |
| Kiro IDE: AI-DLC のレビューやヘルパーが実行されない、チャットに **Approval needed in agent-aidlc-...** パネルが表示される、または `/aidlc` が Kiro のワークフローに渡されてしまう | Kiro の Workflows 機能がオンになっています。エージェントにオフにするよう頼むか、`aidlc config trust --kiro-workflows off --yes` を実行してから **Developer: Reload Window** を実行してください（[Kiro's Workflows feature](harnesses/kiro.md#kiros-workflows-feature-stops-ai-dlcs-reviews-and-helpers) を参照） |
| Kiro CLI: AI-DLC のレビューやヘルパーが実行されず、エージェントが作業を引き渡すツールがないと言う | Kiro CLI の Workflows 機能がオンになっています。エージェントにオフにするよう頼むか、`aidlc config trust --kiro-cli-workflows off --yes` を実行してから Kiro CLI を終了し、このプロジェクトで再度 `kiro-cli --v3` を起動してください（[Kiro's Workflows feature](harnesses/kiro.md#kiros-workflows-feature-stops-ai-dlcs-reviews-and-helpers) を参照） |
| Windows 上の Kiro IDE で、Kiro のターミナルがコマンドプロンプトの場合: コマンドの結果が `AI-DLC reply:` 行で終わるかその後で切れてしまい、ステップに余分なターンがかかる、またはすべてのコマンドが終了コード -1 になる | エージェントに Kiro のターミナルを PowerShell に設定するよう頼むか、`aidlc config trust --kiro-terminal powershell --yes` を実行してから Kiro を再起動してください（[Command Prompt as Kiro's terminal](harnesses/kiro.md#command-prompt-as-kiros-terminal-on-windows) を参照） |
| Windows 上の Kiro IDE で、Kiro のターミナルが PowerShell の場合: フォルダを信頼しているのにすべてのコマンドが終了コード -1 になり、結果が `AI-DLC reply:` 行で終わるか切れてしまう | Kiro のシェル統合が読み込まれていません。たとえば PowerShell の実行ポリシーが `Restricted` や `AllSigned` になっている場合です。Kiro のターミナルで `aidlc doctor` を実行すると（チャットでの `/aidlc --doctor` ではこれを確認できません）、`Kiro terminal` 行に原因が表示されます。Kiro の PowerShell ターミナルで（Windows PowerShell と PowerShell 7 はそれぞれ別のポリシーを持ちます）`Get-ExecutionPolicy -List` と `$ExecutionContext.SessionState.LanguageMode` を実行してください。`MachinePolicy` または `UserPolicy` が設定されている場合、あるいは言語モードが `FullLanguage` でない場合は IT チームに相談してください。`Process` が設定されている場合は、スタートメニューから Kiro を起動してください。有効なポリシーが `CurrentUser` または `LocalMachine` から来ていて `Restricted` または `AllSigned` の場合は、そこで `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned` を実行し、Kiro を再起動してください。Kiro の `terminal.integrated.shellIntegration.enabled` がオフになっている場合や、PowerShell プロファイルに余分な引数がある場合も同じ症状になります（[PowerShell without Kiro's shell integration](harnesses/kiro.md#powershell-without-kiros-shell-integration-on-windows) を参照） |
| Kiro IDE: エージェントが "my memory"（自分の記憶）を根拠に、チェックの回避やフックの手動実行を提案する | その提案は無視し、該当する Kiro のメモリを削除してください（[Kiro のメモリーに古い AI-DLC の助言が残っている](harnesses/kiro.md#kiro-memories-carry-old-ai-dlc-advice) を参照） |
| 状態ファイルが壊れている | `/aidlc --doctor` を実行し、状態テンプレートと比較してください |
| 承認ゲートで止まっている | 返答を入力してください。先へ進むには `/aidlc --stage <target>` を使います |
| Kiro IDE: エージェントが実行したビルド・テスト・プローブのコマンドが `Exit Code: -1` を示すが、AI-DLC 自身のコマンドは動作する | Kiro がそのコマンドを実行していません。あなたの許可を待っているか、あなたが拒否したためです。Kiro のプロンプトで **Allow**（または **Always allow**）を選ぶか、どう進めるかをエージェントに伝えてください |
| Kiro IDE: 承認の質問への返答が認識されない、またはコマンドが終了コード -1 で返ってくる | Kiro がこのフォルダを信頼するか尋ねてきたら **Trust Folder & Continue** を選び（または Restricted Mode のバナーで **Manage**、続いて **Trust** を選び）、コマンドパレットから **Developer: Reload Window** を実行して、続けるよう伝えてください。次のメッセージがまだ記録されない場合は、`/aidlc --doctor` で理由がわかります。Kiro CLI では、いったん終了してそのフォルダで再度 `kiro-cli --v3` を起動してください（フォルダを信頼するか尋ねられ、中身を把握している場合は **Trust, continue** を選びます）（[Kiro IDE hooks not running](#kiro-ide-hooks-not-running) を参照） |
| Kiro IDE または Kiro CLI: 承認時に `aidlc-record-human-turn` がオフになっているため回答が記録されなかったと表示される、または次の AI-DLC ステップで AI-DLC のフックがオフになっていると表示される | `.kiro/hooks/` にあるそのフックのファイルに `"enabled": false` が書かれています。Kiro IDE の Agent Hooks パネルがこれを書き込み、その状態でコミットされたファイルはすべてのクローンに持ち込まれます。AI-DLC がこれを自動でオンに戻すことはありません。Kiro IDE では Agent Hooks でオンに戻してください。Kiro CLI では、表示された行が示すファイルの `"enabled": false` を `true` に変更し、Kiro を終了してそのフォルダで再度 `kiro-cli --v3` を起動してください（フックファイルはセッション開始時に読み込まれます）（[A Kiro hook switched off](#a-kiro-hook-switched-off) を参照） |
| Kiro CLI（または Kiro ACP クライアント）: すべての承認で人間の返答が届いていないと表示され、再起動しても直らない | エンジンがディストリビューションと一致していません。Kiro CLI (V2) には Kiro CLI の v2 エンジン（`kiro-cli --v2`）が必要で、Kiro がエージェントファイルをアップグレードした後であれば v3 も使えます。Kiro には v3（`kiro-cli --v3`）が必要です（[Kiro CLI のフックが動かない](#kiro-cli-hooks-not-running) を参照） |
| AI-DLC がナレッジベースからナレッジドキュメントフォルダ内のファイルを読めないと言う、または `aidlc doctor` がそれらについて警告する | それらのファイルはナレッジベースに追加されていないか、読み取れるテキストを含んでいません。各ファイルについて表示された行の指示に従い（追加するか、その横に `.md`、`.txt`、`.csv` のコピーを保存する）、その後 `/aidlc knowledge sync` を実行してください（[ドキュメントナレッジ](08-knowledge.md#document-knowledge-documentkb) を参照） |
| `aidlc doctor` がこのプロジェクトは同期フォルダ（Google Drive、OneDrive、Dropbox、Box、iCloud Drive）内にあると言う、または誰も編集していないファイルについてレビューや計画の承認が拒否される | 同期クライアントが AI-DLC のチェック対象のファイルを書き換えることがあり、ファイルが変更されると再レビューされるまで承認は止まります。プロジェクトを同期フォルダの外に移すか、そのフォルダの同期を一時停止してから、拒否メッセージが示す手順を実行してください |
| セッションの途中でコンテキストがコンパクションされた | `/aidlc` を実行してチェックポイントから再開してください |
| 監査ログが大きすぎる | そのままにしておいてください。長期プロジェクトの監査ファイルが大きくなるのは設計どおりで、エンジンはそれを読んで、あなたが何を承認し何を完了したかを把握します（[Audit Log Growing Too Large](#audit-log-growing-too-large) を参照） |
| フックが止まっているように見える | `/aidlc --doctor` でロックの所有者を診断してください。[Lock Files Left Behind](#lock-files-left-behind) を参照してください |
| ステータスラインに "ready" と表示される | `aidlc-state.md` に `**Lifecycle Phase**` フィールドがあるか確認してください |
| ステータスラインが表示されない | オンにしたときだけ表示されます: `/aidlc --statusline on`（Claude Code と Codex）。その後 `aidlc doctor` を実行してください。コピーインストールの場合は `bun` が PATH 上にあることを確認してください |
| サブエージェントがタイムアウトした | `/aidlc` を実行して再試行するか、ステージをインラインで実行してください |
| `A program is watching the project folder (most likely a dev server or a test watcher still running)`（Windows） | 開発サーバー（Vite）や実行したままのテストウォッチャーなど、プロジェクトのすべてのフォルダを監視するプログラムがあると、AI-DLC が作業記録のロックフォルダを移動できず、各ステップが約 10 秒待たされます。そのプログラムを停止してください（`npm run dev` を止めても `npm` だけが止まりサーバーは動き続けることがあります。その場合はタスクマネージャーか `taskkill /T /PID <pid>` で終了してください）。そうすれば次のステップは通常どおり進みます。アシスタントまたはそれが送り出したエージェントが会話の中でそのプログラムを起動した場合（たとえばアプリの確認のため）、それを停止して作業を続けます。この行が表示されるのは、どちらも起動していない場合か、停止してもフォルダが解放されなかった場合だけです。Guard Policy が strict の場合、そのプログラムが止まるまでビルドの書き込みは拒否されます。 |
| Windows: `aidlc/` 配下のファイルについて git が `Filename too long` と言う、またはレビューや質問のファイルを書き込めない | プロジェクトフォルダの階層が深すぎます。長いパスが有効になっていないと、Windows のツールと Git は 260 文字を超えるパスを拒否し、AI-DLC の記録はプロジェクトフォルダの深い位置に置かれます。`aidlc config` がそのことを 1 行で伝え、`aidlc doctor` は `Project folder` 行でフォルダの深さと記録が加える長さを示します。プロジェクトを短いフォルダ（たとえば `C:\src\<name>`）に移すか、長いパスを有効にしてください。管理者として PowerShell で `New-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Control\FileSystem -Name LongPathsEnabled -Value 1 -PropertyType DWORD -Force` を実行し、続いて `git config --global core.longpaths true` を実行します。AI-DLC はこの 2 つの設定を読み取り、両方が有効なら何も表示しません。AI-DLC がこれらの設定、あなたのフォルダ、git の設定を変更することはありません。Copilot CLI は、この 2 つの設定にかかわらず 260 文字を超えるパスを読み込むたびに確認を求めます（`copilot -p` は拒否します）。そのため Copilot CLI では、短いフォルダにすることでしかその確認はなくならず、この行はどちらの場合も表示されます。 |
| ワークフローが止まった、または挙動がおかしいので助けが必要 | `/aidlc --doctor --export` を実行し、生成された `.tar.gz` を共有してください（秘匿化済みで、作業成果物は含みません）。AI ツール自身のチャットログにしか現れない内容（拒否、繰り返される質問、承認）も含めたい場合は、`/aidlc-issue-report`（Codex: `$aidlc-issue-report`）で issue レポートを依頼してください。実行の記録とツールのイベントを、名前とテキストをラベルに置き換えたうえで含み、送信前に確認できます。doctor の結果はレポートに含まれますが、アーカイブ自体は送信されません（チームから求められたら `/aidlc --doctor --export` を再度実行してください） |
| Bolt の試行が脇に置かれた | 保存された中断結果の `restore_operation` を、その正確な argv で呼び出すようアシスタントに依頼してください。ヒントは表示専用です。証拠のみの試行には、復元する保存済みファイルはありません。[getting the files back](#a-bolt-attempt-was-set-aside-getting-the-files-back) を参照してください |

---

## Native Install Channel

| 症状またはエラー | 解決策 |
|------------------|------------|
| `Checksum mismatch for <asset>.` または `<asset>: checksum mismatch` | 作業を止めてください。ダウンロードしたディレクトリは再利用しないでください。同じリリースから、リリースアセット一式と `checksums.txt` をもう一度ダウンロードしてください。 |
| `install.sh` の後に `command not found: aidlc` | インストーラーが表示した bin ディレクトリを `PATH` に追加し（通常は `export PATH="$HOME/.local/bin:$PATH"`）、新しいシェルを開いて `aidlc doctor` を実行してください。 |
| `install.ps1` の後に `aidlc` が認識されない | 既定のインストールでは、PowerShell を実行したアカウントのユーザー PATH に登録されます。インストール中にすでに開いていたターミナルアプリや IDE（Kiro、VS Code、Cursor）は起動時の PATH を保持しているため、完全に終了してから開き直してください。その中で新しいタブを開いたり Reload Window を実行したりするだけでは不十分です。[`-NoModifyPath`](18-install-and-lifecycle.md#windows-powershell) を使った場合は、表示された直接実行用のコマンド（通常は `& "$env:LOCALAPPDATA\aidlc\bin\aidlc.exe" config`）を使うか、そのスイッチなしでインストーラーを再実行して PATH を登録してください。 |
| `aidlc` が PowerShell や CMD では動くが、Git Bash（またはその他の MSYS シェル）では動かない | これらのシェルは Unix の `execvp` の規則でコマンドを解決し、`PATHEXT` を無視します。そのため、素の `aidlc` は bin ディレクトリ（通常は `%LOCALAPPDATA%\aidlc\bin`）の `aidlc.exe` は見つけますが、`aidlc.cmd` は決して見つけません。`aidlc.exe` 導入前のインストールでは、代わりに `aidlc.cmd` の横に拡張子なしの `aidlc` ランチャーがあり、これらのシェルでも素の名前で解決されます。どちらもない場合は、インストーラーを再実行するか、（`aidlc` がすでに解決できる PowerShell や CMD から）`aidlc use <version>` を実行して `aidlc.exe` を書き込んでください。Windows で `/hooks` には登録済みと表示されるのに一度も発火しないフックの既知の原因の 1 つがこれで、フックのシェルが `aidlc` を解決できないことです。`aidlc doctor` はこれを失敗した `Windows launcher (Git Bash)` 行として示し、取るべき手順を示します。**AI-DLC が書いたものではない `aidlc` という名前のファイル（たとえば以前に手作りした転送用スクリプト）がすでにその bin ディレクトリにある場合、インストーラー、`aidlc update`、`aidlc use` はターミナル上で一度だけ置き換えてよいか尋ね、あなたのファイルを `aidlc.bak-<time>` としてその横に残します。`--yes`（インストーラーでは `-Yes`）は「はい」と答えます。ターミナルも `--yes` もない場合は、ファイルをそのままにして、退避するよう伝えます。そこにある `aidlc` という名前のディレクトリが置き換えられることはありません。削除するか名前を変更してから、インストーラーまたは `aidlc use <version>` を再実行してください。** |
| `This PowerShell window is running as administrator` | 最も安全にインストールするには、N と答え、PowerShell を通常どおり（"Run as administrator" ではなく）開いて、インストーラーを再度実行してください。それでも管理者として続行するには、y と答えるか、インストーラーに `-Yes` を渡してください。`aidlc uninstall` は何も尋ねません。同じ警告を表示してそのまま進むため、最も安全にアンインストールするには通常の PowerShell ウィンドウから実行してください。AI-DLC はユーザー単位でインストールされ、管理者権限は必要ありません。Windows Server の組み込み Administrator や UAC による昇格のないその他のセッションでは、警告は表示されません。 |
| Windows のインストールで PATH の競合が報告される | ファイルはインストールされていますが、永続的な PATH が別のコマンドを選択しています。`config` には表示されたコマンドのフルパスを使うか、報告された競合を解消してください。終了コードが 0 でも、JSON は `status: "warning"` と `data.ready: false` を報告します。 |
| Windows のインストールで PATH を構成できなかったと報告される | ファイルはインストールされていますが、PATH の手順が終了コード 1 で失敗しました。表示された復旧手順に従うか、`-NoModifyPath` を付けて再実行し、直接実行用のコマンドを使ってください。JSON は `data.installed: true` と `data.ready: false` を報告します。 |
| Windows のアンインストール後もユーザー PATH のエントリが残っている | アンインストールで削除されるのは、`windows-path.json` を通じて所有しているエントリだけです。もともと存在していたエントリや、その所有記録のないインストールでは、ユーザー PATH は変更されません。[アンインストール](18-install-and-lifecycle.md#uninstall) を参照してください。 |
| アンインストール後もファイルが残っている | アンインストールでは、個別に検証したファイルを削除し、ディレクトリは空の場合にのみ削除します。一覧にないファイルや変更されたファイルは保持され、報告されます。古いインストールで所有一覧がないファイルも同様です。報告されたパスを確認してください。`--purge` でも無関係な内容は削除されません。 |
| `--offline requires --from <release-directory>` | オフラインモードがネットワーク上のリリースにフォールバックすることはありません。リリースアセット一式を転送し、そのディレクトリを `--from` / `-From` で渡してください。インストーラーがそれを検証します。 |
| `aidlc` が終了コード 4 で終了する、または Windows のアクティブポインタが無効 | ランチャー（bin 内の `aidlc.exe` または `aidlc.cmd`）は、使用できなかったファイル（アクティブバージョンのマーカー、アクティブなコマンドのターゲット、またはアクティブな実行ファイル）を示す `aidlc:` 行を 1 行表示し、続けて `Rerun the AI-DLC installer (install.ps1) to repair the aidlc command.` と表示します。以前のリリースから `aidlc update` を実行した後は、そのリリースのランチャー（何も表示せずに終了コード 4 で終了するもの）が、`aidlc doctor` と `aidlc uninstall` 以外の次の `aidlc` コマンドで置き換えられるまで残ります。その 1 回のコマンドはまだ古いランチャーを通して実行され、空白を含む値は別々の単語に分割されます。古いランチャーが残っている間、`aidlc doctor` は `Windows launcher` 行を表示します。AI-DLC がそれを置き換えられない場合（たとえばインストール後に `aidlc.cmd` や `aidlc-shim.ps1` が変更された場合）、その行に理由と修正手順が示されます。古いランチャーがコマンドの開始前に終了コード 4 で終了する場合は、インストーラーを再実行してください。`%LOCALAPPDATA%\aidlc\active-executable` は編集しないでください。検証済みの同じインストーラーを再実行するか、正常に動作する保持済みの実行ファイルから `aidlc use <version>` を使ってください。PowerShell がランチャーを `ConstrainedLanguage` モードで実行していることを示す行が表示された場合は、アプリケーション制御ポリシー（AppLocker または WDAC）がそれをブロックしています。管理者に `aidlc-shim.ps1` の許可を依頼するか、`%LOCALAPPDATA%\aidlc\versions\<version>\aidlc.exe` を直接実行してください。 |
| Windows で `aidlc-shim.ps1 cannot be loaded` または `is not digitally signed` と表示される、あるいはインストーラー、`aidlc update`、`aidlc use`、`aidlc rollback` が `command pointer validation failed for <version>: version probe returned 1 ""` で始まるエラーで止まる | グループポリシーの実行ポリシーが `aidlc.cmd` の渡す `-ExecutionPolicy Bypass` を上書きするため、PowerShell がランチャーを実行前に拒否しています。新しいバージョンの有効化を `aidlc.cmd` 経由で確認するリリースでは、その有効化が取り消されます。以前のバージョンがアクティブなまま残り、初回インストールの場合はどのバージョンもアクティブになりません。`Get-ExecutionPolicy -List` を実行してください。上から見て最初に `Undefined` でない行が有効なポリシーなので、`MachinePolicy` が `UserPolicy` より優先されます。その行が `MachinePolicy` または `UserPolicy` で、値が `AllSigned` または `Restricted` であれば、それが原因です。`RemoteSigned` と `Unrestricted` ではランチャーは実行できます。管理者にスクリプトの許可を依頼するか、`%LOCALAPPDATA%\aidlc\versions\<version>\aidlc.exe` を直接実行してください。 |
| `pending Windows uninstall`、または Windows のアンインストール復旧の失敗 | 実行中の AI-DLC コマンドを閉じて `aidlc doctor` を実行してください。保留中のクリーンアップは次のコマンドで再開されます。doctor がクリーンアップの失敗を報告した場合は、示された原因を解消してから、doctor が表示するフルパスで `aidlc uninstall` を再度実行してください（PATH エントリがすでに消えている可能性があるため、通常は `& "$env:LOCALAPPDATA\aidlc\bin\aidlc.exe" uninstall`、bin 内の `aidlc.exe` と `aidlc.cmd` が両方とも AI-DLC がインストールしたままの状態でない場合は記録された `aidlc.cmd` を使います。元のコマンドで `--purge` を使った場合は付けてください）。ファイル削除前に失敗したクリーンアップはディスクの状態から計画し直され、開始済みのものは再開されます。そのコマンドがもう実行できない場合は、インストーラーを再度実行してください。インストーラーはまず保留中のクリーンアップを再試行するため、2 回実行する必要があるかもしれません。ファイル計画が紐付いていない古いジャーナルや改変されたジャーナルは拒否されます。調査のため、一時ジャーナル、クリーンアップスクリプト、マシンフェンス、残っているファイルは保持してください。 |
| `the last AI-DLC uninstall is still finishing` | Windows では、`aidlc uninstall` の終了後に AI-DLC のファイルが削除されます。インストールされたバージョンが 1 つか 2 つなら 1 分未満ですが、それ以上ある場合や遅いマシンではもっとかかります。何も変更されていません。表示された行の指示に従ってインストーラーを再度実行してください。完了せずに止まったクリーンアップは、開始から 10 分後に doctor 以外の最初の AI-DLC コマンドによって再開されます。 |
| Alpine で `libstdc++.so.6` または `libgcc_s.so.1` が見つからないと報告される | Bun と Node.js の musl ビルドが必要とするのと同じランタイム依存関係を `apk add libgcc libstdc++` でインストールし、インストーラーまたはコマンドを再実行してください。完全に静的な Bun musl コンパイルターゲットは現在利用できません。インストーラーはこの対処法を表示しますが、パッケージのインストールはしません。 |
| ハーネスがすでに独自のモデルアクセスを持っているのに、`Providers` が `no recorded answers` とともに `[needs]` を示す | Kiro CLI と Kiro IDE では、モデルアクセスが Kiro に付属しているため、この行は `[ok]` となり、`aidlc config providers` は何も尋ねません。GitHub Copilot と Cursor では、セッションがモデルアクセスを提供するため、回答なしで `[ok]` となります。自分の Amazon Bedrock アクセスを使う場合にのみ `aidlc config providers` を実行してください（Cursor では Bedrock のキーを受け付けるのは IDE だけで、CLI は常に Cursor のバックエンドを使います）。その他のハーネスではこの行は本当に対応が必要です。`amazon-bedrock` と答えるか、`keep current`（`--provider current`）を選んで、ハーネスにすでに設定されているプロバイダを維持して記録してください。`--provider other --acknowledge` は、自分で設定した別のプロバイダを記録します。 |
| `Models` が `no recorded policy` とともに `[needs]` を示す | ポリシーがない場合、ほとんどのエージェントはセッションのモデルと effort を引き継ぐため、高い effort のセッションではエージェントも高い effort で実行されます。エージェントの effort を設定するにはプリセットを記録するか（`aidlc config models --preset balanced --project --yes`）、そのままにして引き継ぎを続けてください。Kiro CLI では、プリセットは対話セッション全体に 1 つの effort を設定します。`aidlc config models` がセッションのモデルを選び、その effort とともに個人の Kiro 設定に保存します。Kiro の中では `/model` と `/effort` で変更できます（[Kiro CLI (V2) session model and effort](harnesses/kiro.md#kiro-cli-v2-session-model-and-effort) を参照）。GitHub Copilot、Cursor、Kiro IDE では、プリセットを設定しても何も変わらないため、この行は `[ok]` となり代わりにホストの名前を示します。モデルと effort はホスト側で選んでください（Kiro では、`aidlc config models --agent` で 1 つのエージェントに独自のモデルと effort を設定できます）。[モデルと推論量の選び方](18-install-and-lifecycle.md#choosing-a-model-and-effort) を参照してください。 |
| `aidlc doctor` で `workspace shell ready` が失敗し、`aidlc config` を再実行しても直らない | 対話的な再実行は既存のプロジェクションを使い、欠けている `aidlc/spaces/default/memory/` を再構築することはありません。シェルが不完全な間は `Workspace` 行を報告し、修正するセクションを提示しません。その行が示すコマンドを実行してください。ネイティブインストールでは `aidlc config --harness <name>` で、インストール済みのランタイムから更新し、ワークスペースのシェルを再作成します。Bun で起動するプロジェクション（`aidlc-copy-runtime-X.Y.Z.tar.gz` の `runtime/<name>/` ルート、またはチェックアウトの `dist/<name>/` ツリーからコピーしたもの）には更新元となるインストール済みランタイムがないため、そのコマンドには `--download` も付きます。これはプロジェクトのリリースに対応する `aidlc-copy-runtime-<version>.tar.gz` を取得して検証します。プロジェクトのフックがネイティブコマンドに切り替わってしまうため、ネイティブ用の `aidlc-runtime-X.Y.Z.tar.gz` や `dist-release/` のバイトが使われることはありません。 |
| `aidlc config` がリリースを必要としていると表示する: `this project is pinned to <version>`、`needs the <version> release files`、`is missing aidlc/spaces/default/memory/`、または `no longer has the MCP server list` | プロジェクトが、このマシンにないリリースのファイルを必要としています。チームメイトが設定した新しいピン、追加しようとしているハーネス、または失われたファイルが原因です。`fix:` 行を実行してください。これは実行したコマンドに `--download` を付けたもので、まさにそのリリースを取得し、チェックサムと（`gh` がインストールされていれば）リリースのアテステーションを検証して、コマンドを完了させます。ネイティブの場合は、`aidlc config --pin` と同様にピン留めされたリリースをインストールして登録します。コピーされたプロジェクトでは `aidlc-copy-runtime-<version>.tar.gz` をダウンロードします。ターミナルでは、config が代わりに確認を求めます。ピン留めされたリリースがすでにインストールされていれば、確認なしで使われます。ネットワークにアクセスできない場合は `offline:` 行に従ってください。示されたファイルを別の場所で取得して `--from` で渡すか、ネイティブの場合は `aidlc config --pin <version> --offline --from <release directory>` を実行します。`--release-base-url` と `--ca-bundle` でミラーを選べます。チェックサムやアテステーションに失敗した場合は、何も変更せずに停止します。 |
| コピーチャネルでインストールしていないプロジェクトで、`aidlc doctor` が Bun を要求する | コピーチャネルのプロジェクションは Bun でフックを実行し、ネイティブインストールは `aidlc` コマンドでフックを実行します。対処法には、プロジェクトがどちらのチャネルかが示されます。Bun を不要にするには、ネイティブのリリースインストーラーで再インストールし、`aidlc config --harness <name>` を再実行してください。 |
| `aidlc doctor` で `Harness trees on different releases` | プロジェクトのハーネスツリーが異なるリリースからインストールされているため、ツールごとにワークフローの挙動が変わる可能性があります。その行が示すコマンドを実行してください。ピン留めされたプロジェクトのツリーはピンのバージョンにそろえられます。ネイティブの場合は、エンジンのリリース（またはピン）と異なる各ツリーについて `aidlc config --harness <name>` を実行します。ピン留めされたコピー型プロジェクトでは `bun <harness-dir>/tools/aidlc.ts config --harness <name> --download` です。ピンがない場合は、最新のツリーの `aidlc-copy-runtime-<version>.tar.gz` を使って `bun <harness-dir>/tools/aidlc.ts config --harness <name> --from <file>` を実行します。ただし config の実行記録がないツリーには、先に一度 `--download` で更新します。進行中の作業は更新の間も継続されるため、終わるのを待つ必要はありません。 |
| `config plan changed after approval` | `aidlc config --dry-run --json` を再実行し、`data.actions` を確認してから、まったく同じソースと動作オプションで新しい `data.planToken` を適用してください。 |
| `aidlc config` から `locally modified` または `managed block was locally modified` | `aidlc config --dry-run --json` を実行して `data.actions` を確認してください。`--force` は、ベースラインが所有するフレームワークのバイトやマネージドブロックを置き換える場合にのみ使ってください。無関係なルートの内容を変更する許可にはなりません。`.claude/settings.json` はエントリ単位でマージされるため `--force` は不要です。更新すると AI-DLC のフックとコマンドの許可エントリが元に戻され、あなたのフック、deny ルール、カスタムのステータスライン、環境変数、その他の設定は保持され、何を戻したかを示す注記が表示されます。廃止された同梱の許可エントリは自動的には削除されません。`.codex/config.toml` も同じようにエントリ単位でマージされます。更新で変更されるのは AI-DLC 自身のキーだけで、あなたのキー、テーブル、コメントはそのまま保持されます。AI-DLC のキーのうちあなたが変更した値も保持され、リリースが別の値を同梱している場合は注記が表示されます。 |
| `aidlc doctor` で `AI-DLC files: N changed in this project` | これらの AI-DLC ファイルがこのプロジェクトで編集されているため、`aidlc config` はそれらを保持し、何かを更新する前に停止します。自分の版を残したい場合は、示されたファイルを別の場所に移してから `aidlc config` を実行してください。同梱の版で上書きしたい場合は `aidlc config --force` を実行してください。違いが改行コードだけのファイル（Windows のチェックアウトでは LF が CRLF に変換されます）は数えられません。 |
| `cannot coexist in one project` | 2 つ目のハーネスが、エンジンディレクトリ（`opencode` / `copilot` は `.aidlc` を使います）または排他的なマネージドブロック（Copilot の `AGENTS.md`）を共有しています。別々のエンジンディレクトリを選び、排他的なブロックを避けてください。Kiro CLI (V2) と Kiro も `.kiro` を共有しますが、`aidlc config --harness kiro`（または `--harness kiro-v2`）でインストール済みのものをその場でもう一方に切り替えられます（Kiro CLI (V2) から Kiro へ、ターミナルなしで、または `--quiet` や `--json` の下で切り替える場合は `--yes` を付けます）。[1 つのプロジェクトで複数のハーネスを使う](18-install-and-lifecycle.md) を参照してください。中立的な `AGENTS.md` ブロックは Kiro CLI、Kiro IDE、Codex、Cursor、OpenCode で共有され、`.gitignore` は同梱のエントリを結合します。 |
| `cannot switch .kiro from <row> to <row>: installed <row> has no ownership baseline`、`has an ownership baseline recorded before it listed only shipped files`、または `has an unusable ownership baseline` | インストール済みの行がどの `.kiro/` ファイルを所有しているかを示す、使用可能な記録がありません。そのため切り替えるとファイルが残ってしまうので、何も変更しません。表示された手順に従ってください。破損した `.kiro/tools/data/aidlc-manifest.json` を退避し（手順にフルパスが示されます）、表示された `config --harness <installed row>` を実行してから、もう一度切り替えを実行します。 |
| `switching .kiro to kiro lets Kiro run hook files AI-DLC does not own` | Kiro は v3 エンジンおよび Kiro IDE で、すべての `.kiro/hooks/*.json` ファイルを実行します。示されたファイルを確認してから、ターミナルでプロンプトに答えるか、表示された `--dry-run` を実行し、それが出力する `--plan-token` で切り替えを適用してください。`config plan changed after approval` または `hook files no longer match this switch's plan` は、切り替えの確認内容がその確認後に変わったことを意味します。もう一度 `--dry-run` を実行し、示された内容を確認してください。 |
| `cannot switch .kiro to kiro: .kiro/hooks is a link or a file, not a directory`、または `Kiro would run hooks through entries that are not regular files in .kiro/hooks` | `.kiro/hooks` をファイルを格納する実際のディレクトリにするか、一覧にある各フックエントリ（リンク、ディレクトリ、その他ファイルでないもの）を通常のファイルに置き換えるか `.kiro/hooks` の外に移してください。その後、もう一度切り替えを実行します。 |
| `predates shared onboarding` | 示されたインストール済みハーネスが選択したリリースより古く（または有効なバージョンが記録されておらず）、そのブロックは共有されていません。プレビュー版は、同じベースバージョンの安定版リリースより前に並びます。中立的な `AGENTS.md` ブロックを共有する別のハーネスを追加する前に、`aidlc config --harness <name>` で更新してみてください。これは古い兄弟ハーネス向けのヒントです。更新後もまだ拒否される場合は、そのブロックは排他的で、1 つのプロジェクトに共存できません。Copilot のブロックは更新後も排他的なままです。現在の排他的なブロックは、代わりに `cannot coexist in one project` を報告します。`--force` でもこの互換性チェックは回避できません。 |
| `refusing to refresh <harness> from a release whose AGENTS.md is not shared` | インストール済みの別のハーネスが中立的なルートブロックを共有していますが、選択した更新元はそれを共有として宣言していません。`AGENTS.md` を共有として宣言しているリリースを使ってください。`--force` でもこのガードは回避できず、プロジェクトのファイルは変更されません。 |
| `shared block is owned by <harness> from a different release` | 共有の `AGENTS.md` ブロックが、選択したリリースではなく、示された兄弟ハーネスのベースラインと一致しています。エラーに示された更新順序に従ってください。選択したハーネスを兄弟と同じリリースから更新するか、先に兄弟を選択したリリースから更新します。拒否された更新によってプロジェクトのファイルが変更されることはありません。 |
| `multiple project harnesses are present; pass one --harness <name>` | 複数のハーネスを持つプロジェクトでは、`aidlc config` を実行するたびに `--harness <name>` で対象を指定する必要があります。ただし、バイパスの記録または解除（他に変更するものがない状態での `aidlc config flags --bypass <NAME>` や `--clear-bypass <NAME>`）は例外で、これはプロジェクトに属するためハーネスの指定は不要です。 |
| `.gitignore` が `preserve (owned by <harness>)` を示す（`aidlc config --dry-run --json` では `action: "preserve"`、`detail: "owned by <harness>"`） | このフォールバックは、所有するハーネスが `tools/data/root-blocks/` を持たないリリースでインストールされた場合に現れます。`aidlc config --harness <owner>` を実行して更新すると、両方のハーネスが 1 つの結合されたブロックに収束します。 |
| `managed block has no ownership baseline` | このブロックは、もう存在しないかベースラインを持たないインストール（たとえば削除されたハーネスツリー）によって書かれたものです。`aidlc config --dry-run --json` を確認してから、`--force` を使って現在の同梱ブロック（共有の `.gitignore` の場合は結合されたエントリ）に置き換えてください。 |
| `has no readable projection descriptor` または `has lost its projection descriptor and ownership baseline` | 別のハーネスを追加する前に、示されたインストール済みハーネスを `aidlc config --harness <name>` で修復してください。共同所有のブロックについては、更新元がそのブロックを共有として宣言し、現在のブロックを変更しない（兄弟のベースラインと一致する）場合に限り、その兄弟のディスクリプタが失われていても同じリリースからの更新が許可されます。これにより、両方の欠けたディスクリプタをハーネスごとに 1 つずつ修復できます。それ以外の場合、`co-owns AGENTS.md` による拒否を解消するには、まず示された兄弟のディスクリプタを復元する必要があります。`--force` でもこのガードは回避できません。スタンプ付き（`aidlc-stamp.json` が存在する）の兄弟がディスクリプタとベースラインの両方を失っている場合、非ユニオンのマネージドブロックを変更する更新は `has lost its projection descriptor and ownership baseline` でブロックされます。先にその兄弟を復元してください。現在のブロックを変更しない同じリリースからの更新は引き続き許可されますが、`--force` でもブロックを変更する更新は許可されません。スタンプがなく共同所有のベースライン上の証拠もないレガシーなツリーは、引き続きハーネスごとに 1 つずつ取り込めます。 |
| `is missing its shipped block copy` | 示されたハーネスのインストールから `tools/data/root-blocks/<marker>` が失われています。`aidlc config --harness <name>` を実行して復元してから、更新を再実行してください。`--force` を使うと、そのハーネスのエントリを含まないブロックが書き込まれます。 |
| 通常の `aidlc config` によるリリース更新で `unowned whole file` | 更新の前に、既存のファイルを手動で移動またはマージしてください。OpenCode の `opencode.json` はここで止まらなくなりました。config はあなたのファイルに AI-DLC のエントリを追加し、それ以外の内容はすべて保持します。 |
| `opencode.json` によって config が `<key> must be a JSON object` または `must be a JSON array` で止まる | AI-DLC は `skills.paths`、`instructions`、`permission` にエントリを追加します。これらのいずれかが別の形（たとえば `"permission": "ask"`）になっている場合、config は何も変更しません。マップまたはリストの形（`"permission": { "bash": "ask" }`）で書き直してから、config を再実行してください。 |
| `<file> is not valid UTF-8; convert its encoding before config` | 示されたルートファイル（`.gitignore` や `AGENTS.md` など）をバックアップし、内容を保ったまま現在のエンコーディングから UTF-8 に変換してから、config を再実行してください。デコードによって情報が失われる場合、AI-DLC は元のバイトに手を付けません。 |
| `managed markers are missing, duplicated, or malformed` | 示されたルートファイルを修復して、対応する `BEGIN AI-DLC` / `END AI-DLC` のペアがちょうど 1 組になるようにするか、壊れた AI-DLC ブロックを削除して `aidlc config` を再実行してください。 |
| `project runtime <version> is incompatible with selected engine <version>` | `aidlc use <version>` を実行して互換性のあるバージョンをインストール・選択するか、`aidlc config` で意図的にプロジェクトを更新してください。 |
| `this project requires <version>, which is not installed completely` | `aidlc config --pin <version>` で、厳密な semver の正確なピンをインストールまたは再インストールしてください。ディスパッチャーはマシンのアクティブなバージョンにフォールバックせず、安全側に失敗します。`aidlc config --unpin` は、チームがプロジェクトのピン留めをやめるつもりの場合にのみ使ってください。 |
| `.aidlc-version must contain one release version id` | コミットされたピンファイルに、余分なテキストや紛れ込んだコマンドなど、1 つのリリース ID 以外のものが含まれています。`aidlc config` とディスパッチャーは、その内容を表示せずに拒否します。ファイルを意図したリリース ID（たとえば `2.10.0`）に修正するか、`aidlc config --unpin` を実行してプロジェクトのピン留めをやめてください。 |
| AI-DLC を依頼したときに `AI-DLC can't run in <folder>. Start the session from your project's folder.` と表示される | セッションが、AI-DLC のマシンインストールやコマンドのディレクトリを含むフォルダ（ホームフォルダなど）、またはその内側のフォルダで開始されています。AI-DLC はそこでは実行されないため、フックは何もせず、そのフォルダでのその他の作業は通常どおり続けられます。AI-DLC を使うには、プロジェクトのフォルダからセッションを開始してください。 |
| 更新が中断され、`aidlc version` がまだ以前のリリースを示している | これは安全に復元された状態で、古いコマンドがアクティブなままです。`aidlc doctor` を実行してから、同じ `aidlc update --version <version>` コマンドを再実行してください。 |
| `another AI-DLC command is still working in <project>.` | 別の AI-DLC コマンド（このプロジェクト上の別のチャット、ツール、またはターミナル）がロックを保持しています。それが終わるのを待ってから、コマンドを再実行してください。他に何も実行していないのに繰り返し表示される場合は、[Transaction lock ownership](#transaction-lock-ownership) に従ってください。 |
| `an AI-DLC command on another computer ... is working in <project> ... or left its lock there.`、または `.../.aidlc-transaction.lock cannot be read.` | ロックが、フォルダを共有している別のコンピュータ（または再起動前のこのコンピュータ）によって残されたか、読み取れない状態です。そこで AI-DLC コマンドが実行されていなければ、表示された行が示すフォルダを削除してから、コマンドを再実行してください。[Transaction lock ownership](#transaction-lock-ownership) を参照してください。 |
| `an earlier AI-DLC command stopped before it finished, so its lock was cleared` | 実行したコマンドについては何もする必要はありません。以前のコマンドは完了していないので、まだ必要であれば再実行してください。 |
| `another AI-DLC command is still changing this machine's install, so this ran on aidlc <version> without waiting for it to finish` | 何もする必要はありません。ピン留めされたプロジェクトのコマンドまたはフックが、更新・アンインストール・その他のマシンの変更を最大 30 秒待ってから、ピン留めされたリリースでそのまま実行されました。他の `aidlc` コマンドが実行されていないのに繰り返し表示される場合は、[Transaction lock ownership](#transaction-lock-ownership) に従ってください。 |
| config 中に `Cannot create an AI-DLC transaction lock` または `Cannot use the filesystem at ...` | プロジェクトのストレージが、AI-DLC に必要な操作をサポートしていません。[Config fails with a storage error](#config-fails-with-a-storage-error) を参照してください。 |
| `existing aidlc is managed by Homebrew` / `Nix`、または配置先のコマンドが `not owned by the AI-DLC installer` | 報告された所有者を通じてアップグレードしてください。別途ネイティブインストールを維持するには、`AIDLC_BIN_DIR` をユーザーが所有する空のディレクトリに明示的に設定してください。このリリース自体は Homebrew や Nix のパッケージを提供しておらず、所有者が混在するコマンドを置き換えることはありません。 |
| `update cache is invalid`、またはマシン設定が拒否される | `aidlc system config global list` を実行してください。示された `%LOCALAPPDATA%\aidlc\aidlc.settings.json`（Windows）または `${XDG_DATA_HOME:-$HOME/.local/share}/aidlc/aidlc.settings.json`（macOS/Linux）だけを修復または削除してください。未知のキーや保存された認証情報は拒否されます。 |
| `HTTPS_PROXY must use HTTP or HTTPS`、またはリリース URL が拒否される | 認証情報・クエリ・フラグメントを含まない HTTP(S) プロキシの URL と HTTPS のリリースミラーを使ってください。ネイティブクライアントは `HTTP_PROXY` ではなく `HTTPS_PROXY` と `NO_PROXY` を読み取り、エラー中の秘密情報らしき URL の部分は伏せ字にします。 |
| 社内 CA の環境でダウンロードに失敗する | `--ca-bundle <absolute-path>` を渡すか、`AIDLC_CA_BUNDLE` を設定してください。カスタム CA を指定する場合、Windows のブートストラップには `curl.exe` が必要です。 |
| プラグインのコマンドで `host inventory unavailable` | 現在のプラグインのルートを注入するホストセッションから sync を実行するか、Claude/Codex のホストレジストリを復元してください。インベントリが欠けている、または不正な形式であることを、内容を安全に削除できる証拠として扱うことはありません。 |
| `Nothing synced: this tool keeps no list of your plugins` | Kiro CLI、Kiro IDE、Cursor、Copilot、opencode にはプラグインの一覧がありません。表示された行が示すコマンドを、`--plugin-root` の後に各プラグインのフォルダを指定して実行してください。 |
| `cannot sync <plugin>` または `cannot prune <plugin>: owned path changed since composition` | プラグインが追加したファイルを編集しています。編集を残したい場合は、そのファイルを別の場所に移してから、表示された行が示すコマンド（同じ `aidlc engine plugin sync`。プラグイン一覧のないホストではプラグインのフォルダを指定し、または `sync --prune-missing`）を再度実行してください。sync はプラグイン自身の版を元に戻します。`--yes` でも所有権のハッシュは上書きできません。 |
| `aidlc system versions prune`、`uninstall`、またはプラグインの prune が `--yes` を要求する | コマンドが対話的な標準入力なしで実行されています。一覧の削除対象を確認してから、`--yes` を付けて再実行してください。整合性による拒否は回避できません。ターミナルでは、これらのコマンドは何も尋ねず、削除対象を表示してから削除を実行します。 |
| `aidlc setup` が不明なコマンドである、または npm でのインストールが利用できない | これらのチャネルは計画中で、まだ提供されていません。リリースインストーラーと `aidlc config` を使ってください。提案段階のやり取りに出てくるコマンドを利用可能なものとして扱わないでください。 |

ネイティブの `aidlc doctor` は、アクティブなコマンドポインタ、ロールバックの可否、
保持されたピンの完全性、古くなったピンの登録、放棄された
トランザクションのステージング、プロジェクトのバージョンのずれ、そしてバイナリチャネルのホスト
フックと権限・信頼のエントリが一貫してネイティブコマンドを選択しているかどうかもチェックします。

### Config warns about an ignore rule hiding committed records

Git リポジトリでは、config はあなた自身の ignore ルールが、コミットして共有するための記録を
隠していないかをチェックします。`aidlc/` のようなルールは記録を隠してしまうため、
config は完了はしますが、最後に `<file>:<line>` と隠されている記録のパス
（`memory/**`、`codekb/**`、`intents.json`、`aidlc-state.md`、
`audit/*.md`）を示す注記を表示します。新しい記録はチームメイトに届きませんが、git がすでに
追跡しているファイルは引き続きコミットされます。初回セットアップと `--quiet` でもこの注記は表示されます。
意図したものでない場合は、無関係な ignore を残したまま、示されたルールの対象を絞ってください。
個人用の作業スペースを Git から除外するためのルールなど、意図的なルールはそのままで構いません。
Git の外、または Git の実行ファイルがない環境では、このチェックは行われません。

### Config fails with a storage error

AI-DLC はハードリンクを作りません。プロジェクトのロックは所有者のスタンプ付きの
`.aidlc-transaction.lock` フォルダで、ファイルを特定の名前に置くのはそこに何もない場合だけで、
排他的な作成とリネームによって行います。そのため exFAT ドライブ、ネットワーク共有、
WSL の `/mnt/c` でも特別な対応は不要です。config がロックなしで書き込みを進めることはありません。

初回のウィザードは、セットアップの選択肢を提示する前に必要なファイルシステム操作を
プローブし、その後一時的なプローブファイルを削除します。プローブが失敗すると、何も書き込まずに
セットアップを停止します。プローブファイル自体を削除できなかった場合は、セットアップがその
ファイル名を示し、それ以外は何も書き込まれていないことを伝えます。人間向けの出力は失敗内容と対処法を示し、
`--quiet` は対処法を出力し、`--json` は構造化された失敗を返します。スクリプトから実行する
`aidlc config --harness <name>` は独自のプローブを行いません。ファイルをリネームできないストレージでは、
最初のリネームの時点で同じ `Cannot use the
filesystem at ...` の行と対処法を表示して停止し、ロールバックして、作成した空のフォルダだけを
残します。プローブに合格しても、アトミック性やクラッシュ時の永続性は保証されません。
[トランザクションと復旧](18-install-and-lifecycle.md#transactions-and-recovery) の機能とサポートの対応表を参照してください。
S3 マウントの名前やストレージエラーだけでは、そのドライバー、バージョン、
オプションは特定できません。

互換性のないマウントの場合は、プロジェクトを互換性のあるローカル
ストレージ（EC2 の EBS ボリューム上の ext4 や XFS など）に移動またはクローンし、そこで `aidlc config` を再実行してください。
作業中のプロジェクトは S3 マウントの外に置いてください。後でアップロードする場合は別の
公開作業となり、複数ファイルのアトミック性は保証されません。同じマウントへのエイリアスや
シンボリックリンクを使っても、その機能は変わりません。`--force`、`--from`、
ロックの削除のいずれでも、欠けているファイルシステムのセマンティクスは補えません。

### Transaction lock ownership

再試行する前に、生きている init/lifecycle の所有者が終わるのを待ってください。次の AI-DLC
コマンドは、所有者が停止しているロックを解除し、そのことを 1 行で伝えます。所有者が
停止しているとみなされるのは、その PID が死んでいる場合、またはシステムがその後その PID を
別のプロセスに割り当てた場合（Windows は PID をすぐに再利用します）です。後者は、開始記録が
ロックに記録されたものと異なるプロセス、または開始記録を残さないリリースのロックであれば、
ロックが書かれた後に開始したプロセスを指します。このコンピュータが再起動前にスタンプしたロックも、
所有者がその起動とともに停止しているため、同じように解除されます。所有者のスタンプが完了していないロック（
`owner.json` がない、空である、または JSON として読める前に途切れている）は、
ロックの取得中に停止したコマンドが残すものです。最初の数秒間は
まだ作業中の別のコマンドとして扱われ、それを過ぎると次のコマンドが解除します。
生きている所有者のロックは保持されます。フォルダを共有している別のコンピュータが
スタンプしたロック、またはコンピュータと起動を記録しないリリースがスタンプしたロックも保持されます。
ここに PID が存在しないことは、所有者が停止した証拠にならないためです。読み取れても有効な
所有者を示さないスタンプも同様です。後者の 2 つの場合は、そこで何も実行されていないことを
確認したうえで、どのフォルダを削除すればよいかが示されます。

手動で診断する場合は、エラー、ロック、示されたステージング・復旧用の
パスを保存してください。`.aidlc-transaction.lock/owner.json`（古いリリースが残したファイルロックの場合は
ロックファイル自体）を確認し、所有者のプロセス、コンピュータ、
マウントの履歴をオペレーターとともに特定してください。1 台のコンピュータ上のコマンドは、
その PID 名前空間を共有している必要があります（同じホスト名を持つ 2 つのコンテナは共有していません）。
修正作業の前に書き込みを行うプロセスを停止し、所有者が不確かな間は証拠を保持してください。
AI-DLC コマンドがまだ実行中の可能性がある間は、再試行を進めるためにプロジェクトのロックや
そのローカルの調整用ゲートを削除しないでください。

---

## Hooks Not Firing

**症状**: ファイルの書き込み後にインテントの `audit/` シャードにエントリが現れない、またはサブエージェントの完了ログがありません。

### Native runtime versus source-generated Bun projection

ソース／開発用の `dist/` プロジェクションは、17 個の TypeScript フックを
`bun` で実行します。ネイティブインストールとバージョン付きのリリースランタイムは、同じフックを
`aidlc` 経由で実行します。ソースから生成したインストールが非対話の PATH 上で Bun を
見つけられない場合、そのフックは発火しません。

```bash
# macOS / Linux
curl -fsSL https://bun.sh/install | bash

# Windows
npm install -g bun
# or: powershell -c "irm bun.sh/install.ps1 | iex"

# Verify
bun --version
```

ソースから生成した `dist/` インストールでは、ハーネスがフックに渡す PATH 上に `bun` が
なければなりません。ターミナルから起動したハーネスは、そのターミナルの
PATH をフックに渡すため、そこで `bun --version` が動けば十分です。Dock、
デスクトップのアイコン、サービスから起動したハーネスにはその PATH が渡されません。フックが
実行されない場合は、`bun --version` が動くターミナルを開き、そこで
`bun <harness-dir>/tools/aidlc.ts doctor` を実行してください（たとえば
`bun .kiro/tools/aidlc.ts doctor`）。その `Runtime hook PATH` 行に、追加すべき
ディレクトリと追加先が示されます。その後ハーネスを再起動してください。それまでは、
そのターミナルからハーネスを起動してください。ネイティブの
Windows PowerShell では、`npm install -g bun` が設定するシステムの PATH エントリで
十分です。

### Kiro IDE hooks not running

Kiro IDE がフォルダのフックを実行するのは、そのフォルダでのコマンド実行を許可し、
その後ウィンドウを再読み込みしてからです。許可するまでは、エージェントが
実行するすべてのコマンドが出力なし・終了コード -1 で返ってくるため、AI-DLC のメッセージは表示されず、
エージェント自身が手順を案内します。Kiro がフォルダを開いた後に信頼した場合は、
コマンドは実行されますが、再読み込みするまでフックは実行されません。その場合 AI-DLC は
作業を始める前に停止し、エージェントが同じ手順を示します。プロジェクトで最初のチャット
メッセージを送る前は、doctor が "AIDLC hooks have not run in this project
yet" と警告しますが、これは想定どおりです。

中身を把握しているフォルダ（自分のプロジェクト、または確認済みのもの）だけを信頼してください。
信頼すると、そのフォルダの `.kiro` フックがあなたのマシン上でコマンドを実行できるようになるためです
（[初回の実行](harnesses/kiro.md#first-run) を参照）。

1. Kiro がこのフォルダを信頼するか尋ねてきたら、**Trust Folder & Continue** を選びます。
   代わりにウィンドウ上部に Restricted Mode のバナーが表示されている場合は、
   その **Manage** を選び、続いて **Trust** を選びます。
2. コマンドパレット（Ctrl+Shift+P、macOS では Cmd+Shift+P）を開き、
   **Developer: Reload Window** を実行します。信頼の設定は再読み込み後に有効になります。
3. チャットで続けるよう伝えます。次のメッセージがまだ記録されない場合は、
   `/aidlc --doctor` で理由がわかります。

### Kiro CLI hooks not running

Kiro CLI の v2 エンジンと v3 エンジンは、異なるフック登録を読み込みます。
エンジンがインストールした AI-DLC のディストリビューションと一致しない場合、フックは 1 つも実行されません。
その結果、すべての承認と確認で人間の返答が届いていないと表示され、レビューでは
出力に書き込みの記録がないと表示され、最初のワークフローステージの後には
doctor が "Hooks have never executed" と報告します。同じエンジンで再起動しても
何も変わりません。

- **Kiro CLI (V2) ディストリビューション**（`--harness kiro-v2`、`.kiro/agents/aidlc.json`）:
  フックは Kiro CLI の v2 エンジンで実行されるため、`kiro-cli --v2` で起動し（Kiro CLI 2.21.4 以降。
  それより古い場合は [Kiro CLI (V2) prerequisites](harnesses/kiro.md#kiro-cli-v2-prerequisites) を参照）、
  `aidlc` エージェントをアクティブにしてください。v3 は AI-DLC が同梱したままのファイルは実行しませんが、
  Kiro がアップグレードした後は実行します。Kiro CLI 2.37 以降は、Kiro
  CLI の設定で `chat.enableAutoAgentUpgrade` が `false` でない限り、v3 での起動時に自動でアップグレードを行います
  （元のファイルはそれぞれ `<name>.json.bak` として保持されます。
  [Kiro CLI (V2) usage](harnesses/kiro.md#kiro-cli-v2-usage) を参照）。
  別のエージェントが選ばれている場合は、`/agent` と入力して `aidlc` を選び、同じチャットで
  そのまま続けてください。その一覧に `aidlc` がない場合（起動時に Kiro が `agent "aidlc" needs
  upgrading for this agent engine` と表示する場合）、セッションは v3 で動いており、
  ファイルがアップグレードされていません。
  `/upgrade-agent` と入力してワークスペースのエージェントを選び、Kiro を終了してこのフォルダで再度
  `kiro-cli --v3` を起動するか、
  `aidlc config --harness kiro` で Kiro（下記）に切り替えるか、Kiro を終了してこのフォルダで
  `kiro-cli --v2 --agent aidlc` で再起動してください。ACP クライアントからは
  `kiro-cli acp --agent-engine v2` を起動してください。
- **Kiro ディストリビューション**（`--harness kiro`、`.kiro/hooks/aidlc-*.json`）: フックは
  Kiro CLI の v3 エンジンでのみ実行されるため、`kiro-cli --v3` で起動してください。Kiro CLI 2.33.0 は
  エンジンをコマンドラインまたは個人の Kiro CLI 設定からのみ取得します。
  `.kiro/settings/cli.json` が同梱する `"chat.agentEngine": "v3"` は無視され
  （`kiro-cli settings --workspace chat.agentEngine` も拒否されます）、これを読み込むのは
  Kiro CLI 2.24.1 から 2.32.x までだけです。ACP クライアントは
  `kiro-cli acp --agent-engine v3` を起動する必要があります。また、`initialize` リクエストで
  `clientCapabilities._meta.kiro.hooks` を `{ enabled: true, v2: true }` として
  宣言する必要があります。この両方の値がないと、v3 はフックを実行しません。

これらの挙動は Kiro CLI 2.21.1 で確認したもので、今後の Kiro CLI で
変わる可能性があります。切り替えた後は、メッセージを送ってから `/aidlc --doctor` を実行してください。
Kiro CLI (V2) ディストリビューションでは、あなたのメッセージが最初のハートビートを残すので、
それ以降 doctor はフックを確認できます。それまでは、このプロジェクトでまだフックが
実行されていないと警告します。

### GitHub Copilot hooks not running

VS Code がプロジェクトのフックを実行するのは、信頼されたワークスペースで **Chat: Use
Hooks** 設定（`chat.useHooks`）がオンになっている場合だけで、組織がこの
設定をオフにすることもできます。プロジェクトがこの設定をしていない場合、`aidlc config` がフォルダの
`.vscode/settings.json` でオンにします。フォルダの値は、オフになっているユーザー設定より優先されます。
Copilot CLI がフックを実行するのは、信頼したフォルダの中だけです。どちらも
フックをスキップしてもチャットで何も知らせないため、AI-DLC が知らせます。あなたのメッセージに対して
フックが 1 つも実行されなかった場合、エージェントがフォルダの設定を自らオンにし、
1 行で知らせます。次のメッセージは同じチャットで送ってください。新しいチャットや再読み込みは
不要です。そのフォルダで最初の Copilot チャットを始める前は、doctor が "AIDLC
hooks have not run in this project yet" と警告しますが、チャットが始まるまでは
想定どおりです。

1. VS Code では、初回にフォルダの作成者を信頼するかどうか VS Code が尋ねるので、
   信頼してください。組織が Chat: Use Hooks をオフにしている場合は、
   管理者にしかオンに戻せません。
2. Copilot CLI では、尋ねられたらフォルダを信頼してください。ヘッドレスの `copilot -p`
   の実行では尋ねられないため、一度フォルダで `copilot` を起動し、"Yes, and
   remember this folder for future sessions" を選んでください。

フォルダの信頼の詳細については [GitHub Copilot](harnesses/copilot.md) を参照してください。

### Codex CLI hooks not trusted

Codex がプロジェクトのフックを実行するのは、`/hooks` 画面でそれらを信頼した後だけで、
フックが変わると再度確認を求めます。Codex で `/hooks` と入力し、`t` を押して
すべてを信頼してから Esc を押し、同じチャットでそのまま続けてください。Codex の外から
これを代わりに行うことはできません。

### opencode plugin not loaded

opencode が AI-DLC のプラグインを読み込むのは、プロジェクトフォルダで普通に起動した場合だけです。
`--pure` を付けた場合やサブフォルダから起動した場合は読み込まれません。opencode を終了し、
プロジェクトフォルダで単に `opencode` として再起動してから、`/aidlc` と入力して
続けてください。エージェントがこの手順を示してあなたの依頼を止めた場合は、代わりに
示された `/aidlc --request <id>` の行を入力してください。その依頼が 1 回だけ
続行されます。

### Claude Code or Codex started in a folder inside the project

どちらのツールも、`src/` のようなプロジェクト内のフォルダから `/aidlc`（Codex では `$aidlc`）を
提供しますが、AI-DLC のフックが動作するのは、ツールをプロジェクトフォルダ自体で起動した場合だけです。
Claude Code はそこではプロジェクトのフックを 1 つも実行せず、
Codex はそのフォルダからフックを実行しようとしますが、そこでは起動できません。どの設定も
`/hooks` での信頼も、これを変えることはできません。ツール自身のチャット記録がそのようなフォルダで
起動されたことを示している場合、エージェントは何も変更せず、1 行で知らせます。ツールを
終了してプロジェクトフォルダで再度起動し、示された `/aidlc --request <id>`
の行を入力してください。依頼が 1 回だけ続行されます。doctor も
同じ手順を示します。チャットがどこで開始されたかを調べるために、AI-DLC はその記録だけを読み、
それ以外は読みません。Claude Code では設定フォルダ（`CLAUDE_CONFIG_DIR`、なければ `~/.claude`）内の
`sessions/<pid>.json`、Codex ではホーム（`CODEX_HOME`、なければ `~/.codex`）の `sessions/` 配下にある
チャットのファイルの 1 行目です。そのような記録がない場合は、フックが実行されないときの通常の手順に従ってください。

### Claude managed policy blocks project hooks

`/hooks` がフックはポリシーで制限されていると報告し、設定済みのフックが 0 個と表示される場合は、`/aidlc --doctor` を実行してください。Claude Code では、doctor は macOS では `/Library/Application Support/ClaudeCode/managed-settings.json` を、Linux/WSL では `/etc/claude-code/managed-settings.json` を、Windows では `%ProgramFiles%\ClaudeCode\managed-settings.json` とそれに続いてレガシーな `%PROGRAMDATA%\ClaudeCode\` の場所を読み込みます。各候補には、隣接する `managed-settings.d/` ディレクトリ内の JSON フラグメントもアルファベット順に含まれます。トップレベルで有効な `allowManagedHooksOnly: true` は、`.claude/settings.json` で宣言されたすべてのプロジェクトフックをブロックします。マネージド設定ファイルが別の場所にある場合は `AIDLC_MANAGED_SETTINGS_PATH` を設定してください。その隣のフラグメントディレクトリも自動的に含まれます。

このマネージド設定を解除できるのは Claude Code の管理者だけです。プロジェクトフックを許可するよう依頼してください。

### Cursor project outside a git repository

Cursor が同じ計画の承認を何度も求めてくる、または選択が記録されなかったと表示する場合は、`/aidlc --doctor` を実行してください。Cursor は git リポジトリ内にないフォルダではプロジェクトフックをスキップすることがあり、フックがなければ承認は一切記録されません。doctor で "project is in a git repository" が失敗した場合は、プロジェクトで `git init` を実行し、Cursor を完全に再起動して、尋ねられたらフォルダを信頼してください。`aidlc config --harness cursor` とコピーインストーラーは、git リポジトリ外のプロジェクトをセットアップするときに同じ案内を表示します。

### Reviewer tool calls refused ("This review cannot open ...")

ユニットごとの Construction レビューの間、reviewer-scope フックは、送り出されたレビュアーのツール呼び出しのうち、兄弟ユニットの `construction/` パスに及ぶもの（stage-protocol-reviewer-unit.md の読み取りスコープの制約）を拒否します。拒否メッセージは現在のユニットを示し、レビュアーに提供されたファイルとそのユニット自身のパスを参照するよう指示します。拒否のたびに `REVIEWER_SCOPE_BLOCKED` 監査行が記録されます。あなた自身のソースツリーに AI-DLC のユニットとは無関係な `construction/` ディレクトリがある（そのため正当なレビュアーの読み取りが拒否されている）場合は、`/aidlc config set guard.reviewer-scope off` で、現在取り組んでいる作業についてレビュアーの読み取りスコープチェックだけを緩和できます（`GUARD_DISABLED` 監査行として記録され、次の作業では再び有効になります）。`AIDLC_DISABLE_REVIEWER_SCOPE_HOOK=1` は、その読み取りスコープの強制をマシン全体で無効にします。どちらの場合も、散文で記述された制約は引き続き適用されます。どちらの設定でも、チームで確保したチェックアウトが別のユニットの `construction/` サブツリーに書き込むことは許可されません。チャットでの一般的な依頼では読み取りスコープチェックは緩和されません。上記のスイッチを入力すると、エージェントがそのための設定コマンドを実行します。スコープを変更しただけでは、実行中のポリシーが緩和されることはありません。レビューが 1 つも進行していないのにレビュアーが拒否される場合は、古いディスパッチ記録が残っています。`/aidlc --doctor` のフックのドロップカウンタ（`reviewer-scope.drops`）を確認し、`<record>/.aidlc-engine/reviewer-dispatch.json` があれば削除してください（6 時間より古い記録は無視され、自動的に削除されます）。

`grep latency construction/U03-scoring/nfr.md | grep endpoint` のような通常のフィルタは許可されます。2 つ目の `grep` はパイプで渡されたテキストを検索するためです。パイプを使っても、ファイルを走査するコマンドは免除されません。再帰的な `grep`、`rg --files`、`rg -f -` には、引き続きスコープ内の検索ルート、または `rg` の場合は現在のユニットに限定した glob が必要です。`-f` で指定するパターンファイルもスコープ内になければなりません。パスを指定しないコマンドが `.` にフォールバックして拒否された場合、メッセージはそのルートが暗黙的に指定されたものであることを示します。

### Sensors are not firing

`/aidlc --status` の **Sensors** 行を確認してください。`classic` スコープの既定値では Sensors はオンです。インテントの上書きやキルスイッチによって Sensors がオフになっている場合、書き込み時の自動チェック、ゲート開始時のチェック、改訂時のチェック、承認時の改訂バックストップチェックは実行されません。`/aidlc --sensors on` でアクティブなインテントを再びオンにできます。`AIDLC_DISABLE_SENSORS=1` はそのインテントの設定より優先されます。自動チェックを再び有効にするには、これを（記録されたプロジェクトのバイパスがあればそれも）解除してください。すべてのフックはインストールされたままで、自動センサーがオフでも、診断用に明示的な `aidlc engine sensor fire` は引き続き使えます。

### Statusline shows a cost segment you don't want (or usage tracking concerns)

Claude Code では、ステージごとのトークン使用量とコストの追跡が既定でオンになっています。fold-usage フックがトランスクリプトの使用量を gitignore されたローカルの台帳（`aidlc/.aidlc-sessions/usage-ledger.json`）に記録し、ステータスラインに `↑<in> ↓<out> $<usd>` が付加され、完了時の監査イベントにはコストの集計が含まれます。どこにも送信されることはありません（メトリクスの送信は、`AIDLC_METRICS_ENDPOINT` による別途のオプトインです）。ローカルの追跡をすべてオフにするには、`AIDLC_DISABLE_USAGE_TRACKING=1` を設定してください。台帳の更新が止まり、ステータスラインのセグメントが消え、完了イベントに集計フィールドが追加されなくなります。既存の台帳はディスクに残るため、履歴も消したい場合は手動で削除してください。フラグを解除すると追跡が再開されます。

**`$<usd>` の数値はローカルでの見積もりであり、請求額ではありません。** 同梱の料金表にある**公開されている定価**から算出されます（[料金表と上書き](../reference/06-hooks-and-tools.md#rate-table-and-overrides) を参照）。Amazon Bedrock が記録されたプロバイダである場合（`CLAUDE_CODE_USE_BEDROCK=1`）、Bedrock が実際に請求する金額は、推論プロファイル、リージョン、サービスティア、交渉済みの価格やサブスクリプション価格によって異なるため、見積もりが請求書と一致しない場合があります。長いワークフローのトークン量の大半はキャッシュ読み取り（割引されたキャッシュ読み取り料金で課金されます）なので、カウントと見積もりは着実に増えていきます。これは実際の使用量であり、水増しではありません。この数値は、個人の目安として扱ってください。

**見積もりを自分の料金で算出するには**、同梱の `.claude/tools/data/model-rates.json` と同じ形の料金ファイルを `AIDLC_MODEL_RATES` に設定してください（100 万トークンあたりの USD、モデルのバージョンごとに 1 つのキーで、`opus-5-5` と `opus-5` は別々の行になります）。一部だけを記述したファイルは、そこに書かれたモデルだけを変更し、それ以外は同梱の既定値のままです。料金は使用量が記録される時点で適用されるため、変更はその時点以降のターンの算出に反映されます。台帳にすでにある合計は、記録時の料金のままです。

**発表中、画面共有中、録画中などに見積もりを表示したくない場合は、コストのセグメントをオフにしてください。** gitignore された `.claude/settings.local.json` にキルスイッチを追加します（トークン／コストのセグメントが消えるだけで、ワークフローには影響しません）:

```json
{
  "env": {
    "AIDLC_DISABLE_USAGE_TRACKING": "1"
  }
}
```

### Hook not configured

フックは、ハーネスのネイティブな設定にプロジェクト全体で登録されます。
Claude では、`.claude/settings.json` に想定どおりの `hooks`
イベントが含まれていることを確認してください。ネイティブのプロジェクトでは、進行中のワークフローを完了してから
`aidlc config --harness claude` を実行して同梱の登録を復元してください。
あなた自身のフックのエントリは保持されます。コピーチャネルのプロジェクトでは、代わりに
`bun .claude/tools/aidlc.ts config --harness claude --from <the runtime/claude root you copied from>` を使います。
`--force` は不要です。登録が復元されたときは注記で知らされます。
フローを変更する登録のイベント、マッチャー、コマンドが変わった場合、または同梱のフックが
接続されなくなった場合、doctor は失敗します。その他の AI-DLC の
登録のずれは警告になります。あなたが追加したフックは、ずれの警告の対象になりません。
チームがそのファイルで設定したステータスラインは保持されます。AI-DLC 自身の
ステータスラインはそこには同梱されていません。各自が
`/aidlc --statusline on` で自分用にオンにします。
手動でコピーした場合は、
同じバージョンの `runtime/<harness>/` アーカイブからハーネスのルート全体を
置き換えてください（コピーで `.gitignore` や `AGENTS.md` が置き換えられることはありません）。
フックのコマンドを 1 つだけ個別に修正しないでください。手動用のアーカイブは Bun 向けの構成で、
ネイティブの `aidlc` 実行ファイルを必要としません。

### Hooks disabled globally (`disableAllHooks`)

Claude Code は、どの設定レイヤーにある `"disableAllHooks": true` にも従います。エンタープライズのマネージド設定、`.claude/settings.local.json`、`.claude/settings.json`、`~/.claude/settings.json`（`CLAUDE_CONFIG_DIR` が設定されている場合はその下の `settings.json`）のいずれでもです。これが設定されていると、ファイルが存在し正しく接続されていても、**すべての**フックが何も知らせずにスキップされるため、ワークフローはあなたの返答を記録できません。`/aidlc --doctor` はこれを検出し、原因のレイヤーを示して **Hooks enabled** 行を失敗とします。その際、Claude Code のレイヤーの優先順位に従い、優先度の高い `false` は優先度の低い `true` を抑止するものとして扱います。あなたのメッセージに対してフックが 1 つも実行されなかった場合、エージェントは作業の前に以下の変更を自ら行い、Claude Code が事前にあなたの確認を求めます。

- 原因のレイヤーが**プロジェクトまたはユーザーのファイル**であれば、このプロジェクトの `.claude/settings.local.json` に `"disableAllHooks": false` を設定してください。このファイルはプロジェクトやユーザーのファイルより優先されるため、ユーザー設定を変えずにそのいずれでのオフにも対処でき、同じチャットのまま有効になります。再起動も `/hooks` も不要です。
- 原因が最も優先度の高いレイヤーである**エンタープライズのマネージド設定**であれば、プロジェクトやユーザーのファイルでは上書きできません。Claude Code の管理者にプロジェクトフックの許可を依頼してください。
- **フックをオフにする設定を付けて Claude Code を起動した**場合（たとえば `--settings '{"disableAllHooks": true}'`）も、プロジェクトのファイルでは上書きできず、doctor もそれを検出できません。その設定なしで Claude Code を再起動してください。
- **このフォルダで Claude Code がすでに開いている間に AI-DLC をセットアップした**場合、そのセッションはフックを読み込んでいません（ネイティブインストールではフックが `aidlc` を呼び出しますが、開いているセッションはまだそれを見つけられません）。Claude Code を終了してこのフォルダで再起動し、エージェントが示した行を入力してください。`/aidlc --request <id>` は止まった依頼を 1 回だけ続行し、`/aidlc` だけを入力すると作業の現在の状態から続行します。

このチェックは、ディスク上のマネージド設定**ファイル**（Linux では `/etc/claude-code/managed-settings.json`、macOS では `/Library/Application Support/ClaudeCode/managed-settings.json`、現在の Windows では `%ProgramFiles%\ClaudeCode\managed-settings.json`。`%PROGRAMDATA%\ClaudeCode\` はレガシーな副次的な場所）と、隣接する `managed-settings.d/` ディレクトリ内の JSON ファイルをアルファベット順に読み込みます。Claude Code がサポートするその他のマネージドチャネル（MDM、Windows レジストリ、リモート／サーバー管理のソース）は**検査しません**。そのため、行が合格しても、チェックが読めた設定ファイルのいずれにおいても解決後の値が `true` でないことを意味するだけで、それらのチャネルに問題がないことを保証するものではありません。マネージド設定ファイルが標準外のパスにある場合は、`AIDLC_MANAGED_SETTINGS_PATH=/path/to/managed-settings.json` でチェックの対象を指定してください。そのファイルの隣にあるフラグメントも含まれます。

### A Kiro hook switched off

Kiro IDE では、Agent Hooks パネルでどのプロジェクトフックもオフにでき、その際 `.kiro/hooks/` 内のそのフックのファイルに `"enabled": false` が書き込まれます。その状態でコミットされたファイルは、すべてのクローンでフックをオフにします。そのようなパネルを持たない Kiro CLI でも同様です。フックは他に何の兆候もなく実行されなくなります。`/aidlc --doctor` は、すべての AI-DLC フックファイルでこのフラグを読み取ります。`aidlc-guard-tool-call` がオフの場合、承認済みの計画、レビュー済みの作業、AI-DLC の記録は保護されません。`aidlc-record-human-turn` がオフの場合、あなたの返答が記録されないため、承認は見えない回答を待ち続け、チャットで入力した `/aidlc` コマンドは直接実行されません。どちらの場合もフックの名前を示す行が失敗となり、`aidlc-continue-workflow` も同様です。これは、Kiro が Stop フックによるブロックを許す環境（Kiro CLI v3。Kiro IDE では記録するだけ）でワークフローを進め続けるフックです。その他の AI-DLC フックがオフになっている場合は、何が失われるかを示す警告になります。Kiro IDE では、Agent Hooks でフックをオンに戻してください。Kiro CLI では、doctor が示すファイルの `"enabled": false` を `true` に変更し、Kiro を終了してそのフォルダで再度 `kiro-cli --v3` を起動してください。Kiro はセッション開始時にフックファイルを読み込みます。次の AI-DLC ステップも、変更ごとに一度、作業の前に 1 行で同じことを伝えます。`aidlc-record-human-turn` がオフの間は、承認時にフォルダを信頼するよう求める代わりにこれらの手順が示されます。フックが自動でオンに戻ることはありません。

---

## Kiro Read Denials <a id="kiro-ide-read-denials"></a>

### Kiro IDE or Kiro CLI v3 denies every read under .kiro/ ("deny fs_read matching")

**症状**: すべてのステージ、エージェント、プロトコルの読み込みが拒否され、ワークフローが止まります:

```text
Tool call denied by user's permissions. Rule: deny fs_read matching ".kiro/" Source: ~/.config/git/ignore
```

Kiro CLI v3 では次のように表示されます:

```text
Blocked by a permission rule (deny fs_read matching ".kiro/"); the user did not reject it. Source: ~/.config/git/ignore.
```

[Kiro IDE と Kiro CLI v3 は](https://kiro.dev/docs/kiroignore/)、git のグローバルな
除外ファイル（git リポジトリ内の場合）と `~/.kiro/settings/kiroignore` に
自動的に従います。Kiro CLI v3 はプロジェクトの `.kiroignore` も適用しますが、
`.gitignore` は適用しません。Kiro IDE では、ワークスペースの ignore ファイルが適用されるのは
`kiroAgent.agentIgnoreFiles` 設定がそれらを指定している場合だけです（既定値には
`.gitignore` が含まれ、`[]` にするとワークスペースのソースが無効になります）。
Kiro は各 ignore ファイルを個別に評価します。プロジェクトの `!.kiro/` による否定は、
グローバルな `.kiro/` ルールを打ち消しません。`git check-ignore` がそのパスを無視されていないと
報告する場合でも同じです。`permissions.yaml` の `fs_read` の allow でも拒否は解除**されません**。
[Kiro はスコープをまたいで deny を優先します](https://kiro.dev/docs/permissions/)。

`/aidlc --doctor` を実行してください。Kiro IDE または Kiro CLI v3 のインストール（`.kiro/agents/aidlc-aidlc.md` が存在する）では、
原因となっているソースを次のように報告します:

```text
Kiro ignore sources: <source>:<line> hides .kiro/
```

doctor は、エンジンがエージェントに `fs_read` で行わせる読み込みを、
エンジン自身の一覧に基づいてテストします。対象は、コンパイル済みのステージグラフで `harness.json` が選択する
すべてのステージについて、ステージファイルと、コンダクターがインラインで保持するペルソナとナレッジ
（Standard と Minimal の深度で、ディレクティブの 8 KiB の
`inline_context_paths` 上限の範囲内）、さらに `stage-protocol.md` とその
`stage-protocol-<name>.md` モジュール、そして各スキルの `SKILL.md` の横にあるファイルです。
プラグインは、どのように合成されたかにかかわらず数えられます。`stage-definition.md` のような
コントリビューター専用のプロトコルファイルは読み込まれません。`SKILL.md` ファイル、コンダクターエージェント（`agents/aidlc-aidlc.md`）、
`aidlc-common/conductor.md`、および `tools/`、`sensors/`、`hooks/`、`scopes/`、
`steering/` は、`fs_read` ではなく Kiro 自身またはエンジンが読み込むため、
数えられません。これらの一部だけを隠すルールは、
件数と影響するフレームワークのフォルダとともに報告されます。たとえば
`hides 11 of 110 framework files (.kiro/agents/)` のようになります。

グローバルなソースが一致すると、doctor は失敗します。プロジェクトの `.gitignore` と `.kiroignore` が一致した場合は
警告になります。doctor はあなたがどの Kiro を実行しているかを判別できず、Kiro IDE で
これらのワークスペースファイルが適用されるかどうかを決める IDE の設定も読めないためです。
警告には、それぞれがどこで適用されるかが示されます。Kiro CLI v3 は `.kiroignore` を適用するため、そこでは
`.kiroignore` の一致によって読み込みが拒否されますが、`.gitignore` は適用されません。

`<source>` は固定の名前です: `~/.config/git/ignore`（または
`$XDG_CONFIG_HOME/git/ignore`）、`core.excludesFile`（パスを知るには
`git config --get core.excludesFile` を実行します）、`~/.kiro/settings/kiroignore`、
`.gitignore`、または `.kiroignore`。doctor の出力はエージェントが読むため、doctor が
ignore ファイルのパス、そのルール、git のエラーテキストを表示することはありません。

doctor が評価できないソースは `not evaluated` として警告され、その `fix:` 行に
対処の方法が示されます:

- **`git is not available`**: `git` を PATH に追加し、doctor を再実行してください。それまでは、
  示されたファイルを手作業で確認してください。git のグローバルな除外ファイルは、git が読み込む
  `core.excludesFile` で、具体的なものから順に次のとおりです。環境変数によるコマンドスコープの設定
  （`GIT_CONFIG_COUNT` と `GIT_CONFIG_KEY_<n>` および
  `GIT_CONFIG_VALUE_<n>`、または `GIT_CONFIG_PARAMETERS`）、リポジトリの設定
  （`.git/config` と `.git/config.worktree`。リンクされた worktree やサブモジュールで
  `.git` がファイルの場合は、その `gitdir:` 行が示す git ディレクトリと、
  その git ディレクトリの `commondir` ファイルが示すディレクトリ）、グローバルな git 設定
  （`~/.gitconfig`、`$XDG_CONFIG_HOME/git/config` または `~/.config/git/config`、あるいは
  `GIT_CONFIG_GLOBAL` が示すファイル）、そしてシステムの gitconfig（
  `GIT_CONFIG_SYSTEM` が示すファイル、なければ git インストールのシステムファイルで、
  `/etc/gitconfig` や Git for Windows のインストール配下の `etc/gitconfig` など。
  `GIT_CONFIG_NOSYSTEM` が true の場合はスキップ）。各ファイルの
  `include.path` と、該当する `includeIf.<condition>.path` のエントリを
  再帰的にたどってください。どれも設定していない場合、ファイルは `$XDG_CONFIG_HOME/git/ignore`、
  `XDG_CONFIG_HOME` が未設定なら `~/.config/git/ignore` です。
- **`git rev-parse exit <n>`** または **`git config exit <n>`**: ディスク上にリポジトリが存在するのに、
  git がこのプロジェクトを拒否しています。プロジェクトで `git status` を実行して
  理由を確認してください。所有者が疑わしい（dubious ownership）場合は、git が表示する
  `git config --global --add safe.directory` コマンドを実行してください。その間に、
  プロジェクトの外（たとえばホームディレクトリ）で `git config --get core.excludesFile` を実行して
  git のグローバルな除外ファイルを特定し（何も出力されなければ
  `$XDG_CONFIG_HOME/git/ignore`、
  `XDG_CONFIG_HOME` が未設定なら `~/.config/git/ignore`）、それを確認してください。
- **その他の理由**、たとえば git コマンドが `did not finish` だった（タイムアウトした）場合:
  示されたファイルを手作業で確認してください。git のグローバルな除外ファイルは、
  `git config --get core.excludesFile` が表示するものです。

シェルで `GIT_CONFIG` が設定されている場合は、
`git config --get core.excludesFile` を実行する前に解除してください。たとえば
`env -u GIT_CONFIG git config --get core.excludesFile` とします（PowerShell では、先に
`Remove-Item Env:GIT_CONFIG` を実行します）。`GIT_CONFIG` は `git config` だけを
別のファイルに向けるため、解除しないと git が適用しないファイルが示されてしまいます。
doctor も同じ方法でこれを解除します。

示された行のルールを削除するか対象を絞ってから、`/aidlc --doctor` を再実行してください。
リポジトリごとの個人的な ignore は、そのリポジトリの `.git/info/exclude` に置いてください。git は
これに従い、Kiro はこれを ignore のソースとして扱いません。別のファイルでの否定や
権限の allow は、原因のルールを修正する代わりにはなりません。

---

## State File Issues

**症状**: オーケストレーターが状態の破損を報告する、またはワークフローが予期しない挙動をします。

### State file missing

状態ファイルは、Initialization の間、または `/aidlc` にスコープが指定されたときに作成されます。

- `/aidlc --status` を実行して、アクティブなワークフローがないことを確認します
- `/aidlc` または `/aidlc <scope>` を実行して、新しいワークフローを開始します

### State file corrupted

`validate-state.ts` フックは、コンパクションのたびに 2 つの必須セクション `## Stage Progress` と `## Current Status` があるかをチェックします。修復するには:

1. `/aidlc --doctor` を実行し、報告された状態、グラフ、フックの問題に対処します
2. 生成された Stage Progress の行が古くなっている場合は、状態の再同期を担うエンジンの経路を再実行します。`/aidlc` でワークフローを開始または再開するか、`/aidlc --scope <scope>` でスコープを変更して、コンパイル済みのグラフとスコープグリッドを再適用します
3. `.claude/knowledge/aidlc-shared/state-template.md` は、セクションとフィールドの取り決めとしてのみ使ってください。テンプレートからステージの行を手で復元しないでください
4. 記録を修復できない場合は、`/aidlc intent archive <name>` で廃止し（`aidlc/spaces/<space>/intents/` 配下の記録ディレクトリは保持されます）、`/aidlc` を実行して新たに開始します

---

## Dispatched Stage Timeouts

**症状**: ディスパッチされたステージ（Reverse Engineering、Practices Discovery、User Stories、Code Generation）がエラーや途中で切れた出力を返します。

### What happens

フレームワークは組み込みの再試行プロトコルに従います:

1. 縮小したコンテキストのプロンプトで**自動的に再試行**します
2. **再試行が失敗した場合**、2 つの選択肢があります:
   - **インラインで実行** — メインの会話でステージを直接実行します（サブエージェントの境界なし）
   - **スキップして後で再訪** — ステージを未完了としてマークし、後で戻ります

### Manual recovery

`/aidlc` を再実行してください。`[-]`（進行中）の状態を検出して、そのステージを続行します。最初からやり直す場合は redo と伝えてください。何が失敗したかを把握するには、`audit/` シャードのエラーエントリを確認してください。

---

## Approval Gate Stuck

**症状**: ワークフローが承認ゲートであなたの返答を待っています。

### How to proceed

求められたら返答を入力してください。選択肢は次のとおりです:

- **Approve** — 次のステージに進みます
- **Request Changes** — 修正のためのフィードバックを伝えます

Kiro IDE で、すでに返答したのにワークフローがまだ待っている場合は、
[Kiro IDE hooks not running](#kiro-ide-hooks-not-running) を参照してください。Kiro CLI では
[Kiro CLI のフックが動かない](#kiro-cli-hooks-not-running) を参照してください。

### Revision loop escape hatch

同じステージで修正が 3 回繰り返されると、3 つ目の選択肢 **Accept as-is** が現れます。これはステージを現状のまま承認して先に進みます。AI-DLC はこの回数を作業の記録に保持するため、中断して再開した後や新しいチャットで続けた場合でも、この選択肢は表示されます。

### Skipping a stage

別のステージに移動するには `/aidlc --stage <target>` を使います。間にあるステージは、状態ファイルで `[S]`（スキップ）としてマークされます。

### A reviewed document needs another change

最終レビューがすでにそのドキュメントを対象にしている場合、レビューが異なる内容を
黙って認証してしまわないよう、直接の編集はブロックされます。

- ステージがアクティブまたは承認待ちの間は、変更内容を説明して
  **Request Changes** を選んでください。この判断はゲートが開く前に記録できます。
- ステージが `[R]` の間は、`/aidlc --stage <slug>` で再開してください。
- ステージが `[x]` になった後は、レビュー済みのソースの状態を復元するか、
  `/aidlc --stage <slug>` で戻ってやり直してください。

ワークスペースのソースだけが変更され、1 回限りの復旧レビューがまだ
使える場合は、古い Review セクションを置き換える前にその復旧リクエストを開始してください。
保留中のリクエストは、古くなった状態が続いている間、そのステージまたはユニットへの書き込みだけを
一時的に許可します。レビュー済みのワークスペースのソースを復元する、判定を記録する、
または別のセッションを開始・再開すると、凍結が再び有効になります。出力ドキュメントのバイトを
復元しても、監査に記録された成果物の古さは解消されません。セッションを
再起動した後は、Review セクションを置き換える前に同じ保留中のリクエストを再試行してください。
対応する判定が記録されるまで、ゲートは閉じたままです。

### A directory source manifest is refused because it contains ignored output

**症状**: ビルド後に、`src/` のようなディレクトリの申告が
`contains ignored application source "src/main.js.map"` と返されます。

Git の ignore ルールだけでは、ファイルはレビューのソース境界から除外されません。
その境界の外にある生成物については、正確なパス（またはその出力
ディレクトリ）をリポジトリルートの `.aidlc-source-paths.json` の `exclude` に追加してから
（たとえば `{"version":1,"exclude":["src/main.js.map"]}`）、レビューを再試行してください。
ディレクトリの申告は、その横にあるソースを引き続き対象にできます。登録された `paths`
のエントリは `exclude` より優先され、境界内に残っているその他の無視されたファイルは
引き続き拒否の対象になります。正確なファイルの申告には、引き続き受け入れ可能な
ソースを指定する必要があります。除外しても、無視された出力がレビューの証拠になるわけではありません。

### An exact source-manifest file is refused because Git ignores it

**症状**: `source-manifest.json` にすでに記載されている正確なファイルが
`is ignored by Git and cannot be source-review evidence` と返されます。

無視された正確なファイルが受け入れられるのは、HEAD がすでにそれを追跡している場合か、
ソースとして明示的に登録されている場合です。Git のインデックスに追加するだけ（
強制追加を含む）や jj だけで追跡しても、HEAD に含まれることにはなりません。
レビュー対象のソースとするファイルの場合は、`{"version":1,"paths":["src/runtime.js"]}` のような登録を、
そのソースリポジトリのルートにある `.aidlc-source-paths.json` にマージしてから、
レビューを再試行してください。既存のエントリは残してください。
登録されたパスは `exclude` より優先されます。
`node_modules/` やセンサーのキャッシュなど、AI-DLC がソースとして扱わないフォルダ配下のファイルは
登録できません。代わりにそれらの正確な申告を省いてください。

ソースではない出力については、正確な申告を省いてください。ソースとして登録するのではなく、
レジストリの `exclude` リストを使って出力をソース境界の外に置いてください。
`/` で終わるディレクトリの申告には、無視された子孫に対する別個のチェックがあり、
正確なファイルの受け入れはそのチェックを変えません。

### Code Generation cannot complete because files changed that no Unit covers

**症状**: Code Generation の承認時に "N changes made during Code
Generation that no Unit's review covers (...)" と返されます。

ステージの間に、どのユニットのソースマニフェストにも記載されていない何かが
ワークスペースで変更されました。表示された行には 3 つの対処方法が示され、アシスタントは
状況に合うものを選びます。変更を行ったユニットがそのパスを記載し、もう一度
レビューを受けます。誰も意図していない変更は元に戻します。ソースではないパス（
リポジトリ内に書き込まれたツールのキャッシュ、ビルド出力、このプロジェクトのコードではない
入れ子のリポジトリ）は、`.aidlc-source-paths.json`（記録されたリポジトリや記録されたチェックアウトの場合は
それ自身のフォルダにあるファイル）の `exclude` に記載して、ソースチェックの対象から外します。除外はすぐに有効になり、
追加のレビューは不要です。その行を削除すると、パスは再び対象に戻ります。
プロジェクトの `.gitignore` がすでに無視しているパスは数えられません。また、
このフォルダで進行中の別の作業が自身のユニットのソースマニフェストに記載しているファイル
（ここで別の作業を構築している 2 つ目のチャット）や、それが記載しているフォルダ配下のものも
数えられません。それは別の作業の変更であり、その作業自身のチェックが対象にします。
完了させない作業をアーカイブすると（`/aidlc intent archive <name>`）、
その申告は終了します。Guard Policy が relaxed または off の場合は、変更は保持され、
代わりに一度だけ知らされます。

ユニットのない計画では、同じ変更に対して "the project source
changed after <reviewer> reviewed it"（1 回限りの復旧レビューを使った後は
"changed again after the one recovery review"）と返され、続けて何が変更されたかと、
ソースではないパスに対する同じ対処方法が示されます。そこでも除外は
すぐに有効になり、追加のレビューは不要です。

### Plan Approval asked twice for the same plan

**症状**: すでに承認した計画について、Code Generation が Plan Approval の質問を
再び提示します。

Plan Approval は、計画、ユニットテストの手順、Testing Contract の内容、
ターゲット、そして現在のステージの試行に結び付きます。`/aidlc` の再実行、
セッションの再起動やコンテキストのコンパクション、Stop フックのプローブ、
`/aidlc --status` では、承認は再び開かれ**ません**。計画のチェックボックスに印を付けても
再び開かれることはなく、レビューを記録しても計画に触れることはありません。

同じターゲットと試行について、計画、テスト手順、Testing Contract を編集して承認が
再び開かれるのは、Guard Policy が `strict` の場合だけです。`relaxed` または
`off` の場合は、更新された内容で作業が続き、元の承認記録はそのまま残ります。
その記録が、あなたが編集を承認したと主張することはありません。実際に有効な設定は
`/aidlc --status` の `Guard Policy:` 行に表示されます。その場合でも、計画をもう一度レビューするよう依頼できます。

計画について一切尋ねられずにコード生成が始まった場合、この作業では計画の承認が
オフになっています。status にはその設定の出どころが表示されます（たとえば
`Plan Approval: off (from scope poc)`）。各計画を示す行には、いつでも計画を
一緒に確認するよう依頼できると書かれています。すべての計画について尋ねるよう
依頼することもできます。[プラン承認](13-customization.md#plan-approval) を参照してください。

Testing Posture、スコープ、テスト戦略、プロジェクトタイプの変更も、同じインテント、
ターゲット、試行の中では同じルールに従います。必要に応じて現在の契約と手順を
更新してください。フェンスが下げられている場合は、それらの入力が変わったという理由だけで
再び承認を求めることなく、実行を続けられます。

再び尋ねられた場合は、何が変わったか、どのルールが当てはまるかを確認してください:

- 計画承認のフェンスがオンの間の、計画の内容または埋め込まれた Testing Contract
  （タスクマーカーへの印付けや、レビュー記録が存在する前に記録されたレビューが残した
  末尾の `## Review` セクションを除く）
- ユニットテスト手順の内容（1 バイトでも）: 手順は開発者にそのまま渡されるため、
  バイト単位で厳密に結び付きます。計画承認のフェンスがオンの場合、承認後に手順へ
  セクションを追記すると承認は再び開かれます
- 計画承認のフェンスがオンの間の、Testing Posture、スコープ、テスト戦略、
  プロジェクトタイプ（ビルドのレビュー後に行った Testing Posture の変更、たとえば
  学びのステップで残すことにしたテストの慣行は、次の計画から適用され、そのビルドの
  承認を再び開くことはありません）
- アクティブなインテント、ユニット、ステージのターゲット
- ステージの試行: 後方へのジャンプ、Request Changes、ゲートでの却下、
  ワークフローの再始動

質問には、何についての質問かが示されます。承認後にそれ以外のコードが動いても（`git
pull` や別のユニットの取り込み）、再び尋ねられることはありません。ビルドは続き、
どのファイルが動いたかが伝えられます。例外は `strict` の下でユニットを並列に
ビルドする場合です。承認後の変更（あなたの編集やコミット、`git pull`）があると、
ユニットのセットアップ前に再び尋ねられ、それを元に戻すか Guard Policy を `relaxed` に
設定するまで、次のユニットの取り込みは止まります（
[Guard Policy](13-customization.md#guard-policy) を参照）。また、あなたの返答から
ビルドまでの間に起きること、
つまり長いチャットのコンテキストのコンパクション、作業の一時停止と再開、別の質問が
先に出てくることでも、再び尋ねられることはありません。先にもう一度確認したい場合は、
これらの直後も含めて "review the plan"（計画をレビューして）と伝えてください。
さらに何かがビルドされる前に、計画が再び承認のために表示されます。

### Plan Approval is not recorded

**症状**: Plan Approval の質問に答えたのに、AI-DLC がまた同じ質問を表示します。

あなたの返答は、この作業に関するどのチャットからでも、あなた自身の言葉（"1"、
"approve"、"looks good"、"rename the handler"）で有効になります。エージェントは
あなたの選択を、あなたの言葉を添えて記録します。質問が再び表示されるのは、
エージェントが選択を記録できなかった場合だけです。最も多いのは、質問が表示される前に
返答が届いた場合や、ハーネスが返答を AI-DLC に渡さなかったチャットで答えた場合です。
表示された質問に答えてください。何度も表示される場合は `/aidlc --doctor` を実行してください。
プロンプトフックが実行されないハーネスでは、返答を一切記録できません。
質問が待っている間に長いチャットのコンテキストがコンパクションされても、
質問は開いたままなので、あなたの返答は引き続き有効です。

プロジェクトのファイルをすべて読み取れない、またはワークスペースのソースを結び付けられない
（ループするリンク、大きすぎて読めないツリー、プロジェクト外のソースパス）と AI-DLC が
伝えた場合、何が起きるかは設定によって異なります。Guard Policy が strict で計画の承認がオンの場合、
ビルドの出発点を示せるものがないため、計画はまだ承認できません。それ以外の場合は、
計画の質問は通常どおり尋ねられ（計画の承認がオフの場合は、計画は書かれたとおりに
ビルドされます）、プロジェクトのファイルをチェックせずにビルドが始まり、その旨を示す行が
1 行表示されます。ファイルをチェックさせるには、メッセージが示すソース境界を
修復してください（問題のパスを `.aidlc-source-paths.json` の除外リストに追加する。これは
`{"version":1,"exclude":["<path>"]}` のような形式です。除外したディレクトリ配下にある
実際のソースはその paths リストで申告する。または壊れたシンボリックリンクを削除する）。その後 `/aidlc`
（Codex では `$aidlc`）を実行してください。`/aidlc --doctor` の "Workspace source boundary binds"
チェックが、失敗しているパスと、これらのどれが当てはまるかを示します。

プロジェクト内に置かれた有効な Git チェックアウトは、リンクされたワークツリー経由も含めて、
jj とオブジェクトストアを共有できます。古い AI-DLC のインストールが、jj が作成した
pack から Git が読み取れるコミットについて "no resolvable HEAD commit" と報告する場合は、
AI-DLC を更新し、プロジェクトのインストールを更新してください。リポジトリの再パックや
jj の設定変更は不要です。HEAD オブジェクトが本当に欠けている、または読み取れない場合は、
引き続き修復が必要です。

計画の Testing Contract が拒否された場合、メッセージは 3 つの原因のいずれかを示します。
ブロックがない、有効な JSON でない、またはレンダリング後に変更された、のいずれかです。
修復方法は 3 つとも同じで、`render` を再実行し、`## Testing Contract` セクション全体を
置き換えます。ブロックが変更されている場合は、通常、シェルコマンドがファイルを書き直して
文字を再エンコードしたことが原因です（たとえば PowerShell の
`Set-Content`）。成果物は代わりにファイル編集ツールで編集します。あなた自身の編集で
ブロックが壊れた場合は、アシスタントがそれを修復し、編集後の計画をビルドするか一度だけ尋ねます。

**旧来の Kiro IDE のウィンドウ。** 入力されたテキストを AI-DLC に渡さない Kiro IDE の
ビルドでは、計画の承認はピッカーでのみ行われ、旧来の記録された判定と回答の
ステップを経由します。その場合、回答はそれが行われたチャットセッションに結び付きます。
プロジェクトのファイルをチェックできない場合でも、それを選ぶか、そう伝えることで、
アシスタントに計画をそのままビルドするよう依頼できます。これはファイルのチェックなしの
あなたの承認として記録されます。Kiro IDE を更新すると、どのチャットからでもあなた自身の言葉で
答えたり、ファイルを編集したりできるようになります。

---

## A Bolt attempt was set aside — getting the files back

復旧処理では、レビュー済みのファイルが再び変更され、許可されている 1 回の再レビューを
すでに使っていた場合、試行が脇に置かれることがあります。復旧の abort の `--discard`
は、古いチェックアウトとブランチを削除する前に、追跡済みファイルと無視されていない未追跡ファイル
（チェックアウトがすでにない場合は残っているブランチの先端）とレビュー済みのソース ref を
ローカルの Git に保存し、新しい試行を開始できるようにします。
`--discard` なしの通常の Abort はチェックアウトをそのまま残し、
`parked_ref: null` を返します。`parked_stamp`、`parked_mode`、`parked_repo`、
`restore_operation`、`restore_hint`、`restore_hint_error`、`parked_excludes` は含まれません。
discard の後、アシスタントは試行を脇に置いた理由を伝え、表示用のヒントを
レンダリングできたかどうかにかかわらず、`restore_operation` がある場合にのみ
復元を提案します。
成功した abort はいずれも `reason: "aborted"` を保持し、渡された
`--reason` のテキストを追加の `abort_reason` フィールドにそのまま返します。試行が退避された場合、
結果には保存された記述子が含まれます:
`parked_ref`、`parked_stamp`、`parked_mode`（`snapshot`、`branch-tip`、または
`evidence-only`）、`parked_repo`（プロジェクトルートの場合は `null`、それ以外は
兄弟リポジトリの名前）。

ファイルが保存されている場合は、復元を依頼してください。アシスタントは保存された結果の
`restore_operation` を使います。スタンプがわかっている場合、ルートは `worktree`、引数は
`["restore", "--slug", slug, "--parked", stamp, "--repo", repo ?? ".", "--intent", recordDirName, "--space", space]` です。
最後のセレクターが所有するインテントを固定します。アシスタントはインストール済みの `{{INVOKE}} engine worktree <args...>`
ルートを、列挙された各引数をそれぞれ独立した argv 引数としてそのまま渡して呼び出し、
シェルコマンドとして連結することは決してありません。`restore_hint` は人が読むための表示テキストにすぎず、
アシスタントの実行入力ではありません。メインのプロジェクトのチェックアウトから
実行する場合の、同等の手動コマンドは次のとおりです:

```bash
aidlc engine worktree restore --slug <slug> --parked <stamp> --repo <name|.> --intent <record-dir-name> --space <space>
```

この操作には、兄弟リポジトリの場合は `--repo <name>`、ルートの場合は
`--repo .` が必ず含まれます。任意の表示用ヒントは、選択されたシェル向けに安全に
レンダリングされます。ネイティブインストールでは上記のプレフィックスを使い、Bun ベースのコピー
インストールでは `bun .claude/tools/aidlc-worktree.ts` を使います（ハーネスディレクトリは
お使いのものに置き換えてください）。ハーネスディレクトリが無効であるなどの理由で安全なレンダリングに
失敗した場合、結果では `restore_hint` が省かれ、代わりに理由を示す `restore_hint_error`
が返されます。型付きの操作は引き続き利用できます。これは証拠だけが保存されたという
意味でも、復元の提案を取り下げるという意味でもありません。

操作に含まれる正確なスタンプ、リポジトリ、所有インテントのセレクターは省かないでください。
`--parked` がないと、restore は最新の保存済み head を選びます。手動で呼び出す場合、1 つの
リポジトリでしか見つからない正確なスタンプは、一般的な slug の曖昧さより先にそのリポジトリを選びます。
それでも選択が曖昧な場合は、doctor が示す正確な操作を、`--repo <name>`
またはプロジェクトルートなら `--repo .` を付けて使ってください。

保存された名前空間はわかっているが discard の記述子が利用できない場合、
フォールバックは `parked_ref` を保持し、名前空間からスタンプを厳密に解析できる場合にのみ
`parked_stamp` を導き出します。それ以外の場合 `parked_stamp` は `null` です。
リポジトリ、保存モード、最新試行の選択を推測する代わりに、`parked_mode: null` と
`parked_repo: null` を報告し、`restore_operation`、`restore_hint`、
`restore_hint_error`、`parked_excludes` を省きます。
その代わり、`recovery_hint` が、doctor を実行して脇に置かれた試行とその正確な復元コマンドを
一覧表示するよう求めます。このヒントは単なる案内であり、実行可能な操作ではありません。
アシスタントは、フォールバックだけを根拠に、どのファイルが保存されたかを主張したり復元を
提案したりしてはなりません。名前空間が保存されなかった場合、`parked_ref` は `null` で、
`parked_stamp`、`parked_mode`、`parked_repo`、`restore_operation`、`restore_hint`、
`restore_hint_error`、`parked_excludes`、`recovery_hint` は含まれません。

レビューの証拠だけが残っていた場合、discard は `parked_mode: "evidence-only"`
と `parked_commit: "-"` を報告します。Abort は ref、スタンプ、モード、リポジトリを保持しますが、
`restore_operation`、`restore_hint`、`restore_hint_error`、
`parked_excludes` は省きます。アシスタントは "Nothing of its
working files remained to save; only its review evidence was kept."（作業ファイルは何も残っておらず保存できませんでした。レビューの証拠だけを保持しました）と伝え、
復元は提案しません。その試行を restore で選択すると、`--raw` を付けても
`no restorable files were parked for <slug> <stamp>; only review evidence was kept` で拒否されます。

復元に成功したら、返された `worktree_path` を開いてください。通常は
`.aidlc/restored/bolt-<id8>_<slug>-<stamp>` です（アップグレード前の試行の場合は記録されている旧来の名前）。これは新しく進行中の試行とは別のものです。
ファイルを復元しても、古い試行が再開されたり、そのレビューが最新になったりすることはありません。
これはローカルでの復旧であり、リモートのバックアップではありません。古いチェックアウトがすでに
なかった場合は、残っていたブランチの先端とレビューの証拠だけが保存されます。ブランチも
なかった場合は、レビューの証拠だけが残ります。

**制限は `parked_mode` によって異なります:** スナップショットは
`parked_excludes: ["ignored files", "eol/text=auto normalization"]` を報告します。無視された
未追跡ファイルは保存されず（追跡済みファイルは含まれます）、保存時に正規化によって
改行コードが変わることがあります。正規化されたバイトは、`--raw` を使っても
元に戻せません。ブランチの先端の場合は代わりに
`["uncommitted files (no working tree existed)"]` を報告し、アシスタントは
"I kept its committed work; there were no uncommitted files to save."（コミット済みの作業は保持しました。保存すべき未コミットのファイルはありませんでした）と伝えます。
スナップショットは保存された blob を直接復元し、ブランチの先端は `--raw` が指定されない限り
通常のチェックアウト時の変換を使います。
フィルターの失敗や raw での復旧の詳細については、[restore コマンドのリファレンス](12-cli-commands.md#aidlc-engine-worktree-restore-recover-files-from-a-set-aside-attempt)
を参照してください。

保存された試行を見つけるには、`/aidlc --doctor`（または `aidlc doctor`）を実行してください。情報表示用の
**Parked attempts** セクションには、slug、スタンプ、経過日数、モード、復元済みチェックアウトの
有無、そして正確な `--parked <stamp>` と明示的な `--repo <name>` または
`--repo .` の引数に続けて所有する `--intent` と `--space` のセレクターを付けた、型付きの復旧操作が一覧表示されます。JSON の各エントリには
`purge_operation` があり、復元可能なエントリにだけ `restore_operation` があります。いずれも
ルートが `worktree` で、引数はシェルコマンドの文字列ではなく argv としてそのまま渡します。
任意の `restore_command` と `purge_command` は安全な人向けの表示テキストです。
レンダリングに失敗した場合は対応するコマンドが省かれ、操作は残したまま、
`restore_command_error` または `purge_command_error` がその理由を説明します。証拠のみの
エントリには、purge の操作と、そのコマンドまたはエラーのフィールドだけがあります。ありえない日付や時刻は、JSON では
`age_days: null` になり、人が読む出力では `unknown` と表示されます。保存された試行は警告や
失敗ではありません。

doctor は、名前空間付きの試行はそのインテントのレジストリ UUID によって、
旧来の試行は `WORKTREE_DISCARDED` の正確な `Parked ref` の来歴によって解決します。
所有者が不明または曖昧なものや、帰属のない旧来の退避は一覧に表示されません。
不要になったら、まず復元済みのチェックアウトを削除してから、次を実行してください:

```bash
aidlc engine worktree purge --slug <slug> --parked <stamp> --repo <name|.> --intent <record-dir-name> --space <space>
aidlc engine worktree purge --slug <slug> --older-than 30 --repo <name|.> --intent <record-dir-name> --space <space>
```

どちらのスタンプセレクターも付けずに purge を実行すると、選択したインテントの Bolt について保存されたすべてのスタンプが削除されます。
`--older-than` は 0 以上の有限の日数を受け付け、UTC のスタンプのタイムスタンプが
それより厳密に古いものを選びます。`-N` サフィックスは無視されます。厳密なカレンダーパーサーは、
ありえない日付や時刻を正規化せずに拒否します。経過日数で絞り込む purge は
解析できないスタンプを残し、`skipped_unparseable` で報告します。この JSON
配列は常に存在し、該当がなければ空です。正確なスタンプ指定またはすべてのスタンプを対象とする purge
では、解析できないスタンプも削除できます。`--older-than` は
`--parked` と組み合わせられません。選択した復元済みチェックアウトが存在する間は、移動されていても
purge は拒否されます。進行中の Bolt のチェックアウトを削除することは決してありません。コピー
インストールでは同じ Bun ツールのプレフィックスを使ってください。[purge のリファレンス](12-cli-commands.md#aidlc-engine-worktree-purge-remove-recovery-refs) を参照してください。

restore と purge は、何かを選択したり変更したりする前に、未知のフラグや重複したフラグを拒否します。
コマンドリファレンスにあるフラグだけを使ってください。復旧用のセレクターと
doctor は、プロジェクトルート（`.`）と、同じ slug の `WORKTREE_CREATED` または
`WORKTREE_DISCARDED` の監査の `Repo` フィールドに記載された有効な Git リポジトリを、
それらの兄弟の名前がシンボリックリンクであっても受け付けます。フレームワークは、
試行のワークツリーや退避を記録したまさにその場所から復旧できます。この受け入れは slug 単位であり、
他の slug の記録がこの slug のリポジトリの集合を広げることはありません。現在または過去の
インテントのリポジトリ一覧に含まれているというだけでは、シンボリックリンクは受け入れられません。
インテントの一覧の候補や、記録されていない検出済みの兄弟は、正規化したパスが
正規化したワークスペースルートの直下にとどまる、実在する直接の子ディレクトリでなければなりません
（`isWorkspaceRepoDir`）。任意のパスや、同じ slug の監査の来歴がないシンボリックリンクの
エイリアスは拒否されます。これらの復旧用セレクターは、進行中の create/discard コマンドや、
必要な人の同意を変えるものではありません。

---

## Context Compaction

**症状**: Claude Code が以前の会話のコンテキストを要約しました。セッションが直近のやり取りを「忘れた」ように感じられることがあります。

### What is preserved

記録ディレクトリの成果物、`aidlc-state.md`、`audit/` シャードはすべてディスク上に残ります。失われるのは、メモリ上の会話のコンテキストと、まだファイルに書き込まれていない進行中の作業だけです。

### How to recover

コンパクションの後で `/aidlc` を実行してください。フレームワークは次のように動作します:

1. `aidlc-state.md` を読み込んでワークフローの位置を把握します
2. 作業が止まったところから続けます

進行中だったステージをやり直させたい場合は、"redo this stage"（このステージをやり直して）と伝えてください。

### The build stops after a compaction

**症状**（GitHub Copilot）: コードの計画を承認してビルドが始まったが、チャットがコンパクションされた後、アシスタントが試みるビルドのステップがすべて拒否されます。

あなたの承認、ワークフローの状態、すでに書き込まれたすべてのファイルは保持されます。チャットから失われるのは、アシスタントがまだファイルに書き込んでいなかった内容だけです。コンパクションの後、アシスタントは何かをビルドする前にステップを読み直す必要があります。アシスタントは自分のコマンドとして `next` を実行し、それによって承認済みのビルドが、あなたに再び尋ねることなくそのまま返されます。その `next` が実行されるまで、`/aidlc --doctor` はそのステップを古くなったものとして、時刻と理由（チャットがコンパクションされた、またはステップの発行後にワークフローの状態が変わった。わかっている場合は何が変わったかも示します）とともに表示します。

---

## Audit Log Growing Too Large

**症状**: 長期のプロジェクトで、このクローンの監査シャードが数千行にまで大きくなりました。

長期プロジェクトのシャードが大きくなるのは設計どおりです。そのままの場所に置いておいてください。エンジンは `audit/` 内のすべてのシャードを読んで、あなたがどのステージを承認し、どのユニットを完了したかを把握します。そのため、シャードを `audit/` の外に移すと、その作業は未完了と見なされ、エンジンが再び割り当ててしまいます。（PreToolUse ガードは、エージェントのファイルツールやシェルツールによる `audit/` への書き込みをすべて拒否します。）

### Git considerations

`audit/` シャードはコミットされます（gitignore されません）。[コミットするものと Gitignore するもの](14-artifacts-reference.md#what-to-commit-vs-gitignore) を参照してください。各クローンは自分専用の `<host>-<clone>.md` シャードに書き込むため、同時に追記してもマージの衝突は起きません。

### Moved, copied, or synced projects

`aidlc/.aidlc-clone-id` には、このクローンのトークンと、最初に使われたホスト名が記録されます。そのため、マシン名が変わったり、フォルダが別のノート PC にコピーまたは同期されたりしても、シャード名は変わらず、作業は 1 つのシャードで続きます。新しく `git clone` したものには、独自のトークンとシャードが割り当てられます。1 つのフォルダの 2 つのコピーは 1 つのクローンなので、2 人（または 2 台のノート PC）が同時に作業する場合は、それぞれに別の `git clone` を用意してください。同期ツールが `<host>-<clone> 2.md` のような競合コピーを残した場合、AI-DLC は元のシャードと共通する行を一度だけ読むため、完了した作業は引き続き完了として数えられ、コピーはそのまま残しておけます。チームでのユニット所有を使っている場合、ユニットの取り込みは、コピーを含め想定外のファイルがあると引き続き止まり、そのファイルを示します（`Unit landing requires a clean source worktree; commit or stash: ...`）。

以前のリリースのときにすでにそのようなコピーがプロジェクトにあり、そのまま作業を続けていた場合、コピーが存在していた間に完了したユニットが、アップグレード後にもう一度割り当てられることがあります（コピーが現れる前に完了したユニットは再び完了として数えられます）。何も削除されません。ユニットのファイルは残っているので、ほとんどのステージでは、もう一度完了させても記録されるだけです。Code Generation のユニットでは、計画の承認をもう一度求められ、ブランチにすでにあるコードから再びビルドすることがあります。これは一度だけで繰り返されることはなく、今後予定されている対応でそれもなくなります。

限られたケースが 1 つあります。独自のワークツリーでビルドされた Code Generation のユニットで、アップグレード時に完了していたがまだマージされていなかったものです。これを再びビルドすると、`Worktree directory already exists` または `resume requires the completed, merged prior Bolt` で止まります。`aidlc engine worktree discard --slug <unit>` を実行してください。このコマンドはそのワークツリーを削除する前に以前の試行のファイルを脇に保存し、次の `/aidlc` でユニットが再びビルドされます。以前のファイルが必要になったら、`aidlc engine worktree restore --slug <unit>` でいつでも別のチェックアウトに戻せます。

---

## Lock Files Left Behind

**症状**: フックが少しの間止まったように見えてからスキップされます。その後の監査エントリが書き込まれません。

監査フックは、同時書き込みを防ぐために `mkdir` ベースのロック（`lib.ts` 経由）を使います。フックが中断されると、ロックディレクトリが残ることがあります。ロックファイルはシステムの一時ディレクトリ（`os.tmpdir()` -- 通常は macOS/Linux では `/tmp/`、Windows では `%TEMP%`）に作成されます。

### Finding stale locks

```bash
# macOS / Linux
ls -la /tmp/.aidlc-*

# Windows (PowerShell)
Get-ChildItem $env:TEMP -Filter ".aidlc-*"
```

ロックディレクトリは、システムの一時ディレクトリ内に `.aidlc-audit-<hash>.lock` と `.aidlc-subagent-<hash>.lock` という名前で作成されます。

### Clearing stale locks

まず `/aidlc --doctor` を実行してください。doctor が自動で削除するのは、終了していることが
証明できる世代、作成世代が一致しなくなった再利用された PID、所有者スタンプが最後まで書かれなかった
（欠けている、空、または途中で切れている）古いロックだけです。一致する、または不明な
生存中の世代、有効な所有者を示さないスタンプ、読み取れないスタンプは、
報告されますが削除されません。

手動でクリーンアップする前に、該当するロックを使っているすべての AI-DLC プロセスを停止し、
所有者とプロジェクトが静止していることを確認して、診断用の証拠を保存してください。
調べるのは名前が示されたロックだけにし、ロックディレクトリを一括削除しないでください。
ロックと、所有者スタンプ付きの `.reap` 復旧ゲートは一時的なもので、必要に応じて
再作成されます。`.gate-mutex` ファイルは永続的なアドバイザリーロックのアンカーで、
一時ディレクトリに空のまま残ることがあります。空のファイルは、古い所有者がいる証拠ではありません。
プロジェクトのトランザクションロックには、[上記の所有者チェック](#transaction-lock-ownership)を使います。

---

## Statusline Issues

### Shows "ready" when workflow is active

ステータスラインは `aidlc-state.md` の `**Lifecycle Phase**` フィールドを読み取ります。そのフィールドがない、または空の場合は `[AIDLC] ready` にフォールバックします。

**対処:** `/aidlc --doctor` を実行して状態ファイルの整合性を確認してください。`## Current Status` セクションに `**Lifecycle Phase**` のエントリがあることを確認してください。

### Shows stale data

想定どおりの動作です。ステータスラインは、次に状態ファイルが書き込まれたとき（通常はステージの遷移時）に更新されます。

### Not appearing at all

1. AI-DLC のステータスラインは、セットアップ時に選んだか、オンにした場合にのみ表示されます:
   `/aidlc --statusline on`（Codex では `$aidlc --statusline on`）を実行してください。Claude
   Code では、あなた自身の `.claude/settings.local.json` に設定されます。組織のマネージド設定が
   選んだステータスラインがある場合は、そちらが優先されます。
2. `aidlc doctor` を実行し、報告されたネイティブコマンドやホストの接続を修復してください。
3. コピーインストールの場合は、Bun がホストプロセスの PATH 上にあることを確認してください。
4. 状態ファイルがない場合、`[AIDLC] ready` と表示されるのは想定どおりです。

---

## Using `--doctor`

`--doctor` ユーティリティコマンドはセットアップを検証します。何かがおかしいと思ったら実行してください:

```
/aidlc --doctor
```

チェックする項目は次のとおりです: 前提条件（`bun`）、フックの利用可否（`settings.json` で接続されたすべてのフレームワークフックが `.claude/hooks/` に存在する必要があり、接続されているのに存在しないフックは明示的に失敗します。有効なカスタムの非 AI-DLC `statusLine` がある場合は、接続されていない `aidlc-statusline.ts` を意図的に対象外とします）、フックがグローバルに無効化されていないこと（Claude Code のいずれかの設定レイヤーで `disableAllHooks: true` に解決されると明示的に失敗します）、マネージドなプロジェクトフックのポリシー（`allowManagedHooksOnly: true`）、プロジェクト構造（`settings.json`）、ワークスペースシェルの準備状況（`.claude/` + `aidlc/spaces/default/memory/`）、状態と監査の整合性（ワークフローの Status を記録された `WORKFLOW_COMPLETED` と、また Stage Progress の各チェックボックスを現在の試行について監査が記録したステージの開始と完了と照合します。監査では開始済みなのにチェックボックスが `[ ]` のままのステージがあると、ワークフローはそのステージの完了を拒否します。警告には変更すべき正確な行が示されます。チームでのユニット所有では、ユニットごとの Construction のチェックボックスは Unit Progress から導出されるため、照合されません）、フックのハートビート、グラフの整合性（循環がなく、すべてのグラフエントリにファイルがある）、**Composed plugin surface**（有効なプラグインのステージがコンパイルされている、コントリビューションのサイドカーとターゲットが有効である、記録された構造的な追加と文章の断片が存在し変更されていない）、選択に応じたプラグイン作成のチェック、全 11 スコープにわたるスコープの検証、**Composed scope durability**（コンポーザーが作成したすべてのスコープが実際の計画に解決されること。グリッドの列がないスコープファイル、まだ投影されていない永続的な `aidlc/scopes/<name>.md` 記録、解決できないスコープを示す実行可能なワークフローはいずれも失敗し、コンパイルで原因に対処できる場合は `graph compile` が対処方法として示されます。裏付けとなる記録がない欠けた列は別に報告されます。コンパイルは、いずれかのステージが宣言しているスコープについてのみ列を出力するためです）、ステージのスキーマとグラフの参照、スコープ間のキーワードの重複。合格するアドバイザリーの行には、グラフの読み込み順によって生成元が曖昧になる消費対象の成果物についての **Duplicate producers**、**Rule drift**（ライフサイクル上古くなった重複は stale-suppressed として別に報告されます）、**Paired sensor coverage**、`HUMAN_TURN` のないステージやゲートの台帳、24 時間を超えて人の対応を待っている承認ゲート、プラグインのアドバイザリーチェック、コミットされていないワークスペースの記録、進行中の compose の質問、新しいバックグラウンドサブエージェントの状態、以前のバージョンが compose の質問用に書き込んだファイル（もう読み込まれないので削除して問題ありません）、そして `repos.json` が存在する場合は、宣言されたリポジトリとマネージドな `.gitignore` のずれが含まれます。2 時間より古いバックグラウンドサブエージェントのエントリは、正確な `rm aidlc/.aidlc-*` の対処方法とともに失敗します。doctor がそれを削除することは決してありません。**Hook drops** は条件付きです。暗黙のうちに機能が低下したフック（たとえばコントリビューションを適用できなかったプラグインの compose や、失敗した再コンパイル）は、重大度タグ付きの行を `<hooks-health>/<hook>.drops` に記録します。`[degraded]` のドロップは doctor を**失敗**させます（そのため CI のゲートが中途半端に適用されたプラグインを検出できます）。それ以外のドロップは、各フックで最も多い理由（それぞれ最初のコロンまで。詳細はファイルに残ります）を示す合格の行になり、そのフックの最新の失敗が 24 時間以内であれば `Hook failures, the latest within the last day` の警告に引き上げられます（1 日経つか、ファイルを削除すると解消されます。`[advisory]` の行で引き上げられることはありません）。フックの通常の判断、たとえば先にあなたが答える必要があるため Stop フックがターンを終了させる場合などは `<hook>.trace` に書かれ、決して数えられません。プラグインの compose フックは実行のたびに drops ファイルを書き直すため、原因を直して再度 compose すれば自然に解消されます。問題がない場合と警告のみの場合のレポートは終了コード 0 で終了し、失敗したチェックが 1 つでもあれば終了コード 1 で終了します。`--verbose` がない場合、正常な行はセクションごとに折りたたまれますが、警告と失敗はすべて表示されたままです。どちらの場合もレポートは標準出力に書き出されます。コアのチェックは**読み取り専用**です。まだインテントのない新しいシェルでは何も作成しないため、最初のインテントを作成する前でも安全に実行できます。プラグインのチェックは、慣例として読み取り専用であることが求められているインストール済みのプラグインのコードを実行しますが、ランタイムがその性質を強制することはできません。インテントが存在すると、doctor は `HEALTH_CHECKED`（と `GUARDRAIL_LOADED`）の監査行を記録します。

Kiro IDE と Kiro CLI v3 では、`.kiro/` を隠すルールがないか、ignore の各ソースを個別にチェックし、ファイルと行を示します。グローバルなソースでの一致は失敗となり、ワークスペースのソースでの一致は警告となります。doctor にはどの Kiro を実行しているかわからないためです。Kiro IDE では `kiroAgent.agentIgnoreFiles` によってそれらが適用されるかどうかが決まり、Kiro CLI v3 は `.kiroignore` を適用しますが `.gitignore` は適用しません。[Kiro Read Denials](#kiro-read-denials) を参照してください。Cursor 以外のすべてのハーネスでは、プロジェクトで AI-DLC のフックが一度も実行されていない場合に "AIDLC hooks have not run in this project yet"（このプロジェクトではまだ AIDLC のフックが実行されていません）という警告も表示します。最初のチャットメッセージの前であれば想定どおりです。メッセージを送った後であれば、その対処にはお使いのツールでの手順が示されます（たとえば [Kiro IDE hooks not running](#kiro-ide-hooks-not-running) や [GitHub Copilot hooks not running](#github-copilot-hooks-not-running) を参照）。Kiro CLI では、フックが一度も実行されていないワークフローは、フックに必要なエンジンを示して "Hooks have never executed" で失敗します。[Kiro CLI のフックが動かない](#kiro-cli-hooks-not-running) を参照してください。

コミットされていない記録の行の隣にある **Workspace record visibility** のアドバイザリーは、
config の後に追加されたユーザーの ignore ルールを検出します。ルールのファイル、
行、隠れているコミット済みの記録のパスを示します。個々の記録を強制的に追加するのではなく、
ルールの対象を絞ってください。この警告は doctor の終了コードを
変えません。記録が見えている場合、Git リポジトリの外にある場合、
または Git が利用できない場合、この行は表示されません。

Claude Code では、doctor はマシンで管理される `managed-settings.json` と、アルファベット順の `managed-settings.d/` のフラグメントも読み取ります。実際に有効な `allowManagedHooksOnly` の値が `true` の場合、組織のポリシーによって、プロジェクトの `.claude/settings.json` で宣言されたすべてのフックがブロックされます。そのポリシーを解除できるのは Claude Code の管理者だけです。そのポリシーが設定されていないのに、ワークフローが進んだ後もハートビートがない場合は、このプロジェクトの `.claude/settings.local.json` で `"disableAllHooks": false` を設定してください。同じチャットで有効になります。
ワークフローに問題がある場合、`--doctor` は構造化された所見（未解決のゲート、ランタイムグラフの欠落、冷えたフック、その他の「先に進まない」原因）を一覧表示する **Workflow diagnosis** セクションも出力します。これは `--doctor --export` がレポートに書き出すのと同じ分析です。

各チェックが何を検証し、失敗をどう修正するかの詳細については、[CLI コマンド](12-cli-commands.md#aidlc-doctor-health-check) を参照してください。

---

## Sharing a Diagnostic Report

ワークフローが止まっている、または挙動がおかしい場合（開かないゲート、先に進まない
ステージ、承認済みのレポートが繰り返し拒否されるなど）に、メンテナーに見てもらいたいときは、
次を実行してください:

```
/aidlc --doctor --export
```

これは新たに `--doctor` を実行してから、小さな**秘匿化済み**の診断レポートを
`aidlc/diagnostics/` に書き出します（`--output <dir>` で変更できます）。システムに `tar` が
あれば、タイムスタンプ付きの `.tar.gz` にまとめます。ない場合はレポートのディレクトリをそのまま残し、
自分で圧縮するよう伝えます。そのアーカイブ（または
ディレクトリ）を共有してください。含まれるのは診断と秘匿化済みの証拠で、**あなたの作業
成果物は含まれません**。ワークスペースのソース、状態・監査・ランタイムグラフの生ファイル、
成果物・コントリビューション・質問・メモリの本文は含まれません。パスは正規化され、
インテント ID とユニット名はハッシュ化され、シークレットらしき値は除去されます。

レポートは監査証跡からワークフローのタイムラインを再構成し、
決定論的な「条件→対処」のルールを実行します。検出する原因のうち最も多い 2 つは次のとおりです:

- **未解決の承認ゲート** — ゲートが一度も解決されていないステージは、
  「先に進まない」原因として最も多いものです。
- **ランタイムグラフの欠落 / 冷えたフック**: ランタイムグラフがない、または
  フックが最後に発火したのが、インテントに記録された最新のステージや承認のステップより
  1 日以上前である場合は、再コンパイルが実行されなかったことを示しています。チャットを開くこと、
  プロンプト、doctor 自身の監査行は作業として数えられないため、誰も作業していない
  インテントが冷えたフックを持つと判定されることはありません。

レポート内の `report.md` には、すべての所見が対処方法とともに一覧表示されます。復旧のための回避手段
（`AIDLC_DISABLE_*` 環境変数など）を示す対処方法には、自動化するのは安全でないという
印が付きます。レポートの内容全体と安全モデルについては、[CLI コマンド](12-cli-commands.md#aidlc-doctor-export-write-a-diagnostic-report)
を参照してください。

---

## Next Steps

- [状態の追跡と監査証跡](10-state-and-audit.md) — 状態ファイルの構造
- [セッション管理](11-session-management.md): コンパクション後の作業の継続
- [CLI コマンド](12-cli-commands.md) — `--doctor`、`--status`、`--stage` の使い方
- [用語集](glossary.md): コンパクション、フックの定義
