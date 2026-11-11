# Learnings Protocol Module

ディレクティブが `directive.protocol_modules` に `learnings` を列挙しているときに、このモジュールを読み込む。適用されるのは `directive.ceremony.learnings === "on"` の間だけである。

## 13. Learnings Ritual

このモジュールが列挙されているとき、人間の承認ゲートに到達するすべてのステージは、**完了メッセージ（section 2）と承認ゲート（section 1）の間に**学びのステップを持つ。このステップが利用者に何かを尋ねるのは、そのステージで利用者があなたを修正した後だけである（下記の When to run）。修正がなければ何も尋ねず、ステージは完了メッセージからそのままゲートへ進む。自動で進む bootstrap の初期化ステージと、単独の `single: true` 実行にはワークフローの承認ゲートがないため、このステップを通らない。未完了の Unit ごとの反復は、ステージの最終ゲート1回までこのステップを延期する。Fowler のハーネスモデルによれば: "when issues recur, feedforward and feedback controls should be improved."（問題が繰り返し起きるなら、フィードフォワードとフィードバックの制御を改善すべきである）。このステップは人間による学習ループである: 利用者が行った修正を、次の実行者が自動的に拾える場所に保持する。

このステップは**ツールが実行主体**である: 利用者が修正したかどうか、それが次回にも当てはまるかどうかは、あなたが利用者自身の言葉から読み取る。決定的なツール（`aidlc-learnings.ts`）が正確な質問を出力し、利用者が残すものを書き込む。あなたは質問を表示し、採用時の矛盾チェックを実行する。残すかどうか、プロジェクト向けかチーム向けかは利用者が決める。

### What changes vs what doesn't

**ステージファイルは変更不可のフレームワーク成果物である。** この儀式は、ステージファイルの `## Steps`、`## Sensors`、`## Learn` の内容を決して編集しない。ステージファイルはフレームワークのリリースとともに配布され、ユーザー層のカスタマイズはハーネスにある。唯一の例外はフロントマターの `sensors:` インポートリストである — センサーのバインド追加では、そこに新しい id を追記する（pull-authoring の2書き込みインストール）。それはインポートリストであって本文の内容ではなく、ステージの変更不可の形は変わらない。ステージファイルはフレームワークとループによって編集されるものであり、フレームワーク専用ではない — ただし増えるのはそのフロントマターのリスト1つだけである。

**ハーネスは変更可能である。** 確定した学びはプラクティスであり、次の2つの面のいずれかに書き込まれる:

- `aidlc/spaces/<space>/memory/project.md`（既定）または `aidlc/spaces/<space>/memory/team.md` — 適切なトピック見出し（例: `## Corrections`、`## Testing Posture`、`## Forbidden`）の下にプラクティス行として追記される。候補をプロジェクトからチームへ広げるのはワンクリックである。これらはリゾルバーが読むのと同じメソッドファイルであり、並行する `*-learnings.md` の面も、部分的な上書き層も、org 層もない（org へ広げる経路はない）。何を学んだかの履歴は、監査シャードとステージごとの日誌に残り、日付付きで積み上がるファイルにはない。
- `{{HARNESS_DIR}}/sensors/aidlc-<id>.md` — 検証チェック用。`matches:` のケイパビリティ glob を持つプロジェクト層のマニフェストで、その id を元のステージの `sensors:` フロントマターリストに追記することで、そのステージにバインドされる。

次回そのステージを実行するとき、解決済みのルールとバインドされたセンサーはコンパイル時に自動的に読み込まれる — 誰もステージファイルの本文を編集することなく、ステージがより良く動く。

### When to run

このモジュールがセッションの早い段階で読み込まれていても、適用されるかどうかは現在のディレクティブが決める。列挙されなくなったら、日誌をつけず、このステップを飛ばす。

このモジュールが列挙されているとき、このステップは Step N-1（完了メッセージの表示）の後、Step N（承認ゲート）の前に来る。ただし、エンジンがそのステージの実際の人間のゲートを出力したときに限る。利用者に何かを尋ねるのは次の2つの場合だけであり、それがいつかを定めるのはこの箇所だけである:

