---
slug: build-and-test
name: Build and Test
phase: construction
execution: ALWAYS
condition: Always executes once after all per-unit stages are finished.
lead_agent: aidlc-quality-agent
support_agents:
  - aidlc-devsecops-agent
mode: inline
produces:
  - build-instructions
  - build-and-test-summary
  - build-test-results
  - cross-unit-traceability
optional_produces:
  - integration-test-instructions
  - performance-test-instructions
  - security-test-instructions
consumes:
  - artifact: code-generation-plan
    required: true
  - artifact: unit-test-instructions
    required: true
  - artifact: code-summary
    required: true
requires_stage:
  - code-generation
sensors:
  - required-sections
  - upstream-coverage
  - type-check
scopes:
  - enterprise
  - feature
  - mvp
  - poc
  - bugfix
  - refactor
  - security-patch
  - infra
  - classic
  - workshop
  - express
inputs: ALL code generation outputs across all units
outputs: "build-instructions.md, build-and-test-summary.md, test-results.md, cross-unit-traceability.md, CONDITIONAL: integration-test-instructions.md (Standard and above), performance-test-instructions.md and security-test-instructions.md (Comprehensive, when that NFR kind exists) (under this stage's record dir, engine-resolved)"
---

# Build and Test

## Steps

### Step 1: Analyze Testing Requirements

すべてのユニットのコード生成の出力を
`<record>/construction/*/code-generation/code-summary.md` から、ユニットごとのテスト
手順を
`<record>/construction/*/code-generation/unit-test-instructions.md` から読み込む。`express` や `infra` のような
ユニットがゼロのスコープでは、
`<record>/construction/code-generation/` 配下にあるステージレベルの同等のファイルを読み込む。

手順を生成する前に、測定可能なすべての品質目標について、ソースを漏れなく網羅した
インベントリを作成する。該当するステージレベルおよびユニットごとのソースをすべて読み込む:

- `nfr-requirements/` 配下のすべての成果物
- `nfr-design/` 配下のすべての成果物
- `code-generation-plan.md` にある現行のすべての `## Testing Contract`。Code Generation の Step 3 の
  フェンスを下げるルールで許可された承認後の編集も含む。
  それらの編集を人間が承認したものとして記述してはならない。
  CI の義務（契約の "Run the selected tests in CI before merge."
  の行、またはメモリのノートの "CI execution before merge"）は、
  プランが CI Pipeline を実行する場合にのみ適用される: つまり `aidlc-state.md` が `ci-pipeline` を、チェックボックスが `[S]` ではない `EXECUTE`
  ステージとして列挙している場合である。プランがそれを実行しない場合は、その
  目標を `N/A` として、理由 "the plan runs no CI Pipeline" とともに記録する: 該当しないため、
  `Unverified` にも失敗にも決してならない。CI
  Pipeline がまだ先にある間（チェックボックスが `[ ]`）は、その目標を満たすのは CI Pipeline
  である: Owning Stage に `ci-pipeline`、Verdict に `N/A`、理由に "CI
  Pipeline, later in this plan, sets up CI to run these tests before merge" を記録する。
  ここでは `Unverified` にも失敗にも決してならない。

各目標について、安定した目標 ID（ソースに ID がない場合はソースのパスとセクションから
導出する）、ソースのパス/セクション、期待値、実際の値を生み出すチェックまたは手順ファイル、
そして Build and Test がローカルで実行できない場合にそれを担う後続の
検証ステージを記録する。
このインベントリから、必要なテストの種類をすべて洗い出す。

### Step 2: Generate Build Instructions

`<record>/construction/build-and-test/build-instructions.md` を作成する:
- 依存関係のインストール手順
- 環境のセットアップ（環境変数、設定ファイル、ローカルサービス）
- ビルドコマンド（コンパイル、バンドル、トランスパイル）
- ビルドの検証手順
- Infrastructure as Code の場合は、変更された IaC のルートまたはアプリごとの静的チェック:
  フォーマットと検証（たとえば `terraform fmt -check` と
  `terraform validate`、`cdk synth`、`cfn-lint`）、およびプロジェクトが実行する
  Policy as Code のルール
- よくあるビルドの問題のトラブルシューティング

### Step 3-7: Generate Test Instructions (Strategy-Aware)

