# Scopes, Depth, and Test Strategy

スコープは**どのステージを実行するか**を制御します。深さ（depth）は各ステージが**どの程度詳しく**成果物を作るかを制御します。テスト戦略は**いくつのテスト**を生成するかを制御します。これらを組み合わせて、包括的なエンタープライズ機能から手早いバグ修正まで、ライフサイクルをタスクに合わせます。

この章は、ルーティングの詳細なリファレンスです。どのような種類のワークフローを実行するか
選んでいる段階なら、同じスコープを利用者向けの言葉で説明している
[ワークフロープロファイル](workflow-profiles.md) から読み始めてください。

---

## The 11 Core Scopes

コアには 11 個の名前付きスコープが同梱されています。各スコープは、ステージの集合、既定の深さ、既定の Guard Policy の値を定義します（`enterprise` では strict、それ以外では off。値の意味と設定方法は [Guard Policy](13-customization.md#guard-policy) を参照してください）。プラグインをインストールするとスコープを追加でき、`aidlc engine plugin select <names>` で、表示するプラグインのスコープを絞り込めます。`plugins` の選択でコアを無効にした場合（`aidlc` を省いた場合）、コアのスコープファイルはインストールされたまま残りますが、コアを再度有効にするまで実行時の有効なスコープにはなりません。Initialization のステージは、有効なすべてのスコープで引き続き実行されます。

以下の各スコープのステージ数は、すべてのスコープが最初に実行する Initialization の 3 つのセットアップステージの後に実行するステージの数です。これは `/aidlc --help` が示す数であり、既存のコードに対してプランの提案と各ステージの開始時に示される数でもあります。新規プロジェクトでは提案から Reverse Engineering が除かれるため、それを実行するスコープでは提案に示される数が 1 つ少なくなります。

### enterprise

**使う場面:** 完全な監査証跡、コンプライアンスレビュー、本番品質の運用が求められる、規制対象のエンタープライズ機能を構築するとき。

- **ステージ:** 30 すべて
- **既定の深さ:** Comprehensive
- **含まれるもの:** コンプライアンス、セキュリティ、運用のすべてのステージ

### feature

**使う場面:** 規模を問わず、33 ステージの完全なライフサイクルで新機能を構築するとき。`--scope feature` または `/aidlc-feature` で明示的に選択します。

- **ステージ:** 30 すべて
- **既定の深さ:** Standard
- **含まれるもの:** すべてのステージ、標準的な成果物の詳しさ

### mvp

**使う場面:** グリーンフィールドの MVP（実用最小限の製品）を構築するとき。後半の運用ステージはスキップしますが、設計と構築はすべて残します。

- **ステージ:** 30 のうち 20
- **既定の深さ:** Standard
- **スキップするもの:** Operation の 7 ステージすべて（デプロイパイプライン、環境のプロビジョニング、デプロイの実行、オブザーバビリティ、インシデント対応、パフォーマンス検証、フィードバック）に加え、Ideation の Market Research、Team Formation、Approval Handoff（10 をスキップ、20 を実行）

### poc

**使う場面:** 実現可能性を素早く証明するとき。Ideation と Inception のほとんどのステージをスキップし、早くコードにたどり着くことに集中します。

- **ステージ:** 30 のうち 5
- **既定の深さ:** Minimal
- **スキップするもの:** Market Research、Feasibility、Team Formation、Mockups、User Stories、ほとんどの Operation ステージ
- **プラン承認:** オフ。コードのプランが書かれると、それを示す 1 行が表示されてコード生成が始まり、その行でいつでもプランを見直すよう依頼できると伝えられます。ビルド前にプランを見せるよう依頼することも、すべてのプランについて確認を求めるよう依頼することもできます

### bugfix

**使う場面:** 特定のバグを修正するとき。インテントの把握からコード生成とテストまでを効率化した経路です。

- **ステージ:** 30 のうち 6
- **既定の深さ:** Minimal
- **含まれるもの:** 検証済みの修正を出荷するための Deployment Pipeline と Deployment Execution
- **スキップするもの:** Market Research、Feasibility、Team Formation、Mockups、ほとんどの設計・アーキテクチャのステージ、環境のプロビジョニング、より広範な運用準備
- **儀式（ceremony）:** ステージの日誌はなく、修正後の "Keep that for next time?" という質問もなく、成果物を書く前に確認するサマリーもありません。センサー、すべてのステージ承認、プラン承認は有効のままで、各ステージはリードエージェントだけで実行されます。この作業でいずれかを戻すには、`/aidlc --learnings on`、`/aidlc --summary-confirmation on`、`/aidlc --collaborators on` と入力します

### refactor

**使う場面:** 機能を変えずに既存のコードを整理または再構成するとき。

- **ステージ:** 30 のうち 7
- **既定の深さ:** Minimal
- **含まれるもの:** Functional Design に加え、Deployment Pipeline と Deployment Execution
- **スキップするもの:** bugfix とほぼ同じ — コード分析、設計、実装、既存の経路でのデプロイに集中します

### infra

**使う場面:** インフラストラクチャを変更するとき（新しい環境、CDK/CloudFormation の更新、コスト最適化）。

- **ステージ:** 30 のうち 12
- **既定の深さ:** Standard
- **スキップするもの:** ユーザー向けのステージ（ストーリー、モックアップ、ユーザーフロー）— アーキテクチャ、インフラストラクチャ、デプロイに集中します
- **IaC を書くステージ:** Code Generation で、Plan Approval のもと、Infrastructure Design をもとに書きます。その後 Build and Test が IaC のチェックを実行し、Environment Provisioning が適用します

### security-patch

**使う場面:** CVE やセキュリティ脆弱性に対応するとき。セキュリティに関係するステージを通る高速な経路です。

- **ステージ:** 30 のうち 7
- **既定の深さ:** Minimal
- **スキップするもの:** Market Research、Team Formation、Mockups、セキュリティ以外の設計ステージ

### classic

**使う場面:** v1 スタイルの進め方を望むとき。Inception と Construction で構成され、各ステージで人間が承認します（Construction では、作業にユニットがある場合はユニットごと）。条件付きステージは実行時にプロジェクトに合わせて調整され、Operation はプレースホルダーのままです。ステージが宣言する実行モードは変わりません。collaborators はオフなので、各ステージはリードエージェントだけで実行されます（`/aidlc --collaborators on` でサポートエージェントが加わります）。

- **ステージ:** 30 のうち 15
- **既定の深さ:** Standard
- **既定のテスト戦略:** Standard
- **スキップするもの:** Ideation のすべてのステージ（1.1-1.7）、CI Pipeline（3.7）、Operation のすべてのステージ（4.1-4.7）
- **キーワード:** なし。明示的に選択されるか、暗黙の既定として使われます
- **儀式:** ウォーキングスケルトンとサマリー確認はオフ。センサーは実行され、学びの手順も実行されます。レビューは advisory です（ステージごとに 1 回のパスで、指摘は承認ゲートで示されます）。明示的な自律モードでは、マージ前の単一のレビューが維持されます。

インテントごとに儀式を上書きするには、`/aidlc --sensors on|off`、`/aidlc --learnings on|off`、`/aidlc --summary-confirmation on|off`、`/aidlc --plan-approval on|off`、`/aidlc --collaborators on|off` を使います。グローバルなキルスイッチ `AIDLC_DISABLE_SENSORS=1`、`AIDLC_DISABLE_LEARNINGS=1`、`AIDLC_DISABLE_SUMMARY_CONFIRMATION=1`、`AIDLC_DISABLE_PLAN_APPROVAL_GUARD=1`、`AIDLC_DISABLE_COLLABORATORS=1` は、それらを強制的にオフにします。オフにできるのはあなただけであるプラン承認を除き、承認ゲート、人間のターンの権限、監査、チームの書き込み保護は引き続き適用されます。[儀式のカスタマイズ](13-customization.md#ceremony-switches) を参照してください。

#### Fewer stops for a workshop or small work

長い実行では、ステージごとの 2 つの確認が停止点を増やすことがあります。修正後の学びの質問と、
サマリー確認です。これらなしでプランを始めるには、要求と一緒に両方のスイッチを入力します。
そうすると、新しい作業はそれらがオフの状態で作成されます:

```
/aidlc workshop --learnings off --summary-confirmation off <the work>
```

| スイッチ | 各ステージで取り除かれるもの | 失うもの |
|--------|-------------------------------|---------------|
| `--learnings off` | エージェントを修正したステージの承認ゲートの前に出る "Keep that for next time?" という質問と、ステージの日誌（`memory.md`） | この作業からは `memory/project.md` にも `memory/team.md` にも何も書き込まれないため、次の実行はこの実行で得た教訓を引き継ぎません。すでにメモリーにあるプラクティスは引き続き読み込まれます |
| `--summary-confirmation off` | ステージが成果物を書く前に、回答のサマリーに対して行う `Looks correct` の確認 | ステージは回答から直接書きます。誤って読み取られた回答は承認ゲートで成果物の中に現れ、その修正にはリビジョンが 1 回かかります |

どちらのスイッチも、ステージの質問や承認ゲートを取り除くことはありません。`bugfix` と
`express` は最初から両方がオフの状態で始まり、`classic` はサマリー確認がオフの状態で始まります。
すでに進行中の作業では、どちらかを自分の言葉で依頼するか、
`/aidlc --learnings off` または `/aidlc --summary-confirmation off` と入力します。サマリー
確認がオフになるのは、あなたが依頼したときだけです。

#### Upgrading an in-flight classic intent

保存済みのステージのプランは保持されます。Operation
ステージが `EXECUTE` として記録されている古い classic インテントは引き続きそれらを実行し、ルーティング、次のステージの検索、
ステータスは記録されたプランに従います。新しい classic インテントは小さいほうのグリッドを使います。
古いインテントに保存済みの儀式のフィールドがない場合、儀式の既定値はすぐに変わります:
Sensors と Learnings は `on` に解決され、Summary Confirmation は classic に従って
`off` に解決されます。サマリー確認を有効にするには `/aidlc --summary-confirmation on` を使うか、
自分の言葉で依頼してください。
保存済みのインテントごとの儀式の選択は引き続き有効です。
環境変数のキルスイッチが引き続き優先されます。

classic のレビュアーの上限もすぐに変わります。レビュアーを伴うステージは、上書きされない限り
advisory のパスを 1 回実行します。作業ごとの `--review` レベルは
スコープの上限を置き換えますが、ステージ自身のレビュークラスを超えることはできません。`--review adversarial`
は adversarial レビューを宣言しているステージでそれを復元し、advisory と宣言されたステージは
advisory のままで、`--review none` は引き続きレベルを下げます。CI
Pipeline と Operation ステージを含む以前の classic のグラフを実行するには、
`aidlc engine scope change --scope workshop` でインテントを workshop に変更します。workshop は
Inception から Operation までの完全なプランを保持し、既定では Minimal のテスト戦略を使います。
必要であれば、そのスコープ変更に `--test-strategy standard` を追加して、本番向けの
Standard のテスト戦略を維持してください。

### workshop

**使う場面:** 確立された Inception から Operation までの完全なライフサイクルと、
より軽い教育向けのテストの下限で、ファシリテーション付きのワークショップやトレーニングラボを実施するとき。

- **ステージ:** 30 のうち 23
- **既定の深さ:** Standard
- **既定のテスト戦略:** Minimal
- **スキップするもの:** Ideation のすべてのステージ（1.1-1.7）
- **キーワード:** `workshop`、`lab`、`training`

### express

**使う場面:** 設計のパスやレビュアーのディスパッチなしで、要件からコードとテストを経て条件付きのデプロイの末尾までを、最も軽い経路で進めたいとき。

- **ステージ:** 30 のうち 7
- **既定の深さ:** Minimal
- **レビューの上限:** なし
- **含まれるもの:** Initialization、条件付きの Reverse Engineering、Requirements Analysis、Code Generation、Build and Test、そして条件付きの Deployment Pipeline、Deployment Execution、Observability Setup の各ステージ
- **スキップするもの:** Ideation、設計、Units Generation、Delivery Planning、CI Pipeline、環境のプロビジョニング、後半の運用ステージ
- **プラン承認:** poc と同様にオフ。各コードプランは、それを示す 1 行の後に書かれたとおりに構築され、その行でいつでも見直しを依頼できると伝えられます

---

## Scope Routing Table

正となるデータは、`.claude/scopes/aidlc-<name>.md` ファイル（スコープの識別）、プラグインのスコープファイル、そして各ステージの `scopes:` frontmatter（所属）にあり、`.claude/tools/data/scope-grid.json` にコンパイルされます。コンパイル済みのグリッドには、現在のプラグイン選択で有効になっているスコープだけが含まれます。ライブのコンパイル済みの表は `aidlc engine gen scope-table` で確認できます（ユーザー向けのワークフローとスコープの 1 行説明は `aidlc engine orchestrate help` で確認できます）。

| スコープ | EXECUTE / 合計 | 深さ | テスト戦略 | 用途 |
|-------|-----------------|-------|---------------|----------|
| `enterprise` | 33 / 33 | Comprehensive | Comprehensive | 規制対象のエンタープライズ機能、完全な監査証跡 |
| `feature` | 33 / 33 | Standard | Standard | 新機能のための完全なライフサイクル |
| `mvp` | 23 / 33 | Standard | Standard | グリーンフィールド、後半の運用をスキップ |
| `poc` | 8 / 33 | Minimal | Minimal | 実現可能性を素早く証明する |
| `bugfix` | 9 / 33 | Minimal | Minimal | 特定のバグを修正してデプロイする |
| `refactor` | 10 / 33 | Minimal | Minimal | 既存のコードを整理してデプロイする |
| `infra` | 15 / 33 | Standard | Standard | インフラストラクチャの変更 |
| `security-patch` | 10 / 33 | Minimal | Minimal | CVE への対応 |
| `classic` | 18 / 33 | Standard | Standard | V1 スタイルの Inception + Construction — 暗黙の既定 |
| `workshop` | 26 / 33 | Standard | Minimal | 教育向けのテストを伴う、ファシリテーション付きのライフサイクル |
| `express` | 10 / 33 | Minimal | Minimal | 要件から条件付きデプロイまで、設計とレビュアーなし |
| （提案） | 状況による | 状況による | 状況による | エージェントが自由記述のインテントから提案し、あなたが確認する |

スコープによって儀式の量は桁違いに異なります。`poc` は狭い単一パスの経路を実行しますが、`feature` は Initialization の後に 30 のステージを 30 の承認ゲートとともに実行し（Reverse Engineering がスキップされる新規プロジェクトではそれぞれ 29）、Construction では 5 つのステージ（4 つの設計ステージと Code Generation）を作業ユニットごとに繰り返します。スコープの確認行には、実際に適用される数（ステージ数、承認ゲート数、ユニットごとのファンアウト）が示されます。これらはコンパイル済みのグリッドとワークスペースのスキャンから計算され、推定されることはありません。グリーンフィールドの作業では Reverse Engineering が除外され、`units-generation` をスキップするスコープでは、ユニットの DAG が存在しないためユニットごとの記述が省かれます。ワークフローが始まる前に、何に同意しているのかがわかります。

> **プロジェクトごとの既定スコープ:** チームは `.claude/settings.json` で `AWS_AIDLC_DEFAULT_SCOPE` を設定することで、プロジェクトの既定スコープをあらかじめ決めておけます。[カスタマイズ § プロジェクトごとの既定スコープ](13-customization.md#per-project-default-scope) を参照してください。

---

## Stage-by-Scope Matrix

上のルーティング表は数を示しています。このマトリクスは、標準の各スコープで**どの**ステージが実行されるかを正確に示すので、ワークフローを始める前にどのステージを通るかを確認できます。✓ はそのスコープでステージが EXECUTE であることを、空のセルは SKIP であることを意味します。ステージの番号と名前は [フェーズとステージ](04-phases-and-stages.md) と一致しています。

<!-- BEGIN scope-stage-matrix: derived from each stage's `scopes:` frontmatter via the compiled scope-grid.json — kept in sync by tests/unit/t244-scope-matrix-doc-sync.test.ts; do not hand-edit cells without re-checking that test -->
| # | Stage | `enterprise` | `feature` | `mvp` | `poc` | `bugfix` | `refactor` | `infra` | `security-patch` | `classic` | `workshop` | `express` |
|---|-------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 0.1–0.3 | Initialization (all 3 stages) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| 1.1 | Intent Capture & Framing | ✓ | ✓ | ✓ | ✓ |  |  |  |  |  |  |  |
| 1.2 | Market Research | ✓ | ✓ |  |  |  |  |  |  |  |  |  |
| 1.3 | Feasibility & Constraints | ✓ | ✓ | ✓ |  |  |  |  |  |  |  |  |
| 1.4 | Scope Definition | ✓ | ✓ | ✓ |  |  |  |  |  |  |  |  |
| 1.5 | Team Formation | ✓ | ✓ |  |  |  |  |  |  |  |  |  |
| 1.6 | Rough Mockups | ✓ | ✓ | ✓ |  |  |  |  |  |  |  |  |
| 1.7 | Approval & Handoff | ✓ | ✓ |  |  |  |  |  |  |  |  |  |
| 2.1 | Reverse Engineering | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |  | ✓ | ✓ | ✓ | ✓ |
| 2.2 | Practices Discovery | ✓ | ✓ | ✓ |  |  |  | ✓ |  | ✓ | ✓ |  |
| 2.3 | Requirements Analysis | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| 2.4 | User Stories | ✓ | ✓ | ✓ |  |  |  |  |  | ✓ | ✓ |  |
| 2.5 | Refined Mockups | ✓ | ✓ | ✓ |  |  |  |  |  | ✓ | ✓ |  |
| 2.6 | Domain Design | ✓ | ✓ | ✓ |  |  |  |  |  | ✓ | ✓ |  |
| 2.7 | Units Generation | ✓ | ✓ | ✓ |  |  |  |  |  | ✓ | ✓ |  |
| 2.8 | Contract Design | ✓ | ✓ | ✓ |  |  |  |  |  | ✓ | ✓ |  |
| 2.9 | Delivery Planning | ✓ | ✓ | ✓ |  |  |  |  |  | ✓ | ✓ |  |
| 3.1 | Functional Design | ✓ | ✓ | ✓ |  |  | ✓ |  |  | ✓ | ✓ |  |
| 3.2 | NFR Requirements | ✓ | ✓ | ✓ |  |  |  | ✓ | ✓ | ✓ | ✓ |  |
| 3.3 | NFR Design | ✓ | ✓ | ✓ |  |  |  | ✓ |  | ✓ | ✓ |  |
| 3.4 | Infrastructure Design | ✓ | ✓ | ✓ |  |  |  | ✓ |  | ✓ | ✓ |  |
| 3.5 | Code Generation | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| 3.6 | Build and Test | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| 3.7 | CI Pipeline | ✓ | ✓ | ✓ |  |  |  | ✓ |  |  | ✓ |  |
| 4.1 | Deployment Pipeline | ✓ | ✓ |  |  | ✓ | ✓ | ✓ | ✓ |  | ✓ | ✓ |
| 4.2 | Environment Provisioning | ✓ | ✓ |  |  |  |  | ✓ |  |  | ✓ |  |
| 4.3 | Deployment Execution | ✓ | ✓ |  |  | ✓ | ✓ | ✓ | ✓ |  | ✓ | ✓ |
| 4.4 | Observability Setup | ✓ | ✓ |  |  |  |  | ✓ |  |  | ✓ | ✓ |
| 4.5 | Incident Response | ✓ | ✓ |  |  |  |  |  |  |  | ✓ |  |
| 4.6 | Performance Validation | ✓ | ✓ |  |  |  |  |  |  |  | ✓ |  |
| 4.7 | Feedback & Optimization | ✓ | ✓ |  |  |  |  |  |  |  | ✓ |  |
| | **Total stages** | **33** | **33** | **23** | **8** | **9** | **10** | **15** | **10** | **18** | **26** | **10** |
<!-- END scope-stage-matrix -->

✓ は静的なスコープへの所属を示します。つまり、そのステージがスコープのプランに含まれることを意味し、無条件に実行されることを意味するわけではありません。CONDITIONAL のステージは、条件が成り立たない場合に実行時にスキップされることがあり（たとえば Reverse Engineering はブラウンフィールドのプロジェクトでのみ実行されます）、保留中のステージは承認されたコンポーザーの提案によって組み替えられることがあります（[コンポーザー](#the-adaptive-composer) を参照）。合成された（カスタムの）スコープはここには載っていません。そのグリッドは標準のスコープと並んで `scope-grid.json` にあります。

---

## The Agent Proposes the Plan

スコープを明示的に指定する必要はありません。やりたいことを説明すれば、
エージェントがあなたの言葉からプランを提案します:

```
/aidlc Build a REST API for inventory management
```

エンジンはあなたの依頼を保持し、AI-DLC がこのプロジェクトに同梱しているすべてのプランとともにエージェントに渡します。
各プランについて、ここで実行するステージ数、何のためのプランか、そして作成者がプランファイルに書いた
ヒントの単語（`scopes/aidlc-<name>.md` の `keywords:`）が渡されます。
ヒントは、あなたの言葉の残りと同様にエージェントが読みます。ツールがそれを照合することはありません。
エージェントはあなたの説明に合うプランを提案し、あなたはそれとともに、コンパイル済みのグリッドから直接作られた
1 つの質問を受け取ります。新規プロジェクトで `/aidlc Fix the timeout on the export endpoint` と入力した場合は
次のようになりました。既存のコードベースでは bugfix は Reverse Engineering も実行するため、
この行は 6 ステージ、6 承認ゲートとなります:

```text
 Plan
 I'd run the "bugfix" plan for: "Fix the timeout on the export endpoint" - 5 stages, 5 approval
 gates. Do you want me to go ahead with it, use a different plan, or tailor one to this task?
 > 1. Go ahead with the "bugfix" plan
      5 stages focused on fixing this specific bug.
   2. Use a different plan
      Pick another ready-made plan.
   3. Tailor a plan to this task
      Have a plan composed specifically for this fix.
   4. Type something.
   5. Chat about this
```

そのまま進めるか、別のプランを使うか（エージェントが他のプランをその長さとともに一覧にするので、1 つを選ぶか名前を入力します）、
あるいはワークフローが始まる前に軌道修正するためにプランを調整します。説明に合う既製のプランがない場合、
エージェントは代わりにプランの調整を提案し（"None of the ready-made plans is an obvious
fit for ..."）、そのまま選べるプランを横に一覧表示します。依頼と一緒に入力した設定
（`--depth minimal`、`--learnings off`）は質問の回答に引き継がれるため、作業はその設定で始まります。
どのプランが提案されても、それがインテントに合っているかを確認してください。選ぶのは質問の場でのあなたです。

---

## The Adaptive Composer

プランが指定も提案もされないときに使われる既定のプランは `classic` です（プロジェクトは
`AWS_AIDLC_DEFAULT_SCOPE` で変更できます）。ユーザー向けのフローでは、
エージェントがあなたの依頼を読み、既製のプランを提案するか、合うものがなければ
プランの調整を提案します。あなたが答えるまで何も始まりません。合成を強制することもできます:

```
/aidlc compose "harden the deployment pipeline and add observability"
/aidlc-compose "same thing, as a typeable shortcut"
/aidlc compose --report sonar.json     # compose from a scan report
/aidlc --new-scope "..."               # force a custom plan even on a stock match
```

コンポーザーエージェントはタスクを読み、実装エントロピーの 5 つの要素（インテントの曖昧さ、コードベースの構造的な不確実性、検証のエントロピー、リスク、未解決の前提）を推定し、最小限で実用的なワークフローを合成します。つまり、成果が依存するすべての成果物をなお生み出せる、必要十分で最小の EXECUTE/SKIP グリッドです。構造の推定は、CodeKB サーバーが設定されインデックス済みの場合は CodeKB MCP のコールグラフとコンポーネント分析に基づきます（オプションの外部ツールであり、AI-DLC には何も同梱されていません）。そうでない場合、コンポーザーは範囲を限定したワークスペースのスキャン（ブラウンフィールド/グリーンフィールド、言語）にフォールバックします。提案は短いオファーとして届きます。どのような種類の変更に見えるか、コンポーザーがどの程度のプロセスを提案するか、そしてそのステップを平易な言葉で説明する 2〜3 文に続いて、プランの名前、ステージ数、承認の質問を示す 1 行があり、承認前に変更できる Guard Policy とスコープ設定の行が並びます。調整されたプランが、その基盤となる標準スコープなら実行するはずのステージや、残したステージが使うステージをスキップする場合（そのスコープが設計上除外しているものを除く）、オファーはそのことを、該当するステージをすべて挙げた 1 行で一度だけ伝えます（"This plan skips Units Generation, which Contract Design, Functional Design and Code Generation use, and Deployment Execution"）。依頼すると、エージェントはスコアの内訳（各要素の LOW/MED/HIGH の帯とその証拠）と、すべての EXECUTE と SKIP の理由を示します。既存のコードでは、調整されたプランには常に Reverse Engineering が含まれます。Reverse Engineering は、再度スキャンする前に以前のスキャンが残したコードマップを確認します。ただし、コンポーザーが CodeKB の証拠を持っていてそれを除外するよう提案する場合は例外です（後述の CodeKB による根拠付けを参照）。プランが作成されると、そのスコアと証拠の取得方法は作業の最初の監査エントリに保存されます。あなたは承認、編集、または却下します。明示的な承認の前には何も書き込まれず、ワークフローも始まりません。プランは承認された作業にだけ適用されるため、あなたが依頼しない限り、合成によってスコープの一覧が増えることはありません。承認すると:

- 提案が標準スコープに一致した（MATCHED）場合、AI-DLC はそのスコープで直接ワークフローを作成します（コードレベルの指摘を多く含むスキャンレポートは、通常このようにして `bugfix` または `security-patch` に振り分けられます）。
- カスタム（CUSTOM）プランの場合、AI-DLC は、最も近い適切な標準スコープ（ウォーキングスケルトンがなく、プランの深さ以外のテスト戦略を持たないもの。これによりプランはゲートで示されたとおりに実行されます。新規プロジェクトでは、該当するものがあれば `bugfix`、`refactor`、`security-patch` ではなく新規作業向けのもの）の上に、プラン独自のステージの変更と深さを加えたワークフローを同じターンで作成し、スコープファイルは書きません。作業の状態には `Plan: <name>`（コンポーザーが提案しゲートで示された名前。名前が提案されなかった場合は `tailored plan`）が記録され、`/aidlc status` がそれを表示します。プランの基盤となるスコープはそこには示されません。カスタムプランのゲートには 2 つ目の承認 **Approve and save as scope** があります。これを選ぶと名前を尋ねられ（コンポーザーが候補を提案します）、プランを再利用可能なスコープとしても保存します。
- 後からプランを保存するには、平易な言葉で依頼します（"save this plan as quick-fix"）。AI-DLC は `aidlc engine scope save --name quick-fix` を実行し、実行中の作業の現在のプランを、その設定と Guard Policy とともにスコープとして書き込み、それを再利用するコマンド（`/aidlc --scope quick-fix "..."`）を示します。保存されたスコープは標準スコープと同じように解決され、グラフの再コンパイルやエンジンの再インストール後も残り、ハーネス間で引き継がれます（[保存したスコープの保存場所](#where-a-saved-scope-is-stored) を参照）。
- front/report のすべての提案は、空でない `creationDescription` を持ちます。compose の依頼にタスクのテキストが含まれていた場合は、そのテキストがそのまま使われます。1 ステップに収まらないほど長い依頼は、代わりにディスパッチの中で保存済みのコピーとして示され、コンポーザーが作業の短い要約を自分の言葉で書きます。そのため、どんな長さの説明でも 1 ステップに収まり、作成時には元の言葉が保持されます。report のみの提案やタスクのない提案では、承認された指摘事項/プランから導出されます。report のみ、またはタスクのない同一ターンでの作成では、それをコンポジションの `--request` id（作業をそのゲートに結びつけるもの）とともに、リテラルの `--` 区切りの後に、シェルで安全な 1 つの argv 値として渡します（シェルで表示する場合は POSIX のシングルクォート）。これにより、状態の Project フィールドとインテント記録の slug は、シェルのメタ文字を含む説明や、フラグで始まる説明も保持できます。compose の承認は、スコープだけで説明がない状態では続行できません。
- すでに他の作業があり、どれも選択されていないプロジェクト（たとえば、インテントの記録がコミットされているリポジトリを新たにクローンした場合）では、AI-DLC はまず、承認されたプランがその作業の一部なのか、別の新しい作業なのか、その作業への変更なのかを尋ねます。別の新しい作業を選ぶと、承認したプランがそのステージの変更と設定とともに作成されます。そこで別の標準プランを指定すると、そのプランが同じ設定で始まりますが、承認したプラン独自のステージの変更は含まれません。いずれの場合も作業は一度だけ始まり、ゲートで伝えた「skip plan approval」は引き続き適用されます。

**ゲートでの設定:** ステージのグリッドに加えて、front/report の提案はプランの Guard Policy と 6 つのスコープ設定（`sensors`、`learnings`、`summary_confirmation`、`plan_approval`、`collaborators`、`review_cap`）を示し、各グループをそれぞれの行に 1 行の理由とともに表示します。これにより、合成されたプランはステージそのものだけでなく、ステージ内部の儀式も削ることができます。小さくてよく理解された修正なら、センサーと学びをオフにし、レビューを advisory に制限して実行するかもしれません。一致した提案は、標準スコープの値から始まります。カスタムプランは独自のステージを選びますが、儀式は `classic` スコープ（合成しない場合に得られるワークフロー）のものから始まります: Guard Policy はオフ、サマリー確認と collaborators はオフ、レビューは advisory、センサー、学び、プラン承認はオンです。コンポーザーは、タスクに理由がある場合にだけ設定を動かし（サマリー確認がオンに戻るのは、タスクが非常に曖昧な場合だけです）、メモリーのレイヤーが strict を保持していない限り Guard Policy をオフのままにします。ゲートで引き上げることができます。いずれの場合も、プランの基盤となる標準スコープと異なる値はこの作業にだけ適用され、新しいスコープファイルは作られません。保存したプランでは、それらの値が frontmatter に保存されます。承認の前に変更を依頼すれば、その変更が行われます。標準プランの Guard Policy を下げると、その値を持つ標準スコープの上のカスタムプランになります。プラン承認はゲートに表示された値を保持するため、カスタムプランは、基盤となる標準スコープが確認なしで構築する場合でもプラン承認をオンのままにします。ゲートで「skip plan approval for this work」と伝えると、作業はプラン承認がオフの状態で、あなたの設定として作成されます。それ以外の設定で承認ゲートが取り除かれることはありません。ワークフローの途中では、平易な言葉で依頼すると（"turn sensors off"、"give this full reviews"）、変更はゲートなしで実行中の作業に適用されます（[儀式のスイッチ](13-customization.md#ceremony-switches) を参照）。作業に設定したレビューレベルはスコープの上限を置き換えるため、レビューを強めてもステージは変わりません。スコープ自身のレベルに設定すると、通常のレビューに戻ります。ワークフロー途中の compose でコンポーザーが自ら提案した設定はゲートに表示され、あなたが承認した場合にだけ適用されます。ワークフローの途中でサマリー確認をオフにするのは、あなた自身が行う変更です。平易な言葉で依頼するか（"skip the looks correct check from now on"）、ゲートで承認すると、エージェントがそれをオフにし、あなたの言葉と元に戻す方法を添えて 1 行で伝えます。プラン承認をオフにできるのはあなただけで、自分の言葉で依頼するか、`/aidlc config set plan-approval off` と入力します。コンポーザーがワークフローの途中でそれをオフにするよう提案することはありません。

**CodeKB による根拠付け（オプション）:** CodeKB は、コードベースの事前計算された構造分析（コールグラフ、コンポーネントの一覧、パッケージ間の結合）を提供する外部の MCP サーバーです。AI-DLC はこれを同梱も必須ともしていません。これがない場合、コンポーザーは範囲を限定したワークスペースのスキャンから構造をスコアリングします。これが通常の経路です。接続した場合、コンポーザーはそれを唯一の構造的な証拠の情報源として使い、提案の中で引用します（`method: codekb`）。接続方法はハーネスによって異なります。Claude Code ではプロジェクトの `.mcp.json` にサーバーを追加します（サブエージェントはセッションの MCP サーバーを引き継ぎます）。Codex では `config.toml` に `mcp_servers` エントリを追加します。opencode では opencode の設定に追加します。Copilot CLI では `~/.copilot/mcp-config.json` に、VS Code では `.vscode/mcp.json` に追加します。Kiro では `"disabled": true` を付けずに `.kiro/settings/mcp.json`（または自分の `~/.kiro/settings/mcp.json`）に追加します。Kiro CLI の v2 エンジンではその後 config を再度実行すると、コンポーザーエージェントに付与されます。v3 エンジンと Kiro IDE では、コンポーザーはそのまま利用できます。CodeKB を、フレームワーク自身の "codekb" ディレクトリ（`aidlc/spaces/<space>/codekb/`）と混同しないでください。後者は Reverse Engineering ステージが書き込むローカルの成果物ストアであり、MCP サーバーとは無関係です。なお、CodeKB の証拠がある場合、コンポーザーは Reverse Engineering のスキップを提案することがあります。その場合、提案は下流のステージがそのローカルストアなしで実行されることを必ず開示しなければならず、ゲートであなたが判断します。

**キーワードの衛生管理:** 保存されたスコープは `keywords: []` を持つため、保存したプランはエージェントのプラン提案ステップにヒントを持ち込みません。今後のプロンプトのためにヒントを与えることは、保存時の明示的な選択です（`scope save --keywords <word,...>`。単語単位で指定します。他のプランがすでに持っているヒントも保存され、保存時に、プラン提案がそれを両方のプランの下に一覧表示することが 1 行で伝えられます）。副作用として付与されることは決してありません。

**実行中の再合成:** ワークフローの途中では、`/aidlc compose` は実行中のワークフローの保留中（PENDING）のステージの組み替えを提案します。不要になったものをスキップし、必要だと気づいた保留中のステージを戻します。コンポーザーは、完了したステージが実際に解決した内容からエントロピーの要素を再推定するため、提案される切り替えのそれぞれに、スコアを動かした証拠が示されます（"feasibility settled the integration questions - risk re-scores MED"）。切り替えは、カーソルより先にある保留中のステージにだけ適用され（完了済みと進行中のステージは凍結されます）、残りのステージが必要な入力を欠くことのないよう厳密に検証され、監査ロックのもとで決定論的な `recompose` 動詞を通じて `RECOMPOSED` 監査イベントとともに適用されます。1 回の承認でステージの変更と設定の両方を扱う場合は、同じ `recompose` が両方を 1 回の書き込みで適用するため、プランが中途半端に変更されたまま残ることはありません。サマリー確認のオフは独自の設定を通じて行われ、あなたが承認した場合、エージェントがその直後に適用します。Construction の最初の EXECUTE ステージは recompose で切り替えられません。これはステージのルーティングのアンカーを保護するためのものであり、最初の設計ステージのレビューを、統合されたスケルトンが動作することの証明にするものではありません。完了した作業は、自身のプランにそれ以上ステージを取り込みません。さらにステージを求めると、最後のステージの後にそれらを実行するプランが示され、選んだプランに切り替えると（`/aidlc --scope <plan>`）、完了した内容を保ったままその場所で作業が再開されます。

動詞をそのまま使う必要はありません。"what else can we cut? we already know this market" のような平易なチャットも、ワークフローの途中では組み替えの依頼として認識され、同じゲートと同じ `recompose` 動詞に回されます。ステージを自分で指定した場合（"can we skip market research?"、"drop market-research and team-formation"）は、ゲートなしでプランがすぐに変更され、返信で元に戻す方法が示されます。`recompose` は、プランが受け入れられない変更は引き続き拒否し、その理由を伝えます。代わりにプランを指定した場合（"switch this to the express plan"）は、`/aidlc --scope express` と同様に、ゲートなしで作業がその場でそのプランに切り替わります。完了した内容は完了したまま残り、返信で何が変わったかが伝えられます。貼り付けたドキュメントの中のテキストが、ステージの指定とみなされることはありません。Claude 以外のハーネスでは、リテラルの `/aidlc compose "<request>"` 動詞が、引き続きドキュメントに記載された確実な経路です。

### Where a saved scope is stored

標準スコープはエンジンに属します。ビルドのたびにハーネスのツリーに再生成されます。合成されたプランから保存したスコープは*あなたのもの*です。エンジンの中にそれを生成するものはないため、あなたの他の作業とともに保存されます:

| | パス | 役割 |
|---|---|---|
| **記録** | `aidlc/scopes/<name>.md` | 永続的なソースです。スコープファイルと、EXECUTE/SKIP のプランを保持する生成されたグリッド領域を 1 つのファイルにまとめたものです。`aidlc/` ツリーの他の部分と一緒にコミットしてください |
| **投影** | `<harness>/scopes/aidlc-<name>.md` + `scope-grid.json` の列 | ランタイムが読むものです。`aidlc engine graph compile` によって記録から再生成されます |

ここから 2 つのことが言えます。どちらも以前は失敗していたことです:

- **エンジンを再インストールしてもスコープは失われません。** ハーネスのツリーを置き換えると投影は破棄されますが、次の `graph compile` が記録から再構築します。これは特にコピーチャネルで重要です。コピーチャネルでは `scope-grid.json` は生成されたファイルであり、上書きコピーのたびに置き換えられるためです。
- **保存したスコープはハーネス間で引き継がれます。** 記録はツールに依存しない共有の `aidlc/` ツリーにあるため、`.kiro/` や `.codex/` で作業している共同作業者も、`graph compile` を 1 回実行すれば、あなたが `.claude/` で保存したスコープを解決できます。

以前のリリースで合成したスコープは、ハーネスのツリーにしか存在しません。アップグレード後の最初の `graph compile` が、その記録を一度だけ書き込みます。それ以降は記録がソースになります。

#### The record's shape

センチネルコメントより上はすべて、自由に編集できます。frontmatter も散文も同様で、独自の Markdown の見出しやフェンス付きコードも含まれます。センチネルの間はすべて生成されたものです。この領域はファイルの**末尾**として扱い、散文はその上に書いてください。`graph compile` がハーネスのツリーに投影するのはその部分だからです。END センチネルの*下*に追記したものは記録に残りますが（既存の記録を書き換えるものはありません）、ハーネスのスコープファイルには引き継がれません:

````markdown
---
name: my-lean-feature
depth: Standard
keywords: []
---

# my-lean-feature

Whatever you want to write, including a "## Stage Grid" heading of your own.

<!-- BEGIN aidlc composed-scope-grid: generated by `aidlc engine graph compile` — reshape the plan through /aidlc compose, not by editing here -->

```json
{ "stages": { "intent-capture": "EXECUTE", "units-generation": "SKIP" } }
```

<!-- END aidlc composed-scope-grid -->
````

プランの組み替えは、領域を編集するのではなく `/aidlc compose` で行ってください。手で編集したグリッドはステージグラフに対して検証されないため、ステージが必要とする入力を欠いてしまう可能性があります。

センチネルより上の部分を編集するのは問題ありませんが、`graph compile` がハーネスのスコープファイルを書くのは、それが**存在しない**場合だけであることに注意してください。編集済みのファイルを上書きすることはありません。そのため、記録の `depth` や `description` を変更した場合は、`<harness>/scopes/aidlc-<name>.md` を削除して再コンパイルし、再投影してください。プラン自体は常に記録から取得されるため、これが影響するのは説明用のフィールドだけです。

#### When a record is unreadable

frontmatter、`name`、または生成されたグリッド領域が欠けている記録は、ファイルパスと必要な編集を示して `graph compile` と `aidlc update` を**停止**させます。コミット済みの作業であるため、黙ってスキップされることはありません。示されたファイルを修復するか、削除してください。ハーネスの投影がまだ存在する場合、次のコンパイルがそこから新しい記録を補完します。`/aidlc --doctor` の **Composed scope durability** チェックは、グリッドの列がないスコープファイル、まだ投影されていない記録、スコープが解決できなくなった実行可能なワークフローを報告します。

---

## The 3 Depth Levels

深さは、各ステージで作られる成果物の詳しさを制御します。スコープが既定の深さを設定しますが、上書きできます。

| 深さ | 成果物の詳しさ | 使う場面 |
|-------|----------------|-------------|
| **Minimal** | 中核となる要点のみ。短いドキュメント、主要な決定、最小限の補足分析。 | 手早い修正、パッチ、概念実証 |
| **Standard** | バランスの取れた詳しさ。完全な要件、根拠付きのアーキテクチャ上の決定、綿密なテスト計画。 | ほとんどの機能と MVP |
| **Comprehensive** | エンタープライズ向けの完全な詳しさ。網羅的な要件、コンプライアンスのマトリクス、詳細な NFR の仕様、完全な監査ドキュメント。 | 規制対象の機能、エンタープライズへのデプロイ |

### How depth affects stages

各ステージで、エージェントはアクティブな深さに応じて出力を調整します:

- **Minimal:** 1〜2 ページの成果物、主要な決定のみ、オプションのセクションはスキップ
- **Standard:** 完全な成果物、すべての必須セクション、簡潔な根拠
- **Comprehensive:** 拡張された成果物、オプションのセクションも含む、詳細な正当化、コンプライアンスの相互参照

### Overriding depth

深さは次の 3 つの時点で変更できます:

1. **`--depth` CLI フラグで** — 起動時に深さを上書きします:
   ```
   /aidlc --depth comprehensive
   /aidlc --scope bugfix --depth standard
   /aidlc --stage code-generation --depth minimal
   ```
2. **スコープの確認時に** - エンジンが提案されたプランの確認を求めたとき、単に確認する代わりに `--depth <level>` と返信します
3. **任意の承認ゲートで** — フィードバックの一部として別の深さを依頼します

エージェントは、セッションごとに 1 回、最初の完了メッセージで、現在の深さ（各ドキュメントにどの程度詳しく書くか）と
テスト戦略（いくつのテストを書くか）、そしてどちらも任意の承認ゲートで変更を依頼できることを
伝えるよう求められています。いつでも、自分の言葉で、または
`/aidlc --depth <level>` で依頼できます。

---

## Specifying Scope Directly

### Explicit scope

```
/aidlc feature
/aidlc bugfix
/aidlc enterprise
```

### Scope with description

```
/aidlc bugfix Fix the login timeout issue
/aidlc poc Build a quick prototype for the search feature
```

### Override scope with utility command

```
/aidlc --scope bugfix
/aidlc --scope enterprise --stage code-generation
```

ジャンプ操作では、`--scope` フラグを `--stage`、`--phase`、`--depth` と組み合わせられます。

### Override depth

```
/aidlc --depth minimal
/aidlc --scope bugfix --depth comprehensive
/aidlc --scope enterprise --depth standard --stage code-generation
```

`--depth` フラグはスコープの既定の深さを上書きします。有効な値: `minimal`、`standard`、`comprehensive`（大文字と小文字は区別されません）。

### Override test strategy

```
/aidlc --test-strategy minimal
/aidlc --depth standard --test-strategy minimal
```

`--test-strategy` フラグは、深さとは独立にテスト戦略を上書きします。詳しい説明は、後述の [3 つのテスト戦略レベル](#the-3-test-strategy-levels) を参照してください。

---

## The 3 Test Strategy Levels

テスト戦略は、**いくつのテスト**を生成するか、そして**どの種類のテスト**を含めるかを制御します。深さとは独立しています。深さは成果物の詳しさ（ドキュメント、図、質問）を制御し、テスト戦略はテストの量だけを制御します。この分離により、テストカバレッジより速さが重要な場合に、Standard の深さの完全なワークフローを Minimal のテストで実行できます。

### Minimal — Nyquist model

信号処理のナイキストレート（信号を再構成するのに必要な最小のサンプリング周波数）に着想を得ています。Minimal のテスト戦略は、すべての要件を検証するのに必要な最小限のテストを生成します。多すぎず、少なすぎずです。

- **特定された要件ごとに 1 つのテスト**（コンポーネント駆動ではなく要件駆動）
- **ハッピーパスの下限:** 対応する要件がない場合でも、すべてのコンポーネントに少なくとも 1 つのハッピーパスのユニットテストを用意します
- **既定はユニットテストです。** bugfix/security-patch の対象を絞った回帰テストでは、
  それが欠陥を再現できる最も狭いレベルであれば、統合テストや E2E を使ってもかまいません。
  無関係なテストの量は Minimal のままです。
- 一般的なプロジェクトで**合計約 5〜15 個のテスト**
- 緩やかな目安です。安全性が重要な状況で必要であれば、エージェントはこれを超えてもかまいません

**最適な用途:** ワークショップ、トレーニングセッション、概念実証、手早いバグ修正など、完全なテストスイートに投資せずに正しさを検証したいあらゆる状況。

### Standard — per-component model

コンポーネント間の境界を検証する、バランスの取れたテストカバレッジです。

- **コンポーネントごとに 5〜8 個のテスト**
- **ユニットテスト + 統合テスト**（コンポーネント間の主要な境界）
- E2E、パフォーマンス、セキュリティのテストは、NFR の要件が明示的に求める場合のみ
- **テストピラミッドの比率:** ユニット約 75% / 統合約 20% / E2E 約 5%
- 緩やかな目安です

**最適な用途:** ほとんどの機能と MVP。テストに過剰に投資することなく、十分なカバレッジが得られます。

### Comprehensive — full coverage model

すべての種類のテストにわたる、綿密なテストカバレッジです。

- **コンポーネントごとに 10〜15 個のテスト**
- **すべての種類のテスト:** ユニット + 統合 + E2E + パフォーマンス（NFR がある場合）+ セキュリティ（NFR がある場合）
- **テストピラミッドの比率**はすべての種類に適用されます
- 緩やかな目安です

**最適な用途:** エンタープライズ機能、規制対象のシステム、テストカバレッジの監査証跡が求められるあらゆる状況。

### How test strategy defaults work

ほとんどのコアスコープは、テスト戦略を深さから引き継ぎます。そのため `classic` は
本番向けの Standard のテストの下限を使い、`express` は要件駆動の Minimal の
下限を使います。`workshop` は教育セッションのために、Standard の深さで明示的な Minimal の上書きを
保持しています。`--test-strategy` でいつでも上書きできます。

### Overriding test strategy

テスト戦略は次の 3 つの時点で変更できます:

1. **`--test-strategy` CLI フラグで** — 起動時に上書きします:
   ```
   /aidlc --test-strategy minimal
   /aidlc --depth standard --test-strategy minimal
   /aidlc --scope bugfix --test-strategy comprehensive
   ```
2. **ワークフローの途中で** — アクティブなワークフローのテスト戦略を変更します:
   ```
   /aidlc --test-strategy comprehensive
   ```
3. **任意の承認ゲートで** — フィードバックの一部として別のテスト戦略を依頼します

### Common depth + test strategy combinations

| 深さ | テスト戦略 | 効果 | 使う場面 |
|-------|--------------|--------|-------------|
| Standard | Standard | 完全な成果物、バランスの取れたテスト | feature、classic、その他の本番向けスコープ |
| Standard | Minimal | 完全な成果物、ナイキストのテスト | ワークショップ、時間を区切ったセッション |
| Minimal | Minimal | 簡素な成果物、要件駆動のテスト | express、手早いバグ修正、パッチ |
| Comprehensive | Comprehensive | すべてが完全 | 規制対象のエンタープライズ機能 |
| Comprehensive | Standard | 完全な成果物、バランスの取れたテスト | 実用的なテストを伴うエンタープライズ |
| Minimal | Comprehensive | 簡素な成果物、綿密なテスト | 確信が必要な重大なバグ修正 |

---

## Choosing the Right Scope

| 状況 | 推奨スコープ |
|-----------|------------------|
| 本番アプリケーションの新機能 | `feature` |
| ゼロから作るグリーンフィールドの製品 | `mvp` または `feature` |
| アプローチの手早い検証 | `poc` |
| 修正すべき既知のバグ | `bugfix` |
| 振る舞いを変えないコードの整理 | `refactor` |
| 新しい AWS 環境や CDK の変更 | `infra` |
| CVE やセキュリティ脆弱性への対応 | `security-patch` |
| コンプライアンスが求められる規制対象の機能 | `enterprise` |
| 最小限の儀式による V1 スタイルの Inception + Construction | `classic` |
| 要件からデプロイまでの軽量な実行 | `express` |
| AI-DLC のワークショップやトレーニングラボ | `workshop` |

迷ったときは、変更の規模でプランを選んでください。小さくてよく理解された変更なら、
すべてのステージを実行する `feature` から始めるのではなく、`express`（既知のバグなら `bugfix`）から始めて、
必要なステージを 1 つ追加してください:

```
/aidlc express --add functional-design <the change>
/aidlc bugfix --add functional-design <what is broken>
```

`--add` はこの作業のプランにだけステージを追加し、スコープの残りはそのまま保つため、
Functional Design を加えた `express` も、レビュアー、センサー、学びの質問、サマリー確認、プラン承認なしで実行されます。
変更の前にまずコンポーネントに名前を付ける必要がある場合は、代わりに `domain-design` を追加してください。
すでに進行中の作業では、`/aidlc --add <stage>` でまだ実行されていないステージを追加できます
（[CLI コマンド](12-cli-commands.md) の `--skip` / `--add` を参照）。
完全なライフサイクルが必要な本番向けの機能には `feature` を、
Ideation や Operation を伴わない v1 スタイルの Inception と Construction の体験には `classic` を選んでください。

---

## Next Steps

- [フェーズとステージ](04-phases-and-stages.md) — 各ステージの内容
- [エージェント](06-agents.md) — どのエージェントがどのスコープに参加するか
- [スキルとランナーコマンド](17-skills.md) — bugfix、express、feature、mvp、security-patch 用の 1 語の `/aidlc-<scope>` ランナー
- [CLI コマンド](12-cli-commands.md) — コマンドの完全なリファレンス
- [用語集](glossary.md) — 用語のリファレンス
