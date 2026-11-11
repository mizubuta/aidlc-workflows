# AI-DLC on GitHub Copilot (CLI + VS Code)

Copilot ランタイムは、このフレームワークのハーネスディストリビューションの 1 つで、**GitHub
Copilot** 向けのものです。1 回のインストールで Copilot の両方の画面に対応します: スタンドアロンの
Copilot CLI（`copilot`）と VS Code のエージェントモードです。GitHub は両者のプロジェクト探索パスを
同じもの（`.github/skills/`、`.github/agents/`、
`.github/hooks/`、ルートの `AGENTS.md`）に統一したため、フレームワークは両者が
読む 1 つのツリーを提供します。決定論的なコアは 1 つで、ハーネスは多数: エンジン、ステートマシン、
監査ログ、グラフ、スウォームのレフェリー、学びのゲートはすべてのディストリビューションでバイト単位で同一であり、
異なるのはシェルだけです。ソース/開発用のツリーは、`bun scripts/package.ts copilot` によって `core/` +
`harness/copilot/` から無視対象のローカル `dist/copilot/` に**生成**されます。手で編集しないでください。

完全なオンボーディングは引き続きルートの `AGENTS.md` にあります: Copilot 固有のセットアップと、
メソッドファイル用のライブな `@` インポートのブロック、続いて中立的なプロジェクトの
ガイダンスです。Copilot が管理するルートのブロックは専用のままであり、中立的なブロックだけを提供する
ハーネスとは共有されません。プロジェクトの指示をマージするときは、これらのインポートを残してください。

## Layout: the engine dir and the .github shell

- **`.aidlc/`** — AIDLC のエンジンツリーです（ツール、フック + Copilot アダプター、
  エージェント、ナレッジ、スコープ、センサー、aidlc-common、そして Copilot CLI のコマンドランチャーである
  `bin/aidlc-engine`）。どちらの Copilot の画面も
  これをスキャンしません。利用者に見えるものはすべて `.github/` に載ります。
- **`.github/`** — ネイティブに読み込まれる、`aidlc` という名前の出力だけです: フックの
  配線（`hooks/aidlc.json`）、14 のペルソナのカスタムエージェント
  （`agents/aidlc-*-agent.md`）、そして完全なスキルツリー（`skills/aidlc*/` —
  オーケストレーター、ステージごとのランナー、スコープのランナー、セッションスキル）です。リポジトリ
  自身の `.github/` の内容（ワークフロー、テンプレート）には手を付けません:
  インストールはこれらのファイルをマージし、プレフィックスによってすべて衝突しません。