`aidlc-state.md` → `**Test Strategy**` から、または分離実行が run-stage の `test_strategy` を持っている場合はそれから、有効なテスト戦略を参照する（stage-protocol.md §8 "Test Strategy" を参照）。戦略のレベルに基づいて、追加のテスト手順ファイルを生成する:

**Minimal 戦略** — 追加のテスト手順ファイルは生成しない。ユニット
テストは Code Generation によってユニットごとにカバーされる。

**Standard 戦略** — 次を生成する:
- `integration-test-instructions.md`: 主要な境界のテスト、ユニット間の相互作用

**Comprehensive 戦略** — 該当するものをすべて生成する:
- `integration-test-instructions.md`: ユニット間の相互作用、外部依存関係の扱い
- `performance-test-instructions.md`（NFR のパフォーマンス要件がある場合）: 負荷テスト、ベンチマーク、リグレッションの検出
- `security-test-instructions.md`（NFR のセキュリティ要件がある場合）: SAST/DAST、認証のテスト、インジェクションのテスト
- 該当する場合はその他の種類（コントラクトテスト、E2E、アクセシビリティ） — 種類ごとに名前を付けたファイルを作成する

すべてのファイルは `<record>/construction/build-and-test/` に置く。

各手順ファイルには次を含めるべきである:
- テストフレームワークのセットアップと設定
- テストの実行方法（コマンド、フラグ、フィルター）
- 戦略のレベルに応じた、期待されるカバレッジ目標
- テストデータの管理と環境のセットアップ

これら 3 つの手順ファイルは `optional_produces` の下に宣言されている。Minimal の
実行ではいずれも書かれず、Standard の実行では統合テストのものだけが書かれるためである。
常に書かれるのは、ビルド手順、サマリー、結果、ユニット間の
トレーサビリティだけである。

これらは緩やかなガイドラインである — コンテキストが求める場合、LLM はどの戦略レベルでも追加の種類のテストを生成できる（例: Minimal の security-patch でも、セキュリティテストの手順が必要になる場合がある）。

### Step 8: Generate Build and Test Summary

`<record>/construction/build-and-test/build-and-test-summary.md` を作成する:
- 全体のビルド状況と前提条件
- テストの種類のインベントリ（どの種類のテストを生成したか）
- プロジェクトと各ユニットに期待されるカバレッジ
- 目標ごとに 1 行を持ち、次の列を持つ `## Target Verification Matrix`:
  Target ID, Source, Expected, Actual, Evidence, Owning Stage, Verdict
- 該当する各目標は、Actual と Evidence を `Pending`、Verdict を
  `Pending` として開始する。`N/A` が有効なのは、ソースのインベントリに該当する
  測定可能な目標が見つからなかった場合（その場合は説明用の `N/A` 行を 1 つ書く）か、
  プランによってここでは該当しなくなる目標の場合だけである。後者は、Step 1 が述べる、CI Pipeline を
  実行しない、またはそれがまだ先にあるプランの CI の義務がそれにあたる。該当する目標が
  `N/A` を使うことは決してできない。
- 準備状況の評価（ビルド可能、テスト可能、デプロイ可能）
- 既知の制約や未解決の項目

### Step 9: Execute Build and Tests

手順ファイルに記載したビルドとテストのコマンドの実行を試みる:

