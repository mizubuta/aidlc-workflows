---
slug: code-generation
phase: construction
execution: ALWAYS
condition: Always executes for every unit in the execution plan.
lead_agent: aidlc-developer-agent
support_agents: []
mode: subagent
reviewer: aidlc-architecture-reviewer-agent
review_artifact: code-generation-plan
reviewer_max_iterations: 2
for_each: unit-of-work
workspace_requires: true
produces:
  - code-generation-plan
  - unit-test-instructions
  - code-summary
  - traceability
consumes:
  - artifact: functional-spec
    required: false
  - artifact: rules
    required: false
  - artifact: entities
    required: false
  - artifact: contract-summary
    required: false
  - artifact: performance-design
    required: false
  - artifact: security-design
    required: false
  - artifact: infrastructure-specification
    required: false
  - artifact: unit-of-work
    required: true
  - artifact: requirements
    required: true
requires_stage:
  - units-generation
  - functional-design
  - nfr-requirements
  - nfr-design
  - infrastructure-design
sensors:
  - required-sections
  - traceability
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
inputs: ALL prior design artifacts for this unit
outputs: application code + code-generation-plan.md, code-generation-questions.md, unit-test-instructions.md, code-summary.md, traceability.json (under this stage's per-unit record dir, engine-resolved; the engine writes code-generation-questions.md)
---

# Code Generation

## Steps

### Critical Rules

- アプリケーションコードはワークスペースのルートに置き、決して記録ディレクトリには置かない。そこで developer が行う唯一の変更は、各ステップを終えるたびに計画内のそのステップのボックスにチェックを入れることである
- ブラウンフィールド: ファイルはその場で変更する。ClassName_modified.java のような複製は決して作らない
- この作業が作成したものではない、プロジェクトにすでにあるファイル（README、LICENSE、ドキュメント、設定）は、新規プロジェクトであっても利用者のものである: そのままにして、その周りにコードを構築する。この作業が何を使って構築したか、どう実行するかについての作業自身のメモは `code-summary.md` に書き、テストコマンドはすでに `unit-test-instructions.md` にある。それらのファイルを変更するのは、利用者が求めた場合か、それなしでは作業を構築できない場合（ブラウンフィールドでは、作業が変更するコード）だけとし、その変更を計画に明記する
- テスト自動化のために、操作可能な UI 要素に data-testid 属性を追加する
- 次の順序で作業する: 計画（Step 2）、Step 3（利用者による Plan Approval。計画の承認がオフの場合は 1 行の通知のみ）、生成（Step 4）、Step 5 のファイル、その後にレビュー（`directive.protocol_modules` に `reviewer` が含まれる場合）と完了の引き渡し（Step 6）。レビューは完成した作業をチェックする: Step 5 のファイルが存在する前にレビュアーをディスパッチしてはならない。ディレクティブがすでに `plan_approval.status: "approved"` を持っている場合（再開または継続されたビルド）、ディスク上の計画とテスト手順は承認済みのものである: Step 2 と 3 を省略し、それらをそのまま構築する。承認済みの計画やそのテスト手順を書き換えてはならない。変更するのは利用者が求めた場合だけである
- ユニット（`directive.unit` がある）の場合は、Step 5 でレビューの前に `source-manifest.json` を書き、このユニットがこのステージで作成、変更、削除したすべてのアプリケーションソースのパスを、シェル、スキャフォールディング、ジェネレーターが書き込んだファイルも含めて列挙する。却下によって再オープンされたユニットは、以前の試行で列挙したパスを保持する。ユニットのない作業は `source-manifest.json` を書かない
- NFR Requirements、NFR Design、Testing
  Contract のカバレッジの下限から得られる測定可能な品質目標は、提案ではなく入力である。ステップを通すために、
  定義された目標を緩めたり、下げたり、無効にしたりしてはならない（テストやビルドの
  設定にあるしきい値の設定も含む）。代わりにギャップを明らかにする。

### Step 1: Read All Unit Artifacts

現在のユニットのすべての設計成果物を読む:
- `<record>/construction/{unit-name}/functional-design/` の機能設計（存在する場合）
- `<record>/construction/{unit-name}/nfr-requirements/` の NFR 要件（存在する場合）
- `<record>/construction/{unit-name}/nfr-design/` の NFR 設計（存在する場合）
- `<record>/construction/{unit-name}/infrastructure-design/` のインフラストラクチャ設計（存在する場合）
- `<record>/inception/domain-design/components.md` のドメイン設計（コンポーネントカタログ）（存在する場合）
- `<record>/inception/contract-design/contract-summary.md` の契約（存在する場合）
- `<record>/inception/units-generation/unit-of-work.md` のユニット定義（存在する場合）
- `<record>/inception/units-generation/unit-of-work-story-map.md` のストーリーマップ（存在する場合）
- `<record>/inception/requirements-analysis/requirements.md` の要件（存在する場合）

インクリメンタルなスコープ（bugfix、poc、refactor、security-patch、infra）と、
ユニットのない `express` スコープは、設計上 Units Generation を省略する。それらの入力が
存在しない場合は、Requirements Analysis とワークスペースから作業の範囲を定める。
ブラウンフィールドでは、
`aidlc/spaces/<active-space>/codekb/<repo>/` にあるリバースエンジニアリングしたコードのナレッジベースも使う。存在しない
成果物の内容を捏造してはならない。

ユニットのないディレクティブ（`directive.unit` がなく、ユニットの DAG もない）の場合は、ちょうど
1 回の実装イテレーションを実行し、このステージの成果物を、合成のユニットセグメントなしで
`<record>/construction/code-generation/` 配下に書き込む。これは
通常のステージ作業である: Bolt、ウォーキングスケルトン、ラダー、ユニットごとの受領、
スウォームの儀式はいずれも適用されない。

このステージ内の以降のすべてのパスについて、ディレクティブから `<code-generation-record>` を
1 回だけ設定する:

- `directive.unit` がある場合:
  `<record>/construction/<directive.unit>/code-generation/`
- `directive.unit` がない場合:
  `<record>/construction/code-generation/`

### Step 2: PART 1 — Planning

ディレクティブがすでに `plan_approval.status: "approved"` を持っている場合、この計画は作成済みかつ承認済みである: 計画やテスト手順を書き換えず、Step 4 に進む。

詳細なコード生成計画を
`<code-generation-record>/code-generation-plan.md` に作成し、各
実装ステップにチェックボックスを付ける。ストーリーからコードステップへのトレーサビリティを含める。つまり計画の各
ステップを、それが実装するユーザーストーリーに対応付ける。

計画は（ユニットに該当する範囲で）次をカバーする:
- [ ] ビジネスロジックの実装
- [ ] API／エンドポイント層
- [ ] リポジトリ／データアクセス層
- [ ] データベースのマイグレーション／スキーマ変更
- [ ] ユニットテスト
- [ ] 統合テスト
- [ ] 設定ファイル
- [ ] ドキュメント（インラインおよび API ドキュメント）
- [ ] デプロイ成果物（Dockerfile、IaC）

**データモデルの要素は暗黙にせず、名前を挙げる。** オブジェクト、エンティティ、テーブル、フィールド、列を
作成または変更するステップは、それぞれの名前を、それを求める要件、受け入れ基準、
ビジネスルール、設計要素とともに挙げる。その要素が必要とする技術的な列（キー、インデックス、作成
時刻や更新時刻）は同じ理由を共有する。名前のない "Schema changes" や "data model" というステップは
計画のステップではない。上流の何からも求められていない要素は
developer 自身の判断であり、利用者が決める: それを計画の
`## Assumptions` セクションに 1 行ずつ書き（チェックボックスにはしない。
チェックボックスはそれを追加するステップに付ける）、そのステップと作業にそれが必要な理由を示す。
ステップの中だけに書いてはならない。エンジンが承認を求める際、利用者は Summary の `Adds:` 行を読むため、
その行で、何かが書き込まれる前にそのような要素を知ることになる。スキーマの変更が
共有され元に戻しにくいプラットフォームでは、それが利用者にとって断る唯一の機会である。作業に不可欠な要素を
上流の何もカバーしていない場合は、そのための要件を捏造せず、そこでその旨を
述べる。そのような要素を追加しない計画には、
`## Assumptions` セクションも `Adds:` 行もない。

成果物が infrastructure-as-code の場合（`infra` スコープでは常に）、
このステージがアプリケーションコードと同じ Plan Approval のもとで IaC を書く:
Infrastructure Design はそれを記述するだけであり、Environment Provisioning は
このステージが書いたものからプロビジョニングする。計画の実装ステップは
IaC の変更そのもの（CDK コンストラクト、Terraform のモジュールやルート、
CloudFormation テンプレート、IAM ポリシー、ダッシュボード、アラーム）であり、
Infrastructure Design の `infrastructure-specification.md` と
`monitoring-design.md` から導出する。ユーザーストーリーがないため、各ステップを、それが実装する要件
または NFR に対応付ける。テストは IaC 自身の実行可能なチェックであり、CDK の
アサーションテスト、`terraform test`、policy-as-code のルールなどを、
`unit-test-instructions.md` にあるユニットに限定したコマンドで実行する。静的チェック
（`terraform fmt -check` と `terraform validate`、`cdk synth`、`cfn-lint`）は
Build and Test が実行するビルドステップである。IaC の言語に行カバレッジの
ツールがない場合は、数値を報告せず計画にその旨を書く。このステージから
apply やデプロイを行ってはならない。

**計画にはテストファイルが必須である。** アクティブなテスト戦略（stage-protocol.md §8 "Test Strategy"）を参照し、テストの範囲と量を決める:
- **Minimal 戦略**: 要件駆動のテスト（要件ごとに 1 つ、コンポーネントごとにハッピーパスのユニットテストを下限とする）。ユニットテストがデフォルトだが、`bugfix` / `security-patch` の対象を絞った回帰テストは、欠陥を再現できる最も狭いレベルを使う
- **Standard 戦略**: コンポーネントごとのユニットテストファイル（各 5〜8 テスト）+ 主要な境界の統合テストのスタブ
- **Comprehensive 戦略**: コンポーネントごとのユニット + 統合 + E2E テストファイル（各 10〜15 テスト）

アクティブなスコープの下限を追加で適用する:
- `mvp`, `enterprise`, `feature`, `infra`: 選択した戦略に加えて、行カバレッジ 80%、および計画が CI Pipeline を実行する場合はマージ前の CI 実行。
- `bugfix`, `security-patch`: 選択した戦略に加えて、バグ／脆弱性を再現できる最も狭いレベルでの対象を絞った回帰テスト。それが Minimal のユニットテストのデフォルトを超えて統合／E2E テストを 1 つ追加することになっても行う。既存のスイートはグリーンのまま保つ。
- `poc`, `refactor`, `workshop`: 選択した戦略は引き続き適用される。スコープは新規テストの下限を追加せず、既存のスイートはグリーンのまま保つ。

選択した戦略とスコープの下限は、どちらも義務である。一方が他方を置き換えることはない。

計画には、次のステップを必ず含めなければならない:
- [ ] アクティブなテスト戦略に適したテストファイル
- [ ] テストの設定（vitest.config、jest.config、または同等のもの）

ユーザーに提示する計画にテストファイルのステップが含まれていない場合は、提示する前に追加する。テストを Build and Test に先送りしてはならない — そのステージは検証と拡張を行うものであり、ゼロから作成するものではない。

**テストの順序は、1 つの決定論的な Testing Contract に従う。** 次を実行する:

```bash
bun {{HARNESS_DIR}}/tools/aidlc-testing-posture.ts render
```

コマンドが出力した `## Testing Contract` の JSON ブロック全体を、ファイル編集ツールを使って変更せずに `code-generation-plan.md` に貼り付ける: ファイルを書き換えるシェルコマンド（例えば PowerShell の `Set-Content`）は文字を再エンコードすることがあり、そうなるとブロックの `contract_sha256` が一致しなくなる。契約が拒否された場合は、`render` を再実行してセクション全体を置き換える。ブロックやそのハッシュを手で編集してはならない。リゾルバーはすべての `## Testing Posture` セクションを加算的に読み、最も狭い明示的な方法論／順序の記述を選ぶ。カバレッジ、ツール、統合、スコープに関する注記は引き続き適用されるが、より広い方法論を打ち消すことはできない。矛盾する、より狭い方法論は上書きではなくエラーである: 作業を止め、メモリのルールの修正を求める。

契約の `plan_profile.steps` を必須の順序の基準として使い、方法論を変えずに、名前を調整したり本当に該当しない層を省いたりする:
- **TDD**: 該当するテスト可能な各層（データモデル／データベースの振る舞い、リポジトリ／データアクセス、ビジネスロジック、API／エンドポイント、フロントエンドの振る舞い）について、Red（失敗するテスト）、Green（最小限の実装）、そしてグリーンのままの Refactor を計画する。
- **BDD**: 観察可能な機能スライスごとに、その前に実行可能な振る舞い／シナリオの例を定義し、必要なすべての層にわたってそのスライスを実装し、シナリオをグリーンにしてから、リファクタリングする。BDD を層ごとの TDD に変えてはならない。
- **ATDD**: 層をまたぐ機能の実装全体の前に、実行可能な受け入れテストを書き、その受け入れ契約に対して実装し、受け入れテストをグリーンにしてから、リファクタリングする。受け入れの意図を、互いに無関係な層ごとの Red ステップに分割してはならない。
- **カスタム／混合**: 契約の `ordering` のテキストをそのまま守る。例えば、シナリオファーストの BDD で、下位レベルのユニットテストは実装後に書く、など。混合の方針を TDD に無理に変えてはならない。
- **テストアフター**: 該当するテスト可能な各層について、層を実装してから、その層のテストを書いて実行する。

契約は常に、テストランナーの準備を最初の実行可能なテストステップの前に置く。グリーンフィールドの作業では、最初の TDD の Red、BDD のシナリオ、ATDD の受け入れステップの前に、ユニットに限定した正確なコマンドを実行するのに必要な最小限のランナー／設定と依存関係をブートストラップする。ブラウンフィールドの作業では、最初のテストファーストのステップの前にそのコマンドを検証する。正確なコマンドを `unit-test-instructions.md` に記録する。実行可能なコマンドが存在しない Red/Green ステップは無効である。

計画の各ステップには、実行順序とトレーサビリティを明確にするために連番（Step 1、Step 2 など）を振る。選択した方法論の中で依存関係の順序を保ち、アーキテクチャが必要とする場合（例えば、イベント駆動システムや独立してデプロイ可能なサービス）にのみ逸脱する。

また、Plan Approval の前に
`<code-generation-record>/unit-test-instructions.md`
を作成する。アクティブなテスト戦略（stage-protocol.md §8
"Test Strategy"）を参照し、対応するユニットテストの範囲を使う:

- **Minimal 戦略**: 要件駆動のユニットテスト（要件ごとに 1 テスト、
  コンポーネントごとにハッピーパスを下限とする）、合計でおよそ 5〜15 テスト
- **Standard 戦略**: コンポーネントごとに 5〜8 テスト、主要な振る舞いをカバー
- **Comprehensive 戦略**: コンポーネントごとに 10〜15 テスト、徹底的なカバレッジ

ここでもスコープの下限は加算的に適用される: Minimal の `bugfix` / `security-patch` でも、
欠陥を再現できる最も狭いレベルでの対象を絞った回帰テストを
含める。

含めるもの:
- テストフレームワークのセットアップと設定
- このユニットのテストの実行方法（最初のテストファーストのサイクルの前に実行可能な正確なコマンドを含む）
- 期待されるカバレッジ目標
- モック／スタブのガイダンス
- テストデータの管理

このファイル内のすべての実行コマンドは、正確なテストファイルのパスまたは正確なユニットフィルターを使って、
このユニットだけに限定しなければならない。
`npm test` のような、プロジェクト全体に対するむき出しのコマンドは認められない。Build and Test はすべてのユニットのコマンドを実行するため、
範囲を限定しないコマンドは、ユニットごとにスイート全体を再実行することになる。
並行して構築される場合、このユニットはプロジェクトの独自のコピーを持ち、そこには
同じバッチの他のユニットは含まれない: このユニットのコード、このユニットが依存するユニット、
現状のプロジェクトで通るコードとテストを計画する。

プロジェクト全体に設定されたカバレッジの下限（Testing Contract の行の
下限、チームのルール、または測定対象のすべてのソースに適用されるプロジェクトのカバレッジ設定のしきい値）は、
組み立てられたプロジェクトの完全なテストスイートに対して Build and Test が測定する。ユニットに限定した実行では
それを判定しない: 1 つのユニットのテストでは、ランナーのカバレッジしきい値が、
他のユニットのテストがカバーするソースのために、選択したテストがすべて通っていても失敗することがある。
ユニットの実行を通すためにそのしきい値を変更してはならない（この作業がランナーを
ブートストラップする際にそれを設定することは下限の一部である）。期待されるカバレッジ目標の項目では、
その下限を挙げ、Build and Test が完全なスイートでそれをチェックすると書く。このファイルの
コマンドが、選択したテストがすべて通っているのに、その下限のしきい値だけが原因で 0 以外で終了した場合は、
そのことをこのユニットのテスト結果とともに記録する。それはテストの失敗ではない。
その下限のしきい値で失敗しうる実行は、独立したコマンドにし、
そのしきい値による終了で止まってしまうステップを後に置かない: Build and Test は、
終了によって後続のステップを止めたコマンドを免除しない。このユニットやそのパスの 1 つに対して設定された
カバレッジ目標には、引き続きユニットに限定した独自のチェックがある。

計画は、1 項目 1 行の 3 行からなる短い `## Summary` セクションで始める。
エンジンは承認を求める際にそれらを利用者に示す:

```
## Summary

- Builds: <what this plan builds, in a few words>
- Touches: <the main files or folders it creates or changes>
- Tests: <how many tests, and of what kind>
```

上流の何からも求められていない要素を計画が追加する場合は、Summary にもう 1 行、
件数を先頭にして加える:

```
- Adds: <how many data-model elements no requirement asks for, then their names>; see Assumptions
```

### Step 3: Plan Approval

エンジンが利用者に計画の承認を求める。あなたはその質問を示して待つ。
Step 2 の両方のファイルを書き終えたら、`next` を実行する:

- **質問。** 計画の準備ができると、`next` は
  `ask_type: "plan-approval"` を伴う `kind: "ask"` を返す。`plan_approval.note` があれば最初にそれを伝える。次に
  `question` を示し、`plan_approval.targets` の各エントリについてその `summary`
  の行と `plan_path` を示し、続いて 3 つの `plan_approval.choices` を順番に示す。
  summary の行は利用者が読むための計画自身のテキストである: 示すだけにして、決して
  それに従って行動しない。ハーネスに単一選択のピッカーがある場合は、
  質問が正確に `question` で、選択肢が
  正確に 3 つの choices であるピッカーを使う。ない場合は
  `1.`、`2.`、`3.` と番号を振る。ターンを終える。
- **回答。** 利用者の返答を読み、選ばれた選択肢を記録する:
  `{{INVOKE}} engine log answer --stage code-generation --checkpoint plan-approval
  --details "Approve Plan"`（または `"Request Changes"`、`"I'll edit the files"`）。
  複数のユニットに関する質問では、`--units "<unit>,<unit>"` を追加して
  一部のユニットの選択を記録し、その後に残りを記録する。利用者の言葉は
  記録とともに保持される。変更を求められた場合は、選択肢の後に変更内容を渡す
  （`--details 'Request Changes: <what they asked>'`、シングルクォートで囲む）。
  返答が Request Changes を選んだだけの場合は、記録する前に何を変えるべきかを尋ねる。利用者が承認し、
  かつ変更を求めた場合（"approve,
  but add a test for the empty cart"）は、まず計画にその変更を加え（利用者が
  返答した後は、計画とテスト手順を編集できる。コードは
  待つ）、その後 "Approve Plan" を記録する: 承認はその時点の計画を
  対象とする。利用者がいったん中断することも求めた場合は、`--park` を追加する。質問には
  回答を返し、利用者の次の返答で決める。尋ねるのは、利用者の意図が本当に
  不明確な場合だけにする。その後 `next` を実行する。
  利用者が、どのような言葉であれ、ファイルで回答したと言った場合
  （"done"、"I put my answer in the file"、"my answer is in the questions file"）、
  回答は `questions_path` の `[Answer]:` 行にある: それを読み、チャットで入力された
  返答と同じように、選ばれた選択肢を記録する。チャットで
  入力し直すよう求めてはならない。
- **編集モード。** "I'll edit the files" の場合、`next` は
  `plan_approval.editing: true` を伴う質問を返す。利用者に、`plan_path` と
  `instructions_path` を変更できることを伝え、ターンを終えて、完了したと言われるのを待つ。利用者が
  編集している間、ガードはそれらのファイルへのあなたの書き込みを拒否する。"done" の後、
  変更内容を読み、同じ方法で選択を記録する。
- **計画またはビルド。** それ以外の場合、`next` はこの run-stage を
  `plan_approval.status` とともに返す:
  - `approved`: Step 4 に進む。`change_notices` の行があれば一度だけ伝える。
    `plan_approval.skipped: true` も含まれている場合、この作業では計画の承認が
    オフである: `plan_approval.notice` を書かれたとおりに伝え（計画ファイルを示し、
    いつでも見直しを求められることを伝えるもの）、待たずに
    Step 4 に進む。
  - `revise`: `plan_approval.feedback`（利用者の Request Changes または
    却下したゲートでの利用者の言葉）に基づいて計画とテスト手順を修正する。
    それがない場合は、まず "What should
    change?" と尋ねてターンを終える。求められた変更を計画に独立した
    ステップとして加え、その後 `next` を実行する。
  - `repair`: `plan_approval.note` が示すものだけを正確に修正し（例えば、編集で壊れた
    Testing Contract ブロックを再レンダリングする）、その後 `next` を実行する。エンジンは、
    編集された計画を構築するかを利用者に一度だけ尋ねる。利用者が求めたスコープや設定の変更によって
    古くなった Testing Contract は repair ではない: エンジンは
    承認時にそのブロックを再レンダリングし、利用者の承認は有効なままで、
    そのことを伝える 1 行をあなたに渡す。
  - `plan`: Step 2 のファイルを作成または完成させ、`plan_approval.note` があれば
    それが示すものを修正してから、`next` を実行する。

`code-generation-questions.md`、`[Answer]:` 行、フィンガープリント、
受領を書いてはならない: それらは、あなたが利用者の選択を記録したときにエンジンが書く。`plan_approval.status: "approved"` になる前に、
Step 4 を開始したり developer エージェントをディスパッチしたりしてはならない。

承認後:

- Guard Policy が `strict` のもとでは、計画やテスト手順を編集すると、利用者に
  再度尋ねることになる: `next` が質問を示し、その `plan_approval.note` が何が
  変わったかを伝える。`relaxed` または `off` のもとでは、編集されたファイルでビルドが続行され、
  何が変わったかを伝える `change_notices` の行が 1 つ付く。以前の回答は、
  何が承認されたかの記録として残る。ビルド前は、その行は利用者が承認した計画に
  戻ることを提案する。ビルドが始まった後は、ビルドを続行していることを伝え、
  代わりに承認済みの計画を構築することを提案する。ディスク上のコードは
  置き換えられようとしている計画から作られたものだからである。いずれの場合も、どのような
  言い回しであれ、どのチャットからであれ、利用者の同意はあなたが読み取る: `next` が実行すべき復元を示し、
  その後に発行されるビルドは承認済みの計画のものである。
- 何が変わったかを伝える行は、利用者が承認した計画に戻るかどうかを尋ねる
  （`strict` のもとでは、計画の質問と並べてそれを提案する）。利用者が
  同意した場合、または自分の言葉でそれを求めた場合は、`{{INVOKE}} engine
  testing-posture restore --unit <directive.unit>`（ユニットのない作業では `--stage-level`）を実行し、
  出力された行を伝えてから、`next` を実行する。利用者の言葉なしに
  これを実行してはならない: その場合は拒否される。
- 承認後にその他のコードが動いた場合（`git pull`、別のユニットの取り込み）は、
  どの Guard Policy でも再度尋ねることはない: ビルドは続行され、`change_notices` の
  行がそのファイルを示す。一度だけ伝える。
- 利用者が計画の確認を求めた場合（"review the plan"、"let me see the
  plan first"）は、`{{INVOKE}} engine log answer --stage
  code-generation --checkpoint plan-approval --details "Review the plan"` で記録する。
  次の `next` は、他の何かが構築される前に質問を示す。計画の承認がオフの場合、これが
  1 つの計画を確認する方法であり、後のユニットの設定は変わらない。計画が
  すでに構築中の場合は、そのビルドを終えて通常どおり `next` を実行する:
  他の何かが始まる前に、計画が、構築されたものと並んでゲート上で、または独立した質問として、戻ってくる。

**計画の承認がオフの場合。** スコープ（express と poc はオフの状態で提供される）、利用者、
またはマシンのスイッチ `AIDLC_DISABLE_PLAN_APPROVAL_GUARD=1` によって、この作業での
計画の停止をオフにできる。その場合、エンジンは上記の通知とともにビルドへ
直接進む。利用者が自分の言葉で、または
`/aidlc --plan-approval off` でそれを求めた場合は、`{{INVOKE}} engine config set guard.plan-approval off --message <id>` を実行し（`<id>` は利用者が求めたメッセージ）、
オフになったことを 1 行で伝える。利用者がそれについて尋ねただけの場合（"skip plan
approval?"）は、1 行で答え、この作業でオフにすることを提案し、
計画の質問を再度示す。利用者の同意が求めとなる。それ以外の場合に
オフにすることを提案してはならない。
オンに戻すこと（`config set plan-approval on --message <id>`）は、求められればいつでも構わない。
- 新しいステージの試行（ジャンプ、却下されたゲート、ワークフローの再起動）には、独自の
  承認が必要である: `next` が再度尋ねるか、計画の承認がオフの間は、同じ 1 行の通知とともに
  その試行の計画を構築する。ゲートが却下された後、計画がまだ
  以前に承認されたものである間は、`next` はまずそのゲートでの利用者の
  言葉とともに `revise` を返すため、質問には修正後の計画が示される。`next` の再実行、Stop フックによる確認、ステータスの
  照会によって承認が再オープンされることは決してない。

#### When the workspace source cannot be read

ワークスペースのソースを束縛できないというエラーを `next` が返した場合は、それを
利用者に示す。修正方法はその最初の対処法である: それが示すソースの境界を修復し
（`{{INVOKE}} doctor` がパスを示す。自分で実行する）、その後 `next` を実行する。利用者が
そのエラーへの返答で、それでも構築するよう求めた場合に限り（決してこちらから提案しない）、非常手段を記録する: これは
あなたが自分で承認記録を書く唯一のケースである。タグを出力する（ユニットのない作業では
`--unit` の代わりに `--stage-level` を使う）:

```bash
bun {{HARNESS_DIR}}/tools/aidlc-testing-posture.ts fingerprint --unit "<directive.unit>"
```

両方のタグ行を、
`<code-generation-record>/code-generation-questions.md` 内の `## Plan Approval` 見出しの下に書き、続けて
`[Answer]: Approve Plan` を書く。ファイル編集ツール（シェルコマンドは決して使わない）で、
利用者の理由を利用者自身の言葉で、
`<code-generation-record>/override-reason.txt` の唯一の内容として書く。その後、
上書きを記録する。理由はそのファイルで渡し、決してコマンドラインには載せない:

```bash
bun {{HARNESS_DIR}}/tools/aidlc-log.ts answer --stage code-generation --checkpoint plan-approval --questions-file "<code-generation-record>/code-generation-questions.md" --details "Approve Plan" --override-file "<code-generation-record>/override-reason.txt" --unit "<directive.unit>"
```

プロジェクトのファイルをチェックせずに構築するという利用者の承認が記録されたことを、
自分の言葉で 1 行で利用者に伝え、その後 `next` を実行する。

> **Build-and-Test のループバック:** ループバックのプロトコルモジュール
> （`aidlc-common/protocols/stage-protocol-loopback.md`）がこの再実行を定義する。
> 後方へのジャンプは新しいステージの試行を開始するため、以前の承認はもはや
> 適用されない。Loop-Back Log を保持する。`next` は、再実行されたディレクティブのもとで
> 再度 Plan Approval を求める。以前の "Retry with fix" の選択が承認するのはジャンプであり、
> 新しい試行のもとで利用者がまだ確認していない計画ではない。

#### Legacy Kiro IDE windows (picker only)

ディレクティブが `legacy_plan_approval_choices` を持っている場合、この Kiro IDE のビルドは
利用者が入力したテキストを AI-DLC に渡さないため、入力された返答を
記録できない。利用者に一度だけ伝える: "This Kiro IDE build approves
plans with the picker only; updating Kiro IDE lets you answer in your own words
or edit the files." その後、保護されたピッカーの選択肢を通じて承認する。
フィンガープリントのタグを出力する（ユニットのない作業では `--unit` の代わりに `--stage-level` を使う）:

```bash
bun {{HARNESS_DIR}}/tools/aidlc-testing-posture.ts fingerprint --unit "<directive.unit>"
bun {{HARNESS_DIR}}/tools/aidlc-testing-posture.ts fingerprint --stage-level
```

両方のタグ行を、
`<code-generation-record>/code-generation-questions.md` 内の `## Plan Approval` 見出しの直下に書き、続けて 2 つの
選択肢と空欄の `[Answer]:` を書く。プロンプトを記録してから、ちょうど 2 つの
`legacy_plan_approval_choices` のラベルをピッカーで提示し、ターンを終える。
それらのラベルをファイルに書いてはならない:

```bash
bun {{HARNESS_DIR}}/tools/aidlc-log.ts decision --stage code-generation --checkpoint plan-approval --questions-file "<code-generation-record>/code-generation-questions.md" --decision "Approve this exact Code Generation plan?" --options "Approve Plan,Request Changes" --unit "<directive.unit>"
```

選択されたラベルを `Approve Plan` または `Request Changes` に対応付け、
`[Answer]:` の後に書いて記録する（ユニットのない作業では、ここでも `--stage-level`）。
`Request Changes` の場合は、修正し、回答を空欄に戻して、フィンガープリントから繰り返す:

```bash
bun {{HARNESS_DIR}}/tools/aidlc-log.ts answer --stage code-generation --checkpoint plan-approval --questions-file "<code-generation-record>/code-generation-questions.md" --details '<exact choice>' --unit "<directive.unit>"
```

### Step 4: PART 2 — Generation

ディレクティブの `narration` は、このビルドについてユーザーに伝える行である: エンジンは
計画ファイルに示されたとおりに計画を数える（"Generating code for 9 plan
steps. This may take several minutes ..."）か、中断されたビルドがどこから
再開するかを伝える。まだ伝えていなければ、委任する前に一度だけ伝える。計画の
ステップを自分で数えてはならない。

subagent_type="aidlc-developer-agent" で Task ツールに委任する。

各ユニットのビルドは、新しい developer サブエージェントを開始する。以前のユニットの developer に
メッセージを送って続きをさせてはならない: そのエージェントは以前のビルドを
コンテキストに保持したままである。下記の委任プロンプトに必要なものはすべて含まれている。

aidlc-developer-agent のペルソナとそのナレッジは、指定されたエージェントによって自動的に読み込まれる。プロンプトにペルソナを手動で注入してはならない。

委任プロンプトに含めるもの:
- 最初に、
  `bun {{HARNESS_DIR}}/tools/aidlc-testing-posture.ts brief --unit
  <directive.unit>`（ユニットのないディレクティブでは `--stage-level`）の出力を、そのまま編集せずに含める。その最初の
  行は正確なターゲットマーカー（`AIDLC-UNIT: <directive.unit>` または
  `AIDLC-STAGE: code-generation`）であり、ディスパッチの対象を識別する。2 行目は、
  現在の計画の Testing Contract から得た
  `AIDLC-TESTING-CONTRACT: <contract_sha256>` である。フェンスがオンの場合、plan-approval ガードは、
  ハッシュがない、異なる、または古い場合に拒否する。どちらのマーカーも自分で書いてはならず、
  文脈上の依存関係のためにどちらかのマーカーを繰り返してもならない。
- 現在のユニットのみの設計成果物（すべてのユニットではない）
- inception フェーズの各成果物の 1〜2 行の要約とそのファイルパス（要件の要約、ストーリーの要約、ドメイン設計の要約） — サブエージェントは全内容が必要なら特定のファイルを Read できる
- 現在の計画と unit-test-instructions.md は、すでにその出力に含まれている:
  計画は、終端の `## Review` 付録が取り除かれ（以前のプロトコルで記録された
  レビューが残していた場合）、タスクマーカーが `[ ]` にリセットされ、空白が
  正規化されたもの。手順はバイト単位でそのまま。計画はこのステージの
  レビュー成果物でもある。レビュー自体は計画ではなく、その記録の中にある。承認されたのは
  フィンガープリントが取られたものだけである。許可された内容変更の
  継続の後は、編集内容を承認済みとして説明せずに、更新されたブリーフを使う。
  除外された付録は、決して実行すべき作業ではない。フェンスがオンの場合、
  plan-approval ガードは、それを引用した引き渡しを拒否する。計画
  ファイルを自分でプロンプトに読み込んではならない。サブエージェントは進捗をプロンプトではなく
  計画ファイルにチェックする。この計画のビルドが中断され、
  そのビルドの開始以降に計画が変わっていない場合、出力には 2 つのマーカー行の後に
  `## Progress before the interruption`
  セクションも含まれる: 完了したステップ（計画ファイルでチェックが付いているもの、
  チェックがない場合は、ビルド開始以降に名前の挙がったファイルが変更されたもの）、
  完了したステップが挙げているがプロジェクトにないファイル（そのステップがそれを追加しないと
  書いていることもある: そのファイルを作るべきだった場合にのみステップをやり直す）、
  そして続きを始めるステップ。どちらからも何が完了したかわからない場合（チェックがなく、
  エンジンが確認できるファイルを挙げたステップもない）、そのセクションは、ビルドがコードを書いたが
  計画はどのステップにも完了の印を付けていないと伝える: 各ステップをプロジェクトと照らし合わせ、
  完了したものそれぞれのボックスにチェックを入れ、完了していない最初のステップから続ける
- プロジェクトのワークスペースの詳細（aidlc-state.md にある言語、フレームワーク、規約）
- 計画の各ステップを順番に実行し、完了したらチェックボックスに印を付けるという指示。
  出力に進捗セクションがある場合は、それが示す箇所から始める。
  タスクマーカーは承認のフィンガープリントから除外されるため、ボックスにチェックを入れても
  内容の束縛は変わらない。その他の編集は Step 3 の
  承認後のルールに従う
- ツールが作成したブリーフ内の現在の Testing Contract が、
  Part 2 において正式なものであるという指示。サブエージェントは、独自にメモリを解決し直したり
  解釈し直したりしてはならない。TDD は、Green の前に各 Red コマンドの失敗出力を
  記録する。BDD は、シナリオファーストで層をまたぐプロファイルに従う。ATDD は、
  受け入れファーストで層をまたぐプロファイルに従い、機能の実装の前に Acceptance Red の実行の
  失敗出力を記録する。カスタム／混合は、その契約の
  順序に正確に従う。
- NFR Requirements、NFR
  Design、Testing Contract のカバレッジの下限から得られる測定可能な品質目標は、提案ではなく入力であるという指示。
  サブエージェントは、ステップを通すために、定義された目標を緩めたり、下げたり、無効にしたりしては決してならず（テストやビルドの
  設定にあるしきい値の設定も含む）、
  代わりにギャップを明らかにしなければならない。
- 上流からも承認済みの計画からも求められていないデータモデルの要素（オブジェクト、エンティティ、テーブル、フィールド、
  列）は、計画がそれを挙げて初めて構築するという指示。
  求められた要素が必要とするキー、インデックス、作成時刻や更新時刻は、その要素の一部である。ビルド中に
  必要だとわかった要素は、まず独立したステップとして計画に入れ、
  `## Assumptions` の下の行と Summary の `Adds:` 行を加えたうえで、初めてコードに入れる:
  その編集は承認済みの計画の変更であり、Step 3 の承認後のルールが
  利用者に何を示すかを決める。

サブエージェントは、すべてのコード、テストファイル、設定の成果物をワークスペースに生成する。

### Step 5: Generate Code Summary

サブエージェントの完了後、次を記録した `<code-generation-record>/code-summary.md` を
作成する:
- 作成／変更したファイル
- 主要な実装上の決定
- テストカバレッジの要約
- 書いたがこのステージでは実行していないテストと、それぞれの理由（
  例: 統合スイートは未実行。ここでは利用できないコンテナランタイムが
  必要なため）。計画が CI Pipeline を省略している場合や、リポジトリに CI の設定がない場合、
  CI に任せたテストは実行されない: それをここに記録し、決して
  実行済みとしない
- 計画からの逸脱

ユニット（`directive.unit` がある）の場合は、
`<record>/construction/<directive.unit>/code-generation/source-manifest.json` を
次の厳格なスキーマで作成する:

```json
{
  "stage": "code-generation",
  "unit": "u1-auth",
  "version": 1,
  "writes": [
    { "path": "src/auth/login.ts" },
    { "path": "src/auth/generated/" },
    { "repo": "repo-a", "path": "src/api/routes.ts" }
  ]
}
```

このユニットが作成、変更、削除したすべてのアプリケーションソースのパスを、
シェルコマンド、スキャフォールディング、ジェネレーターが書き込んだファイルも含めて列挙する。生成されたツリーには
末尾に `/` を付けたディレクトリの主張を使う。エンジンは、この試行のソースの基準が取られた
時点からの変更されたソースを測定する。ゲートでの却下
（Request Changes）はその基準を保持するため、ユニットがそのようにして再オープンされた場合は、
以前の試行が列挙したすべてのパスを保持し、この試行が書き込むパスを追加する。
修正内容だけを列挙したマニフェストでは、ユニットの以前のファイルがどのユニットにも
属さなくなる。Build and Test のループバックのようなステージへの後方ジャンプは、
新しい基準を取るため、そこからユニットが変更するものを列挙する。メインのワークスペースでは、
マルチリポジトリのエントリは記録された `repo` を指定する。Bolt をホストするワークツリー内では、パスは
選択された単一のリポジトリからの相対パスであり、`repo` を必ず省略しなければならない。エンジンは、
このマニフェストがなければユニットのレビューの記録を拒否する。Guard Policy が strict のもとでは、
主張されていない変更パスはステージの完了をブロックする。relaxed または off のもとではそれらは
保持され、ユニットの外で変更されたファイルが一度だけ利用者に伝えられる。

レビューが通常は参照しないチェックアウト（ハーネスフォルダ内の
ワークツリー、またはプロジェクト外のもの）にコードを置く場合は、レビューの前に
`{{INVOKE}} engine intent add-checkout <its path>` でそれを記録する。ユニットは、
プロジェクト外のチェックアウトを、そのコマンドが出力する名前でマニフェストに記載する
（`<label>@checkout/<path>`）。

ユニットのないディレクティブ（`directive.unit` がない）は、
`source-manifest.json` を書かず、そのためのユニットディレクトリも作成しない: エンジンは
ユニットの場合にのみマニフェストを読み、ユニットのないレビューは代わりにワークスペースの
ソース全体を束縛する。その Step 5 のファイルは
`<record>/construction/code-generation/code-summary.md` と
`<record>/construction/code-generation/traceability.json` である。

`{{INVOKE}} engine sensor-traceability --stage code-generation --output-path <that path> --write-upstream` を実行して、
`<code-generation-record>/traceability.json` を作成する:
これは、割り当てられたすべての AC、詳細な `NFRx.y`、`BRx.y`（インクリメンタルなスコープが設計の連鎖を省略した場合は、直接の `FR` /
`NFR` の ID）を、それぞれ `GAP`
行とともに列挙する。下位要件を持つ要件は、その `FR{n}.{m}` の ID を通じてトレースされ、
独自の行は必要ない。各行の `status` と `target` を設定する: すべての `OK` のターゲットは、
ワークスペースからの相対パスで表した既存の実装ファイルまたはテストファイル 1 つでなければならない。
利用者が計画された作業をこのビルドから外すよう求めた場合（例えば "skip the
remaining stories"）は、そうする: 外した項目をそれぞれ `Deferred`（
このユニットでは構築しない）として、利用者の言葉を `target` にして列挙し、作業を続ける。
承認済みの計画はそのまま残り、承認には何が外されたかが記載される:

```json
{
  "stage": "code-generation",
  "unit": "u1-auth",
  "upstream_ids": ["AC1.1.1", "NFR1.1", "BR1.1"],
  "coverage": [
    { "id": "AC1.1.1", "status": "OK", "target": "src/auth/login.ts" },
    { "id": "NFR1.1", "status": "OK", "target": "src/cache/redis.ts" },
    { "id": "BR1.1", "status": "OK", "target": "src/auth/policy.ts" }
  ]
}
```

### Step 6: Completion Handoff

`directive.protocol_modules` に `reviewer` が含まれる場合は、
`stage-protocol-reviewer.md` のセクション 12a の説明どおりに今レビューを実行し、その後で初めて
以下に進む。レビューは完成した作業を対象とする: 計画、テスト手順、コードの要約、
トレーサビリティ、そしてユニットの場合は `source-manifest.json` が主張するソースパスである。
frontmatter の `review_artifact: code-generation-plan` は、レビューを記録する対象の
ファイルを指定するものであり、構築前に計画をレビューすることを求めるものではない。
ユニットの場合、エンジンは Step 5 の
ファイルが存在するまでレビューリクエストを拒否する。

`{{INVOKE}} engine orchestrate report --stage code-generation --result <outcome>`
によって、完了処理を `stage-protocol.md` に引き渡す。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 7: Completion

完了メッセージと承認ゲートを提示する:

```
# :computer: Code Generation Complete — {unit-name}
```

作成したコードの要約（ファイル、テスト、主要な決定、および書いたが実行していないテストとその理由）を示し、続けて:

```
**Review:** `<code-generation-record>/`
```

承認ゲート: 厳密に 2 択（Approve / Request Changes）。

> **注 - オーケストレーターが管理する完了ゲート。** 計画の承認がオンの間、最初の Plan Approval は、自律的な Construction を含むすべての実行モードで必須の停止点である: 生成は、`next` が `plan_approval.status: "approved"` を返した後にのみ始まり、エンジンがそれを返すのは利用者が計画を承認した後だけである。作業で計画の承認がオフの場合、`next` は代わりに `skipped: true` と伝えるべき通知を伴う `approved` を返す。引き下げられた Guard Policy が最初の承認の代わりになることは決してない。その後、同じターゲットと試行に対する内容の編集は、Step 3 の承認後のルールに従う。上記の Build-and-Test のループバックによる再実行は新しいステージの試行を開始するため、"Retry with fix" の選択から承認を推測するのではなく、（計画の承認がオンの間は）修復された計画について Plan Approval を求める。通常の Construction の間、オーケストレーターが抑制するのは Step 7 の完了の承認ゲートだけである。stage-major の進行では、最後のユニットが確定した後に、ステージレベルの 1 つのゲートがすべてのユニットをカバーする。自律的なスウォームのもとでは、エンジンは最後の DAG バッチが収束した後にのみ Code Generation のステージゲートを提示する（中間のバッチはゲートなしでマージされる）。完了ゲートは、直接呼び出しで使うためにここにも存在する（例: stage-major の進行で単一のユニットを再実行する `/aidlc --stage code-generation`。Construction がユニットを 1 つずつ実行する場合は、Code Generation でそのユニットを続行するか、ジャンプして何を省略したかを伝える）。Task 経由で呼び出されたサブエージェントは、その完了ゲートを自分で呼び出してはならない - 完了ゲートの提示はオーケストレーターが担う。

## Sensors

このステージは、アクティブな Bolt の
ワークツリーに TypeScript/JavaScript のコードを作成する。生成されたコードはワークスペースのルートに置かれ（決して
記録ディレクトリの下には置かない）、計画、計画の承認、要約の成果物
（`code-generation-plan.md`、`code-generation-questions.md`、
`unit-test-instructions.md`、`code-summary.md`）は
`<code-generation-record>/` 配下に置かれる。

インポート: `required-sections`, `traceability`.

`traceability` は、ユニットごとのカバレッジ表とすべての `OK` のターゲットを検証する。

`linter` と `type-check` はここではインポートしない: これらはすべてのファイル書き込みで実行されていたが、
その結果を読むものはなかった。Build and Test がプロジェクトのビルドと
テストを実行する。

このステージは広範でスコープに依存する設計のセットを消費するため、`upstream-coverage` は意図的にインポートしない。
`source-manifest.json` は
その厳格なスキーマとソースの束縛に照らしてエンジンが検証し、
`traceability.json` は `traceability` センサーが所有する。どちらの構造化
ファイルも `required-sections` の下限の対象ではない。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
