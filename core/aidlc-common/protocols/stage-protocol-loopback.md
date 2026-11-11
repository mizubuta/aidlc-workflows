# Loop-back Protocol Module

エンジンが `directive.protocol_modules` に `loopback` を列挙したときにこのモジュールを読み込む。対象は、Code Generation と Build and Test のすべてのステップ、およびすべての `invoke-swarm` である。

### Build-and-Test failure loop-back (3.6 → 3.5)

Build and Test（3.6）が診断した失敗の根本原因（ROOT CAUSE）が、生成された
コード、または code-generation で選択されたアプローチにある場合（このステージ
自身のテスト/ビルド用の足場ではなく）、ワークフローはそのアプローチを見限ったりゲートで
行き詰まったりする代わりに、code-generation に戻って修復できる。
これが発動する「タイミング」はステージの Step 9 の失敗エスカレーションラダーが決め、この
サブセクションは「方法」を定義する。これは Critical チェックリスト項目 5 の
"complete the current stage before jumping"（ジャンプする前に現在のステージを完了する）に対する公認の例外である。失敗した build-and-test の
実行は意図的に実行途中のまま残される — そのゲートは提示されず（NOT presented）、有効化されている §13 の
学びの儀式は最終的に成功する実行まで延期（DEFERS）される（`learnings` モジュールが列挙されている場合、ステージの日誌
memory.md はループをまたいで保持される。そうでない場合、日誌も儀式も実行されない）。

**どのユニットが戻るか。** ループバックは、診断が指定した
ユニットについて Code Generation を再オープンする。それ以外のユニットはすべて、完了済みの作業、
レビュー、Plan Approval、チェックポイントの承認を維持し、再実行、再レビュー、
再質問の対象にはならない。すべてのユニットにまたがる原因、または特定のユニットに
絞り込めない原因の場合にのみ、すべてのユニットについて Code Generation を再オープンする。いずれの場合も、Code
Generation より前のステージは作業を維持する。以下で「対象ユニット」とは、再オープンされたユニットを指す。

**ループバックカウンター**は test-results.md の `## Loop-Back Log` の下にある。
`### Loop-back N` エントリの数がそのまま上限の判定に使われる（インテントあたり最大 3 回）。ジャンプの監査行を
解析するのではなくこの成果物の台帳を選んでいるのは、後方へのジャンプを経ても残り（ジャンプはチェックボックスをリセットするが、成果物は決してリセットしない）、
いずれにせよ記録しなければならない診断と同じ場所にあり、最終ゲートで読めるためである。
ジャンプツールが出力する行（すべてのユニットに対する `STAGE_JUMPED`、再オープンされたユニットごとのユニットタグ付き
`GATE_REJECTED`）は、引き続き決定論的な監査の照合手段となる。ログは追記専用である。人間の指示による
後方へのジャンプは上限にカウントされない — このプロトコルが書くエントリだけがカウントされる。

**再実行時のプラン承認。** ジャンプは、再オープンする各ユニットについて新しいステージ試行を
開始するため、そのユニットの以前の Plan Approval（前回の試行に紐付いている）では再実行を
承認できない。
Loop-Back Log を保持する。修復したプランが書かれた後、`next` は生成の前に再度
利用者に Plan Approval を求める（Code Generation Step 3）。人間が
"Retry with fix" を選択したことはループバックジャンプを承認するものであり、新しい試行のもとで
人間がレビューしていないプラン内容を承認するものではない。

**自律ループバックの手順**（モードが `autonomous`、上限に達していない、
影響を見積もった修正が特定済み）:
1. `### Loop-back N — <ISO timestamp>` エントリ（Diagnosis /
   Root-cause stage / Planned fix / Estimated impact）を test-results.md に追記し、対応する
   Deviations エントリをこのステージの memory.md に追記する。