1. **ビルド**: `build-instructions.md` のビルドコマンドを Bash で実行する。出力を取得する。
2. **ユニットテスト**: ステージレベルの
   `<record>/construction/code-generation/unit-test-instructions.md` ファイル（
   存在する場合。Express を含む）と、ユニットごとのすべての
   `<record>/construction/*/code-generation/unit-test-instructions.md` ファイルの両方から実行コマンドを集める。
   同一のコマンドは重複を除き、異なるコマンドをそれぞれ 1 回だけ Bash で実行する。
   ユニットごとのコマンドは、すでにそのユニットにスコープされているはずである。ステージレベルのファイルや
   不正な形式のユニットごとのファイルには、プロジェクト全体のコマンドが含まれている場合がある。そのコマンドは
   1 回だけ実行し、N 回実行してはならない。ステージレベル/ユニットごとの合否の
   結果を、二重に数えずに取得して報告する。
   プロジェクト全体にカバレッジの下限が設定されている場合（Testing
   Contract、チームのルール、または測定対象のすべてのソースに対するプロジェクトのカバレッジ
   設定のしきい値による）、組み立てたプロジェクトに対して、プロジェクト自身のカバレッジ設定で
   プロジェクトのフルスイートのカバレッジコマンドを 1 回実行し
   （同じリポジトリでこのステップですでに実行した同一のコマンドもそれに数える）、そのコマンド、終了コード、レポートを
   `test-results.md` に記録する。
   マルチリポジトリのインテントでは、下限が適用される各リポジトリで、
   そのリポジトリ自身のコマンドとカバレッジ設定を使って 1 回ずつ実行し、
   行の Evidence にリポジトリ名を記す。ユニットにスコープされた実行ではなく、その実行が、Target
   Verification Matrix におけるプロジェクト全体のカバレッジ下限の行の Evidence となる。また、あるリポジトリの実行が
   別のリポジトリの下限の Evidence になることは決してない。ユニットにスコープされたコマンドについて、
   選択したテストがすべて合格し、プロジェクト全体に設定された下限（ランナーがファイルごとにチェックする
   場合でも、測定対象のすべてのソースに適用される下限）のしきい値だけが 0 以外の終了の原因であることが
   出力から分かる場合、そのコマンドは、手順ファイルが新規のものでも承認済みのものでも、合格として扱う:
   その終了コードとこの理由を `test-results.md` に記録する。これが適用されるのは、
   その終了によってコマンドの後続のステップが止まらなかった場合に限る: それより後のステップが
   そのせいで実行されなかった場合（`… && <lint>` のように連結されたもの、コマンドが呼び出す
   スクリプトやライフサイクルフックの中のもの、ビルドツールの後続のタスクやフェーズ。
   どれにあたるかは、コマンドが実行するスクリプト、フック、タスクやフェーズの定義と、その出力から判断する）、
   そのコマンドは例外にならず、0 以外の終了は失敗したコマンドとなる。
   その他の 0 以外の終了（失敗またはエラーになったテスト、1 つのパスやユニットに設定された
   しきい値）や、両方を示していない出力は、失敗したコマンドである。
   1 つのユニットに設定されたカバレッジ目標には、そのユニットのスコープに対する独自のチェックと
   独自の証拠が必要である。フルスイートの実行が合格しても、それが `Met` になるわけではない。
3. **統合テスト**（該当する場合）: 統合テストのコマンドを実行する。結果を取得する。
4. **その他の該当するチェック**: パフォーマンス、セキュリティ、
   コントラクト、E2E、アクセシビリティ、その他の生成された手順ファイルにある、該当するすべてのコマンドを
   実行する。チェックを先送りできるのは、それがデプロイ済みまたは
   本番相当の環境を必要とし、かつ現在の実行プランに、そのチェックを明示的に担う後続の
   検証ステージ（たとえば
   `performance-validation`）が含まれている場合だけである。担当するステージと期待される証拠の
   パスを記録する。先送りした目標は `Unverified` のままであり、
   ステージの成功という結果には寄与できない。後続の担当ステージが予定されていない場合、その目標は
   `Unverified` であり、先送りに成功したことにはならない。
   Infrastructure as Code の場合、実環境に対する変更のプレビュー
   （`terraform plan`、`cdk diff`、CloudFormation の変更セット）は、クラウドの認証情報や
   リモートの状態を必要とする場合にそうしたチェックとなる: プランがそのステージを実行する場合は、Environment
   Provisioning がそれを担う。このステージから適用やデプロイを
   決して行ってはならない。