- **利用者がこのステージであなたを修正し、それが次回にも当てはまる場合。** ゲートで変更を求めた（Request Changes）、または作業中にあなたが行ったことや前提としたことを自分の言葉で修正し、その修正が次回にも通用する作業の進め方（慣習、このプロジェクトやチームのルール）である場合。単発の修正（誤字、名前の誤り、1つのファイルの1つの値）は提示しない。チェック自身の質問への回答（Fix findings または Override blocking sensors。言い方は問わない）も提示しない: それはその指摘に対する判断であり、チームのチェックはチームが設定したとおりに残る。その読み取りは、他の返答を読むのと同じように、利用者自身の言葉からあなたが行う。ツールや単語リストが決めるのではない。各修正について尋ねるのは一度だけである。
- **利用者が次回のために何かを残すよう求めた場合**（"next time, use the shared fixtures"、"from now on, ..."）。利用者がすでにそう言っているので、質問なしで残す（step 5）。チームを名指ししない限り、プロジェクト向けとする。ゲートでの返答の中で承認とともに言われた場合は、返答のとおりに承認を記録し、メモをこのステージの下に残す: 別の承認が記録されるまでは `surface --slug <this stage>` がそれを受け付ける。何をどこに残したかを1行で伝え（"Kept for next time in this project's rules, `aidlc/spaces/<space>/memory/project.md`"）、利用者がチームのルールへ移したり削除したりできるようにする。

それ以外の場合、このステップは何も尋ねない: 日誌のメモについての質問も、"Anything to add?" もしない。完了メッセージからそのままゲートへ進む。

`gate: false` の反復はこのステップを実行しない。bootstrap の初期化ステージは日誌（step 1）だけをつけ、質問、採用、永続化は行わない。単独の `single: true` 実行は日誌をつけず、このモジュールを一切実行しない。Unit ごとの `gate: false` の反復は日誌をつけ、このステップをステージの最終ゲートまで延期する。チームが所有する unit-major のゲートは例外である: 単独の最終ステージゲートを待たずに、出力された各 `directive.unit_gate` で、対応する Unit の識別子とともにこのステップを実行する。`learnings` を列挙する人間の Construction チェックポイントでは、construction プロトコルのチェックポイント手順のとおり、`directive.construction_checkpoint.stages` のステージについて一度だけ実行する。`surface --slug` には修正が行われたステージを指定する。ディレクティブが `approve_together` を持つゲートでは、同じように `approve_together.stages` のすべてのステージについて一度だけ実行し、質問は `directive.stage` の下にログを取る。Request Changes の後、利用者が求めた変更は他と同じく修正である: 改訂を行ったら（ディレクティブがレビュアーを持つ場合はレビューも済ませたら）、`report --result revised` がゲートを再提示する前にそれについて尋ねる。ただし、単発の修正である場合や、すでに尋ねた場合を除く。

### The ritual

1. **作業しながら、ステージごとのメモリーファイルを維持する。** エントリーを `directive.memory_path` に追記する: `<record>/<phase>/<stage>/memory.md`、Unit ごとのステージでは Unit 自身の `<record>/construction/<unit>/<stage>/memory.md`（学びの儀式がオンのときだけ、エンジンが run-stage ディレクティブを出力する際に、同梱のテンプレートから作成する）。このパスは出力専用の書き込み先として扱う: オーケストレーターはそれを読んだり、調べたり、作成したり、初期化したりせず、エントリーを挿入するときはアクティブなハーネスの日誌書き込みの規律に従う。次の4つの標準 H2 見出しを使う:
   - **Interpretations** — ステージの記述が曖昧な箇所で行った選択
   - **Deviations** — 意図的にステージの記述から外れた箇所とその理由
   - **Tradeoffs** — 検討した代替案と、選んだものを選んだ理由
   - **Open questions** — 次の実行の前に確認すべきこと、または注意を促すべき不確かな文脈

   各エントリーは、該当する見出しの下の箇条書きとし、ISO 8601 のタイムスタンプを先頭に付ける（`{{INVOKE}} engine now` が出力する）:
   ```markdown
   - 2026-05-20T10:14:32Z — <one-line summary>; <2-3 sentences of context>
   ```

   メモリーファイルはセッションをまたいで保持される — 停止して再開したステージも、そのログはそのまま残る。ステージが承認されると、メモリーファイルはステージの恒久的な記録の一部として成果物ディレクトリに残る（他の成果物とともにコミットされる）。

   日誌はステージの記録である: そのどれも利用者には提示しない。このステップの質問は、利用者の修正（step 2）からだけ生まれる。

