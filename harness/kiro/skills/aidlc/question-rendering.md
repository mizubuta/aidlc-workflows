# Question Rendering — Kiro CLI harness annex

このファイルは、
`{{HARNESS_DIR}}/aidlc-common/protocols/stage-protocol.md` のセクション "Structured questions" が求める構造化質問を、**この**ハーネスがどのようにレンダリングするかを定義する。
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
stage-protocol の中のそれらの仕様は規範的なプロンプトテンプレートである: 周囲の指示が質問を求める場合、
その内容は必ずこの付属文書を通じてレンダリングしなければならない。この付属文書の対応付けの例は説明用である。
禁止しているのは、実際のオーケストレーションのターンで生のフェンスをエコーすることである。）

## Mechanism

Kiro CLI には構造化質問ツールがないため、すべての構造化質問は
**チャット内の番号付き散文の選択肢**としてレンダリングされ、ユーザーは番号（または
自由記述）で回答する。仕様は次のようにレンダリングする:

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

## Canonical interaction-mode rendering

インタラクションモードの質問は最も一般的な 3 択の仕様であり、合成した Other の逃げ道を
目に見える選択肢 `4` として必ず含めてレンダリングしなければならない。この質問は、ステップがそれを
求めている場合にのみ尋ねる。次のようにレンダリングする:

```
**Questions** - I've created [N] questions at `[file path]`. How would you like to answer them?

1. **Guide me** - Walk through each question interactively here
2. **I'll edit the file** - I'll fill in the answers in the file directly
3. **Chat** - Discuss freely - I'll extract decisions from our conversation
4. **Other** - describe what you want instead

Reply with a number (or just tell me).
```

メッセージの他の場所で Other に言及しても、この 4 番目の番号付き選択肢の
代わりにはならない。

## Pre-send invariant

番号付き散文の構造化質問を**いかなるものでも**送信する前に、レンダリングしたリストを確認し、
次の 3 つの条件がすべて成り立たない限り修正する:

1. 最後の番号付きの行が Other の選択肢である
2. 番号付きの Other の選択肢がちょうど 1 つだけある
3. その番号が、Other 以外の選択肢の数より 1 大きい

ソースにすでに
`X. Other (please specify)` を含むファイルベースの質問では、そのソースの行を最後の番号に
対応付け直せば、この不変条件を満たす。別の行を合成してはならない。番号付きリストの外で Other について
散文で触れたり、ヒントや指示を書いたりしても、この不変条件を満たすことには決してならない。

## Questions AI-DLC asks

AI-DLC が尋ねる質問は、レンダリング済みの状態で届く: プロンプト、番号付きの選択肢、
Other の行、返信のヒントである。それを与えられたとおりに出力し、**ターンを終了する**。質問の
テキストから組み立て直したり、言い換えたり、番号を取り除いたり、
選択肢を Yes/No にまとめたり、推奨を付け加えたり、先に別のツールを実行したり
状態を読み込んだりしてはならない: その質問自体がすでにルーティングの判断である。

そうしたレンダリングはすべて、最後に表示される選択肢が次の番号に続けて
`**Other** - describe what you want instead` となっていない限り無効である。"or tell
me" のような末尾の一文は Other の選択肢ではなく、それで置き換えてはならない。

選択肢を選んだ返信は、ステップの指示どおりにその選択肢のコマンドを実行する。
続行または作り直しの回答で、先に作業の対象を指定する必要がある場合は、
列挙された対象だけを Other 付きの番号付きリストとして提示し、ターンを終了する。再入時のやり直し、
ジャンプ、または最初からやり直す要求だけは、ステージに属さない
`report --result resumed --choice <redo|jump|fresh>` を使う。質問への回答に `report` を決して使ってはならない。

## Mandatory consolidated-summary checkpoint

このチェックポイントは、`directive.ceremony.summary_confirmation === "on"` の場合にのみ適用される。`"off"` の場合は、回答から直接生成する: サマリー確認のプロンプトも、確認のエントリーも、受領記録も作らない。ステージに必須の質問や、Plan Approval やステージの承認ゲートを含むその他の人間の判断は変わらない。

ガイド形式またはチャット形式のファイルベースの Q&A の後（および、Requirements Analysis のように
ステージ定義が明示的に求める場合は常に）、ステージプロトコルは、ステージの成果物を生成する前に
別個の確認を求める。質問ファイルに `## Consolidated Summary Confirmation` を
追記または更新し、サマリー、プロンプト、A/B のファイル用文字プレフィックスを付けない 2 つの選択肢、空の
`[Answer]:` タグを書き込む。そのうえで、この番号付きの質問をチャットでレンダリングする。その際、
サマリーの箇条書きを同じメッセージ内の質問の直上に置き、利用者が何を確認するのかを
読めるようにする:

```
- <each answer, as a summary bullet>

**Confirm** - Does this all look correct before I generate the artifact?

1. **Looks correct** - Generate the artifact from these answers
2. **Request changes** - Revise one or more answers before generation
3. **Other** - describe what you want instead

Reply with a number (or just tell me).
```

永続化される確認には文字の付かない意味上の選択肢が 2 つしかないが、チャットでは番号付きの `3. Other` が
必須である。Other の応答は議論を始め、チェックポイントを再提示する。それによってファイルに選択肢が
追加されることも、保存される `[Answer]:` になることも決してない。