5. **結果の確定と報告**: ループバック、停止して確認、中止、
   受け入れた失敗を含むすべての終了経路で、
   `<record>/construction/build-and-test/test-results.md` と Build and Test
   Summary を作成または更新し、次を含める:
   - ビルドの状況（成功/失敗 + 出力）
   - テストの結果（合計、合格、失敗、スキップ）
   - 失敗の詳細（テスト名、アサーション、スタックトレース）
   - カバレッジレポート（テストフレームワークが対応している場合）
   - 確定した Target Verification Matrix: 実際の値、証拠のパスまたは
     コマンドの出力、担当ステージ、そして該当する目標ごとにちょうど 1 つの最終判定:
     `Met`、`Not Met`、`Unverified` のいずれか。`Pending` が許されるのは
     Step 8 を準備している間だけであり、Step 9 が終了するときには `Pending` の判定を
     1 つも残してはならない。
   - このステージがブラウザの挙動をテスト済みとして報告する箇所（マトリックスの行、
     `cross-unit-traceability.md` のエントリ、サマリー、準備状況の
     評価）ではどこでも、チェックを実行した場所を明記する: Node、DOM エミュレーター（jsdom、
     Happy DOM）、または実際のブラウザエンジン（たとえば Playwright で動かす
     Chromium）。Node や DOM エミュレーターでのテストは、
     組み立てたバンドルを通したジャーニーも含め、統合テストの証拠である: それらは
     `window` レシーバーなしで `fetch` を呼び出すといったネイティブのブラウザの挙動を示せないため、
     ブラウザや E2E の証拠と呼んではならない。ブラウザでの挙動を必要とする目標を `Met` にできるのは、
     実際のブラウザエンジンでの実行だけである。それがない場合は、ここでブラウザを起動できない場合も含め、
     その目標は `Unverified` である。
   - `## Loop-Back Log`（失敗のラダーの第 3 段または第 4 段がループバックを
     発動した場合のみ）: 試行ごとに 1 つの `### Loop-back N — <ISO timestamp>` エントリを置き、
     Diagnosis / Root-cause stage / Planned fix / Estimated impact を記載する。このセクションは
     追記専用であり、このステージの再実行後も必ず残さなければならない（ループバックでの
     再入時には Modify を選び、決して Redo を選ばない — Redo は台帳を消去してしまう）。

**失敗の判定条件**: いずれかのビルドまたはテストのコマンドが失敗した場合、または
該当するいずれかの目標が `Not Met` か `Unverified` である場合、Build and Test は失敗している。例外が 1 つある:
実際のブラウザエンジンで実行されていないことだけが不足であるブラウザの目標は、
承認ゲート（Step 12）で伝え、失敗としては扱わない。失敗の処理に
入る前に、その終了経路で得られるすべての証拠を使ってマトリックスとサマリーを確定する。
定義された品質目標を弱める、緩める、下げる、無効にすることは、
決して許容される修正ではない。

**失敗時**: コマンドの失敗、`Not Met` の目標、`Unverified` の目標に対して、
同じ失敗エスカレーションのラダーを実行する:

1. **ステージ内での修正（最大 2 回）** — 根本原因がこのステージ自身の
   担当範囲（テストの設定、ビルドスクリプト、環境のセットアップ、または実行可能な目標の
   チェック）内にある場合: 失敗の証拠を読み、失敗している設定や
   足場を特定し、修正を適用し、失敗したステップを再実行して、目標の
   マトリックスを更新する。
2. **分類と影響の見積もり** — ステージ内での試行を使い果たした場合、または
   診断が上流を指している場合: 根本原因が、欠陥の大きさにかかわらず、生成された
   ソースコードやテストコードにあるのか、それとも code-generation で
   選んだアプローチ（ライブラリ/バージョン、コンテナイメージ、インスタンスタイプ、
   アルゴリズム、フラグ）にあるのかを判断する。そうであれば、差し替え可能な
   側面（より新しいイメージ、ドライバー、wheel のインデックス、CLI のフラグ）で特定できる修正を探し、その
   影響を見積もる — 工数、金銭的コスト、リスク。影響を見積もっていない工数の想定に基づいて、
   実現可能な道をスコープ外と宣言しては決してならない。
3. **自律的で上限付きのループバック** — `Construction Autonomy Mode:
   autonomous`（aidlc-state.md 内）であり、影響を見積もった修正が存在し、かつ
   test-results.md の `## Loop-Back Log` の下のエントリが 3 件未満の場合: ループバック
   プロトコルのモジュール
   （`aidlc-common/protocols/stage-protocol-loopback.md`）の
   "Build-and-Test failure loop-back" に従う。診断と
   影響を見積もった修正プランを記録し、その後、修正が名指しするユニット
   （原因がすべてのユニットにまたがる場合のみすべてのユニット）の code-generation を再び開き、
   決着を考慮したルートで前方へ再実行する。失敗した実行について、このステージの
   承認ゲートを提示してはならない。
4. **停止して確認** — モードがゲート付き（または未設定）の場合、3 回のループバックの上限を
   使い果たした場合、または特定できる修正が存在しない場合: 失敗を
   test-results.md に記録し、ループバックプロトコルのモジュール
   （`aidlc-common/protocols/stage-protocol-loopback.md`）の
   "Build-and-Test failure loop-back" で定義された、影響を見積もった停止して確認の質問を提示し、
   すべての修正候補をその見積もった影響とともに列挙する。断念するかどうかは人間が下す判断であり、
   決してエージェントの判断ではない。第 2 段で特定できる修正がまったく見つからなかった場合は、
   代わりにそのセクションの修正なしのバリアントを提示する — それは、再試行するための修正を作り出すのではなく、
   "Retry with fix" の選択肢を完全に取り除く。