2. エンジンを通じてジャンプを実行する: 診断が指定した各ユニットについて、
   `bun {{HARNESS_DIR}}/tools/aidlc-orchestrate.ts next --stage
   code-generation --unit <unit>` を実行し、次のユニットに進む前に、その `print` ディレクティブが
   指定するコマンドをそのまま実行する（`aidlc-jump.ts reopen --target
   code-generation ... --units <unit>` で、そのユニットを再オープンし、
   Current Stage を Code Generation に戻す）。すべてのユニットにまたがる原因、または
   ユニットを指定しない原因の場合にのみ、代わりに `next --stage code-generation` を実行する。これは
   `aidlc-jump.ts execute --target code-generation --direction backward
   --scope <scope>` を自ら実行し、すべてのユニットについて Code Generation とそれ以降の
   ステージをリセットして正規の `STAGE_JUMPED` を出力し、Code
   Generation のステップを返す。Construction を
   ステージ単位で実行している場合（aidlc-state.md の `Construction Iteration` が
   `unit-major` でない場合）、1 つのユニットだけを再オープンすることはできない。その
   `next --stage code-generation` を、尋ねずに直ちに実行する。"Retry with fix"
   （または自律の許可）によって既に修復が選択されている。`reopen` の後は
   `next` を再実行し、戻ってきたステップから転送ループを続ける。
   `reopen` や `execute` の呼び出しを決して手で組み立てない。エンジンの print が
   検証済みの形式である。
3. code-generation に再入したら、下記の "Re-entry settlement and review"
   に従う。修正の生成を行う前に、エンジンは上記のとおり修復したプランの Plan Approval を
   求める。Construction の自律が許可されたままであっても、これは人間による停止点である。その後、計画した修正を診断が指定したユニットにのみ（ONLY）適用し、
   決定論的な Artifact Re-use の判断を適用する（stage-protocol.md の Artifact Re-use の下にある "Autonomous failure
   loop-back" を参照）。常設の
   `Construction Autonomy Mode: autonomous` の許可はジャンプによって変わらない。
   すべての対象ユニットに現在の試行での新しいレビューが揃った後、再実行された
   完了処理は現在のチェックポイント/ポリシーのディレクティブに従う。自動的な
   完了では `--user-input` を省略する。コンダクターの判断を人間の回答として
   ラベル付けしてはならない。新しい Plan Approval は依然として必須である。
4. その後、Build and Test は前方への再実行の中で自然に再実行される。その Artifact Re-use の
   問いでは Modify を選び（Redo は決して選ばない — Loop-Back Log が消去されてしまう）、Step 9 を新たに実行し直す。

**Re-entry settlement and review（再入時の決着とレビュー）。** 後方へのジャンプは成果物を保持するが、
その経路は code-generation がこれまでにユニットのライフサイクル台帳を使ったことがあるかどうかで
変わる:

1. **成果物のみのワークフロー** — code-generation のライフサイクル行が一度も
   出力されておらず、Construction のチェックポイントがオフの場合（オンの場合は、最初のユニットから
   レシートが必要となる）、引き続き成果物が決着のシグナルとなる。そのため、再入時の `next`
   呼び出しは、全件カバー済みの `gate: true` の高速パスを出力しうる。そのゲートを提示または自動承認する
   前に（BEFORE）、再入時のオーバーライドを通じて、計画した修正と決定論的な Modify/Keep の判断を
   適用する。
2. **レシートモードのワークフロー** — code-generation のライフサイクル行が 1 つでも存在すると、
   レシートモードは固定される（Construction のチェックポイントがオンの場合は、最初の
   ユニットから適用される）。再オープンによって各対象ユニットの古い
   決着レシートが無効になるため、再入時にはそれらのユニットについてユニットごとの `run-stage` ディレクティブが
   出力される。各対象ユニットについて、
   `unit start` / `unit complete` を発行し直し、対象のユニットには計画した修正を、
   そしてそのユニットの再実行時にインラインで、決定論的な **Modify targeted / Keep rest** の Artifact Re-use
   の判断を適用する。

どちらの経路でも（BOTH）、すべての修正と再利用の判断を終えた後、かつ決着/承認ゲートを提示または
自動承認する前に（BEFORE）、すべての対象ユニットについて code-generation で宣言された
レビュアーをディスパッチし、現在の試行での新しい
`REVIEW_COMPLETED` レシートを記録する。再オープンによって各対象ユニットの
以前のレビューレシートは無効になり（ステージ全体の `STAGE_JUMPED` の後ではすべてのユニットのもの）、
新しいレシートを欠く対象ユニットが 1 つでもある間、エンジンは承認を拒否する。unit-major の反復では、自律スウォームは決して
発動しない。再実行は通常のユニットごとの順次処理に従い、上記のとおりユニットごとにライフサイクルと
レビューのレシートを発行し直す。対象に紐付いた新しい Plan Approval は、修復のための
人間による停止点であり続ける。自律はそれを免除しない。