これは必須の人間によるチェックポイントであり、ステージの承認ゲートではない。
レンダリングする前に、`SKILL.md` にあるこのチェックポイント専用の `aidlc-log.ts decision` コマンドを、
正確な `--questions-file` と、あれば `--unit` / `--single` の
識別子を含めて実行する。提示した後は**ターンを終了し**、ユーザーの応答を待つ。
次に、応答を正確な選択肢ラベルに対応付け直し、`[Answer]: Looks
correct` または `[Answer]: Request changes` を永続化して、対応するチェックポイント専用の `aidlc-log.ts answer`
コマンドを実行する。書き込む前に、ソースの文字、番号付き散文のインデックス、句読点、選択肢の
説明をすべて取り除く。`[Answer]: A. Looks correct`、`[Answer]: 1. Looks correct`、
および自分で選んだ回答は無効である。Request changes の場合、返信の中で
何を変えるべきかがすでに述べられていれば、その言葉がフィードバックとなる。そうでなければ
**"What should change?"** と尋ねて再び**ターンを終了し**、そのフィードバックが届くまでどの回答も
更新しない。その後、フィードバックを記録し、影響を受ける回答を更新し、
このタグを空に戻して、統合サマリーを再度提示する。
ファイルに人間による明示的な
`[Answer]: Looks correct` が含まれ、受領記録のコマンドが成功するまで、成果物を生成してはならない。このチェックポイントを、
その後のレビュアー、学び、承認のステップと決して統合してはならない。

ルール:

- **承認ゲートの `[next stage]`**: 承認の質問では、
  `Continue to [next stage]` のプレースホルダーを `next_stage` フィールドからそのまま
  レンダリングする（例: `Continue to NFR Requirements`）。使うのは、ゲートを開いた返信
  （`report --result awaiting-approval` または `revised`）のフィールドであり、これにはステージ中に行われた
  プランの変更が反映されている。それがなければ run-stage ディレクティブのフィールドを使う。`next_stage` が null の場合は
  `Complete workflow` をレンダリングする。次のステージを決して推測してはならない。
- **承認ゲートの Accept as-is**: その返信、またはゲートを再度表示するステップが、
  ゲートが 3 番目の選択肢を提示するかどうかを示す。
  `accept_as_is: "offered"` の場合は、3 番目に `Accept as-is` を追加し、
  `revision_cycle` から何回目のラウンドかを示す。`"next-revision"` の場合は、選択肢を 2 つのままにし、
  あと 1 回の修正の後にその選択肢が出るという注記を加える。ラウンド数を
  自分で数えては決してならない。
- **ヘッダーを太字にし**、続けてプロンプト、仕様の順序どおりに番号付きの選択肢を並べる。
  質問に推奨の選択肢がある場合は、それを**最初に**並べ、ラベルの末尾に
  "(Recommended)" を付ける。
- **ローカルな番号を振り直す**: メッセージ内の前の番号付きコンテンツやバッチ内の他の質問とは無関係に、
  すべての質問を `1` から始める。
  直前に置くサマリーには番号なしの箇条書きを使う。表示上の `1` は
  ソースの最初の選択肢ラベルに、`2` は 2 番目に、というように対応する。
- **最後の "Other" の逃げ道はちょうど 1 つ**: ソースの選択肢にまだ Other の選択肢が
  含まれていない場合は、表示上の最後の番号として `Other` を追加する。
  ファイルベースの質問がすでに `X. Other (please specify)` で終わっている場合は、
  その既存のソースの選択肢を表示上の最後の番号に対応付け直し、2 つ目の Other を
  追加しない。表示上の番号は、ソースの正確なラベル
  `X. Other (please specify)` に対応付け直される。
- **multiSelect: true** → "Reply with all numbers that apply (e.g. 1, 3)." と伝える。
- **回答の取得**: ユーザーの番号を正確な選択肢の `label` に対応付け直し、
  そのラベルをそのまま記録する（プロトコル: User Input を決して要約しない）。
  明らかにいずれかの選択肢に一致する自由記述の返信は、その選択肢として扱う。
  質問に答えている、利用者自身の言葉による返信は、その人の回答である: それを
  "Other" の回答として、その人の言葉のまま記録する。質問について尋ねている、または
  じっくり話し合いたいという返信の場合にのみ、まず議論する。その後、利用者が
  落ち着いた結論を採用する。
- **ファイルベースの質問**: Markdown のソースでは A-E と X のラベルを保持するが、
  チャットで提示するときはそれらの選択肢を番号付きの散文に対応付け直す。
  ソースの順序を保ち、選ばれた番号を保存済みのラベルに対応付け直す。
  上記の統合サマリーのチェックポイントは明示的な例外である: その
  ファイル上の選択肢にはソースの文字もファイルレベルの Other 行もない。チャットでの
  レンダリングでは最後に番号付きの Other を 1 つ合成し、その `[Answer]:` には
  ソースの文字、チャットの番号、プレフィックス、説明ではなく、正確な意味上のラベルを
  保存する。
  ファイル用の文字を回答のキーとして提示したり、ユーザーにファイル用の文字で
  回答するよう求めたりしては決してならない。
- **バッチ化**: 質問あたりの選択肢の数にハーネス上の制限はないが、バッチは
  読みやすく保つ — 1 メッセージあたり最大 4 問程度とし、選択肢が 5 つ以上ある場合は
  1 メッセージに 1 問を優先する。質問**ファイル**が引き続き正式な記録である。
  複数の質問を含むメッセージでは、質問に Q1、Q2 などのラベルを付け、最後を "Reply with each question's number and the number of your choice (for example Q1: 2, Q2: 1), or just tell me." で締めくくる。文字を使った回答例を決して示してはならない。
- **創発的な選択肢を作らない**: ソースの選択肢を正確にレンダリングし、ソースにまだ Other が
  含まれていない場合にのみ、合成した Other を 1 つ加える。NO
  EMERGENT BEHAVIOR ルールは、仕様だけでなくレンダリングにも適用される。
