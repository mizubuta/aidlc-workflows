---
name: aidlc-issue-report
description: >
  When something went wrong while running AI-DLC, gather what the run left
  behind on this machine (AI-DLC's audit record and state, hook health,
  AI-DLC doctor, the chat logs of Kiro, Claude Code, Codex or Copilot for this
  project) into one folder, with personal, company and project details
  replaced by labels, for the person to send to the AI-DLC team, who analyse
  it. It does not diagnose anything itself. Use it when the person says
  "이슈 보고서 만들어줘", "AI-DLC 가 이상했어 정리해줘", "aidlc 문제 리포트",
  "에러 난 거 모아줘", "report an AI-DLC issue", "AI-DLC bug report", "what went
  wrong with aidlc", even long after the run and even when they cannot say
  what the problem was.
---

# AI-DLC issue report

利用者が AI-DLC を実行し、何か問題が起きた。利用者の役割は、フォルダを AI-DLC チームに送った時点で終わる。チームは
自分たちのマシンでそれを分析する。あなたの仕事はそのフォルダを集めることである — **実行の記録をできる限り多く含め、
個人、会社、プロジェクトを特定しうるものは一切含めない。** 何が問題だったかを診断、順位付け、説明することはしない。
スクリプトが収集とフィルタリングを行い、チームが読む。ワークフローを修正することは決してせず、`/aidlc` や、下記の doctor の 1 回の実行を除く
いかなる AI-DLC コマンドも決して実行せず、何もどこにも送信しない。

出力はフォルダ内のファイルだけであり、それ以外は何もない。GitHub の issue、プルリクエスト、コメントを作成せず、
`gh`、`git push`、いかなるアップロードも実行せず、Slack やメールへの投稿もしない — 利用者に「とにかく登録しておいて」と
頼まれた場合であっても同様である。フォルダを送るかどうかは利用者の判断であり、このスキルの外で行われる。

利用者には短く平易な文で話す。利用者が AI-DLC のコマンドを知っていることは想定しない。コマンドの実行を頼んではならない。

**言語。** このスキルは日本語で書かれているが、あなたは利用者の言語で話す。言語は、この会話で利用者が
あなたに書いた内容（スキルを開始した依頼、その後の各返信）から判定する。利用者が言語を切り替えたら、あなたも切り替える。
利用者の書いた内容から言語が判断できない場合は英語を使う。このスキルがあなたに言わせるセリフはすべて、ここでは
英語で書かれている（必要に応じて日本語訳を添えている）。それを利用者の言語で、逐語訳ではなく意味を保って伝える。フォルダ内のファイルは英語のままにする。

## Why the care about privacy

このフォルダは社外に出る。AI-DLC の記録には、利用者自身の言葉（依頼、回答、フィードバック）、
利用者の名前を含むファイルパス、プロジェクトの内容が詰まっている。そのため、スクリプトは 2 つの層を書き出す:

- `safe.json` — 送信されるデータ。各値は、そのフィールドが AI-DLC 自身の語だけを保持する場合（イベント、ステージ、
  エージェント、ステータス）の AI-DLC 自身の語、数値、時刻、バージョンのいずれかである。利用者が選んだ名前（Unit、インテント、
  ファイル）や、人またはエージェントが書いたものはすべて `<custom-text-12>` のようなラベルになり、同じ値には同じラベルが付く。
  各部分が何を保持するかは `references/data.md` に記載されている。
- `local/` — ラベルが何を表すかと、チェックに必要なもの。決して送信されず、最後に削除される。

何かを引き渡す前に、チェックがフォルダ内のすべてのファイルを書かれたとおりに読み、そのフィールドのルールが書く形式ではない値と、
個人、会社、マシンを特定しうるものを含むカバーの行をすべて拒否する。何が安全かを推測することはせず、拒否する。

## Steps

スクリプトはこのスキルの `scripts/` フォルダにある（以下の `<skill>` はこのスキルのフォルダ）。以下の各コマンドは
`<run>` で始まり、これは任意のフォルダからスクリプトの 1 つを実行する:

- `bun` が見つかる場合はそれを使う。
- 見つからない場合（ネイティブの AI-DLC インストールには Bun が付属しない）は、AI-DLC 自身の `aidlc` コマンドを、そのコマンドに限り
  `BUN_BE_BUN` を設定して使う:
  - macOS、Linux、WSL: `BUN_BE_BUN=1 aidlc <script> …`
  - Windows PowerShell: `& { $env:BUN_BE_BUN = '1'; try { aidlc <script> … } finally { Remove-Item Env:BUN_BE_BUN } }`
  - Windows cmd: `cmd /c "set BUN_BE_BUN=1&& aidlc <script> …"`