**スウォームとの相互作用。** エンジンが `invoke-swarm` を出力するループバックの再実行では、
ジャンプによって新しい正確なステージ試行の `Run floor`
境界トークン（ワークフロー開始、ジャンプ、却下、ステージ開始の境界にわたる `<event>:<timestamp>#<ordinal>`）が確立される。各 `SWARM_UNIT_CONVERGED` 行は
現在のトークンと一致しなければならないため、以前の試行の行はもはやカウントされず、既定ではすべてのユニットが
再ディスパッチされる。`prepare` の前に、`worktree list` と `worktree info`
を使って、選択されたインテントについて以前の試行が残したワークツリーとブランチを特定する
（スウォームの途中でのクラッシュや halt-and-ask はそれらを残したままにする）。
`prepare` は衝突があるとハードエラーになり、
`finalize` は現在の試行の prepare スタンプがないユニットを拒否するため、
新たに `prepare` する前に古いワークツリー/ブランチを破棄する — それらを新しい試行に
取り込んではならない。破棄するのは選択されたインテントの Bolt だけである。別のインテントにある
同じ Unit slug は古い兄弟ではなく、クリーンアップは他のワークツリーのパスでチェックアウトされている
ブランチを拒否する（削除するのではなく、所有者のパスを提示する）。
破棄は各試行を退避させる。そのスナップショットは
`{{INVOKE}} engine worktree restore --slug <slug>` によって、別の復元用チェックアウトで
復元可能なままだが、現在の試行の証拠として扱ってはならない。ユニットごとにワーカーのターンを費やさない:
`prepare` の後、まず（FIRST）すべてのユニットで
`check <unit> --check-cmd '<the convergence check the project runs>'` を
実行する。既にグリーンのユニットにビルダーのターンは不要だが、それを
`finalize --claimed` に入れる前に、その新しいワークツリーで code-generation のレビュアーを
ディスパッチし、現在の試行の終端 `REVIEW_COMPLETED` を記録する。`finalize`
はその後、現在の prepare スタンプ、終端レシート、および現在の
成果物フィンガープリントを検証してから、その主張を受け入れる。ワーカーをディスパッチするのは、
Loop-Back Log の計画した修正が対象とするユニット、またはチェックに失敗したユニットに対してだけであり、
それらの最終的な変更の後に同じレビュアーのパスを実行する。この安価な経路は、
以前の試行のコードがワークツリーの分岐元のベースに含まれていることを前提とする — これが真となるのは、
その試行の git のコードマージが実際に完了した場合だけである。完了していない場合（その
試行が finalize の前に停止した場合）、すべての `check` が赤になり、安価な
経路は、未構築のユニットを黙って構築済みと主張するのではなく、完全な再ディスパッチへと穏やかに
退化する。

**Halt-and-ask、影響見積もり付きのバリアント（gated モードまたは未設定のモード、あるいは上限到達で、かつ
修正候補が特定されている場合）:**

```question
prompt: "Build and Test failed: [short error]. Root cause: [diagnosis]. Candidate fix: [fix] — estimated impact — effort: [effort]; financial cost: [cost]; risk: [risk]. Loop-backs used: [N]/3. How would you like to proceed?"
header: Build Failure
multiSelect: false
options:
  - label: Retry with fix
    description: Jump back to code-generation, apply [fix] (estimated impact — effort: [effort]; financial cost: [cost]; risk: [risk]), re-run.
  - label: Accept failure
    description: Log the failure in test-results.md and proceed to this stage's approval gate.
  - label: Abort
    description: Stop here; the workflow can resume later.
```

**Halt-and-ask、修正なしのバリアント（入れ替え可能などの次元にも
特定可能な修正が存在しない場合）:** "Retry with fix" は完全に省略する — 修正
候補なしでそれを提示すること自体が、このプロトコルが逆方向に禁じている、影響を見積もっていない
諦めの選択肢（リトライ用にでっち上げた修正）になってしまう。次を使う:

```question
prompt: "Build and Test failed: [short error]. Root cause: [diagnosis]. No identifiable fix exists in any swappable dimension (library/version, container image, instance type, algorithm, flag). Loop-backs used: [N]/3. How would you like to proceed?"
header: Build Failure
multiSelect: false
options:
  - label: Accept failure
    description: Log the failure in test-results.md and proceed to this stage's approval gate.
  - label: Abort
    description: Stop here; the workflow can resume later.
```

