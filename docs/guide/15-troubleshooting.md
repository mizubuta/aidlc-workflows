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
| ファイルを手で編集した。エージェントはそれを受け入れるか？ | はい。編集してから **done** と伝えてください（またはそのまま続けてください）。完了済みのステージへの編集はそのまま使われますが、再承認はされません。承認を受けたい場合は `/aidlc --stage <name>` で戻ってください。そのステージとそれ以降のすべてのステージが再び開かれます。各ケースについては [Editing Files Yourself](07-interaction-modes.md#editing-files-yourself) を参照してください |
| Claude のフックがポリシーで制限されている | Claude Code の管理者に、マネージド設定の `allowManagedHooksOnly` を解除するよう依頼してください。プロジェクト設定ではこれを上書きできません |
| Cursor: 承認が一切記録されない | プロジェクトが git リポジトリ内にない場合は、そこで `git init` を実行し、Cursor を完全に再起動してフォルダを信頼してください（[Cursor project outside a git repository](#cursor-project-outside-a-git-repository) を参照） |
| Kiro IDE または Kiro CLI v3: `deny fs_read matching ".kiro/"` | `/aidlc --doctor` を実行し、表示された ignore ファイルから `.kiro/` のルールを削除してください（[Kiro Read Denials](#kiro-read-denials) を参照） |
| Kiro IDE: AI-DLC のレビューやヘルパーが実行されない、チャットに **Approval needed in agent-aidlc-...** パネルが表示される、または `/aidlc` が Kiro のワークフローに渡されてしまう | Kiro の Workflows 機能がオンになっています。エージェントにオフにするよう頼むか、`aidlc config trust --kiro-workflows off --yes` を実行してから **Developer: Reload Window** を実行してください（[Kiro's Workflows feature](harnesses/kiro.md#kiros-workflows-feature-stops-ai-dlcs-reviews-and-helpers) を参照） |
| Kiro CLI: AI-DLC のレビューやヘルパーが実行されず、エージェントが作業を引き渡すツールがないと言う | Kiro CLI の Workflows 機能がオンになっています。エージェントにオフにするよう頼むか、`aidlc config trust --kiro-cli-workflows off --yes` を実行してから Kiro CLI を終了し、このプロジェクトで再度 `kiro-cli --v3` を起動してください（[Kiro's Workflows feature](harnesses/kiro.md#kiros-workflows-feature-stops-ai-dlcs-reviews-and-helpers) を参照） |
| Windows 上の Kiro IDE で、Kiro のターミナルがコマンドプロンプトの場合: コマンドの結果が `AI-DLC reply:` 行で終わるかその後で切れてしまい、ステップに余分なターンがかかる、またはすべてのコマンドが終了コード -1 になる | エージェントに Kiro のターミナルを PowerShell に設定するよう頼むか、`aidlc config trust --kiro-terminal powershell --yes` を実行してから Kiro を再起動してください（[Command Prompt as Kiro's terminal](harnesses/kiro.md#command-prompt-as-kiros-terminal-on-windows) を参照） |
| Windows 上の Kiro IDE で、Kiro のターミナルが PowerShell の場合: フォルダを信頼しているのにすべてのコマンドが終了コード -1 になり、結果が `AI-DLC reply:` 行で終わるか切れてしまう | Kiro のシェル統合が読み込まれていません。たとえば PowerShell の実行ポリシーが `Restricted` や `AllSigned` になっている場合です。Kiro のターミナルで `aidlc doctor` を実行すると（チャットでの `/aidlc --doctor` ではこれを確認できません）、`Kiro terminal` 行に原因が表示されます。Kiro の PowerShell ターミナルで（Windows PowerShell と PowerShell 7 はそれぞれ別のポリシーを持ちます）`Get-ExecutionPolicy -List` と `$ExecutionContext.SessionState.LanguageMode` を実行してください。`MachinePolicy` または `UserPolicy` が設定されている場合、あるいは言語モードが `FullLanguage` でない場合は IT チームに相談してください。`Process` が設定されている場合は、スタートメニューから Kiro を起動してください。有効なポリシーが `CurrentUser` または `LocalMachine` から来ていて `Restricted` または `AllSigned` の場合は、そこで `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned` を実行し、Kiro を再起動してください。Kiro の `terminal.integrated.shellIntegration.enabled` がオフになっている場合や、PowerShell プロファイルに余分な引数がある場合も同じ症状になります（[PowerShell without Kiro's shell integration](harnesses/kiro.md#powershell-without-kiros-shell-integration-on-windows) を参照） |
| Kiro IDE: エージェントが "my memory"（自分の記憶）を根拠に、チェックの回避やフックの手動実行を提案する | その提案は無視し、該当する Kiro のメモリを削除してください（[Kiro memories carry old AI-DLC advice](harnesses/kiro.md#kiro-memories-carry-old-ai-dlc-advice) を参照） |
| 状態ファイルが壊れている | `/aidlc --doctor` を実行し、状態テンプレートと比較してください |
| 承認ゲートで止まっている | 返答を入力してください。先へ進むには `/aidlc --stage <target>` を使います |
| Kiro IDE: エージェントが実行したビルド・テスト・プローブのコマンドが `Exit Code: -1` を示すが、AI-DLC 自身のコマンドは動作する | Kiro がそのコマンドを実行していません。あなたの許可を待っているか、あなたが拒否したためです。Kiro のプロンプトで **Allow**（または **Always allow**）を選ぶか、どう進めるかをエージェントに伝えてください |
| Kiro IDE: 承認の質問への返答が認識されない、またはコマンドが終了コード -1 で返ってくる | Kiro がこのフォルダを信頼するか尋ねてきたら **Trust Folder & Continue** を選び（または Restricted Mode のバナーで **Manage**、続いて **Trust** を選び）、コマンドパレットから **Developer: Reload Window** を実行して、続けるよう伝えてください。次のメッセージがまだ記録されない場合は、`/aidlc --doctor` で理由がわかります。Kiro CLI では、いったん終了してそのフォルダで再度 `kiro-cli --v3` を起動してください（フォルダを信頼するか尋ねられ、中身を把握している場合は **Trust, continue** を選びます）（[Kiro IDE hooks not running](#kiro-ide-hooks-not-running) を参照） |
| Kiro IDE または Kiro CLI: 承認時に `aidlc-record-human-turn` がオフになっているため回答が記録されなかったと表示される、または次の AI-DLC ステップで AI-DLC のフックがオフになっていると表示される | `.kiro/hooks/` にあるそのフックのファイルに `"enabled": false` が書かれています。Kiro IDE の Agent Hooks パネルがこれを書き込み、その状態でコミットされたファイルはすべてのクローンに持ち込まれます。AI-DLC がこれを自動でオンに戻すことはありません。Kiro IDE では Agent Hooks でオンに戻してください。Kiro CLI では、表示された行が示すファイルの `"enabled": false` を `true` に変更し、Kiro を終了してそのフォルダで再度 `kiro-cli --v3` を起動してください（フックファイルはセッション開始時に読み込まれます）（[A Kiro hook switched off](#a-kiro-hook-switched-off) を参照） |
| Kiro CLI（または Kiro ACP クライアント）: すべての承認で人間の返答が届いていないと表示され、再起動しても直らない | エンジンがディストリビューションと一致していません。Kiro CLI (V2) には Kiro CLI の v2 エンジン（`kiro-cli --v2`）が必要で、Kiro がエージェントファイルをアップグレードした後であれば v3 も使えます。Kiro には v3（`kiro-cli --v3`）が必要です（[Kiro CLI hooks not running](#kiro-cli-hooks-not-running) を参照） |
| AI-DLC がナレッジベースからナレッジドキュメントフォルダ内のファイルを読めないと言う、または `aidlc doctor` がそれらについて警告する | それらのファイルはナレッジベースに追加されていないか、読み取れるテキストを含んでいません。各ファイルについて表示された行の指示に従い（追加するか、その横に `.md`、`.txt`、`.csv` のコピーを保存する）、その後 `/aidlc knowledge sync` を実行してください（[Document Knowledge](08-knowledge.md#document-knowledge-documentkb) を参照） |
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
| Windows のアンインストール後もユーザー PATH のエントリが残っている | アンインストールで削除されるのは、`windows-path.json` を通じて所有しているエントリだけです。もともと存在していたエントリや、その所有記録のないインストールでは、ユーザー PATH は変更されません。[Uninstall](18-install-and-lifecycle.md#uninstall) を参照してください。 |
| アンインストール後もファイルが残っている | アンインストールでは、個別に検証したファイルを削除し、ディレクトリは空の場合にのみ削除します。一覧にないファイルや変更されたファイルは保持され、報告されます。古いインストールで所有一覧がないファイルも同様です。報告されたパスを確認してください。`--purge` でも無関係な内容は削除されません。 |
| `--offline requires --from <release-directory>` | オフラインモードがネットワーク上のリリースにフォールバックすることはありません。リリースアセット一式を転送し、そのディレクトリを `--from` / `-From` で渡してください。インストーラーがそれを検証します。 |
| `aidlc` が終了コード 4 で終了する、または Windows のアクティブポインタが無効 | ランチャー（bin 内の `aidlc.exe` または `aidlc.cmd`）は、使用できなかったファイル（アクティブバージョンのマーカー、アクティブなコマンドのターゲット、またはアクティブな実行ファイル）を示す `aidlc:` 行を 1 行表示し、続けて `Rerun the AI-DLC installer (install.ps1) to repair the aidlc command.` と表示します。以前のリリースから `aidlc update` を実行した後は、そのリリースのランチャー（何も表示せずに終了コード 4 で終了するもの）が、`aidlc doctor` と `aidlc uninstall` 以外の次の `aidlc` コマンドで置き換えられるまで残ります。その 1 回のコマンドはまだ古いランチャーを通して実行され、空白を含む値は別々の単語に分割されます。古いランチャーが残っている間、`aidlc doctor` は `Windows launcher` 行を表示します。AI-DLC がそれを置き換えられない場合（たとえばインストール後に `aidlc.cmd` や `aidlc-shim.ps1` が変更された場合）、その行に理由と修正手順が示されます。古いランチャーがコマンドの開始前に終了コード 4 で終了する場合は、インストーラーを再実行してください。`%LOCALAPPDATA%\aidlc\active-executable` は編集しないでください。検証済みの同じインストーラーを再実行するか、正常に動作する保持済みの実行ファイルから `aidlc use <version>` を使ってください。PowerShell がランチャーを `ConstrainedLanguage` モードで実行していることを示す行が表示された場合は、アプリケーション制御ポリシー（AppLocker または WDAC）がそれをブロックしています。管理者に `aidlc-shim.ps1` の許可を依頼するか、`%LOCALAPPDATA%\aidlc\versions\<version>\aidlc.exe` を直接実行してください。 |
| Windows で `aidlc-shim.ps1 cannot be loaded` または `is not digitally signed` と表示される、あるいはインストーラー、`aidlc update`、`aidlc use`、`aidlc rollback` が `command pointer validation failed for <version>: version probe returned 1 ""` で始まるエラーで止まる | グループポリシーの実行ポリシーが `aidlc.cmd` の渡す `-ExecutionPolicy Bypass` を上書きするため、PowerShell がランチャーを実行前に拒否しています。新しいバージョンの有効化を `aidlc.cmd` 経由で確認するリリースでは、その有効化が取り消されます。以前のバージョンがアクティブなまま残り、初回インストールの場合はどのバージョンもアクティブになりません。`Get-ExecutionPolicy -List` を実行してください。上から見て最初に `Undefined` でない行が有効なポリシーなので、`MachinePolicy` が `UserPolicy` より優先されます。その行が `MachinePolicy` または `UserPolicy` で、値が `AllSigned` または `Restricted` であれば、それが原因です。`RemoteSigned` と `Unrestricted` ではランチャーは実行できます。管理者にスクリプトの許可を依頼するか、`%LOCALAPPDATA%\aidlc\versions\<version>\aidlc.exe` を直接実行してください。 |
| `pending Windows uninstall`、または Windows のアンインストール復旧の失敗 | 実行中の AI-DLC コマンドを閉じて `aidlc doctor` を実行してください。保留中のクリーンアップは次のコマンドで再開されます。doctor がクリーンアップの失敗を報告した場合は、示された原因を解消してから、doctor が表示するフルパスで `aidlc uninstall` を再度実行してください（PATH エントリがすでに消えている可能性があるため、通常は `& "$env:LOCALAPPDATA\aidlc\bin\aidlc.exe" uninstall`、bin 内の `aidlc.exe` と `aidlc.cmd` が両方とも AI-DLC がインストールしたままの状態でない場合は記録された `aidlc.cmd` を使います。元のコマンドで `--purge` を使った場合は付けてください）。ファイル削除前に失敗したクリーンアップはディスクの状態から計画し直され、開始済みのものは再開されます。そのコマンドがもう実行できない場合は、インストーラーを再度実行してください。インストーラーはまず保留中のクリーンアップを再試行するため、2 回実行する必要があるかもしれません。ファイル計画が紐付いていない古いジャーナルや改変されたジャーナルは拒否されます。調査のため、一時ジャーナル、クリーンアップスクリプト、マシンフェンス、残っているファイルは保持してください。 |
| `the last AI-DLC uninstall is still finishing` | Windows では、`aidlc uninstall` の終了後に AI-DLC のファイルが削除されます。インストールされたバージョンが 1 つか 2 つなら 1 分未満ですが、それ以上ある場合や遅いマシンではもっとかかります。何も変更されていません。表示された行の指示に従ってインストーラーを再度実行してください。完了せずに止まったクリーンアップは、開始から 10 分後に doctor 以外の最初の AI-DLC コマンドによって再開されます。 |
| Alpine で `libstdc++.so.6` または `libgcc_s.so.1` が見つからないと報告される | Bun と Node.js の musl ビルドが必要とするのと同じランタイム依存関係を `apk add libgcc libstdc++` でインストールし、インストーラーまたはコマンドを再実行してください。完全に静的な Bun musl コンパイルターゲットは現在利用できません。インストーラーはこの対処法を表示しますが、パッケージのインストールはしません。 |
| ハーネスがすでに独自のモデルアクセスを持っているのに、`Providers` が `no recorded answers` とともに `[needs]` を示す | Kiro CLI と Kiro IDE では、モデルアクセスが Kiro に付属しているため、この行は `[ok]` となり、`aidlc config providers` は何も尋ねません。GitHub Copilot と Cursor では、セッションがモデルアクセスを提供するため、回答なしで `[ok]` となります。自分の Amazon Bedrock アクセスを使う場合にのみ `aidlc config providers` を実行してください（Cursor では Bedrock のキーを受け付けるのは IDE だけで、CLI は常に Cursor のバックエンドを使います）。その他のハーネスではこの行は本当に対応が必要です。`amazon-bedrock` と答えるか、`keep current`（`--provider current`）を選んで、ハーネスにすでに設定されているプロバイダを維持して記録してください。`--provider other --acknowledge` は、自分で設定した別のプロバイダを記録します。 |
| `Models` が `no recorded policy` とともに `[needs]` を示す | ポリシーがない場合、ほとんどのエージェントはセッションのモデルと effort を引き継ぐため、高い effort のセッションではエージェントも高い effort で実行されます。エージェントの effort を設定するにはプリセットを記録するか（`aidlc config models --preset balanced --project --yes`）、そのままにして引き継ぎを続けてください。Kiro CLI では、プリセットは対話セッション全体に 1 つの effort を設定します。`aidlc config models` がセッションのモデルを選び、その effort とともに個人の Kiro 設定に保存します。Kiro の中では `/model` と `/effort` で変更できます（[Kiro CLI (V2) session model and effort](harnesses/kiro.md#kiro-cli-v2-session-model-and-effort) を参照）。GitHub Copilot、Cursor、Kiro IDE では、プリセットを設定しても何も変わらないため、この行は `[ok]` となり代わりにホストの名前を示します。モデルと effort はホスト側で選んでください（Kiro では、`aidlc config models --agent` で 1 つのエージェントに独自のモデルと effort を設定できます）。[Choosing a Model and Effort](18-install-and-lifecycle.md#choosing-a-model-and-effort) を参照してください。 |
| `aidlc doctor` で `workspace shell ready` が失敗し、`aidlc config` を再実行しても直らない | 対話的な再実行は既存のプロジェクションを使い、欠けている `aidlc/spaces/default/memory/` を再構築することはありません。シェルが不完全な間は `Workspace` 行を報告し、修正するセクションを提示しません。その行が示すコマンドを実行してください。ネイティブインストールでは `aidlc config --harness <name>` で、インストール済みのランタイムから更新し、ワークスペースのシェルを再作成します。Bun で起動するプロジェクション（`aidlc-copy-runtime-X.Y.Z.tar.gz` の `runtime/<name>/` ルート、またはチェックアウトの `dist/<name>/` ツリーからコピーしたもの）には更新元となるインストール済みランタイムがないため、そのコマンドには `--download` も付きます。これはプロジェクトのリリースに対応する `aidlc-copy-runtime-<version>.tar.gz` を取得して検証します。プロジェクトのフックがネイティブコマンドに切り替わってしまうため、ネイティブ用の `aidlc-runtime-X.Y.Z.tar.gz` や `dist-release/` のバイトが使われることはありません。 |
| `aidlc config` がリリースを必要としていると表示する: `this project is pinned to <version>`、`needs the <version> release files`、`is missing aidlc/spaces/default/memory/`、または `no longer has the MCP server list` | プロジェクトが、このマシンにないリリースのファイルを必要としています。チームメイトが設定した新しいピン、追加しようとしているハーネス、または失われたファイルが原因です。`fix:` 行を実行してください。これは実行したコマンドに `--download` を付けたもので、まさにそのリリースを取得し、チェックサムと（`gh` がインストールされていれば）リリースのアテステーションを検証して、コマンドを完了させます。ネイティブの場合は、`aidlc config --pin` と同様にピン留めされたリリースをインストールして登録します。コピーされたプロジェクトでは `aidlc-copy-runtime-<version>.tar.gz` をダウンロードします。ターミナルでは、config が代わりに確認を求めます。ピン留めされたリリースがすでにインストールされていれば、確認なしで使われます。ネットワークにアクセスできない場合は `offline:` 行に従ってください。示されたファイルを別の場所で取得して `--from` で渡すか、ネイティブの場合は `aidlc config --pin <version> --offline --from <release directory>` を実行します。`--release-base-url` と `--ca-bundle` でミラーを選べます。チェックサムやアテステーションに失敗した場合は、何も変更せずに停止します。 |
| コピーチャネルでインストールしていないプロジェクトで、`aidlc doctor` が Bun を要求する | コピーチャネルのプロジェクションは Bun でフックを実行し、ネイティブインストールは `aidlc` コマンドでフックを実行します。対処法には、プロジェクトがどちらのチャネルかが示されます。Bun を不要にするには、ネイティブのリリースインストーラーで再インストールし、`aidlc config --harness <name>` を再実行してください。 |
| `aidlc doctor` で `Harness trees on different releases` | プロジェクトのハーネスツリーが異なるリリースからインストールされているため、ツールごとにワークフローの挙動が変わる可能性があります。その行が示すコマンドを実行してください。ピン留めされたプロジェクトのツリーはピンのバージョンにそろえられます。ネイティブの場合は、エンジンのリリース（またはピン）と異なる各ツリーについて `aidlc config --harness <name>` を実行します。ピン留めされたコピー型プロジェクトでは `bun <harness-dir>/tools/aidlc.ts config --harness <name> --download` です。ピンがない場合は、最新のツリーの `aidlc-copy-runtime-<version>.tar.gz` を使って `bun <harness-dir>/tools/aidlc.ts config --harness <name> --from <file>` を実行します。ただし config の実行記録がないツリーには、先に一度 `--download` で更新します。進行中の作業は更新の間も継続されるため、終わるのを待つ必要はありません。 |
| `config plan changed after approval` | `aidlc config --dry-run --json` を再実行し、`data.actions` を確認してから、まったく同じソースと動作オプションで新しい `data.planToken` を適用してください。 |
| `aidlc config` から `locally modified` または `managed block was locally modified` | `aidlc config --dry-run --json` を実行して `data.actions` を確認してください。`--force` は、ベースラインが所有するフレームワークのバイトやマネージドブロックを置き換える場合にのみ使ってください。無関係なルートの内容を変更する許可にはなりません。`.claude/settings.json` はエントリ単位でマージされるため `--force` は不要です。更新すると AI-DLC のフックとコマンドの許可エントリが元に戻され、あなたのフック、deny ルール、カスタムのステータスライン、環境変数、その他の設定は保持され、何を戻したかを示す注記が表示されます。廃止された同梱の許可エントリは自動的には削除されません。`.codex/config.toml` も同じようにエントリ単位でマージされます。更新で変更されるのは AI-DLC 自身のキーだけで、あなたのキー、テーブル、コメントはそのまま保持されます。AI-DLC のキーのうちあなたが変更した値も保持され、リリースが別の値を同梱している場合は注記が表示されます。 |
| `aidlc doctor` で `AI-DLC files: N changed in this project` | これらの AI-DLC ファイルがこのプロジェクトで編集されているため、`aidlc config` はそれらを保持し、何かを更新する前に停止します。自分の版を残したい場合は、示されたファイルを別の場所に移してから `aidlc config` を実行してください。同梱の版で上書きしたい場合は `aidlc config --force` を実行してください。違いが改行コードだけのファイル（Windows のチェックアウトでは LF が CRLF に変換されます）は数えられません。 |
| `cannot coexist in one project` | 2 つ目のハーネスが、エンジンディレクトリ（`opencode` / `copilot` は `.aidlc` を使います）または排他的なマネージドブロック（Copilot の `AGENTS.md`）を共有しています。別々のエンジンディレクトリを選び、排他的なブロックを避けてください。Kiro CLI (V2) と Kiro も `.kiro` を共有しますが、`aidlc config --harness kiro`（または `--harness kiro-v2`）でインストール済みのものをその場でもう一方に切り替えられます（Kiro CLI (V2) から Kiro へ、ターミナルなしで、または `--quiet` や `--json` の下で切り替える場合は `--yes` を付けます）。[More than one harness in a project](18-install-and-lifecycle.md) を参照してください。中立的な `AGENTS.md` ブロックは Kiro CLI、Kiro IDE、Codex、Cursor、OpenCode で共有され、`.gitignore` は同梱のエントリを結合します。 |
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
[Transactions and Recovery](18-install-and-lifecycle.md#transactions-and-recovery) の機能とサポートの対応表を参照してください。
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

**症状**: ファイルの書き込み後にインテントの `audit/` シャードにエントリが現れない、またはサブエージェントの完了ログがない。

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
（[First run](harnesses/kiro.md#first-run) を参照）。

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

**`$<usd>` の数値はローカルでの見積もりであり、請求額ではありません。** 同梱の料金表にある**公開されている定価**から算出されます（[Rate table and overrides](../reference/06-hooks-and-tools.md#rate-table-and-overrides) を参照）。Amazon Bedrock が記録されたプロバイダである場合（`CLAUDE_CODE_USE_BEDROCK=1`）、Bedrock が実際に請求する金額は、推論プロファイル、リージョン、サービスティア、交渉済みの価格やサブスクリプション価格によって異なるため、見積もりが請求書と一致しない場合があります。長いワークフローのトークン量の大半はキャッシュ読み取り（割引されたキャッシュ読み取り料金で課金されます）なので、カウントと見積もりは着実に増えていきます。これは実際の使用量であり、水増しではありません。この数値は、個人の目安として扱ってください。

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

**症状**: すべてのステージ、エージェント、プロトコルの読み込みが拒否され、ワークフローが止まる:

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

**症状**: オーケストレーターが状態の破損を報告する、またはワークフローが予期しない挙動をする。

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

**症状**: ディスパッチされたステージ（Reverse Engineering、Practices Discovery、User Stories、Code Generation）がエラーや途中で切れた出力を返す。

### What happens

フレームワークは組み込みの再試行プロトコルに従います:

1. 縮小したコンテキストのプロンプトで**自動的に再試行**する
2. **再試行が失敗した場合**、2 つの選択肢がある:
   - **インラインで実行** — メインの会話でステージを直接実行する（サブエージェントの境界なし）
   - **スキップして後で再訪** — ステージを未完了としてマークし、後で戻る

### Manual recovery

`/aidlc` を再実行してください。`[-]`（進行中）の状態を検出して、そのステージを続行します。最初からやり直す場合は redo と伝えてください。何が失敗したかを把握するには、`audit/` シャードのエラーエントリを確認してください。

---

## Approval Gate Stuck

**症状**: ワークフローが承認ゲートであなたの返答を待っている。

### How to proceed

求められたら返答を入力してください。選択肢は次のとおりです:

- **Approve** — 次のステージに進む
- **Request Changes** — 修正のためのフィードバックを伝える

Kiro IDE で、すでに返答したのにワークフローがまだ待っている場合は、
[Kiro IDE hooks not running](#kiro-ide-hooks-not-running) を参照してください。Kiro CLI では
[Kiro CLI hooks not running](#kiro-cli-hooks-not-running) を参照してください。

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
`contains ignored application source "src/main.js.map"` と返される。

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
`is ignored by Git and cannot be source-review evidence` と返される。

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
Generation that no Unit's review covers (...)" と返される。

ステージの間に、どのユニットのソースマニフェストにも記載されていない何かが
ワークスペースで変更されました。表示された行には 3 つの対処方法が示され、アシスタントは
状況に合うものを選びます。変更を行ったユニットがそのパスを記載し、もう一度
レビューを受ける。誰も意図していない変更は元に戻す。ソースではないパス（
リポジトリ内に書き込まれたツールのキャッシュ、ビルド出力、このプロジェクトのコードではない
入れ子のリポジトリ）は、`.aidlc-source-paths.json`（記録されたリポジトリや記録されたチェックアウトの場合は
それ自身のフォルダにあるファイル）の `exclude` に記載して、ソースチェックの対象から外す。除外はすぐに有効になり、
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

**Symptom**: Code Generation presents the Plan Approval question again for a plan
you already approved.

Plan Approval binds to the plan, unit test instructions, and Testing Contract
content, to the target, and to the current stage attempt. It is NOT reopened by
re-running `/aidlc`, by a session restart or a context compaction, by a Stop-hook
probe, or by `/aidlc --status`. Ticking a plan checkbox does not reopen it
either, and recording a review never touches the plan.

For the same target and attempt, plan, test instruction, or Testing Contract
edits reopen approval only under Guard Policy `strict`. Under `relaxed` or
`off`, work continues with the updated content and the original approval
record stays intact; it does not claim you approved the edits. The
`Guard Policy:` line of `/aidlc --status` shows the effective setting. You can still ask to review the plan again.

If code generation starts without asking you about the plan at all, plan
approval is off for this piece of work: status shows where that came from, for
example `Plan Approval: off (from scope poc)`. The line that names each plan
says you can ask to go over it at any time; you can also ask to be asked about
every plan. See [Plan approval](13-customization.md#plan-approval).

Testing Posture, scope, test strategy, or project type changes follow the same
rule within the same intent, target, and attempt. Refresh the current contract
and instructions as needed; a lowered fence permits continued execution
without asking for approval again solely because those inputs changed.

If you are asked again, check what changed and which rule applies:

- the plan content or embedded Testing Contract while the plan-approval fence
  is on (beyond a ticked task marker, or a terminal `## Review` section left by a
  review recorded before review records existed)
- the unit-test instructions content, any byte of it: the instructions are handed
  to the developer in full, so they bind byte-exactly, and a section appended to
  them after approval reopens it when the plan-approval fence is on
- the Testing Posture, scope, test strategy, or project type while the
  plan-approval fence is on (a Testing Posture change made after the build's
  review, such as a testing practice you keep at the learnings step, applies
  from the next plan and does not reopen approval for that build)
- the active intent, Unit, or stage target
- the stage attempt: a backward jump, a Request Changes, a gate rejection, or a
  workflow restart

The question names what it is about. Other code moving after approval (a `git
pull`, another Unit landing) never asks again: the build continues and you hear
which files moved. The exception is a build of Units in parallel under
`strict`: a change after you approved (your edit or commit, a `git pull`) asks
again before the Units are set up, and stops the next Unit from landing until
you undo it or set Guard Policy `relaxed` (see
[Guard Policy](13-customization.md#guard-policy)). Nor does anything between
your answer and the build:
a long chat compacting its context, parking the work and resuming it, or another
question coming up first. Say "review the plan" if you want to look again first,
including right after any of those: the plan is shown for approval again before
anything more is built.

### Plan Approval is not recorded

**Symptom**: you answered the Plan Approval question, but AI-DLC shows it again.

Your answer counts from any chat on this piece of work, in your own words ("1",
"approve", "looks good", "rename the handler"): the agent records the choice
you made, with your words beside it. The question comes back only when the
agent could not record a choice, most often because the reply arrived before
the question was shown, or in a chat where the harness did not pass it to
AI-DLC. Answer the question it shows. If it keeps coming back, run
`/aidlc --doctor`: a harness whose prompt hook does not run cannot record any
reply. A long chat that compacts its context while the question waits keeps
the question open, so your answer still counts.

If AI-DLC says the project's files cannot all be read, or that the workspace
source cannot be bound (a link that loops, a tree too large to read, or a
source path outside the project), what happens depends on your settings. Under Guard Policy strict with plan approval on, the
plan cannot be approved yet, because nothing could say what the build starts
from. Otherwise the plan question is asked as usual (with plan approval off,
the plan is built as written) and the build starts without a check of the
project's files, with one line that says so. To have the files checked, repair
the source boundary the message names (add the offending path to the exclude
list in `.aidlc-source-paths.json`, which looks like
`{"version":1,"exclude":["<path>"]}`, declare real source under an excluded
directory in its paths list, or remove a broken symlink), then run `/aidlc`
(`$aidlc` on Codex). `/aidlc --doctor` has a "Workspace source boundary binds"
check that names the failing path and says which of these applies to you.

A valid Git checkout kept inside the project can share its object store with
jj, including through a linked worktree. If an older AI-DLC install reports
"no resolvable HEAD commit" for a commit that Git can read from a jj-created
pack, update AI-DLC and refresh the project's install. Repacking the repository
or changing jj settings is unnecessary. A genuinely missing or unreadable
HEAD object still needs repair.

If the plan's Testing Contract is refused, the message names one of three
causes: the block is missing, it is not valid JSON, or it changed after it was
rendered. The repair is the same for all three: re-run `render` and replace the
whole `## Testing Contract` section. A changed block usually means a shell
command rewrote the file and re-encoded its characters (for example PowerShell
`Set-Content`); artifacts are edited with the file-editing tool instead. When
your own edit broke the block, the assistant repairs it and asks you once to
build the edited plan.

**Legacy Kiro IDE windows.** A Kiro IDE build that passes no typed text to
AI-DLC approves plans with the picker only, through the older recorded
decision and answer steps. There an answer binds to the chat session it was
given in. When the project's files cannot be checked, you can still ask the
assistant to build the plan anyway, by picking that or saying so, and it is
recorded as your approval with no check of the files. Updating Kiro IDE lets
you answer in your own words, from any chat, or edit the files.

---

## A Bolt attempt was set aside — getting the files back

A recovery can set aside an attempt when its reviewed files changed again and
its one allowed re-review was already used. The recovery abort's `--discard`
saves tracked files and non-ignored untracked files (or the remaining branch
tip when the checkout is already gone) plus reviewed source refs in local Git
before removing the old checkout and branch, so a fresh attempt can start.
An ordinary Abort without `--discard` leaves the checkout in place and returns
`parked_ref: null`, with no `parked_stamp`, `parked_mode`, `parked_repo`,
`restore_operation`, `restore_hint`, `restore_hint_error`, or `parked_excludes`.
After a discard, the assistant tells you why it set the attempt aside and offers
restoration only when `restore_operation` is present, regardless of whether a
display hint could be rendered.
Every successful abort retains `reason: "aborted"` and echoes the supplied
`--reason` text in the additive `abort_reason` field. When an attempt was parked,
the result includes the saved descriptor:
`parked_ref`, `parked_stamp`, `parked_mode` (`snapshot`, `branch-tip`, or
`evidence-only`), and `parked_repo` (`null` for the project root, otherwise the
sibling repository name).

If files were saved, ask to restore them. The assistant uses the saved result's
`restore_operation`: route `worktree`, args
`["restore", "--slug", slug, "--parked", stamp, "--repo", repo ?? ".", "--intent", recordDirName, "--space", space]` when the
stamp is known. The final selectors pin the owning intent. It invokes the installed `{{INVOKE}} engine worktree <args...>`
route with each listed arg exactly as a separate argv argument, never joined
into a shell command. `restore_hint` is human display text only, not the
assistant's execution input. The equivalent manual command, run from the main
project checkout, is:

```bash
aidlc engine worktree restore --slug <slug> --parked <stamp> --repo <name|.> --intent <record-dir-name> --space <space>
```

The operation always includes `--repo <name>` for a sibling repository or
`--repo .` for the root. Its optional display hint is safely rendered for the
selected shell; a native install uses the prefix above, while a Bun-based copy
install uses `bun .claude/tools/aidlc-worktree.ts`, substituting your harness
directory. If safe rendering fails, for example because the harness directory
is invalid, the result omits `restore_hint` and supplies `restore_hint_error`
with the reason. The typed operation remains available; this does not mean
that only evidence was saved or withdraw the restoration offer.

Do not drop the operation's exact stamp, repository, or owning intent selectors: without
`--parked`, restore chooses the latest saved head. For manual invocation an exact
stamp found in only one repository selects it before generic slug ambiguity;
if selection is still ambiguous, use doctor's exact operation with `--repo <name>`
or `--repo .` for the project root.

If a saved namespace is known but its discard descriptor is unavailable, the
fallback retains `parked_ref` and derives `parked_stamp` from its namespace
only when the stamp parses strictly; otherwise `parked_stamp` is `null`.
It reports `parked_mode: null` and `parked_repo: null` and omits
`restore_operation`, `restore_hint`, `restore_hint_error`, and `parked_excludes`,
rather than guessing a repository, saved mode, or latest-attempt selection.
Instead, `recovery_hint` asks you to run doctor to list set-aside attempts and
their exact restore commands. The hint is plain guidance, not an executable
operation. The assistant must not claim what files were saved or offer restoration
from the fallback alone. If no namespace was saved, `parked_ref` is `null`;
`parked_stamp`, `parked_mode`, `parked_repo`, `restore_operation`, `restore_hint`,
`restore_hint_error`, `parked_excludes`, and `recovery_hint` are absent.

If only review evidence remained, discard reports `parked_mode: "evidence-only"`
and `parked_commit: "-"`. Abort keeps the ref, stamp, mode, and repository but
omits `restore_operation`, `restore_hint`, `restore_hint_error`, and
`parked_excludes`. The assistant says: "Nothing of its
working files remained to save; only its review evidence was kept." It does
not offer restoration. Selecting that attempt with restore, even with `--raw`,
refuses with `no restorable files were parked for <slug> <stamp>; only review evidence was kept`.

After a successful restore, open the returned `worktree_path`, normally
`.aidlc/restored/bolt-<id8>_<slug>-<stamp>` (the recorded legacy name for a pre-upgrade attempt). It is separate from any new live attempt;
restoring files does not resume the old attempt or make its review current.
This is local recovery, not a remote backup. If the old checkout was already
gone, only its remaining branch tip and review evidence could be saved; if its
branch was gone too, only review evidence could remain.

**Limits depend on `parked_mode`:** a snapshot reports
`parked_excludes: ["ignored files", "eol/text=auto normalization"]`. Ignored
untracked files are not saved (tracked files remain included), and normalization
may change line endings while saving; normalized bytes cannot be recovered,
even with `--raw`. A branch tip instead reports
`["uncommitted files (no working tree existed)"]`, and the assistant says:
"I kept its committed work; there were no uncommitted files to save."
Snapshots restore stored blobs directly; branch tips use ordinary checkout
conversions unless `--raw` is given.
See the [restore command reference](12-cli-commands.md#aidlc-engine-worktree-restore-recover-files-from-a-set-aside-attempt)
for filter failures and raw recovery details.

Run `/aidlc --doctor` (or `aidlc doctor`) to find saved attempts. Its informational
**Parked attempts** section lists slug, stamp, age, mode, restored-checkout
presence, and typed recovery operations with exact `--parked <stamp>` and
explicit `--repo <name>` or `--repo .` args, followed by owning `--intent` and `--space` selectors. Every JSON entry has
`purge_operation`; only restorable entries have `restore_operation`. Each has
route `worktree` and args to pass exactly as argv, never a shell command string.
Optional `restore_command` and `purge_command` are safe human display text;
if rendering fails, the corresponding command is omitted and
`restore_command_error` or `purge_command_error` explains why, without removing
the operation. Evidence-only entries have only the purge operation and its
command-or-error fields. Impossible dates or times have `age_days: null` in JSON
and show `unknown` in human-readable output. Saved attempts are not warnings or
failures.

Doctor resolves namespaced attempts through their intent's registry UUID and
legacy attempts through the exact `WORKTREE_DISCARDED` `Parked ref` provenance.
Unknown or ambiguous owners and unattributed legacy parks are not listed.
Once you no longer need one, remove any restored checkout first, then:

```bash
aidlc engine worktree purge --slug <slug> --parked <stamp> --repo <name|.> --intent <record-dir-name> --space <space>
aidlc engine worktree purge --slug <slug> --older-than 30 --repo <name|.> --intent <record-dir-name> --space <space>
```

Purge without either stamp selector removes every saved stamp for the selected intent's Bolt.
`--older-than` accepts nonnegative finite days and selects strictly older UTC
stamp timestamps, ignoring any `-N` suffix. The strict calendar parser rejects
impossible dates and times rather than normalizing them. Age-filtered purge
keeps unparseable stamps and reports them in `skipped_unparseable`. This JSON
array is always present and otherwise empty; exact-stamp or all-stamp purge
can remove unparseable stamps. `--older-than` cannot be combined with
`--parked`. Purge refuses while a selected restored checkout exists, even if
moved; it never removes a live Bolt checkout. Use the same Bun tool prefix for
copy installs. See the [purge reference](12-cli-commands.md#aidlc-engine-worktree-purge-remove-recovery-refs).

Restore and purge reject unknown or duplicate flags before selecting or changing
anything; use only the flags in the command reference. Recovery selectors and
doctor accept the project root (`.`) and valid Git repositories named in the
same slug's `WORKTREE_CREATED` or `WORKTREE_DISCARDED` audit `Repo` fields even
when those sibling names are symlinks: the framework may recover exactly where
it recorded the attempt's worktree or parking. This admission is slug-scoped;
records for other slugs never widen this slug's repository set. Membership in
a current or historical intent's repo list alone cannot admit a symlink.
Intent-list candidates and unrecorded discovered siblings must be real
immediate child directories whose canonical paths stay directly under the
canonical workspace root (`isWorkspaceRepoDir`). Arbitrary paths and symlink
aliases without same-slug audit provenance are refused. These recovery selectors
do not change live create/discard commands or the required human consent.

---

## Context Compaction

**Symptom**: Claude Code summarized earlier conversation context. The session may feel like it "forgot" recent discussion.

### What is preserved

All record-dir artifacts, `aidlc-state.md` and the `audit/` shards persist on disk. Only in-memory conversation context and partial in-progress work not yet written to files is lost.

### How to recover

Run `/aidlc` after compaction. The framework:

1. Reads `aidlc-state.md` to load workflow position
2. Carries on where the work stopped

To have the stage that was in progress done again, say "redo this stage".

### The build stops after a compaction

**Symptom** (GitHub Copilot): you approved the code plan, the build started, and after the chat compacted every build step the assistant tries is refused.

Your approval, the workflow state, and every file already written are kept; only what the assistant had not yet written to a file is gone from the chat. After a compaction the assistant must read its step again before it builds anything. It runs `next` as its own command, which hands the approved build straight back without asking you again. `/aidlc --doctor` shows the step as out of date, with the time and the reason (the chat was compacted, or the workflow state changed after the step was issued, naming what changed when it is known), until that `next` runs.

---

## Audit Log Growing Too Large

**Symptom**: this clone's audit shard has grown to thousands of lines over a long project.

A long project's shard is large by design; leave it where it is. The engine reads every shard in `audit/` to know which stages you approved and which Units you finished, so moving one out of `audit/` makes that work count as not done and the engine hands it out again. (The PreToolUse guard refuses the agent's file and shell tools any write into `audit/`.)

### Git considerations

The `audit/` shards are committed (not gitignored), see [What to Commit vs. Gitignore](14-artifacts-reference.md#what-to-commit-vs-gitignore). Each clone writes its own `<host>-<clone>.md` shard, so concurrent appends never merge-conflict.

### Moved, copied, or synced projects

`aidlc/.aidlc-clone-id` records this clone's token and the host name it was first used on, so the shard name stays the same when the machine's name changes or the folder is copied or synced to another laptop: the work continues in one shard. A fresh `git clone` gets its own token and shard. Two copies of one folder are one clone, so if two people (or two laptops) work at the same time, give each its own `git clone`. If a sync tool leaves a conflict copy such as `<host>-<clone> 2.md`, AI-DLC reads the rows it shares with the original once, so finished work keeps counting and the copy can stay. With team Unit ownership, landing a Unit still stops on any file it did not expect, a copy included: it names the file (`Unit landing requires a clean source worktree; commit or stash: ...`).

If such a copy was already in your project on an earlier release and you kept working, Units you finished while the copy sat there may be handed out once more after you upgrade (the Units you finished before the copy count again). Nothing is deleted: the Unit's files are still there, so for most stages finishing it again only records it; a Code Generation Unit may ask you to approve its plan once more and build again from the code already in your branch. It happens once and never repeats, and a planned follow-up removes even that.

One narrow case: a Code Generation Unit built in its own worktree that had finished but was not yet merged when you upgraded. Building it again stops with `Worktree directory already exists`, or `resume requires the completed, merged prior Bolt`. Run `aidlc engine worktree discard --slug <unit>`: it sets the earlier attempt's files aside before it removes that worktree, then the next `/aidlc` builds the Unit again, and `aidlc engine worktree restore --slug <unit>` brings the earlier files back into a separate checkout whenever you want them.

---

## Lock Files Left Behind

**Symptom**: Hooks appear to hang briefly then skip. Subsequent audit entries are not written.

The audit hooks use `mkdir`-based locking (via `lib.ts`) to prevent concurrent writes. If a hook is interrupted, the lock directory may persist. Lock files are created in the system temp directory (`os.tmpdir()` -- typically `/tmp/` on macOS/Linux, `%TEMP%` on Windows).

### Finding stale locks

```bash
# macOS / Linux
ls -la /tmp/.aidlc-*

# Windows (PowerShell)
Get-ChildItem $env:TEMP -Filter ".aidlc-*"
```

Lock directories are named `.aidlc-audit-<hash>.lock` and `.aidlc-subagent-<hash>.lock` inside the system temp directory.

### Clearing stale locks

Run `/aidlc --doctor` first. It automatically clears only a provably-dead
generation, a reused PID whose creation generation no longer matches, or an old
lock whose owner stamp never finished (missing, empty, or cut off). Matching or
unknown live generations, stamps that name no valid owner, and unreadable stamps
are reported but not removed.

Before any manual cleanup, stop all AI-DLC processes using the affected locks,
confirm ownership and that the projects are quiescent, and preserve diagnostic
evidence. Investigate only the named lock; do not bulk-delete lock directories.
Locks and their owner-stamped `.reap` recovery gates are transient and recreated
as needed. `.gate-mutex` files are persistent advisory-lock anchors and may
remain empty in the temp directory; an empty file is not evidence of a stale
owner. Project transaction locks use the [ownership checks above](#transaction-lock-ownership).

---

## Statusline Issues

### Shows "ready" when workflow is active

The statusline reads the `**Lifecycle Phase**` field from `aidlc-state.md`. If that field is missing or empty, it falls back to `[AIDLC] ready`.

**Fix:** Run `/aidlc --doctor` to check state file integrity. Verify the `## Current Status` section contains a `**Lifecycle Phase**` entry.

### Shows stale data

Expected behavior — the statusline updates when the state file is next written, typically at stage transitions.

### Not appearing at all

1. AI-DLC's status line shows only when you chose it at setup or turned it on:
   run `/aidlc --statusline on` (`$aidlc --statusline on` on Codex). On Claude
   Code it goes into your own `.claude/settings.local.json`; a status line your
   organization's managed settings choose still wins.
2. Run `aidlc doctor` and repair the reported native command or host wiring.
3. On a copy install, verify Bun is on the host process PATH.
4. With no state file, `[AIDLC] ready` is the expected output.

---

## Using `--doctor`

The `--doctor` utility command validates your setup. Run it whenever something seems wrong:

```
/aidlc --doctor
```

It checks: prerequisite (`bun`), hook availability (every framework hook wired by `settings.json` must exist in `.claude/hooks/`, and a wired-but-missing hook fails loudly; a valid custom non-AI-DLC `statusLine` intentionally exempts the unwired `aidlc-statusline.ts`), hooks-not-globally-disabled (a resolved `disableAllHooks: true` in any Claude Code settings layer fails loudly), managed project-hook policy (`allowManagedHooksOnly: true`), project structure (`settings.json`), workspace shell readiness (`.claude/` + `aidlc/spaces/default/memory/`), state/audit consistency (the workflow Status against a recorded `WORKFLOW_COMPLETED`, and each Stage Progress checkbox against the stage starts and completions the audit recorded for the current attempt. A stage the audit shows as started whose checkbox still reads `[ ]` is what makes the workflow refuse to finish it, and the warning names the exact line to change. Under team Unit Ownership the per-unit Construction checkboxes are derived from Unit Progress, so they are not compared), hook heartbeats, graph integrity (no cycles, every graph entry has a file), the **Composed plugin surface** (enabled plugin stages are compiled; contribution sidecars and targets are valid; recorded structural additions and prose fragments remain present and unchanged), selection-aware plugin-authored checks, scope validation across all 11 scopes, **Composed scope durability** (every composer-authored scope resolves to a real plan: a scope file with no grid column, a durable `aidlc/scopes/<name>.md` record not yet projected, or a runnable workflow naming an unresolvable scope all fail, with `graph compile` as the remedy wherever compile can reach the cause; a missing column with no record behind it is reported apart, since compile emits a column only for a scope some stage declares), stage schema + graph references, and keyword overlap across scopes. Passing advisory rows include **Duplicate producers** for consumed artifacts whose producer is ambiguous by graph load order, **Rule drift** (with lifecycle-stale overlaps reported separately as stale-suppressed), **Paired sensor coverage**, stage/gate ledgers with no `HUMAN_TURN`, approval gates waiting for a human for more than 24 hours, plugin advisory checks, uncommitted workspace records, an open in-flight compose question, fresh background-subagent state, a file an earlier version wrote for the compose question (no longer read, safe to delete), and, when `repos.json` exists, declared-repo and managed-`.gitignore` drift. A background-subagent entry older than 2 hours fails with the exact `rm aidlc/.aidlc-*` remediation; doctor never deletes it. **Hook drops** is conditional: a hook that silently degraded (e.g. a plugin compose that could not apply a contribution, or a failed recompile) records a severity-tagged line to `<hooks-health>/<hook>.drops`; a `[degraded]` drop **fails** doctor (so a CI gate catches a half-applied plugin), while any other drop is a passing row naming each hook's most frequent reasons (each up to its first colon; the detail stays in the file), raised to a `Hook failures, the latest within the last day` warning while the hook's latest failure is under 24 hours old (it clears a day later or when you delete the file; an `[advisory]` line never raises it). A hook's normal decisions, such as the Stop hook letting a turn end because you have to answer first, go to `<hook>.trace` and are never counted. The plugin compose hook rewrites its drops file each run, so fixing the cause and re-composing self-clears it. Clean and warnings-only reports exit 0; any failed check exits 1. Healthy rows collapse by section unless `--verbose` is present, while every warning and failure remains visible. The report writes to stdout either way. Core checks are **read-only**: on a fresh shell with no intent yet they create nothing, so the command is safe to run before the first intent is created. Plugin checks execute installed plugin code that is required by convention to be read-only, but the runtime cannot enforce that property. Once an intent exists doctor records a `HEALTH_CHECKED` (and `GUARDRAIL_LOADED`) audit row.

On Kiro IDE and Kiro CLI v3, it also checks each ignore source independently for rules hiding `.kiro/`, naming the file and line. Global-source matches fail; workspace-source matches warn because doctor cannot tell which Kiro you run: Kiro IDE's `kiroAgent.agentIgnoreFiles` governs whether they apply there, and Kiro CLI v3 applies `.kiroignore` but not `.gitignore`. See [Kiro Read Denials](#kiro-read-denials). On every harness but Cursor it also warns "AIDLC hooks have not run in this project yet" when no AI-DLC hook has run in the project. That is expected before your first chat message; after one, its fix names your tool's step (see, for example, [Kiro IDE hooks not running](#kiro-ide-hooks-not-running) or [GitHub Copilot hooks not running](#github-copilot-hooks-not-running)). On Kiro CLI, a workflow whose hooks never ran fails "Hooks have never executed" with the engine its hooks need; see [Kiro CLI hooks not running](#kiro-cli-hooks-not-running).

The **Workspace record visibility** advisory, beside the uncommitted-records
row, catches user ignore rules added after config. It names the rule's file,
line, and hidden committed record paths; narrow the rule rather than
force-adding individual records. This warning does not change doctor's exit
code. The row is absent when the records are visible, outside a Git repository,
or when Git is unavailable.

On Claude Code, doctor also reads the machine-managed `managed-settings.json` and alphabetical `managed-settings.d/` fragments. If the effective `allowManagedHooksOnly` value is `true`, organization policy blocks every hook declared by the project's `.claude/settings.json`; only your Claude Code administrator can lift that policy. When that policy is not set and heartbeats are still absent after workflow progress, set `"disableAllHooks": false` in this project's `.claude/settings.local.json`; it works in the same chat.
When a workflow has issues, `--doctor` also prints a **Workflow diagnosis** section listing structured findings (unresolved gates, a missing runtime graph, cold hooks, and similar "it will not advance" causes) — the same analysis `--doctor --export` writes to its report.

See [CLI Commands](12-cli-commands.md#aidlc-doctor-health-check) for full details on what each check validates and how to fix failures.

---

## Sharing a Diagnostic Report

When a workflow is stuck or misbehaving — a gate that will not open, a stage that
will not advance, an approved report repeatedly refused — and you want a
maintainer to look, run:

```
/aidlc --doctor --export
```

This runs a fresh `--doctor` pass, then writes a small, **redacted** diagnostic
report to `aidlc/diagnostics/` (override with `--output <dir>`). It packages
a timestamped `.tar.gz` when a system `tar` is available; otherwise it keeps the
report directory and tells you to compress it yourself. Share that archive (or
directory) — it carries the diagnosis and redacted evidence, **not your work
product**. No workspace source, raw state/audit/runtime-graph files, or
artifact/contribution/question/memory bodies are included; paths are normalized,
intent ids and Unit names are hashed, and secret-like values are scrubbed.

The report reconstructs the workflow timeline from the audit trail and runs
deterministic condition→remedy rules. The two most common causes it catches:

- **Unresolved approval gates** — a stage whose gate never resolved is the single
  most common "it will not advance" cause.
- **Missing runtime graph / cold hooks**: a runtime graph that is absent, or a
  hook that last fired more than a day before the newest stage or approval step
  recorded on the intent, points at a recompile that did not run. Opening a chat,
  a prompt and doctor's own audit rows are not counted as work, so an intent
  nobody is working on does not read as having cold hooks.

`report.md` inside the report lists every finding with a remedy; a remedy that
names a recovery bypass (such as an `AIDLC_DISABLE_*` env var) is flagged
as not safe to automate. See [CLI Commands](12-cli-commands.md#aidlc-doctor-export-write-a-diagnostic-report)
for the full report contents and safety model.

---

## Next Steps

- [State Tracking and Audit Trail](10-state-and-audit.md) — State file structure
- [Session Management](11-session-management.md): carrying on after compaction
- [CLI Commands](12-cli-commands.md) — `--doctor`, `--status`, `--stage` usage
- [Glossary](glossary.md): definitions for compaction, hook