シェルに `BUN_BE_BUN` を設定したままにしてはならない。設定されたままだと、`aidlc` の実行はすべて AI-DLC ではなく Bun を起動する。`aidlc` も
見つからない場合（AI-DLC のインストール前に開かれたエディターなど）は、代わりに
`~/.local/share/aidlc/active-executable`（Windows: `%LOCALAPPDATA%\aidlc\active-executable`）に記載された実行ファイルを使う。

### 1. When

一度だけ尋ねる: "Roughly when did it happen? It's fine if you don't know."（だいたいいつ起きましたか？ わからなくても大丈夫です。）利用者が既に述べていれば、再度尋ねない。
利用者が示した時刻（「昨日の午後」「今朝」）を、既に知っている日付から計算して **`Z` 付きの UTC** の `--since`/`--until`
（`2026-10-17T08:00:00Z`）に変換する — `date` を実行してはならない。Kiro は引数が `host:port` のように見えるコマンドを拒否し、
`+09:00` のようなオフセットもそれに該当する（Kiro CLI 2.32.0 で確認）。時刻の指定がない場合、
対象期間は最後の AI-DLC の活動より前の 7 日間となる。利用者が述べた内容はフォルダに入れない。

### 2. Collect

まず 1 行で伝える: "I'll run AI-DLC's diagnostics once. It adds one line to the project's record and
refreshes AI-DLC's memory copy. It does not change your code."（AI-DLC の診断を 1 回実行します。プロジェクトの記録に 1 行追加され、AI-DLC のメモリのコピーが更新されます。コードは変更しません。）その後、次を実行する（プロジェクトフォルダ、つまり
`aidlc/` を保持するフォルダを必ず渡す。不明な場合はツールで開いているフォルダ）:

```bash
<run> <skill>/scripts/collect.ts --project <project folder> [--since <ISO>] [--until <ISO>]
```

`--out` は渡さない。フォルダはスクリプトが決める。スクリプトは短い JSON を出力する。そこには `out`（フォルダ。
`<project>/aidlc/diagnostics/issue-reports/<time>/` で、git が無視する AI-DLC 自身の diagnostics フォルダ内にある）、収集したもの、
そして `safeJsonHits`（チェックが拒否する `safe.json` のフィールド。通常はなし）が含まれる。`safe.json` 内の指摘された各値を
`<redacted>` に置き換える — ラベルは常に通過する — か、理由に削除せよとある場合はそのフィールドを削除する（ある部分が持たないキーについては、そのキーを削除する）。`safe.json` の `window.notes` に時刻が除外されたとある場合は、
既定の対象期間が使われたということなので、利用者に 1 行で伝える。`found.aidlcWords` が 0 の場合、このインストールから AI-DLC 自身の語を読み取れなかったため、
フォルダ内の AI-DLC の名前はすべてラベルになっている。利用者に 1 行で伝える。`no-aidlc-project` と出力された場合は、どのフォルダで AI-DLC を実行したかを尋ねる。

### 3. Missing tool versions

`safe.json` の `environment.versionsMissing` には、この実行で使われたツールのうち、ツール自身のチャットログにも
現在のこのマシン上にもバージョンが見つからなかったものが列挙される。それぞれについて、画面から読み取ってもらうよう利用者に一度だけ頼む — コマンドの実行を
頼んではならない。どこにあるかを伝える: Kiro IDE は Kiro メニュー → About Kiro（Windows と Linux: Help →
About）、CLI ツールは起動時に表示されるバナー（Claude Code は `/status` でも確認できる）。常に「わからない」という選択肢を示す。
`environment.toolsUsed` が空の場合、どのチャットログからも実行に使われたツールがわからなかった（このレポートだけを保持していたか、読み取れなかった）ため、使った AI ツールとそのバージョンを併せて尋ねる。
各回答を記録する: 利用者が示したとおりのバージョンを、コマンドではなくファイル書き込みツールで `<out>/local/version-<key>.json`（`<key>` はコマンドのもの）
に書き込み（`{ "version": "1.4.13" }`）、次を実行する

```bash
<run> <skill>/scripts/set-version.ts --out <out> --tool <key>
```

バージョンの形をした値だけが受け付けられる。値が拒否された場合（`not-a-version`）や、利用者がわからない場合は次に
進む — 二度尋ねない。ファイル自体が拒否された場合は、書き直して再実行する。
利用者がツール名を挙げたがバージョンを挙げなかった場合は、ファイルを書かずに実行する。ツールは記録され、バージョンは不明となる。

利用者が除外したい会社、顧客、製品、人物の名前を挙げた場合は、それを隠す。名前はコマンドではなく、ファイル書き込みツールで
`<out>/local/hide-names.json` に書き込む。シェルは名前に含まれる `&` やアポストロフィを独自の構文として解釈するためである
（`[{ "kind": "customer", "name": "<name>" }, …]`、kind は `customer`、`org`、`product`、
`person`、`other` のいずれか）。その後、次を実行する