- **`.vscode/settings.json`**: VS Code の設定 `chat.agent.maxRequests` を 1 つだけ、
  プロジェクトがそれを設定していない場合にのみ追加します（
  [VS Code request cap](#vs-code-request-cap) を参照）。コピー用のランタイムには
  このファイルは含まれません。

## Prerequisites

- **Copilot CLI ≥ 1.0.74 および/または VS Code ≥ 1.130** — 次の機能が検証済みのバージョンです:
  PascalCase でのフック登録（その結果、両方の画面が同一の
  snake_case のペイロードを渡します）、ブロッキングする PreToolUse の拒否チャネル、ブロッキングする
  Stop フック、そして `.github` のスキル/エージェントの探索。
  `copilot --version` / `code --version` で確認してください。VS Code のエージェントフックは Preview
  機能であり、doctor はオプションの Copilot CLI のバージョンしかチェックしないため、
  `code --version` は自分で確認してください。
  [ファシリテーターガイド](../facilitator-guide.md#github-copilot-on-windows)
  には、フックが実行されることを確かめる準備状況のチェックがあります。
- **bun** は、ソース/開発用の `dist/`
  プロジェクションを生成または実行する場合にのみ必要です。ネイティブのインストールとバージョン付きのリリースランタイムは `aidlc` を使います。
- **フォルダーの信頼**: Copilot の各画面がそれぞれ独自にチェックします。
  - Copilot CLI は、`trustedFolders`
    リストがカバーするフォルダー（そのフォルダー自体またはその上位のフォルダー）でのみリポジトリのフックを実行します。このリストは
    `COPILOT_HOME`、なければ `~/.copilot`
    （Windows では `%USERPROFILE%\.copilot`）の下の `config.json` にあります。対話型の `copilot` の実行では、
    プロンプトを受け付ける前にフォルダーの信頼を確認するよう求められます。記録するには "Yes, and
    remember this folder for future sessions" を選んでください。リストがそのフォルダーをカバーしていない場合は、`aidlc config`
    がそれを知らせます。
  - VS Code のエージェントモードは、そのリストを決して読みません。そのフックは、
    信頼されたワークスペース（VS Code Workspace Trust）で、かつ **Chat: Use Hooks**
    設定（`chat.useHooks`）がオンの場合にのみ実行されます。この設定は、組織がオフにできる
    プレビュー機能です。プロジェクトがこの設定をしていない場合、`aidlc config` はフォルダーの
    `.vscode/settings.json` でこれをオンにします。フォルダーの値は、オフになっている
    ユーザー設定より優先されます。スキップされたフックはチャットに
    メッセージを残しません。Agent Debug Logs パネルに表示されます。
  - CLI のリストがフォルダーをカバーしていない場合、`/aidlc --doctor` が警告します。
    VS Code のスイッチは確認できませんが、メッセージに対してフックが 1 つも実行されなかった場合は、
    AI-DLC がチャットでそのことを伝えます（後述の「AI-DLC says when its hooks
    have not run」を参照）。
- **モデルプロバイダー** — このインストールには、モデルを固定するものは何もありません。サインイン済みの
  Copilot はそのまま動作します。BYOK は GitHub の認証がまったくなくても動作します（例: Amazon
  Bedrock の Anthropic 互換エンドポイント:
  `COPILOT_PROVIDER_BASE_URL=https://bedrock-runtime.<region>.amazonaws.com/anthropic`、
  `COPILOT_PROVIDER_TYPE=anthropic`、ベアラートークン、そして
  `COPILOT_MODEL=<catalog name>` + `COPILOT_PROVIDER_WIRE_MODEL=<Bedrock
  model id>` — 一式は `copilot help providers` に記載されています）。VS Code では、
  モデルピッカーまたは Custom Endpoint プロバイダーを使ってください。AI-DLC のすべてのエージェントは、
  Copilot セッションのモデルとエフォートを使うため、それらはそこで選んでください。
  [モデルと推論量の選び方](../18-install-and-lifecycle.md#choosing-a-model-and-effort) を参照してください。

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
aidlc config --harness copilot
aidlc doctor
```

### Versioned manual-copy alternative

[インストールとライフサイクル: コピーチャネル](../18-install-and-lifecycle.md#copy-channel)
の説明に従って特定リリースの `aidlc-copy-runtime-X.Y.Z.tar.gz` をダウンロードして展開し、
`RUNTIME_ROOT` を展開した `runtime/` ディレクトリに設定します。

1. ディストリビューションをプロジェクトにコピーします:

   ```bash
   mkdir -p your-project/.aidlc your-project/aidlc your-project/.github
   cp -R "$RUNTIME_ROOT/copilot/.aidlc/."  your-project/.aidlc/
   cp -R "$RUNTIME_ROOT/copilot/aidlc/."   your-project/aidlc/    # the workspace shell — a sibling of .aidlc/, not inside it
   cp -R "$RUNTIME_ROOT/copilot/.github/." your-project/.github/  # MERGE — everything is aidlc-prefixed, nothing of yours is overwritten
   ```

2. コピーしたもの自身のセットアップを 1 回実行します:
   `cd your-project && bun .aidlc/tools/aidlc.ts config --from "$RUNTIME_ROOT" --harness copilot`。
   これは、AI-DLC の行を `AGENTS.md`（メソッドのインクルードを含む）と
   `.gitignore` の既存の内容の後に追加します。プロジェクトにこれらのファイルがない場合は
   作成します（クローンごとの監査シャードは意図的にコミットされ、
   カーソルとマシンローカルのランタイムは無視されたままです）。これを実行しない場合は、最初のチャットの開始時に AI-DLC が
   それらを追加します。
   VS Code の場合は、`.vscode/settings.json` が `"chat.agent.maxRequests": 200` のキーを設定していなければ、
   それも追加してください（
   [VS Code request cap](#vs-code-request-cap) を参照）。コピー用のランタイムには
   そのファイルが含まれないため、コピーによって自分のファイルが置き換えられることはありません。

3. フォルダーを信頼します: プロジェクトで `copilot` を対話型で 1 回起動し、
   信頼のプロンプトを承認します（または、プロジェクトの絶対パスを
   `~/.copilot/config.json` の `trustedFolders` に追加します）。

4. `/aidlc --doctor` を実行し、続いて `/aidlc` の後に作りたいものを書いて実行します —
   どちらの画面でもかまいません。

Bun 形式のプロジェクションが必要なフレームワーク開発者は、リポジトリをクローンし、
`bun install --frozen-lockfile` と `bun scripts/package.ts` を実行してから、
無視対象のローカル `dist/copilot/` の出力を使ってください。

## What's different on this harness

- **1 回のインストールで 2 つの画面。** スキル、ペルソナ、指示、フックは、
  CLI と VS Code のエージェントモードで同じように動作します。相違点は
  以下で明示しています。
- **ランナーは、入力したときにだけ開始します。** 生成されるすべてのランナー
  （`/aidlc-bugfix`、`/aidlc-<stage>`、`/aidlc-init` など。プラグインの
  ランナーを含む）は `disable-model-invocation: true` を持つため、エージェントが
  自分からランナーを開始することはなく、ランナーの説明は
  Copilot のスキル一覧に載りません。入力すれば、VS Code のチャットと
  対話型の CLI で従来どおり動作します。ヘッドレスの `copilot -p "/aidlc-bugfix ..."` は、入力した行を
  プレーンテキストとしてエージェントに渡します。ルートの `AGENTS.md` が、その
  ランナーのファイルを読んで従うようエージェントに指示するため、ランナーはそれでも実行されます。
- **質問は番号付きの散文の選択肢として表示されます。** どちらの画面にも
  ネイティブのピッカーツールがありますが、ピッカーでの回答はツールの結果として返り、
  人間の存在を確認するガードが必要とする、信頼済みの `UserPromptSubmit` イベントを発火させません。
  セッションで選択されたワークフローが有効な `Status: Running` の状態にある間は、
  マッチャーなしの PreToolUse ガードがそれらのピッカー呼び出しを拒否し、番号付きの散文を表示して
  ターンを終了するようモデルに指示します。実行中のワークフローがない場合は
  （完了済みや使用できない状態を含む）、ネイティブのピッカーには手を付けません。
  回答となるのは人間の次のチャットメッセージです。`[Answer]:` タグを持つ質問ファイルが
  引き続き信頼できる情報源です。
- **自分で入力したものだけが返信として数えられます。** エージェントが作業を
  サブエージェント（レビュアー、ビルダー）に渡すとき、VS Code と Copilot CLI は、
  エージェントが書いた指示を、チャットメッセージと同じ方法でフックに渡します。
  AI-DLC はその指示を認識し、決してあなたのターンとして記録しません: それは
  承認を満たさず、あなたの回答や変更の要求として読まれず、
  入力によるスイッチも適用しません。あなたが入力したものは、サブエージェントの実行中であっても、
  従来どおり数えられます。例外は VS
  Code だけで、チャットでサブエージェントが開始してから、その指示が届くまでの最初の数秒間です:
  その間のメッセージで AI-DLC がどの指示とも照合できないものは、
  その指示とみなされて数えられず、それがあなたのものだった場合は
  もう一度返信するよう求められます。Copilot CLI は指示をチャットに送ることがないため、
  そこでは入力したメッセージがすべて数えられます。作業が保留中のままターンが終わったときに
  AI-DLC がエージェントに送る注記（"The last AI-DLC step
  stopped on a problem: ..." など）も、あなたのターンではありません: Copilot CLI はそれを
  チャットメッセージとして返しますが、AI-DLC はそれを認識し、あなたのものとして記録しません。
- **`/aidlc` とリクエストを入力すると、その言葉が保たれます。** Copilot は、
  スラッシュ名で呼び出したスキルを、入力した行としてではなく、あなたの言葉を埋め込んだ
  スキル自身のテキストとして AI-DLC に渡します。テキストが、インストールされたスキルファイルに
  あなたの言葉を入れたものと完全に一致する場合、AI-DLC はその行
  （`/aidlc` とあなたの言葉。言葉はスキルのテキストが保持している箇所からそのまま取ります）を
  組み立て直します。そのため、エージェントの最初のコマンドにリクエストがまったく含まれていなくても、
  リクエストは記録に残り、作業に届きます。そのファイルと完全には一致しないものは、
  Copilot が渡したとおりに記録されます。
- **AI-DLC のエージェントはステージのルールを受け取り、ビルダーはプランの承認を
  待ちます。** エージェントが AI-DLC のエージェントを VS Code の
  `runSubagent` ツールで開始しても CLI の `task` ツールで開始しても、AI-DLC は現在の
  ステージのルールをそれに渡します。また Code Generation の間は、プランを承認するまで
  developer エージェントを開始しません。エージェントは、どちらの画面でも同じ「先にプランを
  承認してください」という拒否を受け取ります。
- **フックが実行されなかった場合は AI-DLC がそう伝えます。** どちらの画面も、リポジトリのフックを
  チャットに何も表示せずにスキップする（前述のフォルダーの信頼を参照）ため、AI-DLC がそれを
  監視しています。最初のメッセージに対してフックが 1 つも実行されなかった場合、または一度もフックが
  実行されていないワークフローでステージが開始した場合、次のステップは何も
  作業をしません: VS Code では、エージェントが
  フォルダーの `.vscode/settings.json` で Chat: Use Hooks を自らオンにし（VS Code がその
  編集の許可を求めます）、"Fixed. Send your next message here to carry on." と伝えます。同じチャットでの
  次のメッセージはフック付きで実行されます。設定がすでにオンだった場合は、
  組織がそれをオフにしていると伝えます。人間の存在のチェックをオフにしている場合は、
  作業は続行され、エージェントがその旨を 1 回伝えます。そのフォルダーで最初の Copilot チャットを
  する前は、`/aidlc --doctor` が "AIDLC hooks have not run in this project yet" と警告します。
  ステージの後は、同じ手順を示して失敗します。実行はされたものの
  クラッシュしたフックは、それでも操作を通し、エラーの行を
  `.aidlc-engine/hooks-health/<hook>.drops` に残します。doctor はこれを読みます。
- **フックはネイティブに強制します。** アダプター
  （`.aidlc/hooks/aidlc-copilot-adapter.ts`。
  `.github/hooks/aidlc.json` で配線されます）は、コアのガードによるブロックを Copilot の
  `permissionDecision: deny` に変換します — レビュアーの読み取り範囲の制限と
  状態遷移のガードが、実際にツール呼び出しを拒否します。SessionStart と Stop の
  応答は、CLI のトップレベルのフィールドと、VS Code が必要とする
  `hookSpecificOutput` のエンベロープの両方を持ちます。
  CLI では実環境で検証済みです。VS Code のエージェントモードでは、同じ拒否/ブロックの
  チャネルが文書化されており、アダプターは
  `runTerminalCommand`、`createFile`、`editFiles`、`readFile` などの文書化された名前を正規化しますが、
  IDE 側はまだ実環境で検証されていません — 検証されるまでは、IDE での強制は
  ベストエフォートとして扱ってください。
  アダプターは、Copilot ランタイム自身のツール名と形式も読み取ります。
  これらは、名前を変更しないホストが送ることがあるものです: シェルとしての `powershell` と `pwsh`、
  `paths` を伴う `grep`、そしてパッチが
  プレーンテキストまたは `toolArgs` で届く `apply_patch` です。
- **VS Code では、AI-DLC の定型コマンドは Allow のプロンプトなしで実行されます。** VS
  Code のエージェントモードは通常、ターミナルコマンドのたびに "Run command? Allow / Skip" と尋ねるため、
  ワークフローの各ステップがクリックを待つことになります。アダプターは、
  AI-DLC がステージ中に実行する定型コマンドに対して `allow` と答えます: `next`、
  `continue`、`report`、`park`、読み取り専用の `next` の形式、エンジンが指定するフラグ付きの `doctor`（または
  `--doctor`）、`--version`、`--status`、
  `--help`、AI-DLC 自身のチェック（`engine sensor-traceability`、
  `sensor-required-sections`、`sensor-upstream-coverage`、
  `sensor-claim-sources`。これらはファイルを読みます。トレーサビリティのチェックは、ステージがそれを実行して
  ステージ自身のカバレッジ表を開始する場合に、記録内にその表を書き込みもします）、そして
  AI-DLC 自身のコマンド表にあるプロジェクトのコマンド（`engine log`、`engine state`、`engine runtime`、
  `engine learnings`、`engine testing-posture`、`engine intent list`、
  `engine intent add-checkout` など）で、直接、ソースディスパッチャー、コンパイル済み、ツールスクリプトのいずれの書き方でもかまいません。
  コピーチャネルの `aidlc-utility.ts <verb>` も、
  `engine workspace <verb>` の書き方と同じ答えを得ます。答えるのは、次のすべてが成り立つ場合だけです:
  - 呼び出しが VS Code のチャットセッションを持ち、AI-DLC のすべてのガードを通過し、
    ワークフローのコマンドがこのセッションのワークフローと照合されていること。
  - PowerShell、cmd、POSIX シェルのすべてが同じように読む、1 つの単純なコマンドである:
    連結、パイプ、末尾の 1 つの `2>&1` 以外のリダイレクト、
    前置きの環境変数の代入、シェル展開がなく、それらのシェルのいずれかが特別に扱う文字
    （`$`、バッククォート、`%`、`^`、`!`、
    `#`、波括弧、`@`、`\`、タイポグラフィ上の引用符など）も、引用符の中であっても含まないこと。
    引用符で囲まれた語は、スペース、`?`、丸括弧、`;` を含めてよく、ダブルクォートの中では
    アポストロフィも含められます。ダブルクォートで囲まれたテキストは、スペースを含む場合は
    `|`、`&`、`<`、`>` も含められます。たとえば
    `--details "Q1: A - both ends included (closed range)"` や
    `--options "Keep the note|Skip it"` です。引用符の外、シングルクォートの中、
    スペースのないダブルクォートの中では、それらの文字があるとプロンプトが残ります。プレーンな
    ASCII 以外のテキスト（たとえばアクセント付きの文字）もプロンプトを残します。Windows で、
    VS Code のターミナルが PowerShell または cmd の場合、バックスラッシュは単なるパスの
    区切り文字なので、`C:\work\app`、`.aidlc\tools\...`、
    `--project-dir 'C:\work\app'`（エンジンがプロジェクトフォルダーを出力する形式）のようなパスは
    クリックなしで実行されます。プロンプトが残るのは、閉じ引用符の直前のバックスラッシュだけです。
    Git Bash や WSL のターミナルでは、バックスラッシュがあると引き続きプロンプトが残ります。
    PowerShell のターミナルでは、プロジェクトフォルダー
    そのものへのフルパスでの `cd` または `Set-Location` を 1 つだけ先頭に置けます:
    `cd C:\work\app; aidlc engine orchestrate next` は
    `aidlc engine orchestrate next` と同じように実行され、プランが承認待ちの間も同様です。
    それ以外のフォルダー（サブフォルダーを含む）への `cd` はプロンプトを残します。インストールされた
    `aidlc` は、実行されたフォルダーをプロジェクトとみなすためです。コマンドを
    `cd` なしで実行するか、プロジェクトフォルダーそのものに `cd` してください。
  - パスとして読める引数はすべてプロジェクト内にとどまること。
  - AI-DLC に独自に実行するコマンドを渡すオプション（`--check-cmd`）がないこと。
  - 素の `aidlc` はインストールされたランチャーであること: プロジェクトのルートまたは
    `PATH` 上のフォルダーに `aidlc` という名前のファイル（`aidlc.cmd` や、`PATHEXT` に
    列挙された任意の拡張子のもの）がある場合、そのコマンドにはプロンプトが残ります。
    cmd は `PATH` を検索する前に作業フォルダー内のファイルを実行するためです。

  それ以外のものには AI-DLC は答えないため、VS Code のプロンプトか、あなた自身の
  承認設定が適用されます: エージェントがプロジェクトのために書くコマンド（ビルド、
  テスト、`git` など）、マシンレベルのコマンド（`update`、`uninstall`、
  `use`、`config`、`system ...`）、ホストが実行するフック、アダプター、ステータスラインの
  エントリ、そして次の AI-DLC のコマンドです。これらはプロンプトを残すので、実行前に
  1 つずつ確認できます:
  - 作業を破棄またはマージするコマンド: `engine worktree discard`、
    `purge`、`merge`、`unit land`、`engine intent archive`、
    `engine swarm finalize`、`engine bolt abort`（
    `--discard` の有無にかかわらず。Bolt の中止にはあなたの同意が必要なためです）。
  - 表示されるステージ、ゲート、レビューを変えるコマンド:
    最後の質問以降にあなたが返信していない場合の `engine recompose`
    （プランの変更を承認した後、それを適用する recompose は
    クリックなしで実行されます）、`next --skip`、`next --add`、`engine jump execute`、
    `engine scope change`、`engine intent create --skip`、
    `engine intent remove-checkout`（レビューの対象からコードを
    外します）、`engine config set` と `next config set`、
    `engine bolt set-autonomy`、その作業について（あなた自身または Guard
    Policy によって）状態遷移のチェックがオフになっている間の `engine state` のステータス変更
    （その場合は変更が実際に反映されるためです）、そして
    ゲートの設定コマンド（`set-unit-gate-rhythm`、`set-construction-checkpoints`、
    `set-skeleton-stance`、`set-status`）。そのチェックがオンの間は、
    AI-DLC が `engine state` のステータス変更そのものを拒否するため、クリックするものは
    ありません。`set-construction-checkpoints` と、
    あなたのチェックの 1 つ（プランの承認、サマリーの
    確認、フェンス、Guard Policy）に対する `engine config set` は、最後の判断以降にチャットで
    その変更を求めていた場合はクリックなしで実行され、チェックをオンに戻す場合は
    常にクリックなしで実行されます。
  - 進行中の作業を切り替えるコマンド: `engine intent switch`（または
    `engine intent <name>`）と `engine space switch`（または `engine space <name>`）。
  - リモートを通じて claim や承認を共有する、チームの `unit` コマンド
    （`unit merge-status` 以外のすべて）。
  - AI-DLC が提供していないコードを実行するコマンド、またはインストールされたスキルを書き換えるコマンド:
    `engine sensor fire`（チェックが指定するものを何でも実行します）、
    `engine sensor-linter` と `sensor-type-check`（プロジェクトの
    リンターと型チェッカーを実行します）、`engine knowledge onboard`、`sync`、
    `engine workspace document-input --onboard`（ハーネスが指定するドキュメント
    抽出ツールを実行します）、
    `engine plugin sync`、`select`、`build`、`plugin build`、
    `engine gen runners` と `runner-scopes`。

  エンジン自身の適用可否のチェックによってエージェントがスキップできる条件付きステージは、
  クリックなしのままです。`doctor` も同様で、自分で実行したときと同じように、
  リリースフィードから更新チェックを更新することがあります。クリックを省いても、
  あなたの判断が記録されるわけではありません: エンジンがステージの承認を記録する前に、
  ステージの開始以降かつ最後の判断以降に、あなたがチャットメッセージを送っている
  必要があり、それはプロンプトのフックが記録します: ゲートが表示された後のメッセージか、
  エージェントがゲートへの回答として示すメッセージです。このチェックが確認するのは、あなたがターンを取ったことであって、
  あなたが何を意図したかではないため、エージェントの報告内容を読んでください。これは VS Code 自身の
  自動承認を使わないため、組織のポリシーで自動承認がオフになっている環境でも動作します。
  ただし、AI-DLC の他の部分と同様に、チャットのフックが有効である必要があります。Copilot
  CLI ではアダプターは許可の判断を返しません。同じコマンドが CLI で
  プロンプトなしに実行される仕組みは、2 項目後で説明します。
- **Copilot CLI では、作業に関する AI-DLC の記録はプロンプトなしで
  書き込まれます。** `/aidlc` スキルの使用中、Copilot は
  `aidlc/spaces/<space>/intents/` 配下のファイルを "Do you want to
  create ...?" と尋ねずに作成・編集します。スキルがそのパスを許可しているためです（
  `.github/skills/aidlc/SKILL.md` の `allowed-tools`）。Copilot はパスを末尾で照合するため、
  パスが `aidlc/spaces/<name>/intents/` で終わるあなた自身のフォルダーも対象になります。
  あなた自身の `--deny-tool` は引き続き優先され、それ以外のファイル（スペースの `memory/` のファイルを含む）
  への書き込みについては、Copilot は従来どおり確認を求めます。
- **Linux と macOS の Copilot CLI では、AI-DLC の定型コマンドはプロンプトなしで
  実行されます。** VS Code で Allow のクリックなしに実行されるコマンド（上記の
  一覧。条件も同じ）は、
  `.aidlc/bin/aidlc-engine <command>` として実行されます。これは AI-DLC がプロジェクトに
  インストールするランチャーで、`/aidlc` スキルがそれを許可しています（スキルの `allowed-tools` 内の
  `shell(.aidlc/bin/aidlc-engine)`）。ランチャーは、AI-DLC のフックが渡したコマンドだけを
  実行します。それ以外のもの（エージェントが入力したものや、フックがオフの状態で実行されたもの）は、
  代わりに実行すべきプレーンなコマンドを示して停止し、Copilot がそれについて確認を求めます。
  VS Code でプロンプトが残るコマンドはここでも残り、エージェント自身のコマンドも
  同様です。Windows のターミナルでは、AI-DLC のコマンドは従来どおり
  確認が求められます。
  - Copilot はシェルのルールをコマンドの最初の語に対して照合します。これらの
    コマンドは今ではランチャーで始まるため、あなた自身の `--deny-tool=shell(bun)`
    や `shell(aidlc)` ではもう止まりません。止めるには、ランチャーも拒否してください:
    `--deny-tool='shell(.aidlc/bin/aidlc-engine)'`。
  - 組織が Copilot の権限を管理している場合、ランチャーによってダイアログが
    減ることはありません: その
    [enterprise managed settings](https://docs.github.com/en/copilot/reference/enterprise-administrators/enterprise-managed-settings)
    に何らかの権限ルールがあると、Copilot は、それらが許可していないすべてのコマンドについて
    （AI-DLC のものを含め）確認を求めます。管理設定で `Shell(bun *)` や
    `Shell(aidlc *)` だけを拒否しても、AI-DLC のコマンドはブロックされず、Copilot が
    確認を求めるだけです。それらをブロックするには、管理者が bun や aidlc に加えて
    `Shell(.aidlc/bin/aidlc-engine)` と `Shell(.aidlc/bin/aidlc-engine *)` も
    `permissions.deny` で拒否します。
  - `shell(bun)` または `shell(aidlc)` だけを許可する `copilot -p` のスクリプトでは、
    `--allow-tool='shell(.aidlc/bin/aidlc-engine)'` を追加します。これがないと、Copilot は
    ランチャーを拒否します: `-p` には確認を求める相手がおらず、スキルのルールも
    そこでは適用されないためです。
- **コマンドの追跡は厳密かつベストエフォートです。** AI-DLC は、単純で直接的な
  オーケストレーター、ソースディスパッチャー、実際のコンパイル済みの `next`、`continue`、
  `report`、`park` のコマンドを追跡します。末尾の 1 つの `2>&1` はサポートされます。調査用の
  コマンドは、`aidlc` という部分文字列からは分類されません。曖昧なラッパーや、
  引数に有効なシェル展開（`$VAR`、グロブ、
  波括弧展開、先頭の `~`）を含むコマンドは、変更されず追跡もされずに実行されます。シェルが最終的に生成する
  argv を、フックがハッシュできないためです。直接的に見える複合コマンドは拒否されます。
  現在の物理的なプロジェクトの外を指す明示的な `--project-dir` は、
  現在のプロジェクトの調整情報が書き込まれる前に拒否されます。
- **AI-DLC が照合できない `continue` でも先に進みます。** フックが AI-DLC のコマンドについて
  自身の記録を見つけられない、または信頼できない場合（たとえば記録が
  削除された後や、フックとターミナルでプロジェクトのパスの表記が
  異なる場合）、フックはコマンドを拒否せずに実行させ、エンジンが
  ディスクから応答します: 自身の記録が一致すれば次のパートを、一致しなければ現在の
  ステップを返します。"could not match this Copilot
  command" の後に再びパート 1 が表示されることは、もうありません。監査には、そのような通過ごとに
  `COORDINATION_STOOD_ASIDE` の行が 1 つ残ります。VS Code では、このように通過した定型
  コマンドも、Allow のクリックなしで実行されます。
- **どのハーネスでも、継続のリプレイはエンジンが担います。** Copilot は、
  Claude、Codex、Cursor、Kiro、
  Kiro IDE、opencode と同じ、記録ローカルでアトミックな使い捨てのカーソルを使います。まずネイティブのトークン検証が実行され、
  次にエンジンが、アクティブなディレクティブのロックの下で、トークン全体の SHA-256 を比較し、
  stdout の前に正確な後継を公開します。Copilot のセッションの所有権と
  配信の証拠は、そのマーカーを補強しますが、リプレイを担うわけではありません。欠落した、
  不正な形式の、v1 の、共有以前のマーカーは、同じトランザクション内で 1 回だけ
  回復します。新たな `next` はカーソルをリセットします。クラッシュ、移行、ロールバック、ファイルシステムの
  制限については、Developer Reference の共有カーソルの契約を参照してください。
- **Stop は、現在配信済みの Copilot のディレクティブを保持します。** ホストの正確な
  `tool_use_id`、または書き換えられたエンジンの入力を通して運ばれ、
  PostToolUse によって返されるアダプターの ID によって、セッションスコープの Stop と
  Resume の動作のための配信を確定できます。正確な対応付けができない場合は、実行は
  追跡なしで許可され、Post は推測しません。新たな単純な `next` が追跡付きの
  配信を回復します。対応付けが失われても、恒久的な拒否にはなりません。claim が
  試みられた後は、プロジェクト、状態、セッションの所有権による拒否は明示的な拒否となります:
  別のセッションが、所有者の現在のトークンを追跡なしの作業として実行することはできません。
  ワークフローを先に進める `report`（承認、スキップ、完了した
  ステップ）は停止点ではありません: エージェントはすぐに `next` を実行し（または、
  利用者が同じ返信でワークフローをいったんそこで止めるよう求めた場合は park し）、
  それでも停止した場合は、Stop が新たな `next` を、停止を求めた利用者には `park` を
  示すため、次のステージは同じターンの中で開始します。そこでターンを終えるのは、ワークフロー完了の報告と、分離された単一ステージの実行だけです。
- **従来の Resume と会話の待機はセッションスコープです。** Stop は、
  本当の会話としての応答がきれいに終わることを許します。
  2.6.19 より前のインストールが書いた Resume マーカーは、所有者スコープのままです。明示的な `next --resume`
  はそれを置き換え、直接続行します。プロンプトのテキストとルールの内容は、
  調整用のマーカーには保存されません。
- **ホストの証拠は意図的に範囲を限定しています。** 書き換えと、運ばれた ID のエコーは、
  macOS 上の Copilot CLI 1.0.79 の非対話モードで実環境で検証されました。
  VS Code の `tool_use_id`、`updatedInput`、`tool_response` の経路は、
  文書化された Preview の契約に基づいてカバーしていますが、ここでは実環境で検証されていません。Copilot
  のクラウドエージェントは、このリリースでサポートされる AI-DLC の画面の対象外です。
- **ほとんどのステージは、追加の 1 ステップでルールを読み込みます。** VS Code のターミナルツールは、
  コマンドの結果を 20,000 文字までしかそのまま保持しません。それより長いものは
  ファイルに保存され、チャットにはその先頭と末尾しか見えません。AI-DLC は、Copilot で出力する
  すべての指示を 19,000 バイト未満に収めるため、ルールが指示と一緒に
  収まらないステージは先にルールを送り、モデルはそれと一緒に出力された `continue`
  コマンドを実行してからステージを開始します。Construction では、
  これが各ユニットの各ステージで起こります。ネイティブのインストールでは、`aidlc update` を実行するとすぐに、
  進行中のワークフローにもこれが反映されます。
  `aidlc config` での更新は不要です。`.aidlc-version` で以前のリリースに固定された
  プロジェクトは、そのリリースのまま実行され続けるため、固定されたプロジェクトでこれが反映されるのは、
  このリリース以降で `aidlc config --pin` を実行したときです。
- **追加のステップをクリックなしで実行させましょう。** VS Code がターミナルコマンドごとに
  許可を求める場合は、Allow の選択肢から **Configure Auto Approve...** を選び、
  `chat.tools.terminal.autoApprove` 設定に `"aidlc engine orchestrate": true` を
  追加してください。これで VS Code は AI-DLC の
  ワークフローのステップを確認なしで実行し、他のコマンドは通常どおりの
  確認を求めます。ワンクリックの **Allow `aidlc ...` in this Session** や
  **Allow `aidlc ...` in this Workspace** でも動作しますが、これは
  `aidlc config`、`aidlc update`、
  `aidlc uninstall` を含むすべての `aidlc` コマンドのクリックを省いてしまいます。コピーした Bun ランタイムでは、ステップが
  `bun .aidlc/tools/` で始まるため、そこでのワンクリックの選択肢はすべての `bun`
  コマンドを許可してしまいます。AI-DLC 自身は自動承認の設定をインストールしません。
- **フックの配線は、設計上マッチャーなしです**: VS Code はフックのマッチャーを解析しますが
  無視するため、アダプターの各ターゲットは代わりに `tool_name` で自らフィルタリングします —
  マッチャーを使うと、IDE では黙って対象が広がってしまいます。
- **レビュアーの識別情報は、配信されるのではなく対応付けによって得られます**: PreToolUse のペイロードには
  呼び出しごとのエージェントのフィールドがありません。アダプターは
  SubagentStart/SubagentStop（VS Code の `agent_type`/`agent_id`
  フィールドを含む）で委譲を囲み、ちょうど 1 つのサブエージェントがアクティブな場合に識別情報を転送します。
  曖昧な重なりがある場合、その呼び出しについてはフェイルオープンとなります（レビュアーモジュールの散文による制限は
  引き続き適用されます）。
- **ペルソナは `model:` の固定を持ちません。** 2 つの画面では、モデルの
  値の構文が一致しません（CLI はフロントマターの文字列をそのまま BYOK
  プロバイダーに転送し、IDE の表示名はそこで 400 エラーになります）。エージェントはセッションの
  モデルを継承します — このハーネスでのティアのプロジェクションは、型としてモデルを省略します。
- **ワーカーのペルソナは、明示的な組み込みの `tools:` 許可リストを使います。** これは、
  ネストした委譲を防ぐために Copilot の `agent` 委譲ツールを省いています。Copilot
  には「agent 以外のすべて」という形式がないため、委譲されたワーカーは任意の
  MCP ツールを継承しません。
- **AIDLC のプラグインは Copilot ネイティブの画面を使います。** 合成されたプラグインのペルソナと、
  生成されたステージ/スコープのランナーは `.github/{agents,skills}` に配置されます。プラグインの
  選択はそれらのパスを再生成し、`.aidlc/skills` や
  `.opencode/agents` を作成することはありません。
- **セッションの終了**: VS Code は SessionEnd を文書化していないため、共有のフック
  マニフェストは両方のホストでこれを省いています。アダプターは、前のセッションを
  次の SessionStart で、推定した来歴とともに整合させます（codex のパターン）。
- **メソッドのインクルードは AGENTS.md の `@` インポートで読み込まれます**（CLI では
  実環境で検証済み。VS Code は `@` インポートの展開を文書化していますが、そこでは実環境で
  検証されていません）。`@` の行と `.github/agents/` のペルソナの対は、
  アクティブなスペースのメソッドを git で無視されたコピーとしてエンジンが置く
  `aidlc/active-memory/` を読みます。`/aidlc space <name>` が書き換えるのはそのコピーであり、ブロックではありません。
- **ステータスラインはありません**。`/aidlc --status` と、ゲートでの進捗の行を使ってください。
- **Construction のスウォームはサブエージェントへのファンアウトのみです**（`AIDLC_USE_SWARM=1` は
  通知付きの何もしない操作です）。
- **MCP**: 何も同梱されていません。サーバーを追加する場合は、ここで画面ごとに異なる点に注意してください —
  CLI は `~/.copilot/mcp-config.json` を読み、VS Code は `.vscode/mcp.json` を読みます。
  コンダクターはそれらを使えますが、委譲されたワーカーのペルソナは使えません。

## VS Code request cap

VS Code のエージェントモードは、1 ターンで `chat.agent.maxRequests` 回のリクエストを行うと停止し
（デフォルトは 50）、"Continue to iterate?" と尋ねます。チャットは誰かが答えるまで
黙って待つため、50 回を超えるツール呼び出しを容易に行う無人の Construction ステージは、
途中で一時停止したままになります。そのため、`aidlc config --harness
copilot`（初回インストール時と毎回の更新時）は、
プロジェクトの `.vscode/settings.json` に `"chat.agent.maxRequests": 200` を追加します:

- プロジェクトがそのキーを設定していない場合に限ります。チームがすでに設定した値は、
  `--force` を指定しても決して変更されません。
- 他のキー、コメント、レイアウトには手を付けず（ファイルは JSONC です）、
  ファイルがない場合は作成します。
- AI-DLC のものとして記録されるのは、AI-DLC が追加した値だけです。後のリリースが
  この設定を提供しなくなった場合、config は、AI-DLC が書いた値がまだ残っている場合にのみそれを削除し、
  ファイルは AI-DLC が作成した場合にのみ削除します。
  `aidlc uninstall` はプロジェクトのファイルを決して編集しないため、設定は残ります。
- AI-DLC がキーを追加した後、保持している設定ファイルからそれを取り除くかどうかは
  あなたの選択であり、config がそれを再び追加することはありません。設定ファイルが
  まったくないチェックアウトでは、再び追加されます。

この設定はウィンドウスコープであるため、プロジェクトの値がユーザー設定より優先されます。
AI-DLC の `.gitignore` のブロックは `.vscode/*` を git から除外するため、この値は
チェックアウトごとのものです: config は、それが実行された場所に値を追加します。コピーしたプロジェクト
（コピーチャネル）には、このファイルをマージする config の手順がなく、その
ランタイムもファイルを同梱していないため、キーは自分で追加してください。`/aidlc --doctor` は、
プロジェクトの値が 100 未満の場合、未設定の場合（その場合はユーザー設定、なければ VS
Code のデフォルトの 50 が適用されます）、数値でない場合（引用符で囲まれた数値を
含む）、または読み取れない場合に警告し、書くべき行を示します。AI-DLC が
キーを追加した後に、チームが保持している設定ファイルからそれを取り除いた場合は警告しません。

マルチルートのウィンドウは、このウィンドウスコープの設定を、フォルダーの `.vscode/settings.json` からではなく
`.code-workspace` ファイルから読みます。そのため
Copilot のプロジェクトでは、`aidlc system workspace-sync` は、生成する `aidlc.code-workspace` にも
`"settings": { "chat.agent.maxRequests": 200 }` を 1 回だけ書き込みます:
書き込むのは、まだ設定がないファイルに限ります。ファイルの設定にすでにあるキーと
値はチームのものであり、取り除かれたキーも含めてそのまま残ります。workspace-sync はファイルを書き直すため、ファイル内のコメントは保持されません。そのファイルが存在する場合は、doctor がそれもチェックします。
ワークスペースを開くときは常に、そのファイルが優先されるためです。

## Verify

```bash
cd your-project
copilot -p "/aidlc --doctor" -s --allow-all-tools   # or run /aidlc --doctor in VS Code chat
```

doctor は、エンジンツリーとアダプターのすべての依存関係、ルートの
`AGENTS.md`、`.github` の配線ファイル、Copilot CLI のバージョンの下限、
フォルダーの信頼をチェックします。Copilot CLI はオプションです:
VS Code のみのインストールは `Harness CLI: optional copilot is not installed` と報告して
合格します。VS Code は、CLI がない場合に "Install GitHub Copilot CLI? (y/N)" と尋ねる
独自の代替の `copilot` をターミナルの PATH に置くため、doctor と
初回のセットアップはそれを決して実行しません: そのフォルダーを飛ばして本物の
CLI を探し、見つからなければ未インストールと報告します。下限より古い CLI がインストールされている場合は
警告であり、失敗には決してなりません。このハーネスの決定論的なエンジンのテストは
`tests/unit/t248-copilot-packaging.test.ts`、
`t249-copilot-adapter.test.ts`、`t250-copilot-adapter-security.test.ts`、
`t-copilot-directive-budget.test.ts` です。
ライブのジャーニーは `tests/e2e/t-exec-copilot-status.serial.test.ts` と
`tests/e2e/t-exec-copilot-release-journey.serial.test.ts` で、
`AIDLC_COPILOT_EXEC_LIVE=1` によって有効になります。Full Suite は、その両方を Linux、macOS、Windows で実行します。
