# Sensors Protocol Module

エンジンが `directive.protocol_modules` に `sensors` を挙げたとき、つまり `directive.ceremony.sensors` が `on` のときに、このモジュールを読み込む。

### A blocking gate sensor at the approval gate

stage-protocol.md のセクション 2、Part 0 の手順 2 において:
- `directive.ceremony.sensors === "on"` のとき、ブロッキングのゲートセンサーが問題を検出した、または検証済みの合格を出せなかったために report（あるいはユニットのチェックポイントの `verify`）が拒否された場合、承認ゲートは開いていない（NOT open）。チェックポイントでは、拒否メッセージが `log decision` と `log answer` に使う `--unit` と再試行すべき `verify` を示し、利用者が以前のユニットで同じチェックをオーバーライドしていた場合は、質問の直前に伝える1行も示す。それらは書かれたとおりに使い、`<check>` は拒否メッセージの指示どおりに埋める。対話モードでは `{{INVOKE}} engine log decision --stage <slug> --decision "Blocking gate sensor failure" --options "Fix findings,Override blocking sensors"` を実行し、その2つの選択肢を独立した構造化質問として提示する。ターンを終了する（END YOUR TURN）。
- **Fix findings**: 人間がこれを選択した後、`aidlc-log.ts answer --stage <slug> --details "Fix findings"` を記録し（チェックポイントでは拒否メッセージの `--unit` を付ける）、指摘された検出事項または評価の失敗を修正してから、拒否メッセージが示したものをオーバーライドなしで再試行する。
- **Override blocking sensors**: 人間がこれを選択した後、`aidlc-log.ts answer --stage <slug> --details "Override blocking sensors"` を記録し（チェックポイントでは拒否メッセージの `--unit` を付ける）、拒否メッセージが示す再試行を `--override-blocking-sensors --user-input "Override blocking sensors"` 付きで実行する。状態ツールは、提示した選択肢と完全に一致する文言、`HUMAN_TURN`、および対応する decision/answer の記録を要求する。フラグだけでは失敗する。`Construction Autonomy Mode: autonomous` のもとでは、この選択肢を決して提示も試行もしてはならない。無人実行は明示的に停止する。
- **1つのチェックだけをオフにするよう求められた場合**: 1つのチェックだけを切り替える手段はなく、作業のすべてのチェックをまとめて切り替えるもの（`{{INVOKE}} engine orchestrate next --sensors off`）しかない。利用者が1つのチェックをオフにするよう求めたら、どちらの意味かを1回だけ尋ねる。3つ目の選択肢を加えて、チェックの質問を再度記録し（`--options "Fix findings,Override blocking sensors,Switch all checks off for this work"`）、Override はこのステージ（チェックポイントではこのユニット）の検出事項を対象とすることを伝え、選ばれたものを上記のとおり処理する。切り替えの場合は `next --sensors off` を実行し、その手順に従う。どちらの意味かを2回尋ねてはならない。以降のユニットで再びこのチェックをオフにするよう求められた場合は、そのユニットの拒否メッセージが示すとおり、以前の選択を適用する。

### Sensor Imports

このセクションの指示は `directive.ceremony.sensors === "on"` の場合にのみ適用される。`"off"` の場合、`sensors_applicable` は空であり、センサーによる修正、再実行、ゲートのブロック、センサーのオーバーライドに関する指示は一切適用されない。通常の成果物の検証と承認ゲートは引き続き必須である。

ステージの `sensors:` frontmatter リストが、そのステージがインポートするチェックの完全な集合である。
指定された各マニフェストは、ファイルの照合条件、コマンド、時間予算、`fire_on`
のタイミング、`default_severity` を定義する。実行対象になるのは、そのステージがインポートしたセンサーだけである。
`fire_on: write` は照合する書き込みの際に実行され、マニフェストが `blocking` を宣言していても、
このリリースでは助言（advisory）にとどまる。`fire_on: gate` は、ステージが承認ゲートに入る、または再び入るときに、
照合する宣言済みの成果物に対して実行される。その助言的チェックは、それより前、
ドキュメントが書かれて引き渡されるときに1回実行される。レビュー依頼、またはレビューが先に受け取らなかった場合は
ゲートを開く report が、チェックの検出内容と同じコマンドを示す `print` として返ってくる。
その手順はあなたの担当である。それらのドキュメントのうち自分の担当部分を修正し、コマンドを再実行する。
その後は再チェックせずに進み、ゲートが再チェックして残りを伝える。
助言的な結果は監査ログの行を出力するが、ゲートを止めない。Construction の自律モードでエンジン自身が承認するゲートでは、
助言的な証拠を読む者がいないため、ブロッキングセンサーだけが実行される。ブロッキングのゲートセンサーは検証済みの合格を要求する。
検出事項、評価不能、不正な出力、タイムアウトはいずれも、問題が修正されるか、上記の人間による
オーバーライドの流れが完了するまで、ゲートへの進入を拒否する。自律モードでは
ブロッキングセンサーをオーバーライドできない。

失敗したチェックは `SENSOR_FAILED` の監査ログ行を出力し、検出事項を
`<record>/.aidlc-engine/sensors/<stage-slug>/<sensor>-<fire-id>.md` に書き込む。スクリプトが
クラッシュした、または起動できなかったチェックは `SENSOR_COULD_NOT_RUN` を出力する。そのチェックは
何も検査していない。

ゲートが開くとき、または再表示されるとき、エンジンの応答は、宣言済みの出力に対してまだ失敗している各チェックと、
実行できなかったチェックを1行で、`narration` の中で
ステージが生成したものの後の独立した行に示す。それを承認の質問とともに利用者に伝える。
チェックが不合格を報告している、または実行できなかった範囲について、網羅していると主張してはならない（すべての要件を追跡済み、すべてのセクションが
存在、コードが型チェックを通る、など）。
利用者が変更を求めたら、その詳細ファイルに基づいて修正する。ユニット自身のゲートは
そのユニットの出力と書き込みだけを示す。センサーがオフの場合、
チェックについては何も言わない。

`required-sections` は、テンプレートが適用される場合、Markdown 出力のテンプレートにある `##` 見出しを要求する
（stage-protocol.md、Template overrides）。テンプレートがない場合は何も要求しない。

`upstream-coverage` は、出力の散文をステージの `consumes:`
frontmatter と比較する。存在する必須の成果物はすべて参照されていなければならず、
これにより出力がどの上流の入力に基づいたかがわかる。任意のもの
（`required: false`）は、ステージが `cited: true` を付けない限りチェックされない。必須または cited の `consumes:` を持たないステージは、
このチェックに自明に合格する。各ステージファイルにある簡潔な `Imports:` と `Upstream targets:` の行は
ローカルな要約であり、チェックを解決する際は frontmatter が正となる。