2. **修正を表に出す（ツールが質問を出力する）。** このステップが何かを尋ねるとき、または利用者が残すよう求めたメモを残すとき（When to run）にだけ、次を実行する:
   ```bash
   {{INVOKE}} engine learnings surface --slug <stage-slug> --correction '<their change>'
   ```
   尋ねる修正1件ごとに `--correction` を1つ付け、"You asked me to" に続く形で、利用者の言葉で（"name test files after the module they test"）、**Their words on a command line**（stage-protocol.md）のとおりシングルクォートで囲んで書く。そこに挙げられた文字のいずれかを含む修正は、代わりにファイルに入れる: ファイルツールで `<record>/.aidlc-engine/answer-text/correction-1.txt` に書き込み（修正1件につき1ファイル）、その代わりに `--correction-file .aidlc-engine/answer-text/correction-1.txt` を渡す。エンジンがファイルを読み、削除する。ツールは修正1件ごとに1つの質問を `questions` に出力する。`--correction` がなければ何も出力しない。これが、利用者が残すよう求めたメモのための呼び出しである。出力には `selections_file`、`space`、`intent` も含まれる: ワークスペースのアクティブなスペースとインテントを、後で導き直すのではなく、この時点で解決したものである。その両方を step 5 の選択ファイルにそのまま引き継ぐ。`persist` はライブのアクティブインテントのカーソルではなくこれらを使うので、永続化の前に後からインテントを切り替えても、書き込みの帰属を誤ることはない。有効なアクティブインテントのカーソルがないまま複数のインテント記録がある場合は、黙って `intent: null` にするのではなく、ここでハードエラーとなる。

   チームが所有する Unit の Unit ごとのステージ（ディレクティブが `unit` を指定している）では、`--unit <directive.unit>` を付け、ツールがその Unit 自身の日誌を数えるようにする。

3. **`surface` が出力した構造化された質問を尋ねる。** その `questions` には、質問が順に正確な選択肢とともに並んでいる: 修正ごとに、"You asked me to <their change>. Keep that for next time?" と、**Yes, for this project**、**Yes, for the team**、**No** である。それぞれを、挙がっているすべての選択肢を含む独立した構造化された質問として尋ねる。質問が1つだけの場合も同様であり、ツールが複数を受け付けるなら全部を同じ呼び出しで尋ねる。1つを別の質問に畳み込んではならない。利用者の返答ですでに答えが出ている質問は尋ねない: stage-protocol.md section 4 が返答の後にログを取る質問について述べているとおり、ログを取り、その回答を記録する。他の構造化された質問と同様に各質問を記録する: 表示する前に `{{INVOKE}} engine log decision --stage <stage-slug> --decision '<the question as printed>' --option 'Yes, for this project' --option 'Yes, for the team' --option 'No' --question-kind learnings-note` を実行し（ファイルに入れた修正を含む質問はファイルに入れる: `--decision-file .aidlc-engine/answer-text/question-1.txt`）、その後に `{{INVOKE}} engine log answer --stage <stage-slug> --details '<the option they picked>'` を実行する。チームが所有する Unit のゲートでは、両方に `--unit <that Unit>` を付ける。利用者の回答を推測したり自分で選んだりせず、質問のところでターンを終える: 承認ゲートは別の後のターンであり、同じメッセージで表示してはならない。選択肢が1つの構造化された質問を出力してはならない: Claude Code と Codex はそれを拒否する。利用者に求めるのは選択だけである。プラクティス行は利用者の修正からあなたが書き、メソッドファイル（KNOWLEDGE）の適切なプラクティス見出しに振り分ける: テストに関する学び -> `## Testing Posture`、禁止事項 -> `## Forbidden`、一般的なもの -> `## Corrections`（既定）。利用者が見出しや日誌の見出しを選ぶことはない。ツールは書き込む前に見出しの存在を保証する。