バリアントは、ラダーの段階 2 の分類・見積もりステップが影響を見積もった修正候補を実際に
生成したかどうかで選ぶ — テンプレートの形を保つためだけに、影響見積もり付きテンプレートの
`Candidate fix` / `Retry with fix` の欄をプレースホルダーやでっち上げた
内容で埋めてはならない。

"Retry with fix" は、自律ループバックと同じ決着を考慮した手順を、
その再入時のオーバーライドも含めて実行する（下記の Artifact Re-use にある "Gated failure loop-back"
を参照）。成果物のみのワークフローは
全件カバー済みの `gate: true` の高速パスを取りうる。レシートモードのワークフローは代わりに
ユニットごとのディレクティブを再出力する。どちらの経路でも、計画した修正と決定論的な
Modify/Keep の判断は決着/承認ゲートの前に必ず（MUST）適用し、
すべての対象ユニットは、そのゲートが提示される前に、現在の試行での新しいレビューを必ず（MUST）受けなければならない。
人間が承認したリトライは Loop-Back Log のエントリとしてカウントされ、人間は上限を明示的に
オーバーライドできる。各選択肢の
説明には、判明している場合は見積もった影響を必ず含める — 影響を見積もっていない
諦めの選択肢を提示することはプロトコル違反である。

### Artifact Re-use on a failure loop-back replay

失敗ループバックが開始した再実行では、stage-protocol.md の Artifact Re-use の問い（Keep、Modify、Redo）は次のように決定される。

**Autonomous failure loop-back（自律失敗ループバック）**: 再実行が、
`Construction
Autonomy Mode: autonomous` のもとで、ループバックプロトコルモジュール
（`aidlc-common/protocols/stage-protocol-loopback.md`）の Build-and-Test 失敗ループバックによって
開始された場合、3 択の問いは提示されない（NOT presented。このループは
人間なしで実行されることを意図している）。コンダクターは Loop-Back Log の計画した修正から
決定論的に判断する: 修正が対象とするユニットには **Modify**、
それ以外のすべてのユニットには **Keep**、再入時の build-and-test 自体には **Modify**
（そこでは Redo は禁止 — Loop-Back Log が消去されてしまう）。すべての
自動判断は引き続き `aidlc-state.ts reuse-artifact <slug>
--decision <keep|modify> --artifacts "<comma-separated list of existing
artifacts found>"` によって監査される。レシートモードでは、それらの判断を、出力された各
ユニットごとの再実行の中で `unit start` と新しいレビュアー / `unit complete` の間に適用する。
成果物のみのモードでは、ゲート前のオーバーライドを通じて適用する。いずれの場合も、
再実行されたゲートが自動承認される前に、すべての対象ユニットについて現在の試行での新しいレビューが必須である。

**Gated failure loop-back（ゲート付き失敗ループバック）**: 同じオーバーライドは、
`Construction Autonomy Mode: gated`（または未設定）のもとで、ループバック
プロトコルモジュール（`aidlc-common/protocols/stage-protocol-loopback.md`）の Build-and-Test の halt-and-ask で、人間が
"Retry with fix" を選んだ場合にも適用される。成果物のみのワークフローは、
全件カバー済みの `gate: true` ディレクティブに直接到達することがあり、そこでは通常の
Artifact Re-use の問いは決して発動しない。レシートモードのワークフローは代わりに
ユニットごとの再実行ディレクティブを受け取る。高速パスでは、ゲートを提示する前に（BEFORE）、
オーバーライドを通じて計画した修正を適用する。レシートモードでは、ユニットの再実行中に
インラインで適用する。どちらも、修正が対象とするユニットには **Modify**、
それ以外のすべてのユニットには **Keep**、再入時の build-and-test 自体には **Modify** を使い
（そこでは Redo は禁止 — Loop-Back Log が消去されてしまう）、同じ
`aidlc-state.ts reuse-artifact <slug> --decision <keep|modify>
--artifacts "<comma-separated list of existing artifacts found>"` 呼び出しで監査される。それらの
判断の後、すべての対象ユニットについて宣言されたレビュアーをディスパッチし、
決着/承認ゲートを提示する前に（BEFORE）現在の試行での新しいレビューを記録する。
人間は "Retry with fix" を選んだ時点で既に確認の判断を下している。
これは、2 回目の、黙示的な自律の推定ではない。
