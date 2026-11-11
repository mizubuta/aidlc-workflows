# Question Rendering — cursor harness annex

このファイルは、`{{HARNESS_DIR}}/aidlc-common/protocols/stage-protocol.md` のセクション "Structured questions" が求める構造化質問を、
「この」ハーネスがどのように表示するかを定める。
プロトコルとステージのファイルはハーネス中立であり、*構造化質問を提示せよ* と述べ、
フェンス付きの ` ```question ` 仕様ブロックを持つ。この付属文書は、
その契約を具体的な仕組みに結び付ける唯一の場所である。

## Never echo the spec (non-negotiable)

` ```question ` のフェンス付きブロックは、**この付属文書のレンダリングへの入力であり、決して
貼り付けるための出力ではない**。オーケストレーターは、すべての ` ```question ` 仕様を、
以下の「Mechanism」で定義する番号付きの散文として必ずレンダリングしなければならない。フェンス付きブロックやそのフィールド行（`prompt:`、
`header:`、`multiSelect:`、`options:`、`label:`、`description:`）を、チャットの
トランスクリプトにエコー、出力、貼り付け、または「引用して返す」ことは決してしてはならない。ユーザーには生のフェンスを決して見せず、
番号付きの散文によるレンダリングだけを見せる。

フェンスをそのままのテキストとしてエコーすることは、スタイルの選択ではなく**プロトコル違反**である。
エコーすると次の問題が生じる:

- ユーザーが選べる番号付きの選択肢ではなく、回答できないブロックが生まれる。
- 番号付きレンダリングが末尾に追加する "Other" の逃げ道が失われる。
- 同じセッション内の他のすべての正しいレンダリングと一貫しなくなる。

返信の中に 3 連バッククォートの `question` ブロックを書こうとしていることに気付いたら、
**止まる**こと。その内容は番号付きの散文としてレンダリングすべき仕様であり、メッセージ
本文ではない。

これは、以下を含む（ただしこれらに限らない）**すべての**構造化質問の箇所に適用される:

- 承認ゲート（すべてのステージ完了時）
- 質問へのインタラクションモードの選択（Guide me / I'll edit the file / Chat）。
  ステップがそれを求めている場合に限る（それ以外の場合は、ステップの通知を伝え、ステップの指示どおりに
  質問を扱う。stage-protocol.md §3 Step 2）
- ラダーのプロンプト（ウォーキングスケルトン後の自律モード）
- Bolt 失敗時の停止と確認（Retry / Skip / Abort）
- 成果物生成前の統合サマリーの確認
- 修正の後に行うセクション 13 の学びに関する質問（このプロジェクト用に残すか、チーム用に残すか、残さないか）

（` ```question ` のフェンスがそのまま残っていてよいのは、**この**ファイルや stage-protocol のような
フレームワークのドキュメントの中である。それらはチャットの出力ではなく、記述用の仕様だからである。
stage-protocol では、それらの仕様は規範的なプロンプトのテンプレートである: 周囲の指示が
質問を求めるとき、その内容は**必ず**この付属文書を通じて
表示しなければならない。この付属文書の対応付けの例は説明用である。
禁止されているのは、実際のオーケストレーションのターンで生のフェンスをエコーすることである。）

## Mechanism

Cursor には構造化質問のツールがないので、すべての構造化質問は
**チャット内の番号付きの散文の選択肢**として表示され、ユーザーは番号（または
自由記述）で答える。仕様は次のように表示する:

```question
prompt: "[Stage Name] complete. How would you like to proceed?"
header: Approval
multiSelect: false
options:
  - label: Approve
    description: Continue to [next stage]
  - label: Request Changes
    description: Provide revision feedback
```

は次のようになる:

```
**Approval** - [Stage Name] complete. How would you like to proceed?

1. **Approve** - Continue to [next stage]
2. **Request Changes** - Provide revision feedback
3. **Other** - describe what you want instead

Reply with a number (or just tell me).
```

## Mandatory consolidated-summary checkpoint

このチェックポイントは、`directive.ceremony.summary_confirmation === "on"` の場合にだけ適用される。`"off"` の場合は、回答から直接生成する: サマリー確認の問いかけ、確認のエントリ、レシートはいずれもない。必須のステージの質問やその他の人間の判断（Plan Approval やステージの承認ゲートを含む）は変わらない。

ガイド付きまたはチャットのファイルベースの Q&A の後（およびステージの定義が明示的に
それを求める場合、例えば Requirements Analysis では常に）、ステージプロトコルは、ステージの成果物を生成する前に
別個の確認を求める。質問ファイルの
`## Consolidated Summary Confirmation` に、サマリー、
問いかけ、A/B のファイル用の文字の接頭辞を付けない両方の選択肢、そして空の
`[Answer]:` タグを追加または更新し、その後、この番号付きの質問をチャットに表示する。
利用者が何を確認するのかを読めるように、サマリーの箇条書きを同じメッセージ内の
すぐ上に置く:

```
- <each answer, as a summary bullet>

**Confirm** - Does this all look correct before I generate the artifact?

1. **Looks correct** - Generate the artifact from these answers
2. **Request changes** - Revise one or more answers before generation
3. **Other** - describe what you want instead

Reply with a number (or just tell me).
```