```bash
<run> <skill>/scripts/hide.ts --out <out>
```

拒否された場合は、指摘された箇所をファイル内で修正し（`entries` はファイル内の位置を示す）、再実行する。1 文字の名前は
隠せないため、ファイルから取り除いて利用者に伝える。
`notHidden` が列挙された場合、それらの名前（ファイル内の位置で示される）は隠されていない: `aidlc-word` は、その名前が
AI-DLC 自身の語の一部（`BOLT_FAILED` の中の `Bolt`）または AI-DLC のメッセージの固定テキストの一部であること、
`ordinary-word` は日常的な単語（`Will`）であることを示す。それぞれ 1 行で利用者に伝え、それでも隠すかどうかを一度だけ尋ねる。
隠す場合、その名前を含む AI-DLC の語やメッセージ、またはその単語を含む値はそれぞれラベルになり、
チームが読み取れる実行の情報は減る。利用者が「はい」と答えたら、それらの名前だけを `<out>/local/hide-everywhere.json` に書き込み、次を実行する

```bash
<run> <skill>/scripts/hide.ts --out <out> --everywhere
```

続いて、このレポート（`from` に出力されたフォルダ）から新しいレポートを収集する:

```bash
<run> <skill>/scripts/collect.ts --from <from>
```

これにより、同じ対象期間、隠し済みの名前、利用者が示したバージョン、doctor の結果を持つ新しいフォルダが作られる
（doctor は二度実行されない）。それらの名前を含む語は、もはや AI-DLC の語としては読まれない。結果はステップ 2 と同様に読み、
以降はその `out` を使う。`project-not-found` と出力された場合はプロジェクトが移動しているので、`--project <its folder now>` を追加する。
その後、送信しない以前のフォルダを削除する:

```bash
<run> <skill>/scripts/finish.ts --discard --out <the earlier out>
```

`emptyFolderLeft`: 空のフォルダだけが削除できなかった（ウィンドウやシェルが開いている）。利用者に、自分で削除してよいと伝える。
`folder-in-use`: フォルダを開いているもの（ウィンドウ、その中のシェル）を閉じて再実行する。ヒントにフォルダにはもうレポートがないとある場合は、
利用者に、自分で削除してよいと伝える。

レポート自身のフィールド名は、どのレポートでも同じであり、そのまま残る。利用者が「いいえ」と答えた場合は名前をそのままにし、二度と
尋ねない。

### 4. Cover, check, hand over

```bash
<run> <skill>/scripts/cover.ts --out <out>
<run> <skill>/scripts/check.ts --out <out>
```

`cover.ts` は `safe.json` から `aidlc-issue-report.md` を書き出す: 対象期間、バージョン、フォルダの内容、ラベルに
置き換えられたもの。これを編集してはならない。`safe.json` を変更したら再実行する。`check.ts` は、指摘ごとに
ファイル、行または `safe.json` のフィールド、そして理由を出力する — 値は決して出力しない。`safe.json` では、指摘された値を
`<redacted>` に置き換える。`refused`: 送信されるのはカバーと `safe.json` だけなので、それ以外のファイルはフォルダの外に移動する。
`pending`: あなたが書いた回答がまだ適用されていない（スクリプトの前で実行が止まった）ので、指示されたコマンドを実行するか、
利用者が回答を取り消した場合はそのファイルを削除する。
`ok: true` になるまで `cover.ts` と `check.ts` を再実行する。チェックが通るまで、何も引き渡してはならない。

**次に、利用者がカバーを読む。** "Please open `<out>/aidlc-issue-report.md` and read it once. It says what is in
the folder. Is there anything you would not send?"（このファイルを開いて一度読んでください。フォルダの内容が書かれています。送りたくないものはありますか？）利用者が挙げたものを隠し（ステップ 3）、`cover.ts` と `check.ts` を再実行し、
送ってよいと言われたときにだけ先に進む。二度尋ねない。

その後:

```bash
<run> <skill>/scripts/finish.ts --out <out>
```

これは同じチェックをもう一度実行し、`local/` を削除してフォルダを開く。collect が出力したのと同じ `<out>` を渡す。collect が
作ったものではないフォルダや、チェックで何かが見つかるフォルダは拒否され（`ok: false`）、何も削除されない。指摘された箇所を修正して
再実行し、決して `local/` を手で削除しない。利用者に簡潔に伝える: フォルダの内容（カバーから 1 行で）、
名前、パス、利用者が入力したものはすべてラベルに置き換えられたこと（ステップ 3 で名前を残すことを選んだ場合はそれが残されたこと）、
そしてフォルダのパスとともに "Please send the files in this folder to your AI-DLC contact"（このフォルダ内のファイルを AI-DLC の担当者に送ってください）。