4. **採用時の矛盾チェック（書き込みの前に）。** 残す学びの候補それぞれについて、提案するプラクティス行を `org.md` の対応する `## <section>`（振り分けた見出しで照合する — §5 の採用ゲートの1行版）と比較する。この比較はセクション単位の LLM チェックである（ナレッジ → オーケストレーター LLM）。プラクティスが org のガードレールと矛盾する場合は、矛盾する org の文をその場で示す。利用者は**改訂する、この候補をスキップする、またはエスカレーションする**（判断 → 利用者。利用者による上書きの経路はない）。書き込みに進むのは、矛盾のない選択か、利用者がエスカレーションした選択だけである。センサーマニフェストには対応する org のセクションがないため、このチェックを省く。

5. **永続化する（ツールが書き込み、監査を出力する）。** 選択ファイル — `{ stage_slug, space, intent, selections[] }`。`stage_slug`/`space`/`intent` は step 2 の surface 出力からそのまま引き継ぐ — を組み立て、surface 出力が示す `selections_file` のパスにファイルツールで書き込む（gitignore された `.aidlc-engine/learnings/` フォルダ: 記録のもの、またはインテント記録がなければスペースの intents フォルダのもの。作業ごと・ステージごとに1ファイル）。その後、次を呼ぶ:
   ```bash
   {{INVOKE}} engine learnings persist --slug <stage-slug> --selections-json <the selections_file surface returned>
   ```
   選択は、利用者が残した修正ごとに1エントリー（**Yes, for this project** なら `scope: "project"`、**Yes, for the team** なら `"team"`、**No** は何も残さない）と、利用者が残すよう求めたメモごとに1エントリー（チームを名指ししない限り `project`）であり、いずれも `source: "user_addition"` を持つ: 言葉は利用者のものだからである。残すよう求められたメモには独自の候補がない: `candidate_id` に `n1`、`n2` のように付ける。
   `selections[]` の各エントリーは、`candidate_id` で候補を特定する。`surface` が出力し step 3 が対応付けに使うキー `id` も別名として受け付ける。両方を持つエントリーは同じ値でなければならない。学びのエントリーは、`type: "learning"`（`type` がない場合の既定）、`scope`（既定は `project`、または `team`）、振り分けた `heading`、プラクティスの `text` を持つ。センサーのエントリーは、`type: "sensor"`、`origin_stage`、および文字列の `id`、`kind`、`command`、`default_severity`、`description`、`matches` と、任意の `timeout_seconds`（数値）、`category` を含む `manifest_fields` を持つ。どちらのエントリーも、`"orchestrator"`（既定）または `"user_addition"` を設定した任意の `source` フィールドを持てる。これは監査行に記録される。`keep: false` のエントリーは利用者が残さなかった候補であり、除外される。不正な形式のエントリーがあると、監査ロックを取得する前に呼び出し全体が失敗するので、中途半端に書き込まれることはない。

   ツールは、選択ファイルの `stage_slug` と異なる `--slug` を拒否し、その後1つの `withAuditLock` トランザクションの中で、固定されたスペースと null でないインテント記録がまだ存在することを検証する。新しい監査スナップショットと、同じバッチで先に出力されたハッシュの両方に対して重複を排除する。そのために使うマーカーと書き込み形式は、機械照合の都合上、次の原文のまま示す（和訳は直後の注記を参照）: `<!-- cid:<intent-slug>:<stage-slug>:<content-hash> -->` marker whose content hash is the full SHA-256 digest of the learning text. A crashed run therefore recovers without double-appending, while distinct learning text cannot share a truncated persisted identity:
   - **Learning** → appends a practice line under the orchestrator-routed heading in `<scope>.md` (scope ∈ {project, team}): `- <text> (learned YYYY-MM-DD) <!-- cid:... -->`; a field line the engine reads (`Methodology: tdd` or `Ordering: ...` under `## Testing Posture`, `Mode: strict` under `## Guard Policy` or the retired `## Change Control`) gets `<!-- learned YYYY-MM-DD -->` instead, so the date stays out of the value. （和訳: このマーカーのコンテンツハッシュは学びのテキストの完全な SHA-256 ダイジェストである。したがって、クラッシュした実行は二重追記なしに回復でき、異なる学びのテキストが切り詰められた永続 ID を共有することはない。**Learning** の場合は、オーケストレーターが振り分けた見出しの下にプラクティス行を追記する。エンジンが読むフィールド行には、代わりに learned 日付のコメントを付け、日付が値に入らないようにする。）まず見出しの存在を保証し、ファイルにまだない振り分け先の見出しは、例外を投げずに作成する。`RULE_LEARNED` を出力する（`Source: orchestrator | user_addition`、`Heading: <routed>` 付き）。
   - **Sensor** → プロジェクト層の `<project>/{{HARNESS_DIR}}/sensors/aidlc-<id>.md` マニフェストを（利用者が指定した `matches:` glob 付きで）生成し、さらに新しい id を元のステージの `sensors:` フロントマターリストに追記する — 両方の書き込みを同じロックの中で行う。`SENSOR_PROPOSED` を出力する。センサーは次のワークフローのコンパイルからバインドされ、発火する。

   オーケストレーターはルールやセンサーのファイルを直接 `Edit` しない — 学びの書き込みはすべてロックの下でツールを通じて行われるので、`RULE_LEARNED` / `SENSOR_PROPOSED` の監査行が、何を学んだかについての再生可能な信頼できる情報源となる。選択ファイルは再生用の成果物である: クラッシュした persist は、人間に再度尋ねることなく同じ selections-json を再生する。

