# Question Rendering — Kiro IDE harness annex

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

(Literal ` ```question ` フェンスは、**この**ファイルや stage-protocol のような
フレームワークのドキュメントには正当に残る。それらはチャットの出力ではなく、作成用の仕様だからである。
stage-protocol では、それらの仕様は規範的なプロンプトテンプレートである。周囲の指示が質問を
要求する場合、その内容は**必ず**この付属文書を通じてレンダリングする。この付属文書の
対応付けの例は説明用である。禁止されているのは、実際のオーケストレーションのターンで
生のフェンスをエコーすることである。)

## Mechanism

Kiro IDE には構造化質問のツールがないため、すべての構造化質問は
**チャット内の番号付きの散文の選択肢**としてレンダリングされ、ユーザーは番号（または
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

インタラクションモードの質問は最も一般的な3択の仕様であり、合成した Other の逃げ道を
目に見える選択肢 `4` として**必ず**含めてレンダリングする。この質問は、ステップがそれを
求める場合にのみ尋ねる。次のようにレンダリングする:

```
**Questions** - I've created [N] questions at `[file path]`. How would you like to answer them?

1. **Guide me** - Walk through each question interactively here
2. **I'll edit the file** - I'll fill in the answers in the file directly
3. **Chat** - Discuss freely - I'll extract decisions from our conversation
4. **Other** - describe what you want instead

Reply with a number (or just tell me).
```

メッセージ内の別の場所で Other に言及しても、この4番目の番号付き選択肢の
代わりにはならない。

## Pre-send invariant

番号付き散文の構造化質問を送信する前には**必ず**、レンダリングしたリストを確認し、
次の3つの条件がすべて満たされていなければ修正する:

1. 最後の番号付きの行が Other の選択肢である。
2. 番号付きの Other の選択肢がちょうど1つだけ存在する。
3. その番号が、Other 以外の選択肢の数より1大きい。

ソースにすでに `X. Other (please specify)` を含むファイルベースの質問では、
そのソースの行を最後の番号に振り直せば不変条件を満たす。別の行を合成してはならない。
番号付きリストの外にある Other についての散文での言及、ヒント、指示は、
この不変条件を決して満たさない。

## Questions AI-DLC asks

AI-DLC が尋ねる質問は、レンダリング済みの状態で届く: プロンプト、番号付きの選択肢、
Other の行、返信のヒント。それを与えられたとおりに出力し、**ターンを終了する**。
質問のテキストから再導出したり、言い換えたり、番号を削除したり、選択肢を Yes/No に
まとめたり、推奨を追加したり、先に別のツールを実行したり状態を読み込んだりしてはならない。
その質問自体がすでにルーティングの決定である。

そのようなレンダリングはすべて、最後に表示される選択肢が、次の番号の後に
`**Other** - describe what you want instead` が続くものになるまでは無効である。"or tell
me"（または教えてください）のような末尾の文は Other の選択肢ではなく、それで置き換えてはならない。

選択肢を選んだ返信は、ステップの記述どおりにその選択肢のコマンドを実行する。作業を先に指定する必要がある
継続（continue）または再構成（reshape）の回答には、該当する作業だけを Other 付きの番号付きリストとして示し、
ターンを終了する。再入時のみのやり直し（redo）、ジャンプ（jump）、新規開始（start-fresh）の要求には、
ステージ以外の
`report --result resumed --choice <redo|jump|fresh>` を使う。質問への回答に `report` を使ってはならない。

## Mandatory consolidated-summary checkpoint

このチェックポイントは、`directive.ceremony.summary_confirmation === "on"` の場合にのみ適用される。`"off"` の場合は、回答から直接生成する: サマリー確認のプロンプト、確認のエントリ、レシートはいずれも不要である。必須のステージ質問やその他の人間による判断（Plan Approval やステージの承認ゲートを含む）は変わらない。

ガイド形式またはチャット形式のファイルベースの Q&A の後（および、Requirements Analysis のように
ステージ定義が明示的に要求する場合は常に）、ステージプロトコルは、ステージの成果物を生成する前に
別個の確認を要求する。質問ファイルの
`## Consolidated Summary Confirmation` に、サマリー、
プロンプト、A/B のファイル用文字プレフィックスを付けない2つの選択肢、空の
`[Answer]:` タグを追加または更新し、その後、この番号付きの質問をチャットでレンダリングする。
その際、利用者が確認する内容を読めるように、同じメッセージ内のすぐ上にサマリーの箇条書きを置く:

```
- <each answer, as a summary bullet>

**Confirm** - Does this all look correct before I generate the artifact?

1. **Looks correct** - Generate the artifact from these answers
2. **Request changes** - Revise one or more answers before generation
3. **Other** - describe what you want instead

Reply with a number (or just tell me).
```

永続化される確認には文字の付かない2つの意味的な選択肢しかないが、チャットでは番号付きの
`3. Other` が必須である。Other の回答は議論を開始してチェックポイントを再提示する。
ファイルに選択肢を追加したり、保存される `[Answer]:` になったりすることは決してない。

これは必須の人間によるチェックポイントであり、ステージの承認ゲートではない。レンダリングする前に、
`SKILL.md` にあるチェックポイント専用の `aidlc-log.ts decision` コマンドを、
正確な `--questions-file` と、あれば `--unit` / `--single` の
識別情報を含めて実行する。提示した後は**ターンを終了し**、ユーザーの返答を待つ。
その後、返答を正確な選択肢ラベルに対応付け、`[Answer]: Looks
correct` または `[Answer]: Request changes` を永続化し、対応するチェックポイント専用の `aidlc-log.ts answer`
コマンドを実行する。書き込む前に、ソースの文字、番号付き散文のインデックス、句読点、選択肢の
説明を取り除く。`[Answer]: A. Looks correct`、`[Answer]: 1. Looks correct`、
およびエージェント自身が選んだ回答は無効である。Request changes の場合、返信の中で
何を変えるべきかがすでに述べられていれば、その言葉をフィードバックとする。そうでなければ
**"What should change?"**（何を変更すべきですか？）と尋ねて再び**ターンを終了し**、そのフィードバックが届くまで
どの回答も更新しない。その後、フィードバックを記録し、影響を受ける回答を更新し、
このタグを空に戻して、統合サマリーを再度提示する。
ファイルに人間による明示的な
`[Answer]: Looks correct` が含まれ、レシートのコマンドが成功するまでは、成果物を生成してはならない。
このチェックポイントを、後続のレビュアー、学び、承認のステップと決して統合してはならない。

ルール:

- **承認ゲートの `[next stage]`**: 承認の質問では、
  `Continue to [next stage]` プレースホルダーを `next_stage` フィールドからそのまま
  レンダリングする（例: `Continue to NFR Requirements`）。使うのは、ゲートを開いた返信
  （`report --result awaiting-approval` または `revised`）にある値であり、これにはステージ中に行われた
  プランの変更も含まれる。それがなければ run-stage ディレクティブの値を使う。`next_stage` が null の場合は
  `Complete workflow` をレンダリングする。次のステージを決して推測してはならない。
- **承認ゲートの Accept as-is**: その返信、またはゲートを再表示するステップが、ゲートが
  3つ目の選択肢を提供するかどうかを示す。
  `accept_as_is: "offered"` の場合は、3つ目に `Accept as-is` を追加し、
  `revision_cycle` から何回目のラウンドかを示す。`"next-revision"` の場合は、選択肢を2つのままにし、
  あと1回の修正の後に提供される旨の注記を加える。ラウンド数を自分で
  数えてはならない。
- **ヘッダーを太字にし**、次にプロンプト、次に番号付きの選択肢を仕様の順序で並べる。
  質問に推奨の選択肢がある場合は、それを**最初に**並べ、ラベルの末尾に
  "(Recommended)" を付ける。
- **新しいローカルな番号付け**: メッセージ内のそれ以前の番号付きコンテンツや、バッチ内の別の質問とは
  無関係に、すべての質問を `1` から始める。
  直前のサマリーには番号なしの箇条書きを使う。表示上の `1` は
  ソースの最初の選択肢ラベルに、`2` は2番目に、以下同様に対応する。
- **最後の "Other" の逃げ道はちょうど1つ**: ソースの選択肢にまだ Other の選択肢が
  含まれていない場合は、最後の表示番号として `Other` を追加する。ファイルベースの質問が
  すでに `X. Other (please specify)` で終わっている場合は、その既存のソースの選択肢を
  最後の表示番号に振り直し、2つ目の Other を追加しない。表示番号は、正確なソースラベル
  `X. Other (please specify)` に対応付けられる。
- **multiSelect: true** → "Reply with all numbers that apply (e.g. 1, 3)."（該当する番号をすべて返信してください（例: 1, 3））と伝える。
- **回答の取り込み**: ユーザーの番号を正確な選択肢の `label` に対応付け、
  そのラベルをそのまま記録する（プロトコル: User Input を決して要約しない）。
  明らかにある選択肢に一致する自由記述の返信は、その選択肢として扱う。質問に答えている
  利用者自身の言葉による返信は、利用者の回答である: それを "Other" の回答として、利用者の言葉のまま
  記録する。質問について尋ねる返信や、話し合いたいという返信の場合にのみ先に議論し、
  その後、利用者が決めた内容を取り込む。
- **ファイルベースの質問**: Markdown のソースでは A-E と X のラベルを維持するが、
  チャットで提示するときはそれらの選択肢を番号付きの散文に振り直す。
  ソースの順序を保ち、選ばれた番号を保存されたラベルに対応付ける。
  上記の統合サマリーのチェックポイントは明示的な例外である: その
  ファイルの選択肢にはソースの文字もファイルレベルの Other の行もない。チャットでの
  レンダリングでは最後に番号付きの Other を1つ合成し、`[Answer]:` には
  ソースの文字、チャットの番号、プレフィックス、説明ではなく、正確な意味的ラベルを保存する。
  ファイルの文字を回答のキーとして提示したり、ユーザーに
  ファイルの文字で回答するよう求めたりしてはならない。
- **バッチ処理**: 1つの質問あたりの選択肢数にハーネス上の制限はないが、バッチは
  読みやすく保つ — 1メッセージあたり最大で約4問とし、選択肢が5つ以上ある場合は
  1質問につき1メッセージにするほうがよい。質問**ファイル**が引き続き正となる記録である。
  複数の質問を含むメッセージでは、それらに Q1、Q2 などのラベルを付け、最後を次の文で締めくくる: "Reply with each question's number and the number of your choice (for example Q1: 2, Q2: 1), or just tell me."（各質問の番号と選んだ番号を返信してください（例: Q1: 2, Q2: 1）。または、そのまま教えてください。）文字を使った回答例を決して示してはならない。
- **選択肢を勝手に増やさない**: ソースの選択肢を正確にレンダリングし、ソースに Other が
  まだ含まれていない場合にのみ、合成した Other を1つ追加する。NO
  EMERGENT BEHAVIOR ルールは、仕様だけでなくレンダリングにも適用される。