**ループバックの再実行の不変条件**（ループバックプロトコルのモジュール、
`aidlc-common/protocols/stage-protocol-loopback.md`）: 成果物のみの
code-generation のワークフローは、
すべてカバー済みのゲートへ直接決着できる。一方、スティッキーなレシートモードのワークフローは
ユニットごとの作業を再発行する。どちらのルートも、ゲートの前に計画した修正と決定論的な
Modify/Keep の判断を適用し、その後、再び開いたすべての code-generation のユニットについて、
現在の試行の新しいレビューを記録する。再び開くことでそれらのユニットの以前のレビューは無効になり、
置き換えがなければ承認は失敗する。修正が名指ししないユニットは、作業と承認を保持する。ユニット優先の
反復では、再実行は自律的なスウォームではなく、ユニットごとの逐次的な処理を使う。

**単一ステージの実行**: `--single` の実行（`/aidlc --stage build-and-test
--single`）では、第 3〜4 段がジャンプを実行することは決してない — 移動すべきメインワークフローの位置が
存在しないためである。第 2 段で停止し、診断と影響を見積もった選択肢を
test-results.md に記録し、この実行の分離実行サマリーでそれらを提示する。

**成功時**: 実行したすべてのコマンドが合格し、かつ該当するすべての
目標が `Met` である場合（またはインベントリに説明用の `N/A` 行が 1 つだけある場合）にのみ、
Build and Test Summary を、準備状況の評価が成功した結果で更新する。

### Step 10: Cross-Unit Final Coverage Gate

これはステージレベルのゲートであり、Construction フェーズの境界ではない。
`{{INVOKE}} engine sensor-traceability --stage build-and-test` を実行する: これは、この実行で
トレースしたすべての要件と受け入れ基準について、構築済みか、
理由とともに `N/A` とマークされているか、そしてそれを構築するユニットとファイルを出力する。その出力から
`<record>/construction/build-and-test/cross-unit-traceability.md` を書き、
合否の判定、ID ごとのカバレッジ、担当するステージ/ユニット、対象
ファイル、カバーされていないすべての要素を記載する。カバーされていない ID はすべて、
承認ゲートで必ず提示しなければならない build-and-test の指摘事項である。

### Step 11: Completion Handoff

`{{INVOKE}} engine orchestrate report --stage build-and-test --result <outcome>` を通じて、
完了を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 12: Completion

完了メッセージと承認ゲートを提示する:

```
# :hammer: Build and Test Complete
```

生成したすべてのテスト手順セットのサマリー、準備状況の評価を示し、その後:

ブラウザでの挙動を必要とする目標、完了条件、または受け入れ基準が、
Node や DOM エミュレーターでしかチェックされなかった場合は、そのことを 1 回、1 行で伝え、
何をどこで実行したかを明記する。たとえば: "The transfer journey was
checked in Happy DOM, not a real browser."（送金のジャーニーは、実際のブラウザではなく Happy DOM でチェックされました。）利用者は、
それを目の前にしてゲートで判断する。

```
**Review:** `<record>/construction/build-and-test/`
```

承認ゲート: 厳密に 2 択（Approve / Request Changes）。

## Sensors

このステージは、`<record>/construction/build-and-test/` 配下にテスト手順の
Markdown ファイルを生成し、実行の一部としてプロジェクトのビルドと
テストのコマンドを実行する。手順の成果物は、
Markdown の形式をチェックするセンサーが対象とする、エージェントが作成した出力である。ビルド
自体は終了コードと結果のレポートを出力する。

インポート: `required-sections`, `upstream-coverage`, `type-check`.

上流の対象: `code-generation-plan`, `unit-test-instructions`, `code-summary`.

`type-check` は、テスト生成中に変更された、該当する TypeScript/TSX のコードを
検査する。

`linter` は意図的にインポートしていない。標準の lint はこのステージが駆動する
ビルドパイプラインの中で実行されるため、インポートすると指摘が重複する。
ビルドの終了コードが引き続き正式なシグナルである。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
