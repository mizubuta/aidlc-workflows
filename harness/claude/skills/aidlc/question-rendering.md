# Question Rendering — Claude Code harness annex

このファイルは、`{{HARNESS_DIR}}/aidlc-common/protocols/stage-protocol.md` のセクション "Structured questions" が求める構造化された質問を、
このハーネスがどのように表示するかを定める。
プロトコルとステージファイルはハーネスに依存しない。これらは「*構造化された質問を提示せよ*」と述べ、
フェンス付きの ` ```question ` 仕様ブロックを含んでいる。この付属文書は、
その契約を具体的な仕組みに結び付ける唯一の場所である。

## Never echo the spec (non-negotiable)

` ```question ` フェンスブロックは **`AskUserQuestion` ツールへの入力であり、決して
表示用の出力ではない**。オーケストレーターは、すべての ` ```question ` 仕様を
実際の `AskUserQuestion` ツール呼び出しに**必ず**変換し、フェンスブロックやその
フィールド行（`prompt:`、`header:`、`multiSelect:`、`options:`、`label:`、`description:`）を
チャットのトランスクリプトに出力、印字、貼り付け、または「引用して返す」ことを**決して**してはならない。
ユーザーには決して生のフェンスを見せてはならない。ユーザーが見るのはネイティブの
`AskUserQuestion` プロンプトだけである。

フェンスをリテラルのテキストとして出力することは、文体上の選択ではなく **プロトコル違反** である。
それは次の問題を引き起こす:

- ユーザーがクリックも選択もできない、対話性のない文字の壁を生み出す。
- `AskUserQuestion` が提供する組み込みの "Other" という逃げ道が失われる。
- 同じセッション内の他の箇所での正しい表示と一貫しなくなる。

返信に三連バッククォートの `question` ブロックを書こうとしていることに気付いたら、
止まること。その内容はメッセージ本文ではなく、`AskUserQuestion` ツール呼び出しの中に入れるべきものである。

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

Claude Code では、すべての構造化された質問は **`AskUserQuestion`
ツール** で表示する。フェンス付きの ` ```question ` 仕様が入力で、ツール呼び出しが
出力であり、決してその逆ではない。仕様のフィールドを 1:1 で対応付ける:

| 仕様のフィールド | AskUserQuestion のフィールド |
|------------|----------------------|
| `prompt` | `questions[0].question` |
| `header` | `questions[0].header` |
| `multiSelect` | `questions[0].multiSelect` |
| `options[].label` | `questions[0].options[].label` |
| `options[].description` | `questions[0].options[].description` |

例 — 次の仕様は:

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

次のように表示される:

```
AskUserQuestion({
  questions: [{
    question: "[Stage Name] complete. How would you like to proceed?",
    header: "Approval",
    multiSelect: false,
    options: [
      { label: "Approve", description: "Continue to [next stage]" },
      { label: "Request Changes", description: "Provide revision feedback" }
    ]
  }]
})
```

## Mandatory consolidated-summary checkpoint

このチェックポイントは、`directive.ceremony.summary_confirmation === "on"` の場合にのみ適用される。`"off"` の場合は、回答から直接生成する。サマリー確認のプロンプト、確認エントリ、レシートはいずれも行わない。必須のステージ質問や、Plan Approval とステージの承認ゲートを含むその他の人間による判断は変わらない。

ガイド付きまたはチャットによるファイルベースの Q&A の後（および Requirements Analysis のように、
ステージ定義が明示的に求める場合は常に）、ステージプロトコルは、ステージの成果物を生成する前に
別途確認を行うことを求める。質問ファイルの
`## Consolidated Summary Confirmation` に、サマリー、プロンプト、A/B のファイル用の文字接頭辞を付けない
2つの選択肢、および空の `[Answer]:` タグを追記または更新し、その後
`AskUserQuestion` を通じて意味上の2つの選択肢を表示する。質問のテキスト自体はサマリーの
箇条書きから始まるため、利用者は確認する内容を選択画面の中で読める。記録される判断は
最後の行だけのままである:

```
AskUserQuestion({
  questions: [{
    question: "- <each answer, as a summary bullet>\n\nDoes this all look correct before I generate the artifact?",
    header: "Confirm",
    multiSelect: false,
    options: [
      {
        label: "Looks correct",
        description: "Generate the artifact from these answers"
      },
      {
        label: "Request changes",
        description: "Revise one or more answers before generation"
      }
    ]
  }]
})
```

これは必須の人間によるチェックポイントであり、ステージの承認ゲートではない。
表示する前に、`SKILL.md` にあるこのチェックポイント専用の `aidlc-log.ts decision` コマンドを、
正確な `--questions-file` と、あれば `--unit` / `--single` の識別子を含めて実行する。
提示した後はターンを終了し（END THE TURN）、ユーザーの応答を待つ。
その後、`[Answer]: Looks correct` または `[Answer]: Request changes` を正確にそのまま保存し、
対応するチェックポイント専用の `aidlc-log.ts answer` コマンドを実行する。書き込む前に、
元の文字、句読点、選択肢の説明はすべて取り除く:
`[Answer]: A. Looks correct`、`[Answer]: 1. Looks correct`、`[Answer]: A`、および
自分で選んだ回答は無効である。Request changes の場合、利用者の返信がすでに
何を変えるべきかを述べていれば、その言葉をフィードバックとする。そうでなければ
**"What should change?"**（何を変更すべきですか？）と尋ねて再びターンを終了し（END THE TURN）、そのフィードバックが届くまでどの回答も更新しない。その後、フィードバックを記録し、影響を受ける回答を更新し、このタグを
空に戻して、統合サマリーを再度提示する。ファイルに人間による明示的な `[Answer]: Looks correct` が含まれ、
レシートコマンドが成功するまで、成果物を生成しない。このチェックポイントを、後続の
レビュアー、学び、承認のステップと決して統合しない。

## Harness-specific behaviors

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
- **バッチの上限**: 1回の `AskUserQuestion` 呼び出しにつき質問は最大4つ、1つの質問につき
  選択肢は最大4つ、そして **1つの質問につき選択肢は最低2つ**。質問ファイル内の質問で
  それより選択肢が多いものは、代わりに番号付きリストとして尋ねる
  （stage-protocol.md Step 3a。複数選択の場合は末尾を "Reply with all
  numbers that apply (e.g. 1, 3)." とする）。質問ファイルは、完全な
  選択肢一式を正式な記録として保持する。選択肢が1つだけの呼び出しは決して送らない。
  ユーザーが回答する前にツールがそれを拒否するためである。
- **"Other" による逃げ道**: `AskUserQuestion` には組み込みの "Other" 選択肢があり、常に
  利用できる。対話的なバッチでは、仕様の選択肢リストに明示的な Other 選択肢を
  追加しては**ならない**。Claude Code はその行に "Type something" というラベルを付けるため、
  利用者に言及するときは常にその名前を使う: "Pick "Type
  something" on any question to answer in your own words or talk it through."（質問ファイルでは、
  プロトコル §3 に従い、引き続きすべての質問の末尾を
  `X. Other (please specify)` とする。ファイル形式は
  ハーネスに依存しないためである。）
- **回答の取得**: ユーザーの選択は、選択肢のラベルそのものとして返される。
  それをそのまま記録する（プロトコル: User Input を決して要約しない）。
- **長いプロンプト**: 質問本文は端末の全幅で表示され、
  適切に折り返される（複数行の折り返しは各リリース前に macOS で検証済み）。
  長いパスの場合のフォールバックについては、
  `{{HARNESS_DIR}}/knowledge/aidlc-shared/worktree-info-schema.md` を参照する。
