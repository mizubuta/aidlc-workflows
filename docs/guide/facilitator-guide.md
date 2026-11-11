# Facilitator Guide

このページは、AI-DLC のワークショップを運営する人や、チームが最初のワークフローを
進めるのを支援する人のためのものです。当日までに確認すべきこと、実行を小さく保つ方法、
チームが行き詰まったときの対処法を扱います。参加者は代わりに
[オンボーディング](onboarding.md)を読んでください。

---

## Before the workshop: the readiness pass mark

チームに AI-DLC をインストールしてもらい、問題があれば報告してもらうだけでは不十分です。
フックが一度も実行されないマシンは何も報告しません。また、ほとんどのハーネスでは、
インストール直後の `aidlc doctor` は、フックが実行される機会を得る前の段階で合格してしまいます。
マシンの準備が整ったと言えるのは、実際のワークフローのステージを 1 つ実行し、
doctor がフックの発火を示したときだけです。

以下の手順を、各参加者のマシンで、チームが作業するフォルダの中で実行してください。
フォルダの信頼設定とフックの設定はフォルダごとなので、使い捨てのフォルダで確認しても
証明できることは少なくなります。

1. **プロジェクトをセットアップする。** プロジェクトフォルダでターミナルを開き、
   `aidlc config --harness <name>`（例: `copilot`）を実行します。
   `Setup check - N of M sections need you.` という表が表示され、指摘された項目を
   順に案内するよう提案されます。
   - 合格: `Runtime` 行が `[ok]` と `hook PATH ready` を示し、
     `Trust` 行が `[ok]` を示すこと。`Runtime` 行が
     `on this shell's PATH only` を示す場合も、そのターミナルからツールを起動するなら
     合格です。Copilot では、Copilot CLI がフォルダを信頼していない場合 `Trust` 行が `[needs]` を示します。
     また、VS Code 自体のスイッチは確認できないため、それらは手作業で確認してください。
     [GitHub Copilot on Windows](#github-copilot-on-windows) を参照してください。
   - `Models` や `Providers` が `[needs]` を示していても、フックは止まりません。
     何と答えればよいかは、[トラブルシューティング](15-troubleshooting.md#native-install-channel)の
     `Models` 行と `Providers` 行を参照してください。
   - GitHub Copilot、Cursor、Kiro IDE では、`Models` 行が `[ok]` を示し、
     ホストの名前を表示します（例: `every agent uses your GitHub Copilot
     session's model and effort`）。そこで選ぶモデルと effort を、当日までにチームと
     合意しておいてください。Opus を使えないチームは
     medium effort から始めるべきです。
     [モデルと effort の選び方](18-install-and-lifecycle.md#choosing-a-model-and-effort)を参照してください。
2. **インストールを確認する。** `aidlc doctor` を実行します。
   - 合格: サマリー行が `0 problems` を示すこと。
   - すべての警告を読んでください。`Runtime hook PATH` の警告は、ホストが
     `aidlc` を見つけられないままフックを起動する可能性があることを意味します。まずこれを修正してください。GitHub Copilot
     と Kiro IDE では、そのフォルダで最初のチャットを行う前は `AIDLC hooks have not run in this project yet` が
     表示されるのが想定どおりです。フックが実行されるかどうかはステップ 4 でわかります。
3. **ステージを 1 つ実行する。** 同じフォルダでハーネスを開き、
   使い捨てのワークフローを開始します。例:

   ```text
   /aidlc express Add a --version flag to the CLI
   ```

   （Codex CLI では `$aidlc`。）initialization のステージは自動的に実行されます。
   最初の実際のステージが何かを尋ねてきたら（質問または承認）、
   ハーネスの表示形式に合わせて答えてください。ピッカーが表示される場合（Claude Code、Codex CLI）は選択肢を選び、
   選択肢に番号が振られている場合（GitHub Copilot など）はチャットに番号または回答を入力します。
   GitHub Copilot で、エージェントがこのプロジェクトでは AI-DLC のフックが実行されていないと
   言った場合は、ワークショップの前に、エージェントが挙げたスイッチを修正してください。
4. **フックが発火したことを確認する。** ターミナルに戻り、
   `aidlc doctor --verbose` を実行します。
   - 合格: `Hooks last fired:` 行に、フック名が現在の時刻とともに列挙されること
     （例: `plan-approval-guard` と `continue-workflow`）。
   - 不合格: `Hooks have never executed although this workflow has progressed`、
     またはフックが一度も発火していないことを示す `Hook heartbeat data` 行は、
     このマシンでホストが AI-DLC のフックを実行していないことを意味します。ワークショップの前に
     修正してください。
   - 準備未完了: `Hook failures, the latest within the last day` の警告は、
     フックがその時点で報告できなかった失敗に遭遇したことを意味します。その行に
     理由が示されます。
   - 準備未完了: `Your replies are not being recorded: ...` の警告は、
     あなたの回答が AI-DLC に届いていないため、後で承認が拒否されることを意味します。
     その修正行に、このツールでの手順が示されます。
5. **テストを片付ける。** チャットで、テスト用の作業の名前を指定して
   `/aidlc intent archive <name>` を実行します（`/aidlc intent` で名前の一覧が表示されます）。何も
   削除されません。次の `/aidlc` では、どの作業を続けるかを尋ねられるか、作業が
   残っていない場合は新しい作業の説明を求められます。フックが部分的にしか動作しない
   マシンでは、チャットがこれを拒否することがあります。その場合は、自分のターミナルで代わりに
   `aidlc engine intent archive <name>` を実行してください。

ワークショップの 1 週間前に、各チームに参加者のマシンでのステップ 1、2、4 の出力を
求めてください。チームにとっては数分の手間で、フック、PATH、フォルダの信頼設定が
正しく接続されているかどうかがわかります。

`workshop` スコープ（`/aidlc workshop`）は、ファシリテーター付きのグループセッション
そのもののためのものです。23 ステージを実行するものであり、準備状況のチェックではありません。

---

## Before the workshop: bring a vision note and a tech-environment note

各チームに、最初のセッションに 2 つの短いメモを持ってくるよう依頼してください。これにより、
最初の 1 時間が質問に答える時間から、回答を確認する時間に変わります。

- **ビジョンメモ**: 何を誰のために作るのか、最初のリリースに含める機能、
  含めないもの、既にわかっている未解決の問いを 1 段落で書いたもの。
  当日は最初の依頼に含めます:
  `/aidlc workshop Read ./vision.md and build what it describes`
  （Codex CLI では `$aidlc workshop ...`）。[ビジョンドキュメントの書き方](writing-inputs/vision-document-guide.md)を参照してください。
- **技術環境メモ**: 言語とバージョン、フレームワーク、テストツール、
  クラウドとデプロイモデル（または「ローカルのみ」）、使用禁止のライブラリとその
  理由および代替、セキュリティの基本、そして典型的なエンドポイント、関数、テストの
  短い例を 1 つずつ。最初の実行の前に、
  `aidlc/spaces/default/knowledge/aidlc-shared/technical-environment.md` として保存してください。
  すべてのステージがここを読み込みます。
  [技術環境ドキュメントの書き方](writing-inputs/technical-environment-guide.md)を参照してください。

各ガイドは短縮版から始まっており、ワークショップにはそれで十分です。既存のコードベースで
作業するチームは、この作業で変更してはならないものをビジョンメモに
追記してください。

---

## Keep side tasks small: name the scope

単なる `/aidlc <description>` では、エージェントがその文言からプランを提案するか、
プランの作成を申し出ます。また、既定の `classic` はセットアップ後に 30 ステージ中 15 ステージを実行します。
平易な言葉で書かれたバグ報告には通常 `bugfix` が提案されます。エージェントが代わりにプランの作成を申し出た場合も、
その申し出の横にあるリストでは `bugfix` が最初に挙げられますが、既定を選ぶと
はるかに大きなワークフローが実行されます。最初の単語としてスコープを指定してください:

| サイドタスクの内容 | 入力 | ステージ数 |
|---|---|---|
| 原因も修正方法もわかっているバグ | `/aidlc bugfix <what is broken>` | 6 / 30 |
| 小さな変更、またはエンドツーエンドの簡単な試行 | `/aidlc express <the change>` | 7 / 30 |

`feature` はすべてのステージ（30 / 30）を実行します。本番向けの機能には適していますが、
サイドタスクには向きません。すべてのスコープについては[ワークフロープロファイル](workflow-profiles.md)を参照してください。

サイドタスクは専用のプロジェクトフォルダで開始し、チームのメインのワークフローと
混ざらないようにしてください。

キックオフで参加者に次のことを伝えてください: 計画したユースケース以外のサイドタスクには、
既定ではなく `bugfix` か `express` を使うこと。

---

## Fewer questions: depth

深さ（depth）は、各ステージが尋ねる質問の数と、ドキュメントの長さを決めます。
`Standard` の深さでは、ステージは約 5〜8 問を目安とし、
`Minimal` では約 2〜4 問を目安とします。これらは目標値であって上限ではありません。曖昧な
回答や矛盾があれば、引き続き追加の質問が行われます。

- ワークフローの途中では `/aidlc --depth minimal` と入力します。実行中の
  ワークフローの深さが変わります。
- 開始時には、スコープの後に追加します（例:
  `/aidlc classic --depth minimal <the work>`）。

`bugfix`、`express`、`poc`、`refactor`、`security-patch` は既定で
`Minimal` です。`classic`、`feature`、`workshop` は既定で `Standard` です。
[スコープと深さ](05-scopes-and-depth.md#the-3-depth-levels)を参照してください。

---

## Fewer stops: learnings and summary confirmation

`workshop` では、すべてのステージで 2 つの確認が行われます。`Looks correct` のサマリー
確認と、グループがエージェントを訂正した後の "Keep that for next time?"（次回のために残しますか？）という学びの質問です。
会場では、これはグループにとって必要以上に多くの停止になりがちです。作業の開始時に
両方をオフにしてください:

```
/aidlc workshop --learnings off --summary-confirmation off <the work>
```

質問と承認ゲートは引き続き実行されます。学びをオフにすると、実行から何もプロジェクトや
チームのメモリに保存されません。サマリー確認をオフにすると、読み違えた回答は成果物が書かれる前ではなく、
承認ゲートで発見されることになります。学びのループを教えることがセッションの一部である場合は、
学びをオンのままにしてください。
[停止を減らす](05-scopes-and-depth.md#fewer-stops-for-a-workshop-or-small-work)を参照してください。

---

## Construction: one Unit at a time

Inception で作業が複数の Unit に分割された場合、Construction はそれらを
2 つの方法で実行できます:

- `unit-major`: 1 つの Unit が設計ステージと Code Generation を終えてから、
  次の Unit が始まります。チームは動作するコードをより早く目にできます。
- `stage-major`: すべての Unit が 1 つのステージを終えてから、いずれかの Unit が
  次のステージに進みます。そのため、Unit が 6 つあるチームは各ステージで 6 セットの質問に答えることになります。
  スコープがウォーキングスケルトンを構築する場合、その最初の Unit は残りより先に
  コードまで進みます。

作業を Unit に分割してコードを書くスコープを持つ、新しい単独のワークフローは、作成時に
`Construction Iteration: unit-major` を記録します。Construction が
始まる前に、その作業の `aidlc-state.md`（
`aidlc/spaces/<space>/intents/` 配下）を開き、その行を確認してください。
`stage-major` となっている場合、またはその行がない場合（これも stage-major を意味します）は、
Inception の間に切り替えてください。エージェントに依頼するか、プロジェクトフォルダで次を実行します:

```bash
aidlc engine state set-construction-iteration unit-major
```

これが `Select Construction Execution: serial` で拒否された場合、その作業は
Unit を並列に構築するよう設定されています。まず
`aidlc engine state set-construction-execution serial` を実行してから切り替えてください。

Construction が始まった後は、チャットで伝えてください（「ここからは 1 つずつ
Unit を構築して」）。エージェントは直ちに切り替えて元に戻す方法を伝え、
既に完了した Unit はその作業を維持します。
[Construction の順序と実行](12-cli-commands.md#construction-order-and-execution)を参照してください。

---

## Recovery playbook

チームが行き詰まっているのは、リトライ後も同じ拒否が返ってくる場合、エージェントが
同じコマンドを繰り返して同じエラーを受け取る場合、またはエージェントが実行する
`aidlc` コマンドがすべて拒否される場合です。待ってもそのようなループは解消されず、エージェントが
考え抜いて抜け出すこともできません。

### 1. Stop the loop

エージェントを停止します（チャットの停止ボタン、またはターミナル型ハーネスでは Esc）。
リトライのたびに時間とクレジットを消費します。

### 2. Capture a diagnostic report from your own terminal

（エージェント経由ではなく）自分でターミナルを開き、プロジェクトフォルダに移動して、
次を実行します:

```bash
aidlc doctor --export
```

AI-DLC のチェックは、エージェントが行うことに対して作用します。自分のターミナルで入力したコマンドは
エージェントの操作ではないため、エージェントが試すコマンドがすべて拒否される場合でも
これは動作します。doctor は `aidlc/diagnostics/` 配下に秘匿化したレポートを書き出し、
そのパスを表示します。レポートにはソースコードもドキュメント本文も含まれません。問題を報告するときに
添付してください。
[診断レポートの共有](15-troubleshooting.md#sharing-a-diagnostic-report)を参照してください。

### 3. Read the fix lines

`aidlc doctor` は、すべての問題と警告の下に `fix:` 行を出力します。何よりも先に
それに従ってください。

### 4. Match the symptom

| チームに見えている状況 | 対処 |
|---|---|
| ワークフローが待機したまま先に進まない | 質問または承認ゲートが表示されている場合は、あなたを待っています。表示されている内容、列挙された選択肢のいずれか、またはゲートでは **Approve** / **Request Changes** で答えてください。どちらも表示されておらず、エージェントがバックグラウンドで作業がまだ実行中だと言っている場合は、完了を待ちます。`/aidlc --status` で現在位置がわかります。ステージから抜けるには、`/aidlc --stage <slug>` でジャンプします。 |
| 回答や承認が記録されない、または doctor がフックが一度も実行されていないと言う | ホストが AI-DLC のフックを実行していません。doctor が挙げる項目（PATH、フォルダの信頼設定、フックの承認、選択したエージェント）を修正し、ハーネスを再起動して、準備状況の合格基準をもう一度実行してください。 |
| Code Generation 中にエージェントの `aidlc` コマンドが拒否され、それらが `cd <path>;` で始まっている | 各 `aidlc` コマンドを、前に何も付けずに単独で実行するようエージェントに依頼してください。チェックは、コマンドの前の `cd` が既存のフォルダをフルパスで指定している場合はそれを読み取り、そのフォルダで実行されたものとしてコマンドを判定します（Windows では Git Bash の `cd /c/path` もフルパスとみなされます）。相対パス、変数、`cd C:\path` のような引用符なしの Windows パス（`;` で連結された PowerShell の行でのみ読み取られる）など、読み取れない `cd` は拒否されます。`aidlc --status`、`aidlc doctor`、`aidlc engine orchestrate next` は単独であれば許可されます。古いビルドでは現在のフォルダから出るすべての `cd` が拒否され、この状態では単なる `aidlc --status` や `aidlc doctor` も拒否するものがあります。その場合は次の行に進んでください。 |
| プランは承認済みなのに、Code Generation の作業が `Plan Approval authority is ambiguous or stale`（Guard Policy が `off` の場合: `cannot select one approval target`）で拒否される | まず、拒否メッセージが示すコマンドを、書かれたとおりに単独でエージェントに実行させてください。拒否メッセージは、このインストールでの実行方法どおりに表記されています。ネイティブインストールでは `aidlc engine orchestrate next`、コピーされたインストールでは `bun <harness-dir>/tools/aidlc-orchestrate.ts next` です。拒否メッセージがプランは既に承認済みだと言っている場合、誰も再承認する必要はありません。同じ拒否が返ってくる場合は、下記の最終手段を使ってください。`/aidlc --plan-approval off` や Guard Policy `off` ではこの拒否は解除されませんが、3 つの例外があります。Guard Policy `off` では、チャットのコンパクションで保留されたビルドステップは、プランが承認済みであれば自動的に続行されるため、この拒否は現れません。Guard Policy `relaxed` または `off` で Code Generation のゲートが開いているときも同様で、承認済みのプランから作業が続行されます。Unit を並列に構築している間は、フェンスを下げると保留の旨を示す行とともに書き込みが通過し、フェンスがオンの場合は Unit のコードは `.aidlc/worktrees/` 配下の準備済みワークツリーに置くべきもので、そこでは通過します。 |
| 会話が長くなった、または混乱している | 新しいチャットを開き、`/aidlc --resume` と入力します。ワークフローは古い会話からではなく、ディスク上のファイルから続行されます。 |

### Last resort: switch the Plan Approval check off

`AIDLC_DISABLE_PLAN_APPROVAL_GUARD` は、Code Generation 中に作業を拒否するチェックを
オフにします。人がセッションを見守っている間だけ、そして誤った拒否を
回避するためだけに使ってください。

最も手早い方法は再起動を必要としません。チャットで伝えてください（「このプロジェクトのプラン承認
チェックをオフにして」）。エージェントが下記のコマンドを実行し、あなたが依頼したばかりなので
チェックはそれを通過させます。または、プロジェクトフォルダのターミナルで自分で
実行します:

```bash
aidlc config flags --bypass AIDLC_DISABLE_PLAN_APPROVAL_GUARD --local --yes
```

これはワークフローの実行中でも機能し、エージェントの次の操作から、このマシン上のこのプロジェクトの
すべてのワークフローでチェックがオフになります。その後 AI-DLC はチャットで、そのチェックの目的、
オフになっていることとその時刻を伝え、オンに戻すことを提案します。オフの間は、すべてのチャットの開始時に
これを繰り返します（セッション開始時のコンテキストを表示しない opencode を除く）。チームが
問題を乗り越えたらすぐに、その提案に自分の言葉で答えるか、`--bypass` の代わりに
`--clear-bypass` を付けて同じコマンドを実行してください。

または、ハーネスの起動時の環境に `AIDLC_DISABLE_PLAN_APPROVAL_GUARD=1` を設定します。
これは、その環境で起動したハーネスのすべてのワークフローに適用されます。フックはその環境から値を
読み取るため、ハーネスを起動する前に設定してください:

- まずハーネスのウィンドウをすべて閉じます。
- Windows PowerShell と VS Code の場合:

  ```powershell
  $env:AIDLC_DISABLE_PLAN_APPROVAL_GUARD = "1"
  code C:\path\to\project
  ```

- macOS または Linux の場合: `AIDLC_DISABLE_PLAN_APPROVAL_GUARD=1 code /path/to/project`
  （または `code` の代わりにハーネス自身のコマンド）。

チームが問題を乗り越えたらすぐに、変数なしでハーネスを再起動してください。

いずれの方法でもチェックがオフの間は、ワークフローの監査証跡にオフであったことが
記録されます（`GUARD_DISABLED`）。

### When to pivot

AI-DLC のデバッグは、チームが参加した目的ではありません。このプレイブックを一通り試した後も
チームが同じ失敗に 2 回遭遇した場合は、中断してください。診断レポートを保管し、
チームを代替プラン（例えば、そのチームのマシンで準備状況の合格基準を満たしたハーネス）に
移行させます。切り替えが数分で済むよう、代替プランはワークショップの前に合意しておいてください。

---

## How strongly each harness enforces the workflow

AI-DLC はどこでも同じエンジンを実行しますが、ホストごとにフックに与えられる
権限が異なります。弱い行ほど、ワークフローはエージェントが指示に従うことに
より依存し、失敗を見逃しやすくなります。

| ハーネス | 質問の表示方法 | AI-DLC はエージェントの操作を拒否できるか | ターンの終了時 | フックが実行される条件 | 知っておくべき制約 |
|---|---|---|---|---|---|
| Claude Code | ネイティブのピッカー。ピッカーでの回答はあなたのターンとしてカウントされる | はい、すべてのチェック | ステップが報告されるまでワークフローを継続させる | セットアップ後にそのフォルダで Claude Code を起動しており、`"disableAllHooks": true` の設定や組織のポリシーで停止されていないこと（doctor が確認する） | 特になし |
| Codex CLI | ピッカー。フォールバックとして番号付きの文章。ピッカーでの回答はカウントされる | はい、すべてのチェック | ワークフローを継続させる | プロジェクトのフックが信頼されていること（対話的な信頼確認を 1 回行うか、同梱の信頼シードを使う） | AI-DLC のステータスラインはプロジェクト単位の選択で、プロジェクト内の全員に適用される（`$aidlc --statusline on`） |
| GitHub Copilot（CLI と VS Code） | 番号付きの文章。回答を入力する。ピッカーでの回答はカウントされないため、ワークフローの実行中はピッカーは拒否される | はい、Copilot の拒否チャネルを通じて。CLI では実環境で検証済み。VS Code ではドキュメント上は可能だが、実環境ではまだ検証されていない | ワークフローを継続させる | VS Code では、ワークスペースが信頼されており、Chat: Use Hooks がオンであること（組織がオフにできる）。Copilot CLI では、フォルダが `trustedFolders` に含まれていること。`aidlc` がホスト起動時の PATH 上にあること | ステータスラインなし。doctor はそのフォルダで Copilot のチャットが開始されるまで警告し、あなたのメッセージに対してフックが実行されなかった場合はチャットがそう伝える。それでも、実際のステージとあなたの返信が完全な準備状況チェックとなる |
| Cursor | 番号付きの文章 | はい | ターンを保持できない。リマインダーはフォローアップメッセージとして届く | プロジェクトが git リポジトリ内にあり、フォルダが信頼されていること（doctor がリポジトリを確認する） | ヘッドレスの `agent -p` の実行では承認ゲートを通過できない。ステータスラインなし |
| Kiro CLI | 番号付きの文章 | はい | 対話型セッションではワークフローを継続させるが、`--no-interactive` の実行では継続させない | `aidlc` エージェントがアクティブであること | ステータスラインなし。セッション終了時やコンパクション前のフックなし |
| Kiro IDE | 番号付きの文章 | 部分的: Plan Approval、review-freeze、状態遷移チェック、ターミナルコマンドチェック。レビュアーの読み取り範囲チェックはフックではない。作業でそのチェックがオンの間、Kiro は AI-DLC が書き出すレビュアーのコピーを通じて、各 Unit のレビュアーをその Unit に限定する | ターンを保持できない。エージェント自身の指示によってワークフローを継続させる | フォルダが信頼されており、ウィンドウが再読み込みされ、`aidlc` エージェントが選択されていること（doctor はフックが実行されるまで警告する） | ステータスラインなし |
| opencode | 番号付きの文章 | はい | ターンを保持できない。リマインダーは新しいプロンプトとして送信される | AI-DLC プラグインがインストールされていること（doctor が確認する） | セッション終了イベントなし。ステータスラインなし |

ステータスラインがない場合は、`/aidlc --status`（Codex CLI では `$aidlc --status`）
と各ゲートの進捗行を使ってください。詳細は
[その他のハーネスでの実行](harnesses/README.md)にある各ハーネスの章を参照してください。

---

## GitHub Copilot on Windows

多くのワークショップでは、Windows 上の VS Code で Copilot を使います。当日までに
次の点を確認してください:

- **通常の PowerShell ウィンドウからインストールする。** "Run as
  administrator"（管理者として実行）で開いたウィンドウは使いません。その後 VS Code を閉じて開き直し、ターミナルとフックが
  新しい PATH を認識するようにします。[インストール](18-install-and-lifecycle.md#install)と
  [Windows PowerShell](18-install-and-lifecycle.md#windows-powershell) を参照してください。
- **VS Code のバージョン。** `code --version` を実行します。AI-DLC には 1.130 以降が必要です。
  doctor が確認するのは任意の Copilot CLI のバージョンだけで、VS Code のバージョンは確認しません。
- **フォルダの信頼設定。** Copilot CLI は、`config.json` の
  `trustedFolders` リストがカバーするフォルダ（そのフォルダまたはその上位フォルダ）でのみリポジトリのフックを実行します。
  プロジェクトフォルダで一度 `copilot` を実行して "Yes, and remember this
  folder for future sessions"（はい、今後のセッションのためにこのフォルダを記憶する）を選ぶか、フォルダのフルパスを自分で追加してください。
  doctor は CLI と同じ場所のファイル（Windows では `%USERPROFILE%\.copilot`、
  または `COPILOT_HOME`）を読み、フォルダがカバーされていない場合は警告します。
  VS Code はそのリストを決して読みません。VS Code のフックは、信頼されたワークスペースで、
  Chat: Use Hooks 設定がオンの場合にのみ実行されます（組織がオフにできます）。
  doctor はどちらも確認できないため、両方を VS Code で確認してください。
- **`aidlc` コマンドを単独で実行する。** セッションの開始時に、エージェントに次のように
  伝えてください: 「各 aidlc コマンドを、前に cd を付けずに単独で実行して」。
  Code Generation 中は、`cd C:\path && aidlc --status` のようなコマンドは拒否されますが、
  `aidlc --status` 単独であれば許可されます。
- **入力して答える。** チャットに番号または回答を入力してください。ワークフローの実行中は、
  ピッカーでの回答は拒否されます。
- **各 Windows マシンで[準備状況の合格基準](#before-the-workshop-the-readiness-pass-mark)
  を実行する。** VS Code がそのマシンでフックを実行することを証明できる唯一のチェックです。

Copilot のその他のセットアップについては、
[GitHub Copilot での AI-DLC](harnesses/copilot.md) を参照してください。

---

## Related reading

- [オンボーディング](onboarding.md) - 参加者の最初の 1 週間
- [複数チームでの Construction とワークショップモード](workshop-mode.md) - 複数のチームが 1 つのインテントの Unit を構築する場合
- [ワークフロープロファイル](workflow-profiles.md) - すべてのスコープとその使いどころ
- [ビジョンドキュメントの書き方](writing-inputs/vision-document-guide.md)と[技術環境ドキュメントの書き方](writing-inputs/technical-environment-guide.md) - 2 つの事前準備メモ
- [トラブルシューティング](15-troubleshooting.md) - 症状と対処法
