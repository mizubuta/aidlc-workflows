# Question Rendering — opencode harness annex

このファイルは、`{{HARNESS_DIR}}/aidlc-common/protocols/stage-protocol.md` のセクション "Structured questions" が求める構造化された質問を、
このハーネスがどのように表示するかを定める。
プロトコルとステージファイルはハーネスに依存しない。これらは「*構造化された質問を提示せよ*」と述べ、
フェンス付きの ` ```question ` 仕様ブロックを含んでいる。この付属文書は、
その契約を具体的な仕組みに結び付ける唯一の場所である。

## Never echo the spec (non-negotiable)

` ```question ` フェンスブロックは **この付属文書による表示への入力であり、決して
貼り付ける出力ではない**。オーケストレーターは、すべての ` ```question ` 仕様を
下記の Mechanism が定める番号付きの文章として**必ず**表示し、フェンスブロックやその
フィールド行（`prompt:`、`header:`、`multiSelect:`、`options:`、`label:`、`description:`）を
チャットのトランスクリプトに出力、印字、貼り付け、または「引用して返す」ことを**決して**してはならない。
ユーザーには決して生のフェンスを見せてはならない。ユーザーが見るのは番号付きの
文章による表示だけである。

フェンスをリテラルのテキストとして出力することは、文体上の選択ではなく **プロトコル違反** である。
それは次の問題を引き起こす:

- ユーザーが選べる番号付きの選択肢ではなく、回答できないブロックを生み出す。
- 番号付きの表示が末尾に加える "Other" という逃げ道が失われる。
- 同じセッション内の他の箇所での正しい表示と一貫しなくなる。

返信に三連バッククォートの `question` ブロックを書こうとしていることに気付いたら、
止まること。その内容は番号付きの文章として表示すべき仕様であり、メッセージ本文ではない。

これは **すべての** 構造化された質問の箇所に適用される。次のものを含むが、これらに限らない:

- 承認ゲート（すべてのステージ完了時）
- 質問の対話モードの選択（Guide me / I'll edit the file / Chat）。
  ステップがそれを求める場合に限る（それ以外の場合は、そのステップの案内を伝え、
  ステップの指示どおりに質問を受け付ける。stage-protocol.md §3 Step 2）
- ラダープロンプト（ウォーキングスケルトン後の自律モード）
- Bolt 失敗時の停止と確認（Retry / Skip / Abort）
- 成果物生成前の統合サマリーの確認
- 修正を受けた後のセクション 13 の学びに関する質問（このプロジェクト用に残す、チーム用に残す、または残さない）

（リテラルの ` ```question ` フェンスが、このファイルや stage-protocol のような
フレームワークのドキュメントに残っているのは正当である。それらはチャット出力ではなく、
作成用の仕様だからである。stage-protocol では、それらの仕様は規範的なプロンプトテンプレートであり、
周囲の指示が質問を求める場合、その内容は**必ず**この付属文書を通じて表示する。
この付属文書の対応付けの例は説明用である。
禁止されているのは、実際のオーケストレーションのターンで生のフェンスを出力することである。）

## Mechanism

opencode には構造化された質問のためのツールがないため、すべての構造化された質問は
**チャット内の番号付きの文章による選択肢** として表示し、ユーザーは番号（または
自由記述）で回答する。仕様は次のように表示する:

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

これは次のようになる:

```
**Approval** - [Stage Name] complete. How would you like to proceed?

1. **Approve** - Continue to [next stage]
2. **Request Changes** - Provide revision feedback
3. **Other** - describe what you want instead

Reply with a number (or just tell me).
```

## Mandatory consolidated-summary checkpoint

このチェックポイントは、`directive.ceremony.summary_confirmation === "on"` の場合にのみ適用される。`"off"` の場合は、回答から直接生成する。サマリー確認のプロンプト、確認エントリ、レシートはいずれも行わない。必須のステージ質問や、Plan Approval とステージの承認ゲートを含むその他の人間による判断は変わらない。

ガイド付きまたはチャットによるファイルベースの Q&A の後（および Requirements Analysis のように、
ステージ定義が明示的に求める場合は常に）、ステージプロトコルは、ステージの成果物を生成する前に
別途確認を行うことを求める。質問ファイルの
`## Consolidated Summary Confirmation` に、サマリー、プロンプト、A/B のファイル用の文字接頭辞を付けない
2つの選択肢、および空の `[Answer]:` タグを追記または更新し、その後
この番号付きの質問をチャットに表示する。サマリーの箇条書きは同じメッセージ内でその直上に置き、
利用者が確認する内容を読めるようにする:

```
- <each answer, as a summary bullet>

**Confirm** - Does this all look correct before I generate the artifact?

1. **Looks correct** - Generate the artifact from these answers
2. **Request changes** - Revise one or more answers before generation
3. **Other** - describe what you want instead

Reply with a number (or just tell me).
```