6. **承認ゲートへ進む。** このステップは助言的かつ追加的である: 利用者が答えた後、ゲートを妨げることは決してない。`report --result awaiting-approval`（Request Changes の後は `report --result revised`）を実行し、ゲートを提示する。

### Routing decision tree

```
Is it a correction they kept, or a note they asked you to keep?
└── Learning → a practice line under the routed heading in <scope>.md
    Heading routed by fit (testing → ## Testing Posture, prohibition →
      ## Forbidden, general → ## Corrections); ensure-exists before write.
    Scope derived from the user's keep + optional promote:
    ├── default                       — project.md
    └── promote scope (project→team)  — team.md   (no org tier)

Is the improvement a verification check?
└── Sensor (two-write install): scaffold a project-tier manifest at
    {{HARNESS_DIR}}/sensors/aidlc-<id>.md with a matches: glob, AND append its id to
    the originating stage's sensors: frontmatter list (one locked transaction).
    The matches: glob is a capability filter — stages: [<id>] is the binding.
```

### What goes where — quick reference

| 利用者が言ったこと | 行き先 |
|---|---|
| Request Changes: "reuse the auth module, don't rewrite it"、その後 **Yes, for this project** | `project.md ## Corrections`（プラクティス行、`(learned YYYY-MM-DD)`） |
| 作業中の修正: "write the acceptance criteria as Given/When/Then"、その後 **Yes, for the team** | `team.md ## Testing Posture`（プラクティス行） |
| ゲートで: "approve, and next time use TDD for the generators"（質問なし） | `project.md ## Testing Posture`（プラクティス行） |
| Request Changes: "the title is Greeter, not Greeting"（単発の修正） | 尋ねず、何も残さない |
| チェック: "ADRs should carry Security and Compliance headings" | センサーマニフェスト `aidlc-<id>.md`（`matches:` glob）。その `sensors:` フロントマターでステージにバインドされる |

### Why stage files stay immutable

理由は2つある: (1) フレームワークによるステージファイルのアップグレードが、ワークフロー実行時の編集と衝突する。(2) 同じステージが多くのプロジェクトで実行されるため、ステージファイルの本文を変更すると、すべてのワークフローがフレームワークの方法論を互いに相容れない方向へずらしてしまう。ハーネス層（ルール、学び、センサー）は組み合わせられるように設計されている — 多数の小さな追加が衝突なく積み重なる。ステージファイルの本文はそうではない。センサーをバインドするフロントマターの編集が唯一認められた例外である: それは `sensors:` インポートリスト（形は変更不可、中身は変更可）を増やすだけであり、`## Steps` / `## Sensors` / `## Learn` の本文には決して触れない。
