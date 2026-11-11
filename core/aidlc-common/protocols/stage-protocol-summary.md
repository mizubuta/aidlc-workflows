# Summary Protocol Module

エンジンが `directive.protocol_modules` に `summary` を列挙している場合、つまりそのステップがサマリー確認を求めている場合（`directive.ceremony.summary_confirmation` が `on`）に、このモジュールを読み込む。これは stage-protocol.md のセクション 3 が Step 3a で指定しているチェックポイントであり、Step 3b と 3c も同じように使う。

- **生成前の統合サマリー**: 以下のチェックポイントは、ステップがサマリー確認を求めている場合にのみ適用する。求めていない場合は、確認のプロンプト、確認エントリ、受領記録なしで、回答から直接生成する。確認がオンの場合は、すべての質問に回答が
  得られた後、すべての回答の統合サマリーを順序なしの箇条書きで提示する（番号付きリストは決して使わない）。利用者は自分が読めるものを確認する。箇条書きは、質問レンダリング付録が示すとおり、質問そのものの中か、同じメッセージ内でその直前に置く。ツールの出力やファイルの中だけに置いてはならない。次に
  `bun {{HARNESS_DIR}}/tools/aidlc-review-brief.ts summary --stage "<directive.stage>" --questions-file "<questions-path>"` を実行する。
  ユニット単位のステージでは `--unit "<directive.unit>"` を追加する。この構造化質問を提示する前に、
  その簡潔な判断ブリーフを一字一句そのまま出力する。ブリーフには、
  ステージ、確認対象の質問ファイルと成果物、そして両方の選択肢の
  正確な効果が記載されている:
  ```question
  prompt: "Does this all look correct before I generate the artifact?"
  header: Confirm
  multiSelect: false
  options:
    - label: Looks correct
      description: Generate the artifact from these answers
    - label: Request changes
      description: Revise one or more answers before generation
  ```
  提示する前に、`<slug>-questions.md` に専用の **Consolidated Summary Confirmation**
  エントリを追記または更新する。このエントリには、このプロンプト、**ファイル用の文字プレフィックスを
  付けない**両方の選択肢、空の `[Answer]:` タグを記載する:
  ```markdown
  - Looks correct
  - Request changes

  [Answer]:
  ```
  この確認エントリは、通常のファイルベースの {{OPTION_LETTERS}}/X
  ラベルの例外である。そのタグは利用者が応答した後にのみ記入し、正確に
  `[Answer]: Looks correct` または `[Answer]: Request changes` を格納する。書き込む前に、元の
  文字、チャットの番号、句読点、選択肢の説明はすべて取り除く。
  `[Answer]: A. Looks correct` や `[Answer]: 1. Looks correct` は無効である。
  提示する前に、チェックポイントのプロンプトを記録する:
  `{{INVOKE}} engine log decision --stage <slug>
  --checkpoint summary-confirmation --questions-file "<questions-path>"
  --decision "Does this all look correct before I generate the artifact?"
  --options "Looks correct,Request changes"`。ユニット単位のステージでは `--unit "<directive.unit>"` を、
  単独実行では `--single` を追加する。この確認を地の文だけで尋ねては決してならない。ハーネスは、ターンが終わる前に回答可能な構造化
  質問をレンダリングしなければならない。

  利用者が応答したら、その返信を読み、利用者が選んだ選択肢を
  確認の `[Answer]:` タグ（`Looks correct` または `Request changes`）に書き込み、
  人間の応答に裏付けられた受領記録を
  `{{INVOKE}} engine log answer --stage <slug>
  --checkpoint summary-confirmation --questions-file "<questions-path>"
  --details "Looks correct"`（または `--details 'Request changes: <what they asked
  to change>'`。下記のとおりシングルクォートで囲む）で記録する。その際、同じ `--unit` / `--single`
  の識別子を使う（stage-protocol.md セクション 1 の "Reading the person's reply at a checkpoint" を参照）。ツールは、
  自分で選んだ回答、対応するプロンプト記録のない応答、記録以降に利用者の
  返信がない応答、または格納された選択が記録しようとするものと異なる質問ファイルを
  拒否する。プロンプトを記録する前に利用者が返信していた場合（サマリーを先に見せた場合）は、
  それに回答したメッセージを指す `--message <id>` を追加し、利用者に再度尋ねない。
  自分の言葉を伴わない **Other** の選択は、stage-protocol.md セクション 1 の Other-escape
  ルールに従う。それについて話し合い、確認を再提示し、タグと
  受領記録には手を付けない。それ以外のすべての返信は、そこにある返信の読み取りルールに従う。

  選択が **Request changes** の場合は、兄弟の
  `## Requested Changes Feedback` 質問を空の `[Answer]:` 付きで追記する。利用者の
  返信がすでに何を変更すべきかを述べている場合（受領記録の出力がそれを
  `feedback` として繰り返す）、その言葉がフィードバックである。再度尋ねずにフォローアップのタグに
  書き込む。そうでない場合は、自由記述の質問
  **"What should change?"**（何を変更すべきですか?）を直接尋ね、ターンを終了する（END THE TURN）。人間が
  そのフィードバックを提供するまで、何も修正してはならない。尋ねて得たフィードバックは
  通常の `aidlc-log.ts decision` / `answer` のペアで記録し、フォローアップのタグに書き込み、
  関連する回答タグを更新し、確認エントリを空の `[Answer]:` にリセットし、
  サマリーを再提示する。人間が明示的に **Looks correct** を選び、受領記録のコマンドが
  成功した後にのみ、成果物の生成に進む。
  以降の Request Changes のサイクルごとに、別の兄弟フィードバックセクションを追記する。
  それらのセクションは時系列順に保持する。ステージに
  `Assumption Confirmation` セクションがある場合は、フォローアップ質問が変換されたときに
  サマリー後の本文と回答を置き換える。重複する見出しを追記してはならない。
  フォローアップ質問は確認済みの意味内容を変えるため、成果物を再保存したり
  レビューを依頼したりする前に、統合サマリーを再度提示し、新しい確認の受領記録を記録する。
  サマリーがすでに確認済みのファイルにおける後続の質問ラウンド（修正、
  または承認後に再開されたステップ）も同じ形をとる。その質問は、サマリーの後に置く新しい `## Q<n>` セクションであり、
  最後のものから続けて番号を付ける。独立したセクションや 2 つ目のサマリーにしてはならない。1 つの
  Consolidated Summary Confirmation セクションの箇条書きを、新旧すべての回答を
  網羅するように書き直し、その `[Answer]:` 行を空にして、上記のとおり再度確認する。