これは必須の人間のチェックポイントであり、ステージの承認ゲートではない。表示する前に、
`SKILL.md` にあるチェックポイント固有の `aidlc-log.ts decision` コマンドを、
正確な `--questions-file` と、あれば `--unit` / `--single` の
識別情報を含めて実行する。提示した後はターンを終え、ユーザーの返答を待つ。
その後、返答を正確な選択肢のラベルに対応付け、`[Answer]: Looks
correct` または `[Answer]: Request changes` を保存し、対応するチェックポイント固有の `aidlc-log.ts answer`
コマンドを実行する。書き込む前に、ソースの文字、番号付き散文の番号、句読点、選択肢の
説明を取り除く。`[Answer]: A. Looks correct`、`[Answer]: 1. Looks correct`、
自分で選んだ回答は無効である。Request changes の場合、利用者の返答が
すでに何を変えるべきかを述べていれば、その言葉がフィードバックである。そうでなければ
**"What should change?"** と尋ねて再びターンを終え、そのフィードバックが届くまで
どの回答も更新しない。その後、フィードバックを記録し、影響を受ける
回答を更新し、このタグを空に戻し、統合サマリーを再度提示する。
ファイルに人間の明示的な
`[Answer]: Looks correct` が含まれ、レシートのコマンドが成功するまでは、成果物を生成しない。この
チェックポイントを、後のレビュアー、学び、承認の手順と統合してはならない。

ルール:

- **承認ゲートの `[next stage]`**: 承認の質問では、
  `Continue to [next stage]` のプレースホルダを `next_stage` フィールドからそのまま表示する
  （例: `Continue to NFR Requirements`）。使うのは、ゲートを開いた返答
  （`report --result awaiting-approval` または `revised`）の値で、これにはステージ中に行われた
  計画の変更が反映されている。それがなければ run-stage ディレクティブの値を使う。
  `next_stage` が null の場合は `Complete workflow` と表示する。次のステージを推測してはならない。
- **承認ゲートの Accept as-is**: その返答、またはゲートを再表示するステップが、
  ゲートが3つ目の選択肢を提供するかどうかを示す。
  `accept_as_is: "offered"` の場合は、3つ目に `Accept as-is` を加え、
  `revision_cycle` から何回目かを示す。`"next-revision"` の場合は選択肢を2つのままにし、
  あと1回修正した後にそれが出てくるという注記を加える。
  回数を自分で数えてはならない。
- **見出しを太字にし**、次に問いかけ、次に番号付きの選択肢を仕様の
  順序で示す。質問に推奨の選択肢がある場合は、それを「最初」に置き、ラベルの末尾に
  "(Recommended)" を付ける。
- **ローカルな番号を毎回振り直す**: メッセージ内の前の番号付きの内容や、まとめて出す別の質問とは無関係に、
  すべての質問を `1` から始める。
  直前のサマリーには番号なしの箇条書きを使う。表示上の `1` は
  ソースの1番目の選択肢のラベルに、`2` は2番目に、以下同様に対応する。
- **常に "Other" の逃げ道を**最後の番号として加える — 仕様の
  選択肢にはそれが含まれない（Claude Code では UI が提供するが、ここでは自分で
  表示する）。
- **multiSelect: true** → "Reply with all numbers that apply (e.g. 1, 3)." と言う。
- **回答の取り込み**: ユーザーの番号を正確な選択肢の `label` に対応付け、
  そのラベルを逐語的に記録する（プロトコル: User Input を要約してはならない）。
  明らかに選択肢に一致する自由記述の返答は、その選択肢として扱う。質問に答えている
  利用者自身の言葉による返答は、利用者の回答である: それを
  "Other" の回答として、利用者の言葉のまま記録する。質問について尋ねたり、
  話し合いたがったりしている返答だけは、先に話し合う。その後、利用者が
  決めたものを受け取る。
- **ファイルベースの質問**: Markdown のソースでは A-E と X のラベルを保持するが、
  チャットで提示するときはそれらの選択肢を番号付きの散文に対応付け直す。
  ソースの順序を保ち、選ばれた番号を保存されたラベルに対応付け直す。
  上記の統合サマリーのチェックポイントは明示的な例外である: その
  選択肢にはソースの文字がなく、その `[Answer]:` には、ソースの文字、チャットの番号、接頭辞、説明ではなく、
  正確な意味上のラベルを保存する。
  ファイルの文字を返答のキーとして提示したり、ファイルの文字で答えるよう
  ユーザーに求めたりしてはならない。
- **まとめて出す**: 1つの質問あたりの選択肢の数にハーネスの制限はないが、まとめたものは
  読みやすく保つ — 1メッセージあたり最大でおよそ4問とし、選択肢が5つ以上の場合は1つの質問ごとに
  1メッセージにするのが望ましい。質問「ファイル」が引き続き正式な記録である。
  複数の質問を含むメッセージでは、それらに Q1、Q2 などのラベルを付け、最後を次の文で締める: "Reply with each question's number and the number of your choice (for example Q1: 2, Q2: 1), or just tell me." 文字を使った回答例を示してはならない。
- **選択肢を作り出さない**: 仕様の選択肢（+ Other）を正確に表示する。