これは必須の人間によるチェックポイントであり、ステージの承認ゲートではない。
表示する前に、`SKILL.md` にあるこのチェックポイント専用の `aidlc-log.ts decision` コマンドを、
正確な `--questions-file` と、あれば `--unit` / `--single` の識別子を含めて実行する。
提示した後はターンを終了し（END THE TURN）、ユーザーの応答を待つ。
その後、応答を正確な選択肢ラベルに対応付け直し、`[Answer]: Looks
correct` または `[Answer]: Request changes` を保存して、対応するチェックポイント専用の `aidlc-log.ts answer`
コマンドを実行する。書き込む前に、元の文字、番号付き表示の番号、句読点、選択肢の
説明はすべて取り除く。`[Answer]: A. Looks correct`、`[Answer]: 1. Looks correct`、
および自分で選んだ回答は無効である。Request changes の場合、利用者の返信がすでに
何を変えるべきかを述べていれば、その言葉をフィードバックとする。そうでなければ
**"What should change?"**（何を変更すべきですか？）と尋ねて再びターンを終了し（END THE TURN）、そのフィードバックが届くまでどの回答も更新しない。その後、フィードバックを記録し、影響を受ける
回答を更新し、このタグを空に戻して、統合サマリーを再度提示する。
ファイルに人間による明示的な
`[Answer]: Looks correct` が含まれ、レシートコマンドが成功するまで、成果物を生成しない。このチェックポイントを、後続の
レビュアー、学び、承認のステップと決して統合しない。

ルール:

- **承認ゲートの `[next stage]`**: 承認の質問では、
  `Continue to [next stage]` プレースホルダを `next_stage` フィールドからそのまま表示する
  （例: `Continue to NFR Requirements`）。使うのは、ゲートを開いた返信
  （`report --result awaiting-approval` または `revised`）のもので、ステージ中に行われた
  計画変更も反映されている。それがなければ run-stage ディレクティブのものを使う。`next_stage` が null の場合は
  `Complete workflow` を表示する。次のステージを決して推測しない。
- **承認ゲートの Accept as-is**: その返信、またはゲートを再表示するステップが、
  ゲートに3つ目の選択肢があるかどうかを示す。
  `accept_as_is: "offered"` の場合は、3つ目に `Accept as-is` を加え、
  `revision_cycle` から何回目のラウンドかを示す。`"next-revision"` の場合は、選択肢を2つのままにし、
  あと1回の修正の後に提示される旨の注記を加える。ラウンド数を
  自分で数えてはならない。
- **ヘッダーを太字にし**、続けてプロンプト、仕様の順の番号付き選択肢を示す。
  質問に推奨の選択肢がある場合は、それを**最初**に並べ、ラベルの末尾に
  "(Recommended)" を付ける。
- **質問ごとに番号を振り直す**: メッセージ内で先に出た番号付きの内容や、バッチ内の
  別の質問とは関係なく、すべての質問を `1` から始める。
  直前に置くサマリーには番号なしの箇条書きを使う。表示上の `1` は
  元の最初の選択肢ラベルに、`2` は2番目に対応し、以下同様である。
- **常に "Other" という逃げ道を**最後の番号として追加する。仕様の
  選択肢には決して含まれていない（Claude Code では UI が提供するが、ここでは
  自分で表示する）。
- **multiSelect: true** の場合 → "Reply with all numbers that apply (e.g. 1, 3)." と伝える。
- **回答の取得**: ユーザーが選んだ番号を正確な選択肢の `label` に対応付け直し、
  そのラベルをそのまま記録する（プロトコル: User Input を決して要約しない）。
  明らかにある選択肢に一致する自由記述の返信は、その選択肢として扱う。
  質問に答えている利用者自身の言葉による返信はその人の回答であり、
  "Other" の回答として本人の言葉のまま記録する。質問について尋ねる返信や、
  話し合いたいという返信の場合に限り、先に話し合い、その後に決まった内容を
  回答として受け取る。
- **バッチ**: 1つの質問あたりの選択肢数にハーネス上の上限はないが、バッチは
  読みやすく保つ。1メッセージにつき質問は最大4つ程度とし、選択肢が5つ以上の場合は
  1メッセージに1つの質問とするのが望ましい。質問ファイルが正式な記録であることは変わらない。
  複数の質問を含むメッセージでは、それらに Q1、Q2 のようにラベルを付け、末尾を次の文で締めくくる: "Reply with each question's number and the number of your choice (for example Q1: 2, Q2: 1), or just tell me." 回答例に文字を使って示しては決してならない。
- **選択肢を新たに作らない**: 仕様の選択肢（と Other）だけを正確に表示する。
