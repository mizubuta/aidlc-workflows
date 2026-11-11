# Swarm Protocol Module

### Before initial protected prepare

Bolt のディレクトリとブランチには、選択されたインテントの `bolt-<id8>_<slug>` という
識別子を使う。ここで `<id8>` は、ユニットのクレームと同じレジストリ UUID の接尾辞である。これにより、
並行するインテントの間で同名のユニットが分離される。ユニットの slug から再構成した名前ではなく、
出力されたパスとブランチ名を使う。レジストリ UUID なしでの作成は拒否される。Construction の前に、
インテントを採用（adopt）するか作り直す。命名と、アップグレード前のレガシー Bolt の来歴に基づく完了については、
[Bolt の識別子](../../knowledge/aidlc-shared/worktree-info-schema.md#bolt-identity)を参照する。

保護された Code Generation では、承認済みの親アプリケーションソースが、最初の
`prepare` の前にコミットされ、選択されたワークツリーのベースから再現できなければならない。これはレガシーの自律ワークフローにも
新しいチェックポイントワークフローにも適用される。自律性も緩和された Change Control も、ソースを免除しない。
未コミットのフレームワークの記録は、それ自体が無関係なファイルをコミットする理由にはならない。
この要件が対象とするのは、承認済みのアプリケーションソースのスナップショットである。

`prepare` は、最初の子ワークツリーを作成する前に、出力されたすべてのユニットについて読み取り専用の事前チェックを行う。
承認済みのアプリケーションソースが未コミットの場合は、対処可能な「コミットして再試行」のエラーを表面化する。
この拒否によって作成された子はなく、破棄すべき孤立物もない。承認済みのソースをコミットするよう人間に依頼するか、
明示的に許可された場合にのみ自分でコミットする。prepare を満たすために **決して自動的にコミットしない**。
ソースがコミットされたら、現在の Plan Approval のエビデンスで再試行する。承認済みのソースが変わった場合は、
ソースの事前チェックの対処法に従う。承認後の計画内容の編集は、Code Generation Step 3 の承認後ルールに従う。
フェンスが下げられていれば、再承認なしで続行できる。
コミットそのものが承認を与えることは決してない。

スケルトンからスウォームへの移行には、この明示的なコミットステップが含まれる。インラインのスケルトンが
統合チェックと人間のチェックポイント承認を通過した後、並列バッチを準備する前に、
承認済みのスケルトンのアプリケーションソースをコミットする。スケルトンのチェックポイントや自律性の付与を
通過したからといって、このステップを省略してはならない。

### Continuing a partially completed batch

継続時、`next` はバッチの全ユニットセットまたは残りのユニットセットを出力することがある。
現在のチェックポイントのリビジョンにネイティブな準備の記録がある場合、エンジンは
`resume_existing` を省略し、準備済みのワーカーをこの継続経路に戻す。
出力されたセットが狭くなっても、それだけで新しい計画や承認を求めることにはならない。

現在の試行ですでに準備済みの残りのユニットについては、親と、記録されたワークツリーの両方で
既存の承認を検証する:

```bash
{{INVOKE}} engine testing-posture verify --unit "<unit>"
{{INVOKE}} engine testing-posture verify --unit "<unit>" --project-dir "<recorded worktree>"
```

両方のチェックが `execution_allowed: true`（終了コード 0）を返したら、保護されたワーカーブリーフと保持された
ワークツリーで続行する。そのユニットについては最初の計画作成、Plan Approval、`prepare` をスキップし、
以下のワーカー/チェック/レビュアー/finalize のステップに従う。メンバーが 1 つしか残っていなくても、
元のグループのマニフェストとレシートは保持する。`ok: false` は、継続が許可されている状態と共存しうる。
これは現在の内容が承認されていないことを正直に示すものであり、実行が許可されている場合は
新たな承認のための停止を必要としない。
継続のメッセージには `reason` を使う。`approval_reason` は古くなったバインディングの詳細な診断であり、
実行の拒否ではない。
ディレクトリが存在するだけでは承認にはならない。検証に失敗した場合は、既存の作業を保持したまま、
指名された修復を行う必要がある。現在の準備が存在しないユニットは、
最初の計画/承認/prepare の手順に従う。

同じユニットと試行における、承認後の計画、テスト指示、Testing Contract の編集には、
Code Generation Step 3 の承認後ルールを使う。フェンスが下げられている場合は、現在のツール生成ブリーフで続行し、
編集を承認済みと付け替えることなく、元の承認エビデンスを保持する。
ワーカーに有効な plan-approval フェンスは、検証済みの稼働中の親インテントから得られる。
既存のワーカーは、次のチェックでフェンスの引き下げや引き上げを観測する。
継続が許可されるかどうかの判断に、コピーされたワーカーの設定を使ってはならない。
成果物の欠落や不正な形式の contract JSON は、自動的な新しい承認の儀式ではなく、
実行前の修復を必要とする。

以前の失敗には、引き続き halt-and-ask の Retry/Abort の判断を使う。同じ承認済みバッチを継続しても、
その人間による停止はなくならない。バッチのチェックポイントでの明示的な Request Changes には、
代わりに以下のリビジョン手順を使う。

### Resuming a reviewed batch after Request Changes

エンジンが `resume_existing: true` を出力した場合、少なくとも 1 つの保留中のユニットが、現在の却下リビジョンのために
まだ準備または回復を必要としている。まず親の `execution_allowed` を検証する。却下によって以前の承認が無効になった場合、
`plan_approval.units` は各ユニットを `status: "revise"` とともに列挙し、利用者の
チェックポイントでの言葉を `feedback` として持つ。それらの計画とテスト指示を改訂し（要求された変更を
それ自体の計画ステップとして追加する）、`next` を実行する。これにより、改訂した計画の
新たな Plan Approval が求められる。同じインテント、ユニット、試行について、現在の承認
または許可された承認後の継続がすでに存在する場合は、中断された準備を再試行するときに、その実際の承認エビデンスを保持する。
セットアップを再試行するためだけに回答を消去したり、レシートを置き換えたりしてはならない。
`execution_allowed: true` と `ok: false` の組み合わせは継続を許可するものであり、
編集された内容が承認されたことや、古い試行を復活できることを主張するものではない。

通常の `{{INVOKE}} engine swarm prepare` 呼び出しに、ディレクティブのバッチ番号と正確なユニットセットとともに
`--resume-existing` を追加する。すでに準備済みのメンバーは、ツールの冪等な経路によって保持される。
同じフラグで、次のサポートされるワークツリーの状態に対応する:

- **子がまだ存在する:** ツールはそのソースを検証して保持し、現在の実行エビデンスを再確立する前に、
  同じ却下リビジョンの古いフレームワークのメタデータをアーカイブする。拒否を回避するために、
  そのワークツリーを削除したり、通常の prepare で代用したりしてはならない。
- **ネイティブなソースの取り込みによって子が削除された:** そのユニットの以前のソースが親に到達したことを
  永続的な取り込みのエビデンスが証明している場合、ツールはすでに取り込まれた親のソースから
  新しい子をフォークできる。却下リビジョンを保持し、そのリビジョンの実際の Plan Approval をバインドする。
  古い承認を復活させたり、元の子が残っているふりをしたりはしない。
- **準備済みのリビジョンの子が明示的に破棄された:** ネイティブな破棄は、その子自身の作成と開始の後に
  行われていなければならない。ツールは、ワーカーを再作成する前に、それを以前の取り込みと現在の承認に対応づける。
  ディレクトリが存在しないことや、別の/古いワーカーの破棄だけでは不十分である。

準備が成功した後、以降の `next` は、リビジョン内のほかのユニットがすでに取り込まれている場合も含め、
別の Plan Approval なしでそれらのワーカーを継続する。

ネイティブなソースの取り込みが成功すると子が削除されることがあるため、マージ後のすべてのワークツリーが
保持されると約束してはならない。出力された再開ルートとツールの回復メッセージに従う。ソースを承認済みの
開始点と整合させられない場合は、拒否を説明し、人間が解決する間、残りの作業を保持する。
異なるソースツリーの承認を推測してはならない。

### A Unit that voids its one recovery receipt

自律的なユニットが唯一の回復レシートを無効にした場合は、`finalize` の前に停止する。
そのユニットを `--claimed` に入れず、マージせず、halt-and-ask の継ぎ目を通じて人間に Retry/Abort の判断を提示する。
Retry の場合は、メインのワークスペースに戻り、古い Bolt を中止して退避/破棄し、そのユニットについて
元の batch/base/repo 引数で、現在の `aidlc-swarm.ts prepare` ステップを再実行する。
新しいワークツリーと `BOLT_STARTED` の境界は、収束を主張することなくレビューの集計をリセットする。
決して `GATE_REJECTED` を合成しない。
破棄では、ライブのチェックアウトとブランチを削除する前に、追跡対象ファイルと無視されていない未追跡ファイル
（チェックアウトがなくなっている場合は残っているブランチの先端）と、レビュー済みのソース参照を退避する。
返された `restore_operation` が存在する場合は、古いレビューの権限を復元することなく、
別のチェックアウトに退避した作業を回復する。レビューのエビデンスしか残っていなかった場合は、
復元すべき保存済みの作業ファイルがないため、`restore_operation` も
`restore_hint` も返されない。

### Retry after a native worker discard

レビュアー枯渇時の手順が Retry を求める場合は、そのユニットについてネイティブな
中止/破棄の手順を使い、その後 `next` を呼び出す。その現在の準備ルートに従う。
未解決のチェックポイントリビジョンには `--resume-existing`、最初のバッチには通常の `prepare` を使う。
現在有効な Plan Approval を保持する。1 つのワーカーが破棄されたというだけで、グループのレシートを置き換えてはならない。

ネイティブな破棄がワーカーのコミット済みの承認済みベースラインを記録している場合、
その後ほかのユニットが親に取り込まれていても、ツールはその正確なソースと承認を復元できる。
実行中のほかのワーカーは、ソース、レシート、レビューの試行を保持する。
置き換えのワーカーには、新しいレビュアーの試行のために新しい Bolt の開始が与えられる。
そのソースのバインディングがない場合は、通常のコミット済み親の事前チェックが引き続き適用され、
必要な修復が示される。古い作成や存在しないディレクトリから承認を推測してはならない。

すべての `invoke-swarm` ディレクティブと、`directive.swarm_settled === true` を持つ
すべての `run-stage` について、このモジュールを読み込む。アクティブなハーネスのサブセクションだけを使う。

**イテレーションと承認済みユニット。** Unit-major は引き続き逐次的であり、決してスウォームを呼び出さない。
明示的な `Construction Execution: swarm` を持つ stage-major では、対象となる Code Generation のバッチが展開される前に、
skeleton-on が最初のユニットの実際の統合チェックポイントを完了して承認する。
`directive.units` をそのまま使う。チェックポイントですでに承認されたインラインのユニットは、
以降のスウォームの作業から除外される。計画作成、有効になっている要約の確認（ステージプロトコルの
`directive.ceremony.summary_confirmation === "on"` ルール）、Plan Approval は、
出力された各ユニットについて引き続き必須である。
エンジンは、計画が待機中であるバッチのすべてのユニットについて、1 つの Plan Approval の質問をする
（Construction モジュールの **Grouped Plan Approval**）。

### Batch checkpoint

`swarm_checkpoint` を持つ `run-stage` は、次のバッチの前に完了の判断を待っている完了済みのバッチである。
本体、レビュアー、通常のゲート、`swarm_settled` の処理の前に、ここで分岐する。
このオブジェクトは `batch`、`units`、`fingerprint`、`ready`、`approved`、`human_required`、`errors` を持つ。
正確なバッチ番号と、出力されたユニットをカンマで連結したものを使う。ディレクティブが `run-stage` であるというだけで、
それらを作り直したり、レビューを再実行したりしてはならない。

準備完了には、各ユニットの現在のネイティブな `SWARM_UNIT_CONVERGED` レシートが、
インテントの承認済み Construction Verification Command と一致する `Command SHA-256` を持っていることが必要である。
バッチの承認もそのダイジェストをバインドする。ダイジェストを持たない古いレシートや、新たに承認されたコマンドの場合は、
まだ承認されていないバッチを承認する前に、新たなネイティブ検証が必要になる。すでに承認されたバッチは、
利用者が新しいコマンドを承認しても承認を保持する。新しいコマンドは、まだ承認されていないバッチをチェックする。

```bash
{{INVOKE}} engine bolt swarm-checkpoint --action status --batch <N> --units "<comma-separated emitted Units>"
```

`ready` が false の場合は、指摘されたエビデンスのエラーを説明し、欠落しているソースの取り込み、検証、レビューを、
それぞれを担う手順で修復する。検証や承認をでっち上げてはならない。status が `ready: true` を報告した後でのみ、
guided/gated の完了は `swarm-checkpoint --action ask` を発行し、以下の人間との質疑応答フローを使ってよい。
`ask` は準備ができていないバッチを拒否するが、例外が 1 つある。`changed_after_check` が true の場合
（厳格な Guard Policy のもとで、バッチのファイルや出力がチェック後に変更された場合）、`ask` は利用者に
Request Changes だけを提示し、利用者がいつでもバッチを差し戻せるようにする。緩和された、またはオフの
Guard Policy のもとでは、そのような変更は代わりに保持される。バッチは準備完了のままで、`ask`
と `approve` は利用者に伝えるべき 1 行を `notices` で返す。自動完了は記録された自律ポリシーを使い、
`ask` を必要とせず、`--user-input` を省略する。

人間によるバッチのチェックポイントでは、`directive.protocol_modules` に `learnings` が列挙されている場合にのみ、
セクション 13 の学びのステップを行う。このモジュールが列挙されている場合は、バッチのユニットについて
利用者が行った修正についてのみ尋ねる（いつ尋ねるかはモジュールが定める）。利用者が残すものだけを永続化し、
その後、別の質問と別のターンとして、以下の `ask` でバッチの承認を開始する。
自動バッチでは、利用者が行った修正は、`directive.protocol_modules` に `learnings` が列挙されている場合に限り、
次の人間のチェックポイントまたは最終の引き渡しまで待つ。モジュールがない場合は、日誌をつけず、
学びの質問もしない。人間が必要な場合は、そのままバッチの承認手順に進む。
記録処理の決着が、すでに処理済みの儀式を繰り返すことは決してない。

`ready: true` を確認した後、**Approve** / **Request Changes** を提示する前に、
質問を現在のバッチのフィンガープリントと、ユニットごとの `Command SHA-256` のセットにバインドする。
コマンドは自分が実行されているセッションを自ら見つけるので、セッションを渡さず、
決してセッションを探してはならない:

```bash
{{INVOKE}} engine bolt swarm-checkpoint --action ask --batch <N> --units "<Units>"
```

その後、選択肢を提示して人間を待つ。省略していない、記録された完全なコマンドを使って "Verified with
`<full command>` (exit 0). Approve this completed batch?"（訳: 完全なコマンドで検証済み（終了コード 0）。この完了したバッチを承認しますか?）を表示する。
検証コマンドツールの出力から正規の `command` を、バッククォートを保持できる区切り文字の
コードスパンにコピーする。このチェックポイントの質問に対する人間の返答は、このチャットでも新しいチャットでも、
そこから読み取ったアクションを承認する。承認するか、変更を求められた内容とともに却下する
（利用者の言葉は記録とともに保持される。補足したい場合は `--reason` を追加する）。
返答が Request Changes を選んだだけの場合は、記録する前に何を変えるべきかを尋ねる。
新しいチャットで入力された返答は、その質問が最も新しく尋ねられた質問である間、その質問のために保持される。
そこで承認または却下のコマンドを、セッションを指定せずに実行し、決して再度尋ねない。
別の質問への返答は数えない。利用者が承認し、かつ変更を求めた場合は、承認してから変更を行い、
何を変えたかを 1 行で伝える。修正を求められた各レビュー指摘について、`approve` に
`--fix-finding "<review-artifact>#R-NN"` を追加し、却下された各指摘について、`reject` に
`--reject-finding '<review-artifact>#R-NN=<their reason>'` を追加する。
この応答は 1 回限りで、このバッチ、正確なユニットセット、現在のフィンガープリント、
ユニットごとのコマンドダイジェストのセットにバインドされる。swarm の `finalize` を再実行すると、
このインテントについて、どのセッションのものであれ、未解決のチェックポイントの質問と取得済みのチェックポイントの応答が
すべて取り下げられる。新たな検証とソースの取り込みの後、新しいディレクティブを取得し、`ready: true` を確認して、
もう一度尋ねる。`finalize` の前に取得された返答は、新しいエビデンスを承認できない。
人間による Request Changes には、自律ポリシーのもとでも、常にこの準備完了時の質疑応答フローが必要である。

```bash
# Only after a real human Approve:
{{INVOKE}} engine bolt swarm-checkpoint --action approve --batch <N> --units "<Units>" --user-input 'Approve'
# Only when human_required is false:
{{INVOKE}} engine bolt swarm-checkpoint --action approve --batch <N> --units "<Units>"
# Only after a real human Request Changes:
{{INVOKE}} engine bolt swarm-checkpoint --action reject --batch <N> --units "<Units>" --user-input 'Request Changes' --reason '<human feedback>'
```

バッチの承認または却下の後、`next` を再実行する。承認によって進むのはバッチのルーティングであり、
Code Generation ステージ全体ではない。フィンガープリントが変わった場合やアクションが拒否された場合は、
でっち上げたユーザーの回答ではなく、現在のエビデンスと別のディレクティブが必要である。
バッチの完了がどのように承認されるかにかかわらず、Plan Approval はすべてのユニットについて必須のままである。
グループ化された Plan Approval が変えるのは提示の仕方だけである。

**決着済みスウォームへの再入。** `swarm_settled: true` は、完了したユニットの本体とレビューのレシートに対する再入である。
まず Construction モジュールのメタデータルーティングを適用する。`swarm_checkpoint` には上記の
Batch checkpoint の手順を使い、`construction_checkpoint` にはユニットのチェックポイントのアクションを使う。
`construction_policy.completion_only` のディレクティブでは、本体、質問、レビュアー、学びをスキップし、
`--user-input` なしで `awaiting-approval`、続いて `approved` を報告してから、`next` を実行する。
通常の決着では、ビルダーやレビュアーを再度ディスパッチしてはならない。残りの完了の提示には
`human_completion_required` を使う。値が false の場合は、定型の人間による完了/学びの質問をスキップし、
ユーザー入力をでっち上げずに報告する。ポリシーがない場合は、レガシーの承認手順を維持する。
`directive.protocol_modules` に `learnings` が列挙されている場合にのみステージレベルの学びの儀式を行い、
その後に承認ゲートを行い、人間の結果を報告する。このルールはセッション再開後にも同様に適用される。

**finalize 後のソースの取り込み。** 各 `finalize` 呼び出しの後、`next` または人間のゲートの前に、
ステータスが `converged` で `merge_failures` に含まれない各結果行について、`bun {{HARNESS_DIR}}/tools/aidlc-worktree.ts merge --slug
<that converged result row's bolt_slug> --target <that row's landing_branch>
--strategy squash` を実行する。`landing_branch` は、prepare がそのリポジトリのメインのチェックアウトで
チェックアウトされているのを見つけたブランチであり、ユニットはフォーク元のベースが何であれそこに取り込まれる。
これを持たない行は、そこでチェックアウトされているブランチに取り込まれる。マージは、一意で永続的なソースの権威から
作成元のリポジトリを回復する。そのインテントは、選択されたワークフローのインテントのままである。
`--intent`/`--space` は、それらがセッションのアクティブなワークフローを指す場合にのみ渡す。
swarm は、変更や監査イベントの出力の前に不一致を拒否する。マージは不変の `Source Commit` を消費し、
利用者自身の git でそれを取り込み（利用者が行う任意のコミットと同様に、利用者のフック、フィルタ、マージドライバが実行される）、
`SWARM_SOURCE_MERGED` を出力する。現在の収束処理では、その行が存在するまでバッチは進まない。

利用者の git は、取り込まれる内容を変えることがある。ファイルを整形し直すフック、フィルタ、マージドライバは
利用者のものである。取り込みはその結果を保持し、行にはレビュー済みのコミット、取り込まれたコミット、差異のあるファイル
（`Landed changes`、`Driver merges`）が記録され、マージは差異ごとに 1 行を `notices` で返す
（例: "Your pre-commit hook changed 3 files while landing Unit
auth-mfa: ..."）。各行を 1 回伝えて先に進む。どの Guard Policy のもとでも、それを拒否したり、
元に戻したり、それについて尋ねたりしてはならない。コミットを拒否するフックの場合は、フック自身の `output`、
`staged` のパス、`next` のステップとともに `status: "commit-refused"` が返され、
メインのチェックアウトは git が残したとおりの状態で、取り込みはステージされたままになる。
利用者のフックがそのユニットの取り込みを拒否したことと、その理由を、`output` フィールドの言葉で 1 行で伝える。
ステージされた取り込みの中で指摘された点を修正し（メインのチェックアウトでファイルを編集して `git add` するか、
コミットメッセージのルールであれば `--message` を渡す）、同じマージを再度実行する。マージは
ステージされた取り込みを完了し、修正を `Landed changes` に記録する。修正できない場合は停止し、
チェックアウトを git が残したままにして、`staged` が列挙するものを伝える。

それより前に取り込まれたユニットと衝突する取り込みは、git がマージできなかったファイル（`conflict_files`）と
`next` のステップとともに `status:
"conflict"` を返し、取り込みをメインのチェックアウトにステージしたままにする。
各ファイルで両側を統合し、各ユニットがその承認済みの計画が求めたものを保持するようにする。一方のユニットの側を
落としてはならない。バッチが進行中の間、それらはメインのチェックアウトで書き込みが許される唯一のファイルである。
その後それらを `git add` し（削除されたままにするものは `git rm`）、同じマージを再度実行する。マージは取り込みを完了し、
統合したファイルを示す通知を 1 つ返すので、それを 1 回伝える。2 つの計画がそこで異なるものを求めている場合は、
停止して、どのファイルが衝突しているかと、そのチェックアウトで `git reset --merge` を実行すれば取り込みが取り消されることを伝える。
それは利用者が求めた場合にのみ実行する。変更対象のファイルに、メインのチェックアウトで未コミットの変更
（たとえば、利用者がまだコミットしていない AI-DLC による `.gitignore` の更新）があるために git が開始しなかった取り込みは、
それらのファイルとステップを示す。それを 1 行で伝え、利用者の指示があった場合にのみその変更をコミットしてから、
同じマージを再度実行する。

`[merge-succeeded:<sha>]` より前のその他のゼロ以外の結果は、ワークツリーを保持し、
それ自身のステップを示す。それに従い、`finalize` を再実行せずに同じマージを再度実行する。
ステップが示されない場合は停止し、チェックアウトをそのままにして 1 行で伝える。メッセージに
`[merge-succeeded:<sha>]` が含まれ、`SWARM_SOURCE_MERGED` が存在する場合、レビュー済みのソースは
すでに権威あるものとなっており、失敗したのはクリーンアップだけである。同じマージを再実行すると、
ソースを再適用したり権威を重複させたりすることなく、クリーンアップのみの整合処理が行われる。
マーカーは存在するが `SWARM_SOURCE_MERGED` が存在しない場合（取り込み後に記録を書き込めなかった場合）は、
マージを再試行してはならない。ワークツリーを保持し、示されたステージの再開、または人間が明示的に承認したバイパスの
対処法に従う。

Bolt のブランチが別のワークツリーのパスでチェックアウトされている場合も、クリーンアップは拒否され、
そのブランチと保持/退避された参照は保持される。所有者のパスを表面化する。
別のインテントの同名のユニットを、このバッチのクリーンアップとして扱ってはならない。

**回復可能な中止/破棄。** 以下のすべてのハーネスの回復経路において、古い Bolt の中止と破棄とは、
退避/破棄を意味する。ライブのチェックアウトとブランチを削除する前に、追跡対象ファイルと無視されていない未追跡ファイルの
スナップショットを取り（チェックアウトがなくなっている場合は残っているブランチの先端を保持し）、
レビュー済みのソース参照を退避する。コンダクターは人間の選択を得て、返された回復コマンドを
変更せずに実行しなければならない。返された `restore_operation` が存在する場合は、
`restore/bolt-<id8>_<slug>-<stamp>` 上の隔離された `.aidlc/restored/bolt-<id8>_<slug>-<stamp>` チェックアウトに
退避した作業を回復し、新しいライブの Bolt を決して上書きしない。保存された引数は正確な slug、スタンプ、
リポジトリを選択し、その後に `--intent <record-dir-name> --space <space>` を追加するので、
以降の実行が別のアクティブなインテントにずれることはない。レビューのエビデンスしか残っていなかった場合は、
復元すべき保存済みの作業ファイルがないため、`restore_operation` も `restore_hint` も返されない。
名前空間付きの復元とレガシーの復元はどちらも、選択されたインテントの正確な `WORKTREE_DISCARDED` の `Parked ref`
とスタンプの来歴を必要とする。レガシーの復元はレガシーの名前を保持する。復元された成果物とレシートは、
現在の試行のエビデンスではない。再試行には、以下で説明するとおり、引き続き新たな `prepare` と
レビューの境界が必要である。

### 12b. Autonomous Code Generation Plan Contract

`code-generation` に対する `invoke-swarm` ディレクティブが変えるのは、生成が実行される場所であり、
計画作成と Plan Approval が行われるかどうかではない。`aidlc-swarm.ts
prepare` の前に:

すでに準備済みの部分的なバッチを継続するセッションでは、まずこのモジュールの
**Continuing a partially completed batch** ルールを適用する。有効な残りのワーカーは
元の計画と承認を保持し、ステップ 4 の保護されたブリーフに進む。それらの質問をリセットしたり、
最初の準備を再度実行したりしてはならない。

最初の承認の後、同じインテント、ユニット、試行に対する計画、テスト指示、Testing Contract の編集は、
Code Generation Step 3 の承認後ルールに従う。`plan-approval` フェンスが下げられている場合、ビルドは
編集されたファイルで続行する。フェンスがオンの場合、`next` は利用者に再度尋ねる。ほかのコードが動いても、
再度尋ねることはない。これはグループの承認済みメンバーにも適用される。最初の承認、ソースの再現性、
新しい試行の承認、完了ゲートは変わらない。成果物の欠落や不正な形式の contract JSON は、
実行前に修復しなければならない。`next` が修復内容を示す。

1. `directive.plan_approval.status` が `plan` の間、それが列挙する各ユニットについて、
   メインのワークスペースで Code Generation Part 1 を書く。計画（`## Summary` を含む）、
   埋め込みの `## Testing Contract`、テスト指示であり、Code Generation Step 3 の説明どおり、
   各エントリの `status`、`note`、`feedback` に従って作業する。
   その後 `next` を実行する。質問ファイルやフィンガープリントを書いてはならない。エンジンが尋ねる。
2. 列挙されたすべての計画の準備ができると、`next` は、まだ待機中のすべてのユニットについて 1 つの `plan-approval` の質問を返す
   （Construction モジュールの **Grouped Plan Approval** を参照）。それを表示してターンを終え、
   利用者が回答した後に `next` を実行する。利用者が承認したユニットは完了である。
   差し戻されたユニットは、利用者の言葉とともに `plan_approval.units` に戻ってくる。
   計画作成中にワークツリーをフォークしたり、実装ワーカーをディスパッチしたりしてはならない。
3. `next` が `plan_approval.status: "approved"` を持つこの `invoke-swarm` を返した場合にのみ
   `prepare` を呼び出す。最初の保護された prepare の前に、承認済みの親アプリケーションソースは、
   以降の並列バッチの前のインラインスケルトンのソースも含めて、コミットされ再現可能でなければならない。
   このモジュールの **Before initial protected prepare** ルールは、レガシーの自律性にも
   新しいチェックポイントにも同様に適用される。子を作成する前にバッチ全体を事前チェックし、
   決して自動的にコミットしない。スウォームの Code Generation では、`prepare` はワークツリーを作成する前に、
   計画、テスト指示、埋め込みのコントラクト、回答、ターゲットにバインドされたフィンガープリント、
   現在のステージの試行、計画されたソース、人間が所有するレシートを検証する。
   メモリの Testing Posture、スコープ、テスト戦略、プロジェクトタイプの入力が変わった場合は、
   必要に応じて現在のコントラクトと指示を更新し、同じ承認後ルールを適用する。
   フェンスが下げられていれば、再承認なしで続行できる。同じインテント、ユニット、試行について
   `next` を再実行しても、承認が再度開かれることはない。
4. すべてのワーカーブリーフは、
   `bun {{HARNESS_DIR}}/tools/aidlc-testing-posture.ts brief --unit <unit>` の出力から、
   一字一句編集せずに始まる。その出力は、正確に次の内容で始まる:

   ```text
   AIDLC-UNIT: <unit>
   AIDLC-TESTING-CONTRACT: <contract_sha256 from that unit's current plan>
   ```

   そして、承認内容の射影（終端の `## Review` 付録より前のすべての行で、その付録は一切含まない。
   タスクマーカーは `[ ]` にリセットされ、空白は正規化される。再生された計画には、
   以前のプロトコルで記録されたレビューの付録がまだ含まれている場合がある）を使った現在の計画と、
   現在の `unit-test-instructions.md` をバイト単位でそのまま含む。どちらのマーカー行も
   自分で書いてはならず、計画ファイルを決してブリーフに読み込んではならない。
   フィンガープリントは付録を除外するため、そのバイト列は実行すべき作業ではない。
   フェンスがオンの場合、plan-approval ガードはそれらを引用した引き渡しを拒否する。
   現在の承認または許可された承認後の継続には、ツールが生成したブリーフを使う。
   でっち上げた承認で代用してはならない。ワーカーへのその他のコンテキストは、コマンドの出力の後に続ける。
   ワーカーは、自分のワークツリー内の計画ファイルで自分の進捗を読み、チェックを付ける。
   ワーカーは、Bolt 内のレビューの前に、ブリーフが指定する記録フォルダにユニットの
   `source-manifest.json` を作成し、作成、変更、削除するすべてのアプリケーションソースのパスを
   列挙しなければならない。Bolt は選択された単一のリポジトリであるため、親インテントが複数のリポジトリを
   記録している場合でも、これらのパスはワークツリーからの相対パスであり、`repo` を省略する。
   現在のブリーフにある Testing Contract が権威を持つ。ワーカーはメモリを再解決せず、
   再試行には現在のツール生成ブリーフを使う。フェンスがオンの場合、plan-approval ガードは、
   マーカーが欠けている、古い、または計画と異なる委譲されたワーカーを拒否する。
   フックを実行できないヘッドレスのワーカーハーネスも、`prepare` とこの必須の
   ブリーフコントラクトによって引き続き保護される。

4 つの義務がすべて満たされた後でのみ、通常のスウォームの
prepare/fan-out/check/review/finalize ループが実行される。

### Claude Code

エンジンは、対象となる Construction バッチをスウォーム実行に選択した。新しいワークフローでは実行方式が明示されており、gated または autonomous の完了承認を使える。実行フィールドを持たないレガシーのワークフローは、自律性に基づくルートを維持する。**あなた（稼働中の `/aidlc` セッション）がコンダクターである。展開と再試行ループはあなたが所有する。`aidlc-swarm.ts` はあなたが参照する決定論的な審判であり、決してループの所有者ではない。** **ステップ (1) の前に、上記の §12b "Autonomous Code Generation Plan Contract" を読んで従う。計画作成、エンジンの Plan Approval の質問、2 つのワーカーブリーフのマーカーは、出力されたすべてのユニットについて必須である。** 以下のステップの前に、上記の **Continuing a partially completed batch**、**Before initial protected prepare**、**Resuming a reviewed batch after Request Changes** を適用して、継続か準備かを選択する。自律性の付与によって自動コミットが許可されることはない。 (1) バッチを **`prepare`** する: `{{INVOKE}} engine swarm prepare --batch <n> --units <directive.units joined by comma> [--base <branch>] [--repo <name>]` は、バッチの事前チェックの後でのみ、最初の隔離されたワークツリーを作成する（`--base` がない場合は、チェックアウトされているブランチからフォークする）。`--resume-existing` を付ける場合は、上記の 2 つの再開経路を使う。ディレクティブに `repo` フィールドがある場合は、`--repo` にその値を渡す。ディレクティブが `repo` を省略している MULTI-REPO のインテントでは、このバッチが対象とする兄弟リポジトリについて `--repo <name>` を指定する（記録されたセットは `/aidlc intent --json`.repos から読む）。マルチリポジトリのインテントでは、これがないと `prepare` はエラーになる。 (2) **`AIDLC_USE_SWARM` に従って展開する:** 未設定 / `"1"` 以外 → 最低限の方式。1 つのアシスタントメッセージで N 個の並列 `Task` 呼び出しをユニットごとに 1 つずつ発行し、それぞれがプロジェクトの収束チェックに合格するまで、自分のワークツリーでユニットを実装する。`="1"` → インラインの Dynamic Workflow（`Workflow({script, args})`、バッチは `args` に入れる）を作成し、その JS がユニットごとの `pipeline` と反復回数の上限を所有する。`="1"` なのに Workflow ツールが利用できない場合は、**はっきり告知したうえで最低限の方式に降格し**、次の審判呼び出しで `--degraded-from ultracode` を渡して、ツールが `SWARM_DEGRADED` を出力するようにする。 (3) `check` と `finalize` はどちらも、チェックポイントの有無にかかわらず、インテントの記録された、人間が承認した **Construction Verification Command** を使う。`--check-cmd` は省略する（指定する場合は一致しなければならない）。承認がない場合は、どちらかを実行する前に、チェックポイントモジュールの検証コマンドの手順と `set-construction-verification-command` を完了する。これは利用者に 1 回だけ尋ねる。各ユニットのワーカーのターンの後、**`check <unit> [--test-file <protected spec>]`** を参照する。終了コード `0` = 本当に収束した（実際のチェックに合格し、保護されたファイルは改ざんされていない。緩和された、またはオフの Guard Policy のもとでは、変更された保護ファイルは保持され、その 1 行が `change_notices` で返されるので、利用者にその行を伝える）。ゼロ以外 = まだであり、再試行するかエスカレーションするかはあなたが判断する（ナレッジ）。 (4) ループが落ち着いたら、**`finalize --batch <n> --units <all> --claimed <the units you believe converged> [--reasons <unit>=<unsatisfiable|budget-exhausted|cap-exhausted>,…]`** は、マージの前にクレームされたすべてのユニットを再検証し（誤ってクレームしたユニットは拒否される。嘘をつくコンダクターへのガード）、本当に合格したものを直列にマージする。クレームしなかったユニットについては、`--reasons` で *なぜ* 断念したかを帰属させる（あなたのナレッジによる判断。根本的に構築不可能な場合は `unsatisfiable`、ultracode のトークン上限で止まった場合は `budget-exhausted`。列挙されていない断念ユニットはデフォルトで `cap-exhausted` になる）。ツールはあなたの帰属を忠実に記録するが、クレームされたのに失敗したユニットの `error` 判定をそれで上書きさせることは決してない。 **`finalize` の終了コードで分岐する:** `0` → このバッチのレビュー済みの記録エビデンスとメタデータが収束した。上記の必須のソース取り込みステップの後、まだステージを報告せずに `next` を再実行する。エンジンはまず、完了したバッチについて `swarm_checkpoint` を返すことがある。次のバッチの前にそれを解決する。そうでなければ、次の `invoke-swarm` か最終の `run-stage` の決着ディレクティブを返す。その決着ディレクティブの場合にのみ、上記のメタデータを考慮した決着済みスウォームのルールを適用し、そのライフサイクルの結果を報告する。途中のバッチの後に承認済みと報告すると、後続のバッチが構築されないままステージが完了してしまう。 `2` → 失敗のエンベロープを返す（ユニットが充足不能、クレームされたのに失敗、改ざん、またはマージの失敗）。**主導権を取り戻す**: 停止し、halt-and-ask の継ぎ目を通じて人間を再び関与させる（`aidlc-common/protocols/stage-protocol-construction.md` § "Halt-and-ask on failure"。失敗時は自律モードにかかわらず常に停止して尋ねる）。 `merge_failures` のユニット（収束したがマージバックに失敗したもの。マージが成功するまで `SWARM_UNIT_CONVERGED` の行は記録されない）については、阻害要因を解決し、そのユニットに範囲を絞って `finalize` を再実行する。ワークツリーは保持されており、`release-merge` は冪等なので、再試行は純粋な再呼び出しである。スウォームが決してコンダクターの手を離れることはない。審判は判定、マージ、監査を所有し、あなたは展開と再試行の判断を所有する。 *（任意: 人間は自律性の付与時に `/goal` と入力して、審判のトランスクリプト出力をキーとする条件まで実行させることができる。ただし収束の判定者としてではない。判定者は引き続き `finalize` の終了コードである。）*

**自律的なレビュアーの境界。** `invoke-swarm` が `directive.reviewer` を持つ場合、`check` に合格したというだけでは、そのユニットを `finalize` でクレームすることはできない。そのユニットの `prepare` で作成されたワークツリーで、レビュアーのコントラクト（stage-protocol-reviewer.md §12a）に従う。`{{INVOKE}} engine log review --stage "<directive.stage>" --unit "<unit>" --reviewer "<directive.reviewer>" --iteration <n> --project-dir "<worktree>"` で `REVIEW_REQUESTED` を記録し、`directive.stage_file` とそのワークツリーのユニットの成果物およびコントラクトに対してレビュアーをディスパッチし、同じコマンドに `--verdict <READY|NOT-READY>` を加えて `REVIEW_COMPLETED` を記録する。`--project-dir` がワークツリーを対象とする間、ロガーはメインのワークスペースにとどまる。これは、マルチリポジトリのワークツリーが選択された兄弟リポジトリだけを含む場合にも機能する。NOT-READY の判定では、同じワークツリーでリードを再度呼び出し、収束チェックを再実行し、`directive.reviewer_max_iterations` までレビュアーを繰り返す。唯一の回復レシートが再び無効になった場合は、そのユニットを `--claimed` に入れず、`finalize` も実行しない。人間による Retry/Abort の判断のために停止する。Retry の場合は、メインのワークスペースに戻り、古い Bolt を中止して破棄してから `next` を呼び出し、元の batch/repo の選択で、そのユニットについて上記の **Retry after a native worker discard** に従う。新しい `BOLT_STARTED` の境界は、収束を主張することなくレビューの集計をリセットする。ユニットを `--claimed` に入れるのは、終端のレビューレシートが存在した後だけにする。`finalize` はマージの前にレシートを検証し、その後それをメインの監査にマージする。すべてのユニットが収束した後に `next` が決着の `run-stage` を返した場合は、レビュアーを再度ディスパッチしてはならない。ユニットごとのレシートがすでにステージをカバーしているので、上記のメタデータを考慮した決着済みスウォームの完了手順だけを適用する。これは自律的な Construction の中でのモデルの作業であり、人間への別のプロンプトではない。

**再試行の破棄が成功した後。** `--discard` による中止が成功し、古い試行が退避されたことが
確認された後、`prepare` を再実行する前に、この SAY 行を使う。`On your go-ahead I` は、
人間が Retry を選択した場合にのみ使う。それ以外の場合は `I` を使い、行われていない人間による対処の選択を
ほのめかしてはならない。中止が失敗した場合や試行が退避されなかった場合は、保存されたスナップショットを告知してはならない。

返された `parked_mode` から `[saved-files text]` を選択する:

- `snapshot`: "I saved a snapshot of its tracked files and non-ignored untracked files. Ignored files are not saved, and the snapshot may normalize line endings."（訳: 追跡対象ファイルと無視されていない未追跡ファイルのスナップショットを保存しました。無視されたファイルは保存されず、スナップショットでは改行コードが正規化される場合があります。）
- `branch-tip`: "I kept its committed work; there were no uncommitted files to save."（訳: コミット済みの作業を保持しました。保存すべき未コミットのファイルはありませんでした。）
- `evidence-only`: "Nothing of its working files remained to save; only its review evidence was kept."（訳: 保存すべき作業ファイルは何も残っていませんでした。レビューのエビデンスだけを保持しました。）
- `null`: `[saved-files text]` を省略する。フォールバックの記述子は、何が保存されたかを確定しない。

**SAY:** "[On your go-ahead I|I] set aside the previous attempt at [Unit] because the work changed again after its re-check, and I'm starting a new attempt. [saved-files text] If you want the previous attempt back, ask me to restore it."（訳: 再チェック後に作業が再び変更されたため、[Unit] の以前の試行は脇に置き、新しい試行を開始します。[saved-files text] 以前の試行を戻したい場合は、復元するよう依頼してください。）

`restore_operation` がない場合は、最後の申し出 "If you want the previous attempt back, ask me to restore it." を省略する。エビデンスのみの試行について、復元操作をでっち上げてはならない。

人間が後でその試行を戻すよう求めた場合は、保存された中止結果の
`restore_operation` を使う。その `worktree` ルートを
`{{INVOKE}} engine worktree <args...>` で呼び出し、列挙された各 `args` 要素を、正確にそのまま
個別の argv 引数として渡す。それらの引数を連結してシェルコマンドにしたり、
slug だけの選択を作り直したりしてはならない。`restore_hint` は人間向けの表示テキストにすぎず、
決して実行の入力ではない。安全なレンダリングに失敗した場合（たとえば、ハーネスの
ディレクトリが無効な場合）は、ヒントは省略され、`restore_hint_error` がその理由を説明する。
操作は引き続き利用でき、復元の申し出も引き続き適用される。
復元が成功した後、返された復元先のパスを平易に告知する:
**SAY:** "I restored the previous attempt at [returned restored path]."（訳: 以前の試行を [returned restored path] に復元しました。）
復元によって古い試行が再開されたり、そのレビューが現在のものになったりすることはない。

---

### Kiro CLI

エンジンは、対象となる Construction バッチをスウォーム実行に選択した。新しいワークフローでは実行方式が明示されており、gated または autonomous の完了承認を使える。実行フィールドを持たないレガシーのワークフローは、自律性に基づくルートを維持する。**あなた（稼働中の `/aidlc` セッション）がコンダクターである。展開と再試行ループはあなたが所有する。`aidlc-swarm.ts` はあなたが参照する決定論的な審判であり、決してループの所有者ではない。** **ステップ (1) の前に、上記の §12b "Autonomous Code Generation Plan Contract" を読んで従う。計画作成、エンジンの Plan Approval の質問、2 つのワーカーブリーフのマーカーは、出力されたすべてのユニットについて必須である。** 以下のステップの前に、上記の **Continuing a partially completed batch**、**Before initial protected prepare**、**Resuming a reviewed batch after Request Changes** を適用して、継続か準備かを選択する。自律性の付与によって自動コミットが許可されることはない。 (1) バッチを **`prepare`** する: `{{INVOKE}} engine swarm prepare --batch <n> --units <directive.units joined by comma> [--base <branch>] [--repo <name>]` は、バッチの事前チェックの後でのみ、最初の隔離されたワークツリーを作成する（`--base` がない場合は、チェックアウトされているブランチからフォークする）。`--resume-existing` を付ける場合は、上記の 2 つの再開経路を使う。ディレクティブに `repo` フィールドがある場合は、`--repo` にその値を渡す。ディレクティブが `repo` を省略している MULTI-REPO のインテントでは、このバッチが対象とする兄弟リポジトリについて `--repo <name>` を指定する（記録されたセットは `/aidlc intent --json`.repos から読む）。マルチリポジトリのインテントでは、これがないと `prepare` はエラーになる。 (2) **`subagent` ツールで展開する**: バッチ内のすべてのユニットを 1 回の委譲でまとめて委譲する（バッチ全体でかまわない。同時実行数はハーネスのキューイングの関心事である）。ユニットごとに `aidlc-developer-agent` を対象とする並列タスクを 1 つずつ用意し、それぞれがプロジェクトの収束チェックに合格するまで、自分のワークツリーでユニットを実装する。このハーネスでは、サブエージェントによる展開が唯一のスウォームモードである。`AIDLC_USE_SWARM=1` はここでは効果がない（Workflow ツールは存在しない）。設定されている場合は、そのことを声に出して伝えたうえで展開を進め、次の審判呼び出しで `--degraded-from ultracode` を渡して、ツールが `SWARM_DEGRADED` を出力するようにする。 (3) `check` と `finalize` はどちらも、チェックポイントの有無にかかわらず、インテントの記録された、人間が承認した **Construction Verification Command** を使う。`--check-cmd` は省略する（指定する場合は一致しなければならない）。承認がない場合は、どちらかを実行する前に、チェックポイントモジュールの検証コマンドの手順と `set-construction-verification-command` を完了する。これは利用者に 1 回だけ尋ねる。各ユニットのワーカーのターンの後、**`check <unit> [--test-file <protected spec>]`** を参照する。終了コード `0` = 本当に収束した。ゼロ以外 = まだであり、再試行するかエスカレーションするかはあなたが判断する。 (4) ループが落ち着いたら、**`finalize --batch <n> --units <all> --claimed <the units you believe converged> [--reasons <unit>=<unsatisfiable|budget-exhausted|cap-exhausted>,…]`** は、マージの前にクレームされたすべてのユニットを再検証し（嘘をつくコンダクターへのガード）、本当に合格したものを直列にマージする。 **`finalize` の終了コードで分岐する:** `0` → このバッチのレビュー済みの記録エビデンスとメタデータが収束した。上記の必須のソース取り込みステップの後、まだステージを報告せずに `next` を再実行する。エンジンはまず、完了したバッチについて `swarm_checkpoint` を返すことがある。次のバッチの前にそれを解決する。そうでなければ、次の `invoke-swarm` か最終の `run-stage` の決着ディレクティブを返す。その決着ディレクティブの場合にのみ、上記のメタデータを考慮した決着済みスウォームのルールを適用し、そのライフサイクルの結果を報告する。途中のバッチの後に承認済みと報告すると、後続のバッチが構築されないままステージが完了してしまう。 `2` → 失敗のエンベロープ。**主導権を取り戻す**: 停止し、halt-and-ask の継ぎ目を通じて人間を再び関与させる（`aidlc-common/protocols/stage-protocol-construction.md` § "Halt-and-ask on failure"）。 `merge_failures` のユニット（収束したがマージバックに失敗したもの。マージが成功するまで `SWARM_UNIT_CONVERGED` の行は記録されない）については、阻害要因を解決し、そのユニットに範囲を絞って `finalize` を再実行する。ワークツリーは保持されており、`release-merge` は冪等なので、再試行は純粋な再呼び出しである。そのために `prepare` を再実行してはならない。スウォームが決してコンダクターの手を離れることはない。

**自律的なレビュアーの境界。** `invoke-swarm` が `directive.reviewer` を持つ場合、`check` に合格したというだけでは、そのユニットを `finalize` でクレームすることはできない。そのユニットの `prepare` で作成されたワークツリーで、stage-protocol-reviewer.md §12a に従う。`{{INVOKE}} engine log review --stage "<directive.stage>" --unit "<unit>" --reviewer "<directive.reviewer>" --iteration <n> --project-dir "<worktree>"` で `REVIEW_REQUESTED` を記録し、`directive.stage_file` とそのワークツリーのユニットの成果物およびコントラクトに対してレビュアーエージェントに委譲し、同じコマンドに `--verdict <READY|NOT-READY>` を加えて `REVIEW_COMPLETED` を記録する。`--project-dir` がワークツリーを対象とする間、ロガーはメインのワークスペースにとどまる。これは、マルチリポジトリのワークツリーが選択された兄弟リポジトリだけを含む場合にも機能する。NOT-READY の判定では、同じワークツリーでリードを再度呼び出し、収束チェックを再実行し、`directive.reviewer_max_iterations` までレビュアーを繰り返す。唯一の回復レシートが再び無効になった場合は、そのユニットを `--claimed` に入れず、`finalize` も実行しない。人間による Retry/Abort の判断のために停止する。Retry の場合は、メインのワークスペースに戻り、古い Bolt を中止して破棄してから `next` を呼び出し、元の batch/repo の選択で、そのユニットについて上記の **Retry after a native worker discard** に従う。新しい `BOLT_STARTED` の境界は、収束を主張することなくレビューの集計をリセットする。ユニットを `--claimed` に入れるのは、終端のレビューレシートが存在した後だけにする。`finalize` はマージの前にレシートを検証し、その後それをメインの監査にマージする。すべてのユニットが収束した後に `next` が決着の `run-stage` を返した場合は、レビュアーに再度委譲してはならない。ユニットごとのレシートがすでにステージをカバーしているので、上記のメタデータを考慮した決着済みスウォームの完了手順だけを適用する。これは自律的な Construction の中でのモデルの作業であり、人間への別のプロンプトではない。

**再試行の破棄が成功した後。** `--discard` による中止が成功し、古い試行が退避されたことが
確認された後、`prepare` を再実行する前に、この SAY 行を使う。`On your go-ahead I` は、
人間が Retry を選択した場合にのみ使う。それ以外の場合は `I` を使い、行われていない人間による対処の選択を
ほのめかしてはならない。中止が失敗した場合や試行が退避されなかった場合は、保存されたスナップショットを告知してはならない。

返された `parked_mode` から `[saved-files text]` を選択する:

- `snapshot`: "I saved a snapshot of its tracked files and non-ignored untracked files. Ignored files are not saved, and the snapshot may normalize line endings."（訳: 追跡対象ファイルと無視されていない未追跡ファイルのスナップショットを保存しました。無視されたファイルは保存されず、スナップショットでは改行コードが正規化される場合があります。）
- `branch-tip`: "I kept its committed work; there were no uncommitted files to save."（訳: コミット済みの作業を保持しました。保存すべき未コミットのファイルはありませんでした。）
- `evidence-only`: "Nothing of its working files remained to save; only its review evidence was kept."（訳: 保存すべき作業ファイルは何も残っていませんでした。レビューのエビデンスだけを保持しました。）
- `null`: `[saved-files text]` を省略する。フォールバックの記述子は、何が保存されたかを確定しない。

**SAY:** "[On your go-ahead I|I] set aside the previous attempt at [Unit] because the work changed again after its re-check, and I'm starting a new attempt. [saved-files text] If you want the previous attempt back, ask me to restore it."（訳: 再チェック後に作業が再び変更されたため、[Unit] の以前の試行は脇に置き、新しい試行を開始します。[saved-files text] 以前の試行を戻したい場合は、復元するよう依頼してください。）

`restore_operation` がない場合は、最後の申し出 "If you want the previous attempt back, ask me to restore it." を省略する。エビデンスのみの試行について、復元操作をでっち上げてはならない。

人間が後でその試行を戻すよう求めた場合は、保存された中止結果の
`restore_operation` を使う。その `worktree` ルートを
`{{INVOKE}} engine worktree <args...>` で呼び出し、列挙された各 `args` 要素を、正確にそのまま
個別の argv 引数として渡す。それらの引数を連結してシェルコマンドにしたり、
slug だけの選択を作り直したりしてはならない。`restore_hint` は人間向けの表示テキストにすぎず、
決して実行の入力ではない。安全なレンダリングに失敗した場合（たとえば、ハーネスの
ディレクトリが無効な場合）は、ヒントは省略され、`restore_hint_error` がその理由を説明する。
操作は引き続き利用でき、復元の申し出も引き続き適用される。
復元が成功した後、返された復元先のパスを平易に告知する:
**SAY:** "I restored the previous attempt at [returned restored path]."（訳: 以前の試行を [returned restored path] に復元しました。）
復元によって古い試行が再開されたり、そのレビューが現在のものになったりすることはない。

---

### Kiro IDE

エンジンは、対象となる Construction バッチをスウォーム実行に選択した。新しいワークフローでは実行方式が明示されており、gated または autonomous の完了承認を使える。実行フィールドを持たないレガシーのワークフローは、自律性に基づくルートを維持する。**あなた（稼働中の `/aidlc` セッション）がコンダクターである。展開と再試行ループはあなたが所有する。`aidlc-swarm.ts` はあなたが参照する決定論的な審判であり、決してループの所有者ではない。** **ステップ (1) の前に、上記の §12b "Autonomous Code Generation Plan Contract" を読んで従う。計画作成、エンジンの Plan Approval の質問、2 つのワーカーブリーフのマーカーは、出力されたすべてのユニットについて必須である。** 以下のステップの前に、上記の **Continuing a partially completed batch**、**Before initial protected prepare**、**Resuming a reviewed batch after Request Changes** を適用して、継続か準備かを選択する。自律性の付与によって自動コミットが許可されることはない。 `directive.verification_command_authorized` が `false` で `directive.plan_approval.status` が `approved` の場合、インテントにはまだ承認済みの **Construction Verification Command** がない。バッチの最初の `prepare` を実行しようとするとき（上記の継続または再開の経路ではない場合）は、ワーカーブリーフとその `prepare` より前に、チェックポイントモジュールの検証コマンドの手順を完了し（コマンドに対する現在の承認レシートがすでに存在する場合は、再度尋ねずにセッターを実行する）、まず `next` を再実行する。利用者がコマンドを承認したのに `next` がまだ `false` を返す場合、または利用者がコマンドを後で選ぶことを求めた場合は、再度尋ねずにステップ (1) に進む。`check` に到達したときはステップ (3) が適用される。Kiro IDE でも Kiro CLI v3 でも同様に、ここで尋ねる。そうすれば各ワーカーのブリーフが承認済みのテストコマンドを示し、回答がワーカーとそのレビュアーの間に落ちることがなくなる。Kiro CLI v3 では、そこで尋ねると新しいチャットターンが始まり、利用者がすべてのサブエージェントに対して承認済みのリクエストについて、レビュアーが再度尋ねてしまう。 (1) バッチを **`prepare`** する: `{{INVOKE}} engine swarm prepare --batch <n> --units <directive.units joined by comma> [--base <branch>] [--repo <name>]` は、バッチの事前チェックの後でのみ、最初の隔離されたワークツリーを作成する（`--base` がない場合は、チェックアウトされているブランチからフォークする）。`--resume-existing` を付ける場合は、上記の 2 つの再開経路を使う。ディレクティブに `repo` フィールドがある場合は、`--repo` にその値を渡す。ディレクティブが `repo` を省略している MULTI-REPO のインテントでは、このバッチが対象とする兄弟リポジトリについて `--repo <name>` を指定する（記録されたセットは `/aidlc intent --json`.repos から読む）。マルチリポジトリのインテントでは、これがないと `prepare` はエラーになる。 (2) **`subagent` ツールで展開する**: バッチ内のすべてのユニットを 1 回の委譲でまとめて委譲する（バッチ全体でかまわない。同時実行数はハーネスのキューイングの関心事である）。ユニットごとに `aidlc-developer-agent` を対象とする並列タスクを 1 つずつ用意し、それぞれがプロジェクトの収束チェックに合格するまで、自分のワークツリーでユニットを実装する。このハーネスでは、サブエージェントによる展開が唯一のスウォームモードである。`AIDLC_USE_SWARM=1` はここでは効果がない（Workflow ツールは存在しない）。設定されている場合は、そのことを声に出して伝えたうえで展開を進め、次の審判呼び出しで `--degraded-from ultracode` を渡して、ツールが `SWARM_DEGRADED` を出力するようにする。 (3) `check` と `finalize` はどちらも、チェックポイントの有無にかかわらず、インテントの記録された、人間が承認した **Construction Verification Command** を使う。`--check-cmd` は省略する（指定する場合は一致しなければならない）。承認がない場合は、どちらかを実行する前に、チェックポイントモジュールの検証コマンドの手順と `set-construction-verification-command` を完了する。これは利用者に 1 回だけ尋ねる。各ユニットのワーカーのターンの後、**`check <unit> [--test-file <protected spec>]`** を参照する。終了コード `0` = 本当に収束した。ゼロ以外 = まだであり、再試行するかエスカレーションするかはあなたが判断する。 (4) ループが落ち着いたら、**`finalize --batch <n> --units <all> --claimed <the units you believe converged> [--reasons <unit>=<unsatisfiable|budget-exhausted|cap-exhausted>,…]`** は、マージの前にクレームされたすべてのユニットを再検証し（嘘をつくコンダクターへのガード）、本当に合格したものを直列にマージする。 **`finalize` の終了コードで分岐する:** `0` → このバッチのレビュー済みの記録エビデンスとメタデータが収束した。上記の必須のソース取り込みステップの後、まだステージを報告せずに `next` を再実行する。エンジンはまず、完了したバッチについて `swarm_checkpoint` を返すことがある。次のバッチの前にそれを解決する。そうでなければ、次の `invoke-swarm` か最終の `run-stage` の決着ディレクティブを返す。その決着ディレクティブの場合にのみ、上記のメタデータを考慮した決着済みスウォームのルールを適用し、そのライフサイクルの結果を報告する。途中のバッチの後に承認済みと報告すると、後続のバッチが構築されないままステージが完了してしまう。 `2` → 失敗のエンベロープ。**主導権を取り戻す**: 停止し、halt-and-ask の継ぎ目を通じて人間を再び関与させる（`aidlc-common/protocols/stage-protocol-construction.md` § "Halt-and-ask on failure"）。 `merge_failures` のユニット（収束したがマージバックに失敗したもの。マージが成功するまで `SWARM_UNIT_CONVERGED` の行は記録されない）については、阻害要因を解決し、そのユニットに範囲を絞って `finalize` を再実行する。ワークツリーは保持されており、`release-merge` は冪等なので、再試行は純粋な再呼び出しである。そのために `prepare` を再実行してはならない。スウォームが決してコンダクターの手を離れることはない。

**自律的なレビュアーの境界。** `invoke-swarm` が `directive.reviewer` を持つ場合、`check` に合格したというだけでは、そのユニットを `finalize` でクレームすることはできない。そのユニットの `prepare` で作成されたワークツリーで、stage-protocol-reviewer.md §12a に従う。`{{INVOKE}} engine log review --stage "<directive.stage>" --unit "<unit>" --reviewer "<directive.reviewer>" --iteration <n> --project-dir "<worktree>"` で `REVIEW_REQUESTED` を記録し、`directive.stage_file` とそのワークツリーのユニットの成果物およびコントラクトに対してレビュアーエージェントに委譲し、同じコマンドに `--verdict <READY|NOT-READY>` を加えて `REVIEW_COMPLETED` を記録する。`--project-dir` がワークツリーを対象とする間、ロガーはメインのワークスペースにとどまる。これは、マルチリポジトリのワークツリーが選択された兄弟リポジトリだけを含む場合にも機能する。NOT-READY の判定では、同じワークツリーでリードを再度呼び出し、収束チェックを再実行し、`directive.reviewer_max_iterations` までレビュアーを繰り返す。唯一の回復レシートが再び無効になった場合は、そのユニットを `--claimed` に入れず、`finalize` も実行しない。人間による Retry/Abort の判断のために停止する。Retry の場合は、メインのワークスペースに戻り、古い Bolt を中止して破棄してから `next` を呼び出し、元の batch/repo の選択で、そのユニットについて上記の **Retry after a native worker discard** に従う。新しい `BOLT_STARTED` の境界は、収束を主張することなくレビューの集計をリセットする。ユニットを `--claimed` に入れるのは、終端のレビューレシートが存在した後だけにする。`finalize` はマージの前にレシートを検証し、その後それをメインの監査にマージする。すべてのユニットが収束した後に `next` が決着の `run-stage` を返した場合は、レビュアーに再度委譲してはならない。ユニットごとのレシートがすでにステージをカバーしているので、上記のメタデータを考慮した決着済みスウォームの完了手順だけを適用する。これは自律的な Construction の中でのモデルの作業であり、人間への別のプロンプトではない。

**再試行の破棄が成功した後。** `--discard` による中止が成功し、古い試行が退避されたことが
確認された後、`prepare` を再実行する前に、この SAY 行を使う。`On your go-ahead I` は、
人間が Retry を選択した場合にのみ使う。それ以外の場合は `I` を使い、行われていない人間による対処の選択を
ほのめかしてはならない。中止が失敗した場合や試行が退避されなかった場合は、保存されたスナップショットを告知してはならない。

返された `parked_mode` から `[saved-files text]` を選択する:

- `snapshot`: "I saved a snapshot of its tracked files and non-ignored untracked files. Ignored files are not saved, and the snapshot may normalize line endings."（訳: 追跡対象ファイルと無視されていない未追跡ファイルのスナップショットを保存しました。無視されたファイルは保存されず、スナップショットでは改行コードが正規化される場合があります。）
- `branch-tip`: "I kept its committed work; there were no uncommitted files to save."（訳: コミット済みの作業を保持しました。保存すべき未コミットのファイルはありませんでした。）
- `evidence-only`: "Nothing of its working files remained to save; only its review evidence was kept."（訳: 保存すべき作業ファイルは何も残っていませんでした。レビューのエビデンスだけを保持しました。）
- `null`: `[saved-files text]` を省略する。フォールバックの記述子は、何が保存されたかを確定しない。

**SAY:** "[On your go-ahead I|I] set aside the previous attempt at [Unit] because the work changed again after its re-check, and I'm starting a new attempt. [saved-files text] If you want the previous attempt back, ask me to restore it."（訳: 再チェック後に作業が再び変更されたため、[Unit] の以前の試行は脇に置き、新しい試行を開始します。[saved-files text] 以前の試行を戻したい場合は、復元するよう依頼してください。）

`restore_operation` がない場合は、最後の申し出 "If you want the previous attempt back, ask me to restore it." を省略する。エビデンスのみの試行について、復元操作をでっち上げてはならない。

人間が後でその試行を戻すよう求めた場合は、保存された中止結果の
`restore_operation` を使う。その `worktree` ルートを
`{{INVOKE}} engine worktree <args...>` で呼び出し、列挙された各 `args` 要素を、正確にそのまま
個別の argv 引数として渡す。それらの引数を連結してシェルコマンドにしたり、
slug だけの選択を作り直したりしてはならない。`restore_hint` は人間向けの表示テキストにすぎず、
決して実行の入力ではない。安全なレンダリングに失敗した場合（たとえば、ハーネスの
ディレクトリが無効な場合）は、ヒントは省略され、`restore_hint_error` がその理由を説明する。
操作は引き続き利用でき、復元の申し出も引き続き適用される。
復元が成功した後、返された復元先のパスを平易に告知する:
**SAY:** "I restored the previous attempt at [returned restored path]."（訳: 以前の試行を [returned restored path] に復元しました。）
復元によって古い試行が再開されたり、そのレビューが現在のものになったりすることはない。

---

### Codex CLI

エンジンは、対象となる Construction バッチをスウォーム実行に選択した。新しいワークフローでは実行方式が明示されており、gated または autonomous の完了承認を使える。実行フィールドを持たないレガシーのワークフローは、自律性に基づくルートを維持する。**あなた（稼働中の `$aidlc` セッション）がコンダクターである。展開と再試行ループはあなたが所有する。`aidlc-swarm.ts` はあなたが参照する決定論的な審判であり、決してループの所有者ではない。** **ステップ (1) の前に、上記の §12b "Autonomous Code Generation Plan Contract" を読んで従う。計画作成、エンジンの Plan Approval の質問、2 つのワーカーブリーフのマーカーは、出力されたすべてのユニットについて必須である。** 以下のステップの前に、上記の **Continuing a partially completed batch**、**Before initial protected prepare**、**Resuming a reviewed batch after Request Changes** を適用して、継続か準備かを選択する。自律性の付与によって自動コミットが許可されることはない。 (1) バッチを **`prepare`** する: `{{INVOKE}} engine swarm prepare --batch <n> --units <directive.units joined by comma> [--base <branch>] [--repo <name>]` は、バッチの事前チェックの後でのみ、最初の隔離されたワークツリーを作成する（`--base` がない場合は、チェックアウトされているブランチからフォークする）。`--resume-existing` を付ける場合は、上記の 2 つの再開経路を使う。ディレクティブに `repo` フィールドがある場合は、`--repo` にその値を渡す。ディレクティブが `repo` を省略している MULTI-REPO のインテントでは、このバッチが対象とする兄弟リポジトリについて `--repo <name>` を指定する（記録されたセットは `$aidlc intent --json`.repos から読む）。マルチリポジトリのインテントでは、これがないと `prepare` はエラーになる。 (2) **`spawn_agent` で展開する**: 1 ターンで、バッチ内のユニットごとに `aidlc-developer-agent` サブエージェントを 1 つずつ生成し（それらは並行して動く）、それぞれがプロジェクトの収束チェックに合格するまで、準備されたワークツリーでユニットを実装させ、その後 `wait_agent` でそれらを待つ。サブエージェントはこのセッションのフォルダで起動する。各サブエージェントに `prepare` から得たユニットのワークツリーのパスを渡し、そのパスの下でのみ作業させる（書き込むすべてのファイルと、実行するすべてのコマンドの作業ディレクトリ）。このハーネスでは、サブエージェントによる展開が唯一のスウォームモードである。`AIDLC_USE_SWARM=1` はここでは効果がない（Workflow ツールは存在しない）。設定されている場合は、**そのことを声に出して伝え**、展開を進め、次の審判呼び出しで `--degraded-from ultracode` を渡して、ツールが `SWARM_DEGRADED` を出力するようにする。 (3) `check` と `finalize` はどちらも、チェックポイントの有無にかかわらず、インテントの記録された、人間が承認した **Construction Verification Command** を使う。`--check-cmd` は省略する（指定する場合は一致しなければならない）。承認がない場合は、どちらかを実行する前に、チェックポイントモジュールの検証コマンドの手順と `set-construction-verification-command` を完了する。これは利用者に 1 回だけ尋ねる。各ユニットのワーカーのターンの後、**`check <unit> [--test-file <protected spec>]`** を参照する。終了コード `0` = 本当に収束した（実際のチェックに合格し、保護されたファイルは改ざんされていない。緩和された、またはオフの Guard Policy のもとでは、変更された保護ファイルは保持され、その 1 行が `change_notices` で返されるので、利用者にその行を伝える）。ゼロ以外 = まだであり、再試行するかエスカレーションするかはあなたが判断する（ナレッジ。同じワーカーに `send_input` すると、そのワーカーが継続する）。 (4) ループが落ち着いたら、**`finalize --batch <n> --units <all> --claimed <the units you believe converged> [--reasons <unit>=<unsatisfiable|budget-exhausted|cap-exhausted>,…]`** は、マージの前にクレームされたすべてのユニットを再検証し（誤ってクレームしたユニットは拒否される。嘘をつくコンダクターへのガード）、本当に合格したものを直列にマージする。クレームしなかったユニットについては、`--reasons` で *なぜ* 断念したかを帰属させる。ツールはあなたの帰属を忠実に記録するが、クレームされたのに失敗したユニットの `error` 判定をそれで上書きさせることは決してない。 **`finalize` の終了コードで分岐する:** `0` → このバッチのレビュー済みの記録エビデンスとメタデータが収束した。上記の必須のソース取り込みステップの後、まだステージを報告せずに `next` を再実行する。エンジンはまず、完了したバッチについて `swarm_checkpoint` を返すことがある。次のバッチの前にそれを解決する。そうでなければ、次の `invoke-swarm` か最終の `run-stage` の決着ディレクティブを返す。その決着ディレクティブの場合にのみ、上記のメタデータを考慮した決着済みスウォームのルールを適用し、そのライフサイクルの結果を報告する。途中のバッチの後に承認済みと報告すると、後続のバッチが構築されないままステージが完了してしまう。 `2` → 失敗のエンベロープを返す。**主導権を取り戻す**: 停止し、halt-and-ask の継ぎ目を通じて人間を再び関与させる（`aidlc-common/protocols/stage-protocol-construction.md` § "Halt-and-ask on failure"。失敗時は自律モードにかかわらず常に停止して尋ねる）。 `merge_failures` のユニット（収束したがマージバックに失敗したもの。マージが成功するまで `SWARM_UNIT_CONVERGED` の行は記録されない）については、阻害要因を解決し、そのユニットに範囲を絞って `finalize` を再実行する。ワークツリーは保持されており、`release-merge` は冪等なので、再試行は純粋な再呼び出しである。スウォームが決してコンダクターの手を離れることはない。審判は判定、マージ、監査を所有し、あなたは展開と再試行の判断を所有する。

**自律的なレビュアーの境界。** `invoke-swarm` が `directive.reviewer` を持つ場合、`check` に合格したというだけでは、そのユニットを `finalize` でクレームすることはできない。そのユニットの `prepare` で作成されたワークツリーで、stage-protocol-reviewer.md §12a に従う。`{{INVOKE}} engine log review --stage "<directive.stage>" --unit "<unit>" --reviewer "<directive.reviewer>" --iteration <n> --project-dir "<worktree>"` で `REVIEW_REQUESTED` を記録し、`directive.stage_file` とそのワークツリーのユニットの成果物およびコントラクトに対してレビュアーのロールを生成し、同じコマンドに `--verdict <READY|NOT-READY>` を加えて `REVIEW_COMPLETED` を記録する。`--project-dir` がワークツリーを対象とする間、ロガーはメインのワークスペースにとどまる。これは、マルチリポジトリのワークツリーが選択された兄弟リポジトリだけを含む場合にも機能する。NOT-READY の判定では、同じワークツリーでリードを再度呼び出し、収束チェックを再実行し、`directive.reviewer_max_iterations` までレビュアーを繰り返す。唯一の回復レシートが再び無効になった場合は、そのユニットを `--claimed` に入れず、`finalize` も実行しない。人間による Retry/Abort の判断のために停止する。Retry の場合は、メインのワークスペースに戻り、古い Bolt を中止して破棄してから `next` を呼び出し、元の batch/repo の選択で、そのユニットについて上記の **Retry after a native worker discard** に従う。新しい `BOLT_STARTED` の境界は、収束を主張することなくレビューの集計をリセットする。ユニットを `--claimed` に入れるのは、終端のレビューレシートが存在した後だけにする。`finalize` はマージの前にレシートを検証し、その後それをメインの監査にマージする。すべてのユニットが収束した後に `next` が決着の `run-stage` を返した場合は、レビュアーを再度生成してはならない。ユニットごとのレシートがすでにステージをカバーしているので、上記のメタデータを考慮した決着済みスウォームの完了手順だけを適用する。これは自律的な Construction の中でのモデルの作業であり、人間への別のプロンプトではない。

**再試行の破棄が成功した後。** `--discard` による中止が成功し、古い試行が退避されたことが
確認された後、`prepare` を再実行する前に、この SAY 行を使う。`On your go-ahead I` は、
人間が Retry を選択した場合にのみ使う。それ以外の場合は `I` を使い、行われていない人間による対処の選択を
ほのめかしてはならない。中止が失敗した場合や試行が退避されなかった場合は、保存されたスナップショットを告知してはならない。

返された `parked_mode` から `[saved-files text]` を選択する:

- `snapshot`: "I saved a snapshot of its tracked files and non-ignored untracked files. Ignored files are not saved, and the snapshot may normalize line endings."（訳: 追跡対象ファイルと無視されていない未追跡ファイルのスナップショットを保存しました。無視されたファイルは保存されず、スナップショットでは改行コードが正規化される場合があります。）
- `branch-tip`: "I kept its committed work; there were no uncommitted files to save."（訳: コミット済みの作業を保持しました。保存すべき未コミットのファイルはありませんでした。）
- `evidence-only`: "Nothing of its working files remained to save; only its review evidence was kept."（訳: 保存すべき作業ファイルは何も残っていませんでした。レビューのエビデンスだけを保持しました。）
- `null`: `[saved-files text]` を省略する。フォールバックの記述子は、何が保存されたかを確定しない。

**SAY:** "[On your go-ahead I|I] set aside the previous attempt at [Unit] because the work changed again after its re-check, and I'm starting a new attempt. [saved-files text] If you want the previous attempt back, ask me to restore it."（訳: 再チェック後に作業が再び変更されたため、[Unit] の以前の試行は脇に置き、新しい試行を開始します。[saved-files text] 以前の試行を戻したい場合は、復元するよう依頼してください。）

`restore_operation` がない場合は、最後の申し出 "If you want the previous attempt back, ask me to restore it." を省略する。エビデンスのみの試行について、復元操作をでっち上げてはならない。

人間が後でその試行を戻すよう求めた場合は、保存された中止結果の
`restore_operation` を使う。その `worktree` ルートを
`{{INVOKE}} engine worktree <args...>` で呼び出し、列挙された各 `args` 要素を、正確にそのまま
個別の argv 引数として渡す。それらの引数を連結してシェルコマンドにしたり、
slug だけの選択を作り直したりしてはならない。`restore_hint` は人間向けの表示テキストにすぎず、
決して実行の入力ではない。安全なレンダリングに失敗した場合（たとえば、ハーネスの
ディレクトリが無効な場合）は、ヒントは省略され、`restore_hint_error` がその理由を説明する。
操作は引き続き利用でき、復元の申し出も引き続き適用される。
復元が成功した後、返された復元先のパスを平易に告知する:
**SAY:** "I restored the previous attempt at [returned restored path]."（訳: 以前の試行を [returned restored path] に復元しました。）
復元によって古い試行が再開されたり、そのレビューが現在のものになったりすることはない。

---

### Cursor

エンジンは、対象となる Construction バッチをスウォーム実行に選択した。新しいワークフローでは実行方式が明示されており、gated または autonomous の完了承認を使える。実行フィールドを持たないレガシーのワークフローは、自律性に基づくルートを維持する。**あなた（稼働中の `/aidlc` セッション）がコンダクターである。展開と再試行ループはあなたが所有する。`aidlc-swarm.ts` はあなたが参照する決定論的な審判であり、決してループの所有者ではない。** **ステップ (1) の前に、上記の §12b "Autonomous Code Generation Plan Contract" を読んで従う。計画作成、エンジンの Plan Approval の質問、2 つのワーカーブリーフのマーカーは、出力されたすべてのユニットについて必須である。** 以下のステップの前に、上記の **Continuing a partially completed batch**、**Before initial protected prepare**、**Resuming a reviewed batch after Request Changes** を適用して、継続か準備かを選択する。自律性の付与によって自動コミットが許可されることはない。 (1) バッチを **`prepare`** する: `{{INVOKE}} engine swarm prepare --batch <n> --units <directive.units joined by comma> [--base <branch>] [--repo <name>]` は、バッチの事前チェックの後でのみ、最初の隔離されたワークツリーを作成する（`--base` がない場合は、チェックアウトされているブランチからフォークする）。`--resume-existing` を付ける場合は、上記の 2 つの再開経路を使う。ディレクティブに `repo` フィールドがある場合は、`--repo` にその値を渡す。ディレクティブが `repo` を省略している MULTI-REPO のインテントでは、このバッチが対象とする兄弟リポジトリについて `--repo <name>` を指定する（記録されたセットは `/aidlc intent --json`.repos から読む）。マルチリポジトリのインテントでは、これがないと `prepare` はエラーになる。 (2) **`task` ツールで展開する**: バッチ内のすべてのユニットを 1 ターンで委譲する。ユニットごとに `aidlc-developer-agent` を対象とする並列タスクを 1 つずつ用意し、それぞれがプロジェクトの収束チェックに合格するまで、自分のワークツリーでユニットを実装する。このハーネスでは、サブエージェントによる展開が唯一のスウォームモードである。`AIDLC_USE_SWARM=1` はここでは効果がない（Workflow ツールは存在しない）。設定されている場合は、そのことを声に出して伝えたうえで展開を進め、次の審判呼び出しで `--degraded-from ultracode` を渡して、ツールが `SWARM_DEGRADED` を出力するようにする。 (3) `check` と `finalize` はどちらも、チェックポイントの有無にかかわらず、インテントの記録された、人間が承認した **Construction Verification Command** を使う。`--check-cmd` は省略する（指定する場合は一致しなければならない）。承認がない場合は、どちらかを実行する前に、チェックポイントモジュールの検証コマンドの手順と `set-construction-verification-command` を完了する。これは利用者に 1 回だけ尋ねる。各ユニットのワーカーのターンの後、**`check <unit> [--test-file <protected spec>]`** を参照する。終了コード `0` = 本当に収束した。ゼロ以外 = まだであり、再試行するかエスカレーションするかはあなたが判断する。 (4) ループが落ち着いたら、**`finalize --batch <n> --units <all> --claimed <the units you believe converged> [--reasons <unit>=<unsatisfiable|budget-exhausted|cap-exhausted>,…]`** は、マージの前にクレームされたすべてのユニットを再検証し（嘘をつくコンダクターへのガード）、本当に合格したものを直列にマージする。 **`finalize` の終了コードで分岐する:** `0` → このバッチのレビュー済みの記録エビデンスとメタデータが収束した。上記の必須のソース取り込みステップの後、まだステージを報告せずに `next` を再実行する。エンジンはまず、完了したバッチについて `swarm_checkpoint` を返すことがある。次のバッチの前にそれを解決する。そうでなければ、次の `invoke-swarm` か最終の `run-stage` の決着ディレクティブを返す。その決着ディレクティブの場合にのみ、上記のメタデータを考慮した決着済みスウォームのルールを適用し、そのライフサイクルの結果を報告する。途中のバッチの後に承認済みと報告すると、後続のバッチが構築されないままステージが完了してしまう。 `2` → 失敗のエンベロープ。**主導権を取り戻す**: 停止し、halt-and-ask の継ぎ目を通じて人間を再び関与させる（`aidlc-common/protocols/stage-protocol-construction.md` § "Halt-and-ask on failure"）。 `merge_failures` のユニット（収束したがマージバックに失敗したもの。マージが成功するまで `SWARM_UNIT_CONVERGED` の行は記録されない）については、阻害要因を解決し、そのユニットに範囲を絞って `finalize` を再実行する。ワークツリーは保持されており、`release-merge` は冪等なので、再試行は純粋な再呼び出しである。そのために `prepare` を再実行してはならない。スウォームが決してコンダクターの手を離れることはない。

**自律的なレビュアーの境界。** `invoke-swarm` が `directive.reviewer` を持つ場合、`check` に合格したというだけでは、そのユニットを `finalize` でクレームすることはできない。そのユニットの `prepare` で作成されたワークツリーで、stage-protocol-reviewer.md §12a に従う。`{{INVOKE}} engine log review --stage "<directive.stage>" --unit "<unit>" --reviewer "<directive.reviewer>" --iteration <n> --project-dir "<worktree>"` で `REVIEW_REQUESTED` を記録し、`directive.stage_file` とそのワークツリーのユニットの成果物およびコントラクトに対してレビュアーのタスクをディスパッチし、同じコマンドに `--verdict <READY|NOT-READY>` を加えて `REVIEW_COMPLETED` を記録する。`--project-dir` がワークツリーを対象とする間、ロガーはメインのワークスペースにとどまる。これは、マルチリポジトリのワークツリーが選択された兄弟リポジトリだけを含む場合にも機能する。NOT-READY の判定では、同じワークツリーでリードを再度呼び出し、収束チェックを再実行し、`directive.reviewer_max_iterations` までレビュアーを繰り返す。唯一の回復レシートが再び無効になった場合は、そのユニットを `--claimed` に入れず、`finalize` も実行しない。人間による Retry/Abort の判断のために停止する。Retry の場合は、メインのワークスペースに戻り、古い Bolt を中止して破棄してから `next` を呼び出し、元の batch/repo の選択で、そのユニットについて上記の **Retry after a native worker discard** に従う。新しい `BOLT_STARTED` の境界は、収束を主張することなくレビューの集計をリセットする。ユニットを `--claimed` に入れるのは、終端のレビューレシートが存在した後だけにする。`finalize` はマージの前にレシートを検証し、その後それをメインの監査にマージする。すべてのユニットが収束した後に `next` が決着の `run-stage` を返した場合は、レビュアーを再度ディスパッチしてはならない。ユニットごとのレシートがすでにステージをカバーしているので、上記のメタデータを考慮した決着済みスウォームの完了手順だけを適用する。これは自律的な Construction の中でのモデルの作業であり、人間への別のプロンプトではない。

**再試行の破棄が成功した後。** `--discard` による中止が成功し、古い試行が退避されたことが
確認された後、`prepare` を再実行する前に、この SAY 行を使う。`On your go-ahead I` は、
人間が Retry を選択した場合にのみ使う。それ以外の場合は `I` を使い、行われていない人間による対処の選択を
ほのめかしてはならない。中止が失敗した場合や試行が退避されなかった場合は、保存されたスナップショットを告知してはならない。

返された `parked_mode` から `[saved-files text]` を選択する:

- `snapshot`: "I saved a snapshot of its tracked files and non-ignored untracked files. Ignored files are not saved, and the snapshot may normalize line endings."（訳: 追跡対象ファイルと無視されていない未追跡ファイルのスナップショットを保存しました。無視されたファイルは保存されず、スナップショットでは改行コードが正規化される場合があります。）
- `branch-tip`: "I kept its committed work; there were no uncommitted files to save."（訳: コミット済みの作業を保持しました。保存すべき未コミットのファイルはありませんでした。）
- `evidence-only`: "Nothing of its working files remained to save; only its review evidence was kept."（訳: 保存すべき作業ファイルは何も残っていませんでした。レビューのエビデンスだけを保持しました。）
- `null`: `[saved-files text]` を省略する。フォールバックの記述子は、何が保存されたかを確定しない。

**SAY:** "[On your go-ahead I|I] set aside the previous attempt at [Unit] because the work changed again after its re-check, and I'm starting a new attempt. [saved-files text] If you want the previous attempt back, ask me to restore it."（訳: 再チェック後に作業が再び変更されたため、[Unit] の以前の試行は脇に置き、新しい試行を開始します。[saved-files text] 以前の試行を戻したい場合は、復元するよう依頼してください。）

`restore_operation` がない場合は、最後の申し出 "If you want the previous attempt back, ask me to restore it." を省略する。エビデンスのみの試行について、復元操作をでっち上げてはならない。

人間が後でその試行を戻すよう求めた場合は、保存された中止結果の
`restore_operation` を使う。その `worktree` ルートを
`{{INVOKE}} engine worktree <args...>` で呼び出し、列挙された各 `args` 要素を、正確にそのまま
個別の argv 引数として渡す。それらの引数を連結してシェルコマンドにしたり、
slug だけの選択を作り直したりしてはならない。`restore_hint` は人間向けの表示テキストにすぎず、
決して実行の入力ではない。安全なレンダリングに失敗した場合（たとえば、ハーネスの
ディレクトリが無効な場合）は、ヒントは省略され、`restore_hint_error` がその理由を説明する。
操作は引き続き利用でき、復元の申し出も引き続き適用される。
復元が成功した後、返された復元先のパスを平易に告知する:
**SAY:** "I restored the previous attempt at [returned restored path]."（訳: 以前の試行を [returned restored path] に復元しました。）
復元によって古い試行が再開されたり、そのレビューが現在のものになったりすることはない。

---

### opencode

エンジンは、対象となる Construction バッチをスウォーム実行に選択した。新しいワークフローでは実行方式が明示されており、gated または autonomous の完了承認を使える。実行フィールドを持たないレガシーのワークフローは、自律性に基づくルートを維持する。**あなた（稼働中の `/aidlc` セッション）がコンダクターである。展開と再試行ループはあなたが所有する。`aidlc-swarm.ts` はあなたが参照する決定論的な審判であり、決してループの所有者ではない。** **ステップ (1) の前に、上記の §12b "Autonomous Code Generation Plan Contract" を読んで従う。計画作成、エンジンの Plan Approval の質問、2 つのワーカーブリーフのマーカーは、出力されたすべてのユニットについて必須である。** 以下のステップの前に、上記の **Continuing a partially completed batch**、**Before initial protected prepare**、**Resuming a reviewed batch after Request Changes** を適用して、継続か準備かを選択する。自律性の付与によって自動コミットが許可されることはない。 (1) バッチを **`prepare`** する: `{{INVOKE}} engine swarm prepare --batch <n> --units <directive.units joined by comma> [--base <branch>] [--repo <name>]` は、バッチの事前チェックの後でのみ、最初の隔離されたワークツリーを作成する（`--base` がない場合は、チェックアウトされているブランチからフォークする）。`--resume-existing` を付ける場合は、上記の 2 つの再開経路を使う。ディレクティブに `repo` フィールドがある場合は、`--repo` にその値を渡す。ディレクティブが `repo` を省略している MULTI-REPO のインテントでは、このバッチが対象とする兄弟リポジトリについて `--repo <name>` を指定する（記録されたセットは `/aidlc intent --json`.repos から読む）。マルチリポジトリのインテントでは、これがないと `prepare` はエラーになる。 (2) **`task` ツールで展開する**: バッチ内のすべてのユニットを 1 ターンで委譲する。ユニットごとに `aidlc-developer-agent` を対象とする並列タスクを 1 つずつ用意し、それぞれがプロジェクトの収束チェックに合格するまで、自分のワークツリーでユニットを実装する。このハーネスでは、サブエージェントによる展開が唯一のスウォームモードである。`AIDLC_USE_SWARM=1` はここでは効果がない（Workflow ツールは存在しない）。設定されている場合は、そのことを声に出して伝えたうえで展開を進め、次の審判呼び出しで `--degraded-from ultracode` を渡して、ツールが `SWARM_DEGRADED` を出力するようにする。 (3) `check` と `finalize` はどちらも、チェックポイントの有無にかかわらず、インテントの記録された、人間が承認した **Construction Verification Command** を使う。`--check-cmd` は省略する（指定する場合は一致しなければならない）。承認がない場合は、どちらかを実行する前に、チェックポイントモジュールの検証コマンドの手順と `set-construction-verification-command` を完了する。これは利用者に 1 回だけ尋ねる。各ユニットのワーカーのターンの後、**`check <unit> [--test-file <protected spec>]`** を参照する。終了コード `0` = 本当に収束した。ゼロ以外 = まだであり、再試行するかエスカレーションするかはあなたが判断する。 (4) ループが落ち着いたら、**`finalize --batch <n> --units <all> --claimed <the units you believe converged> [--reasons <unit>=<unsatisfiable|budget-exhausted|cap-exhausted>,…]`** は、マージの前にクレームされたすべてのユニットを再検証し（嘘をつくコンダクターへのガード）、本当に合格したものを直列にマージする。 **`finalize` の終了コードで分岐する:** `0` → このバッチのレビュー済みの記録エビデンスとメタデータが収束した。上記の必須のソース取り込みステップの後、まだステージを報告せずに `next` を再実行する。エンジンはまず、完了したバッチについて `swarm_checkpoint` を返すことがある。次のバッチの前にそれを解決する。そうでなければ、次の `invoke-swarm` か最終の `run-stage` の決着ディレクティブを返す。その決着ディレクティブの場合にのみ、上記のメタデータを考慮した決着済みスウォームのルールを適用し、そのライフサイクルの結果を報告する。途中のバッチの後に承認済みと報告すると、後続のバッチが構築されないままステージが完了してしまう。 `2` → 失敗のエンベロープ。**主導権を取り戻す**: 停止し、halt-and-ask の継ぎ目を通じて人間を再び関与させる（`aidlc-common/protocols/stage-protocol-construction.md` § "Halt-and-ask on failure"）。 `merge_failures` のユニット（収束したがマージバックに失敗したもの。マージが成功するまで `SWARM_UNIT_CONVERGED` の行は記録されない）については、阻害要因を解決し、そのユニットに範囲を絞って `finalize` を再実行する。ワークツリーは保持されており、`release-merge` は冪等なので、再試行は純粋な再呼び出しである。そのために `prepare` を再実行してはならない。スウォームが決してコンダクターの手を離れることはない。

**自律的なレビュアーの境界。** `invoke-swarm` が `directive.reviewer` を持つ場合、`check` に合格したというだけでは、そのユニットを `finalize` でクレームすることはできない。そのユニットの `prepare` で作成されたワークツリーで、stage-protocol-reviewer.md §12a に従う。`{{INVOKE}} engine log review --stage "<directive.stage>" --unit "<unit>" --reviewer "<directive.reviewer>" --iteration <n> --project-dir "<worktree>"` で `REVIEW_REQUESTED` を記録し、`directive.stage_file` とそのワークツリーのユニットの成果物およびコントラクトに対してレビュアーのタスクをディスパッチし、同じコマンドに `--verdict <READY|NOT-READY>` を加えて `REVIEW_COMPLETED` を記録する。`--project-dir` がワークツリーを対象とする間、ロガーはメインのワークスペースにとどまる。これは、マルチリポジトリのワークツリーが選択された兄弟リポジトリだけを含む場合にも機能する。NOT-READY の判定では、同じワークツリーでリードを再度呼び出し、収束チェックを再実行し、`directive.reviewer_max_iterations` までレビュアーを繰り返す。唯一の回復レシートが再び無効になった場合は、そのユニットを `--claimed` に入れず、`finalize` も実行しない。人間による Retry/Abort の判断のために停止する。Retry の場合は、メインのワークスペースに戻り、古い Bolt を中止して破棄してから `next` を呼び出し、元の batch/repo の選択で、そのユニットについて上記の **Retry after a native worker discard** に従う。新しい `BOLT_STARTED` の境界は、収束を主張することなくレビューの集計をリセットする。ユニットを `--claimed` に入れるのは、終端のレビューレシートが存在した後だけにする。`finalize` はマージの前にレシートを検証し、その後それをメインの監査にマージする。すべてのユニットが収束した後に `next` が決着の `run-stage` を返した場合は、レビュアーを再度ディスパッチしてはならない。ユニットごとのレシートがすでにステージをカバーしているので、上記のメタデータを考慮した決着済みスウォームの完了手順だけを適用する。これは自律的な Construction の中でのモデルの作業であり、人間への別のプロンプトではない。

**再試行の破棄が成功した後。** `--discard` による中止が成功し、古い試行が退避されたことが
確認された後、`prepare` を再実行する前に、この SAY 行を使う。`On your go-ahead I` は、
人間が Retry を選択した場合にのみ使う。それ以外の場合は `I` を使い、行われていない人間による対処の選択を
ほのめかしてはならない。中止が失敗した場合や試行が退避されなかった場合は、保存されたスナップショットを告知してはならない。

返された `parked_mode` から `[saved-files text]` を選択する:

- `snapshot`: "I saved a snapshot of its tracked files and non-ignored untracked files. Ignored files are not saved, and the snapshot may normalize line endings."（訳: 追跡対象ファイルと無視されていない未追跡ファイルのスナップショットを保存しました。無視されたファイルは保存されず、スナップショットでは改行コードが正規化される場合があります。）
- `branch-tip`: "I kept its committed work; there were no uncommitted files to save."（訳: コミット済みの作業を保持しました。保存すべき未コミットのファイルはありませんでした。）
- `evidence-only`: "Nothing of its working files remained to save; only its review evidence was kept."（訳: 保存すべき作業ファイルは何も残っていませんでした。レビューのエビデンスだけを保持しました。）
- `null`: `[saved-files text]` を省略する。フォールバックの記述子は、何が保存されたかを確定しない。

**SAY:** "[On your go-ahead I|I] set aside the previous attempt at [Unit] because the work changed again after its re-check, and I'm starting a new attempt. [saved-files text] If you want the previous attempt back, ask me to restore it."（訳: 再チェック後に作業が再び変更されたため、[Unit] の以前の試行は脇に置き、新しい試行を開始します。[saved-files text] 以前の試行を戻したい場合は、復元するよう依頼してください。）

`restore_operation` がない場合は、最後の申し出 "If you want the previous attempt back, ask me to restore it." を省略する。エビデンスのみの試行について、復元操作をでっち上げてはならない。

人間が後でその試行を戻すよう求めた場合は、保存された中止結果の
`restore_operation` を使う。その `worktree` ルートを
`{{INVOKE}} engine worktree <args...>` で呼び出し、列挙された各 `args` 要素を、正確にそのまま
個別の argv 引数として渡す。それらの引数を連結してシェルコマンドにしたり、
slug だけの選択を作り直したりしてはならない。`restore_hint` は人間向けの表示テキストにすぎず、
決して実行の入力ではない。安全なレンダリングに失敗した場合（たとえば、ハーネスの
ディレクトリが無効な場合）は、ヒントは省略され、`restore_hint_error` がその理由を説明する。
操作は引き続き利用でき、復元の申し出も引き続き適用される。
復元が成功した後、返された復元先のパスを平易に告知する:
**SAY:** "I restored the previous attempt at [returned restored path]."（訳: 以前の試行を [returned restored path] に復元しました。）
復元によって古い試行が再開されたり、そのレビューが現在のものになったりすることはない。

---

### GitHub Copilot

エンジンは、対象となる Construction バッチをスウォーム実行に選択した。新しいワークフローでは実行方式が明示されており、gated または autonomous の完了承認を使える。実行フィールドを持たないレガシーのワークフローは、自律性に基づくルートを維持する。**あなた（稼働中の `/aidlc` セッション）がコンダクターである。展開と再試行ループはあなたが所有する。`aidlc-swarm.ts` はあなたが参照する決定論的な審判であり、決してループの所有者ではない。** **ステップ (1) の前に、上記の §12b "Autonomous Code Generation Plan Contract" を読んで従う。計画作成、エンジンの Plan Approval の質問、2 つのワーカーブリーフのマーカーは、出力されたすべてのユニットについて必須である。** 以下のステップの前に、上記の **Continuing a partially completed batch**、**Before initial protected prepare**、**Resuming a reviewed batch after Request Changes** を適用して、継続か準備かを選択する。自律性の付与によって自動コミットが許可されることはない。 (1) バッチを **`prepare`** する: `{{INVOKE}} engine swarm prepare --batch <n> --units <directive.units joined by comma> [--base <branch>] [--repo <name>]` は、バッチの事前チェックの後でのみ、最初の隔離されたワークツリーを作成する（`--base` がない場合は、チェックアウトされているブランチからフォークする）。`--resume-existing` を付ける場合は、上記の 2 つの再開経路を使う。ディレクティブに `repo` フィールドがある場合は、`--repo` にその値を渡す。ディレクティブが `repo` を省略している MULTI-REPO のインテントでは、このバッチが対象とする兄弟リポジトリについて `--repo <name>` を指定する（記録されたセットは `/aidlc intent --json`.repos から読む）。マルチリポジトリのインテントでは、これがないと `prepare` はエラーになる。 (2) **サブエージェントへの委譲で展開する**: バッチ内のすべてのユニットを 1 ターンで委譲する。ユニットごとに `aidlc-developer-agent` を対象とする並列の委譲を 1 つずつ行い、それぞれがプロジェクトの収束チェックに合格するまで、自分のワークツリーでユニットを実装する。このハーネスでは、サブエージェントによる展開が唯一のスウォームモードである。`AIDLC_USE_SWARM=1` はここでは効果がない（Workflow ツールは存在しない）。設定されている場合は、そのことを声に出して伝えたうえで展開を進め、次の審判呼び出しで `--degraded-from ultracode` を渡して、ツールが `SWARM_DEGRADED` を出力するようにする。 (3) `check` と `finalize` はどちらも、チェックポイントの有無にかかわらず、インテントの記録された、人間が承認した **Construction Verification Command** を使う。`--check-cmd` は省略する（指定する場合は一致しなければならない）。承認がない場合は、どちらかを実行する前に、チェックポイントモジュールの検証コマンドの手順と `set-construction-verification-command` を完了する。これは利用者に 1 回だけ尋ねる。各ユニットのワーカーのターンの後、**`check <unit> [--test-file <protected spec>]`** を参照する。終了コード `0` = 本当に収束した。ゼロ以外 = まだであり、再試行するかエスカレーションするかはあなたが判断する。 (4) ループが落ち着いたら、**`finalize --batch <n> --units <all> --claimed <the units you believe converged> [--reasons <unit>=<unsatisfiable|budget-exhausted|cap-exhausted>,…]`** は、マージの前にクレームされたすべてのユニットを再検証し（嘘をつくコンダクターへのガード）、本当に合格したものを直列にマージする。 **`finalize` の終了コードで分岐する:** `0` → このバッチのレビュー済みの記録エビデンスとメタデータが収束した。上記の必須のソース取り込みステップの後、まだステージを報告せずに `next` を再実行する。エンジンはまず、完了したバッチについて `swarm_checkpoint` を返すことがある。次のバッチの前にそれを解決する。そうでなければ、次の `invoke-swarm` か最終の `run-stage` の決着ディレクティブを返す。その決着ディレクティブの場合にのみ、上記のメタデータを考慮した決着済みスウォームのルールを適用し、そのライフサイクルの結果を報告する。途中のバッチの後に承認済みと報告すると、後続のバッチが構築されないままステージが完了してしまう。 `2` → 失敗のエンベロープ。**主導権を取り戻す**: 停止し、halt-and-ask の継ぎ目を通じて人間を再び関与させる（`aidlc-common/protocols/stage-protocol-construction.md` § "Halt-and-ask on failure"）。 `merge_failures` のユニット（収束したがマージバックに失敗したもの。マージが成功するまで `SWARM_UNIT_CONVERGED` の行は記録されない）については、阻害要因を解決し、そのユニットに範囲を絞って `finalize` を再実行する。ワークツリーは保持されており、`release-merge` は冪等なので、再試行は純粋な再呼び出しである。そのために `prepare` を再実行してはならない。スウォームが決してコンダクターの手を離れることはない。

**自律的なレビュアーの境界。** `invoke-swarm` が `directive.reviewer` を持つ場合、`check` に合格したというだけでは、そのユニットを `finalize` でクレームすることはできない。そのユニットの `prepare` で作成されたワークツリーで、stage-protocol-reviewer.md §12a に従う。`{{INVOKE}} engine log review --stage "<directive.stage>" --unit "<unit>" --reviewer "<directive.reviewer>" --iteration <n> --project-dir "<worktree>"` で `REVIEW_REQUESTED` を記録し、`directive.stage_file` とそのワークツリーのユニットの成果物およびコントラクトに対してレビュアーのタスクをディスパッチし、同じコマンドに `--verdict <READY|NOT-READY>` を加えて `REVIEW_COMPLETED` を記録する。`--project-dir` がワークツリーを対象とする間、ロガーはメインのワークスペースにとどまる。これは、マルチリポジトリのワークツリーが選択された兄弟リポジトリだけを含む場合にも機能する。NOT-READY の判定では、同じワークツリーでリードを再度呼び出し、収束チェックを再実行し、`directive.reviewer_max_iterations` までレビュアーを繰り返す。唯一の回復レシートが再び無効になった場合は、そのユニットを `--claimed` に入れず、`finalize` も実行しない。人間による Retry/Abort の判断のために停止する。Retry の場合は、メインのワークスペースに戻り、古い Bolt を中止して破棄してから `next` を呼び出し、元の batch/repo の選択で、そのユニットについて上記の **Retry after a native worker discard** に従う。新しい `BOLT_STARTED` の境界は、収束を主張することなくレビューの集計をリセットする。ユニットを `--claimed` に入れるのは、終端のレビューレシートが存在した後だけにする。`finalize` はマージの前にレシートを検証し、その後それをメインの監査にマージする。すべてのユニットが収束した後に `next` が決着の `run-stage` を返した場合は、レビュアーを再度ディスパッチしてはならない。ユニットごとのレシートがすでにステージをカバーしているので、上記のメタデータを考慮した決着済みスウォームの完了手順だけを適用する。これは自律的な Construction の中でのモデルの作業であり、人間への別のプロンプトではない。

**再試行の破棄が成功した後。** `--discard` による中止が成功し、古い試行が退避されたことが
確認された後、`prepare` を再実行する前に、この SAY 行を使う。`On your go-ahead I` は、
人間が Retry を選択した場合にのみ使う。それ以外の場合は `I` を使い、行われていない人間による対処の選択を
ほのめかしてはならない。中止が失敗した場合や試行が退避されなかった場合は、保存されたスナップショットを告知してはならない。

返された `parked_mode` から `[saved-files text]` を選択する:

- `snapshot`: "I saved a snapshot of its tracked files and non-ignored untracked files. Ignored files are not saved, and the snapshot may normalize line endings."（訳: 追跡対象ファイルと無視されていない未追跡ファイルのスナップショットを保存しました。無視されたファイルは保存されず、スナップショットでは改行コードが正規化される場合があります。）
- `branch-tip`: "I kept its committed work; there were no uncommitted files to save."（訳: コミット済みの作業を保持しました。保存すべき未コミットのファイルはありませんでした。）
- `evidence-only`: "Nothing of its working files remained to save; only its review evidence was kept."（訳: 保存すべき作業ファイルは何も残っていませんでした。レビューのエビデンスだけを保持しました。）
- `null`: `[saved-files text]` を省略する。フォールバックの記述子は、何が保存されたかを確定しない。

**SAY:** "[On your go-ahead I|I] set aside the previous attempt at [Unit] because the work changed again after its re-check, and I'm starting a new attempt. [saved-files text] If you want the previous attempt back, ask me to restore it."（訳: 再チェック後に作業が再び変更されたため、[Unit] の以前の試行は脇に置き、新しい試行を開始します。[saved-files text] 以前の試行を戻したい場合は、復元するよう依頼してください。）

`restore_operation` がない場合は、最後の申し出 "If you want the previous attempt back, ask me to restore it." を省略する。エビデンスのみの試行について、復元操作をでっち上げてはならない。

人間が後でその試行を戻すよう求めた場合は、保存された中止結果の
`restore_operation` を使う。その `worktree` ルートを
`{{INVOKE}} engine worktree <args...>` で呼び出し、列挙された各 `args` 要素を、正確にそのまま
個別の argv 引数として渡す。それらの引数を連結してシェルコマンドにしたり、
slug だけの選択を作り直したりしてはならない。`restore_hint` は人間向けの表示テキストにすぎず、
決して実行の入力ではない。安全なレンダリングに失敗した場合（たとえば、ハーネスの
ディレクトリが無効な場合）は、ヒントは省略され、`restore_hint_error` がその理由を説明する。
操作は引き続き利用でき、復元の申し出も引き続き適用される。
復元が成功した後、返された復元先のパスを平易に告知する:
**SAY:** "I restored the previous attempt at [returned restored path]."（訳: 以前の試行を [returned restored path] に復元しました。）
復元によって古い試行が再開されたり、そのレビューが現在のものになったりすることはない。
