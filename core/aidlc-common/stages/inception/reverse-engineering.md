---
slug: reverse-engineering
phase: inception
execution: CONDITIONAL
condition: Execute when project is brownfield. On rerun the Step 1 guard checks store freshness (codekb-scope-diff) - verified-CURRENT stores may be reused by human choice, anything else rescans. Skip for greenfield projects.
lead_agent: aidlc-developer-agent
support_agents:
  - aidlc-architect-agent
mode: pipeline
produces:
  - business-overview
  - architecture
  - code-structure
  - api-documentation
  - component-inventory
  - technology-stack
  - dependencies
  - code-quality-assessment
  - reverse-engineering-timestamp
consumes: []
requires_stage:
  - state-init
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
  - mvp
  - poc
  - bugfix
  - refactor
  - security-patch
  - classic
  - workshop
  - express
inputs: <record>/aidlc-state.md
outputs: "aidlc/spaces/<active-space>/codekb/<repo>/ (9 artifacts: business-overview.md, architecture.md, code-structure.md, api-documentation.md, component-inventory.md, technology-stack.md, dependencies.md, code-quality-assessment.md, reverse-engineering-timestamp.md)"
---

# Reverse Engineering

このステージは `mode: pipeline`（stage-protocol-ensemble.md §5）で実行される。これは、各リンクが作業成果物を直接前に進めるチェーンである。リンクは正確に `directive.pipeline.links` であり、エンジンはディレクティブの実効的な `support_agents` からそれを組み立てるので、協働者のスイッチがチェーンを左右する。全メンバーがそろっている場合、チェーンは2つのリンクからなる: 開発者のリード（リンク 1）がスキャンして構造化された結果を返し、アーキテクト（リンク 2、最後のリンク）がその結果を統合して 9 つの成果物を書く。

**アーキテクトがオフになっている場合（このスコープで協働者がオフ）、`directive.pipeline.links` は開発者のリードだけになり、開発者は単独で作業する: スキャンと、結果の 9 つの成果物への統合の両方を自分で行う**（stage-protocol-ensemble.md セクション 5）。いずれの場合も、最後のリンクが `produces[]` の成果物を完成させる。それと、実際にディスパッチした各エージェントについて記録されたリンクとが、パイプラインの契約である: パイプラインのステージには貢献ファイルはない。再開時は `directive.pipeline.completed` を読み、欠けている最初のリンクだけをディスパッチする。複数リポジトリのエントリは `<repo>:<agent>` として修飾される。以下のストアのチェック、スナップショット、ステージング、公開、リンクの記録は黙って行う: 利用者が聞くのは、スキャンが利用者のコードで見つけたことだけである。

## Steps

### Step 1: Check Conditions

`<record>/aidlc-state.md` を読み、次を確認する:
- プロジェクトの種類が brownfield である

プロジェクトが brownfield でない場合は、`{{INVOKE}} engine orchestrate report --stage reverse-engineering --result skipped --reason '<reason>'` を実行する。エンジンがスキップを記録し、スコープ内の次のステージへ進む。プロジェクトの種類を決めるのは利用者であり、スキャンではない: 利用者がこれは既存のコードだと言っている場合はスキップせず、`{{INVOKE}} engine orchestrate next --project-type brownfield` を実行し、返されたものに従う。

#### Resolve the intent's repo set (multi-repo)

このステージは、インテントが関わる**リポジトリごと**に実行される。再利用やスキャンの判断をする前に、インテントのレジストリ行からリポジトリの完全な集合を解決する:

1. アクティブなインテントの `repos` 配列を `aidlc/spaces/<active-space>/intents/intents.json`（`uuid`/`slug` がアクティブなインテントに一致する行）から読む。これはインテント作成時に取得された集合である（明示的な `--repos a,b`、または兄弟ディレクトリの自動検出）。
2. **記録されていないプロジェクトルートのリポジトリ:** `repos` がない、または空の場合、RE はワークスペースのルートに対して1回実行される。そのハンドオフとレシートにはリポジトリの修飾を付けない。
3. **登録済みのリポジトリ（1つ以上）:** 記録されたすべてのリポジトリについて Step 1 のガードの判断を解決し、スキャン対象に選ばれた各リポジトリについて Steps 2-3 を1回ずつ実行する。そのリポジトリの兄弟ディレクトリ（`<workspace>/<repo>/`）をスキャンし、ハンドオフと両方のレシートをそのリポジトリの正確な識別子で修飾し、9 つの成果物を `codekb-path --repo <repo>` が出力するディレクトリ（スペースレベルの `aidlc/spaces/<active-space>/codekb/<repo>/`。Step 3 を参照）に書く。各リポジトリの codekb は独立しているので、選ばれたスキャンは並列のサブエージェントとして実行してよい。

以下の手順では、`<repo>` は判断またはスキャンを処理中のリポジトリを指す。

スキャン対象に選ばれた各リポジトリについて、Steps 2-3 は1つの独立したレシートのチェーンである。インテントがそのリポジトリの識別子を記録している場合は、リポジトリがちょうど1つの集合も含め、両方のレシートのコマンドに `--repo <repo>` を付ける。省くのは、記録されていないプロジェクトルートのリポジトリの場合だけである。

#### Rerun guard: check each existing store before scanning

codekb は、インテント間で共有されるスペースレベルのストアである。フル再スキャンは 9 つの成果物すべてを置き換え、フォーカススキャンは既存のストアにマージするので、ナレッジはインテントをまたいで蓄積される。解決した集合のすべてのリポジトリについて、読み取り専用のチェックを実行する:

```
{{INVOKE}} engine workspace codekb-scope-diff --repo <repo>
```

- **NO_STORE** - このリポジトリの初回スキャン。Step 2 へ進む。質問はしない。
- **CURRENT** - ストアの分析済みパスは、構築されて以来変わっていない。記録されたカバレッジがこのインテントの領域に役立ちそうであれば、下の再利用の質問を提示する。このインテントが明らかにストアの分析済みパスの外にあるコードを対象としている場合は、再利用の選択肢を省き、再スキャンかフォーカススキャンかだけを尋ねる。
- **STALE / UNVERIFIED / UNKNOWN_SCOPE** - ストアのナレッジが古い、検証できない、またはスコープの追跡より前のものである。下の再スキャンの質問を、再利用の選択肢を含めずに提示する。
- **SCANNED** - この実行ですでにリポジトリをスキャンし、ストアを公開している: それについては何も尋ねず、何も記録しない。スキャンすべきリポジトリが残っていなければ、Step 4 へ進む。

再利用の質問（CURRENT で、カバレッジがインテントに合う場合）- ツールの出力（ストアのインテント、分析済みパス）を問いかけに組み込み、人間が根拠に基づいて判断できるようにする:

```question
prompt: "An up-to-date code knowledge base exists for <repo> (built by intent <store-intent>; verified unchanged). Deep coverage: <analyzed paths>. Reuse it, or rescan?"
header: "Code KB"
multiSelect: false
options:
  - label: "Reuse existing knowledge base"
    description: "Skip the scan; downstream stages read the current store as-is"
  - label: "Full rescan"
    description: "Rebuild the store covering the whole repo (replaces all 9 artifacts)"
  - label: "Focused scan"
    description: "Scan this intent's area and extend the store; preserve prior prose outside it, demoting unverifiable deep coverage to shallow"
```

再スキャンの質問（STALE / UNVERIFIED / UNKNOWN_SCOPE、またはカバレッジが合わない CURRENT の場合）- 判定の行を問いかけに含める:

```question
prompt: "A code knowledge base exists for <repo> but <verdict summary - e.g. its analyzed paths have changed since it was built / it does not cover this intent's area>. A full rescan replaces it; a focused scan merges into it. How should the scan run?"
header: "Code KB"
multiSelect: false
options:
  - label: "Full rescan"
    description: "Rebuild the store covering the whole repo (replaces all 9 artifacts)"
  - label: "Focused scan"
    description: "Scan this intent's area and extend the store; preserve prior prose outside it, demoting unverifiable deep coverage to shallow"
```

リポジトリごとに1つの判断（再利用、フル再スキャン、フォーカススキャン）を記録する。再利用の判断は、別のリポジトリがまだスキャンを必要とするかもしれない間は、ステージを report したり前進させたりはしない。スキャンを選んだ場合は、その範囲も記録する。その選択が開発者へのブリーフを決め、Step 3 のスコープブロックが、スキャンが実際にカバーした範囲を記録する。

人間による再利用の判断のたびに、その直後にそのリポジトリの現在の試行の免除を記録する:

```
bun {{HARNESS_DIR}}/tools/aidlc-state.ts reuse-artifact reverse-engineering --decision keep --artifacts "<codekb-path output>" [--repo <repo>] [--single]
```

再利用する登録済みリポジトリ1つにつき1行を使う。記録されていない単一リポジトリのワークスペースでは、`--repo` を省く。単独実行（`directive.single === true`）では `--single` を付ける。ツールは、正規の 9 つの成果物からなる完全なストアが存在し、まだ `CURRENT` であることを検証し、その行をこの合成の試行に結び付ける。完了チェックは、それを受け入れる前に、成果物の権威と鮮度を独立に再検証する。

Step 2 の直前に、スキャン対象に選ばれたすべてのリポジトリについて、compare-and-swap のスナップショットを1つずつ取る:

```
bun {{HARNESS_DIR}}/tools/aidlc-utility.ts codekb-snapshot --repo <repo> --paths <source paths> --json
```

`<source paths>` は次のように選ぶ:

- フル再スキャン: `./`。
- CURRENT のストアのフォーカススキャン: ストアの既存の `analyzed.paths` と、予定しているフォーカスのパスの和集合。したがって、フルのストアでは `./` を使う。
- STALE、UNVERIFIED、UNKNOWN_SCOPE、NO_STORE のストアのフォーカススキャン: 予定しているフォーカスのパス。

返された `store_generation`、`source_fingerprint`、`paths`、`head_commit` をリポジトリごとに保持する。最初の3つは、統合を、共有 CodeKB の正確な世代と、スキャンがこれから調べるソースのバイト列の両方に結び付ける。開発者が後でスナップショットの `paths` の外にある `analyzed.paths` のエントリを報告した場合は、その結果を破棄し、拡張したパス集合に対してスナップショットとスキャンをやり直す。対応するスキャン前のソーススナップショットなしに、スキャン後に検証済みのカバレッジを広げてはならない。

すべてのリポジトリの判断が解決した後にだけ:

- 通常のワークフロー実行ですべてのリポジトリが再利用される場合は、ステージを skipped としてちょうど1回 report する: `{{INVOKE}} engine orchestrate report --stage reverse-engineering --result skipped --reason "codekb reuse: all resolved stores CURRENT, human chose reuse"`。
- 単独実行（`directive.single === true`）ですべてのリポジトリが再利用される場合は、メインワークフローの skipped の report を呼び出さない。再利用したリポジトリのサマリーをオーケストレーターの単独ステージランナーの分岐に返す。単独実行の再利用の行がそのパイプラインの証拠を満たし、単一の `report --single --stage "reverse-engineering" --result completed` はその分岐が担う。
- スキャンが必要なリポジトリが1つでもある場合は、スキップを report しない。フル/フォーカススキャンのリポジトリについてだけ Steps 2-3 へ進み、再利用する各リポジトリのストアは変更しない。再利用の行がそれらのリポジトリを免除し、スキャンしたリポジトリには引き続き両方のリンクが必要である。単独実行では、`directive.pipeline.record` のリンクのコマンドに `--single` が付く。

### Step 2: Developer Code Scan

aidlc-developer-agent を指定して Task ツールに委任する:
- subagent_type="aidlc-developer-agent"
- エージェントのペルソナとナレッジは自動的に読み込まれる。ペルソナを手動で注入してはならない。
- aidlc-state.md のワークスペースの状態をコンテキストとして含める

コンダクターはストアと再利用の判断を担うが、このディスパッチの前にアプリケーションのソースを調べたり、リポジトリを列挙したり、ファイル一覧を事前に作ったりはしない。それは開発者のリンクと重複するからである。開発者には、リポジトリのルート、インテント、選んだ範囲、アクティブな Minimal/Standard/Comprehensive の深さ、`directive.pipeline.record` にあるそのリンクのコマンドが指定する正確なハンドオフのパス（下記のパス）、Run Record 用のスナップショットの `head_commit` を渡す。ソースの範囲は開発者が見つける。

開発者へのブリーフには、Step 1 のガードで選んだスキャン範囲（フル再スキャン = リポジトリ全体、フォーカススキャン = インテントの領域で、ブリーフ中で明示的に名指しする）を示し、スキャン結果の Scan Coverage セクション（re-artifacts.md のテンプレート）に、実際に深く分析したもの、ざっと目を通したもの、開かずに除外したものを列挙するよう求める。リポジトリのスナップショットの `paths` を含める。深く分析した結果は、その集合の中に必ずとどめる。

スキャン対象に選ばれた各リポジトリについて、開発者は `<repo>` のコードベース（兄弟ディレクトリ `<workspace>/<repo>/`。単一リポジトリのインテントではコードベース全体）をスキャンし、次を調べる:
- すべてのパッケージ、モジュールとその目的
- ビルドシステム、設定、依存関係
- 外部および内部の API（エンドポイント、契約、メソッド）
- フレームワーク、ライブラリとそのバージョン
- テストディレクトリ、テストフレームワーク、カバレッジの設定
- コード品質の指標（リンティング、CI/CD、ドキュメント）
- 技術的負債の兆候

AI-DLC 自身のインストールはプロジェクトのコードではない。ブリーフでは、それをスキャンしたり文書化したりしないよう開発者に伝える:

- ハーネスのディレクトリ `.claude/`、`.kiro/`、`.codex/`、`.cursor/`、`.opencode/`、`.aidlc/`、および `aidlc/` ワークスペース
- AI-DLC が `.github/`（Copilot）と `.agents/`（Codex）の下に書くエージェント、フック、スキル: `aidlc` の名前が付いたもの、および `SKILL.md` のフロントマターに `generated-by: aidlc-runner-gen` と書かれたすべてのスキル（ステージランナー。プラグインのステージを含む）
- AI-DLC が丸ごと書くルートのファイル: インストールされたハーネスが `<harness directory>/tools/data/aidlc-projection.json` に列挙するファイル丸ごとのルート統合（コピーされた Cursor ランタイムが持ち込む `install.ts`）
- `AGENTS.md` や `.gitignore` などの共有ルートファイルのうち AI-DLC が印を付けたセクション、AI-DLC が `.mcp.json` に追加する MCP サーバー、opencode の `opencode.json` 内の AI-DLC のエントリ（`<harness directory>/tools/data/aidlc-manifest.json` の `rootContributions` に挙げられている）

`.github/`、`.agents/`、`AGENTS.md`、`.gitignore`、`opencode.json` の残りの部分はプロジェクト自身のものであり、通常どおりスキャンする。

人が書いたものだけをスキャンするよう開発者に伝える: ファイルツールで1フォルダずつファイルを探し、リポジトリの `.gitignore` ファイルを読んでそれが無視するものを除外し、ビルド出力、依存関係のフォルダ、IDE やツールのキャッシュは、何も無視していない場合でも（たとえばプロジェクトファイルの横にある .NET の `bin/` と `obj/`）開かずにスキップする。スキャンはコードを読むだけである: 利用者が求めない限り、プロジェクトのテスト、ビルド、プログラムは一切実行しない。`{{HARNESS_DIR}}/knowledge/aidlc-developer-agent/code-analysis-guide.md` の "What to Skip" セクションがそれらを列挙し、そうした名前のフォルダが手書きのコードを含む場合を、フォルダの横にあるファイルからどう見分けるかを説明している。

開発者は、`{{HARNESS_DIR}}/knowledge/aidlc-developer-agent/re-artifacts.md` の Developer Code Scan Template に従って、構造化されたスキャン結果を書く:

- 記録されていないプロジェクトルートのリポジトリ（リポジトリが登録されておらず、`codekb-path` がプロジェクトフォルダに付ける名前を出力する場合）: `<record>/inception/reverse-engineering/developer-scan.md`。その名前を `--repo` として渡す場合もこれを使い、再スキャンが以前のハンドオフを置き換えるようにする
- 登録済みのリポジトリ（リポジトリがちょうど1つの集合を含む）: `<record>/inception/reverse-engineering/developer-scan-<repo>.md`

このファイルが永続的なパイプラインのハンドオフである。開発者が返すサマリーは、ハンドオフのパスと懸念事項だけを示し、スキャンの本文は繰り返さない。

開発者の返答を読んだ後、ハンドオフのファイルが存在し、`## Developer Code Scan Results`、`### Scan Coverage`、`## Handoff Summary` を含むことを検証する。その後、アーキテクトをディスパッチする前に、`directive.pipeline.record` にあるそのコマンドでリンク 1 を記録する。

記録は、この試行が上記のパスに書いたハンドオフから行い、その後ハンドオフの名前を変えたり、移動したり、編集したりしてはならない。Guard Policy が strict の場合、却下や再開で古いファイルを再利用することはできず、ハンドオフに何らかの変更があれば、開発者とアーキテクトを再び実行することになる。

### Step 3: Architect Synthesis

**この手順は、`aidlc-architect-agent` が `directive.pipeline.links` に含まれている場合にだけ実行する。**
開発者が単独で作業する場合（協働者がオフで、リンクが開発者のリードだけの場合）は、アーキテクトをディスパッチしない。代わりに、開発者のリードが、この手順で説明する 9 つの成果物の完全な候補を自分で作らなければならない: Step 2 のスキャンとこの手順の統合仕様の両方を含むブリーフ（同じ書き込み方のルール、下記の同じステージングディレクトリの契約）で開発者をディスパッチし、開発者のリンクだけを記録して Step 4 へ進む。この手順の残りは、全メンバーがそろっている場合の経路である。

aidlc-architect-agent を指定して Task ツールに委任する:
- subagent_type="aidlc-architect-agent"
- エージェントのペルソナとナレッジは自動的に読み込まれる。ペルソナを手動で注入してはならない。
- 開発者のスキャンのハンドオフは、本文ではなくパスを渡す。アーキテクトがそのファイルを読む
- テンプレートのパス `{{HARNESS_DIR}}/knowledge/aidlc-developer-agent/re-artifacts.md` を渡す: アーキテクトはそれに基づいて 9 つの成果物と Scope of Analysis ブロックを書く
- aidlc-state.md のワークスペースの状態を含める
- Run Record 用にスナップショットの `head_commit` を渡す
- ステージングディレクトリ（`stage`）、そこにステージング済みのもの（`staged`）、インテントの slug（`intent`）のために、`{{INVOKE}} engine workspace codekb --repo <repo> --json` を渡す
- プロジェクトのソースを確認するときは開発者のルールに従うようアーキテクトに伝える: AI-DLC 自身のインストールと、`{{HARNESS_DIR}}/knowledge/aidlc-developer-agent/code-analysis-guide.md` の "What to Skip" セクションがスキップするすべてのフォルダ（たとえばプロジェクトファイルの横にある .NET の `bin/` と `obj/`）は、列挙も開くこともせず、成果物の中でそれらのファイルを一切挙げない

アーキテクトは、スキャン結果を 9 つの成果物の完全な候補に統合する:
1. **business-overview.md** — ビジネスドメイン、目的、主要な機能
2. **architecture.md** — システムアーキテクチャ、パターン、コンポーネント間の関係（Mermaid 図を含む）。ビジネストランザクションがコンポーネントをまたいでどのように実装されているかを示す Interaction Diagrams セクション（シーケンス図またはフロー図）を必ず含める。
3. **code-structure.md** — パッケージ/モジュールの構成、ファイルの分類、コードのパターン
4. **api-documentation.md** — 外部および内部の API の表面、エンドポイント、契約
5. **component-inventory.md** — 責務と依存関係を含む、コンポーネントの完全な一覧
6. **technology-stack.md** — 言語、フレームワーク、ライブラリとそのバージョン
7. **dependencies.md** — 外部の依存関係、パッケージをまたぐ内部の依存関係
8. **code-quality-assessment.md** — テストカバレッジ、リンティング、CI/CD、ドキュメントの品質、技術的負債
9. **reverse-engineering-timestamp.md** - リバースエンジニアリングを実施した時点（日付、利用可能ならコミットハッシュ）を `## Run Record` 見出しの下に記録し、最後は必ず構造化された `## Scope of Analysis` ブロックで終える。どちらも `{{HARNESS_DIR}}/knowledge/aidlc-developer-agent/re-artifacts.md` のテンプレートに基づく。開発者の Scan Coverage から、またフォーカスのマージの場合は既存のストアから、下記のルールに従って埋める - 記録するのは実際に深く検証されたものであり、目指したものではない。これが、Step 1 の再実行ガードが読む鮮度/陳腐化のマーカーである。

Step 1 で記録した書き込み方を選ぶ:

- **既存のストアがある場合のフォーカススキャン（NO_STORE 以外のすべての判定）:** 統合の前に、既存の 9 つの成果物とストアの Scope of Analysis ブロックを読む。新たに分析した領域をカバーするセクションを更新または拡張し、その外にある以前のセクションは保持する。この実行のフォーカスした結果だけから成果物を作り直してはならない。
  - **CURRENT:** `analyzed.paths` と `analyzed.components` を、ストアとこの実行の和集合にする。CURRENT の `kind: full` のストアは `kind: full` のままで、`analyzed.paths` に `./` を保持する。それ以外の場合は `kind: partial` を使い、partial のブロックには決して `./` を入れない。
  - **STALE / UNVERIFIED:** `analyzed.paths` と `analyzed.components` を、この実行だけから設定する。以前の散文は保持するが、その深いカバレッジは再検証できなかったため、ストアの以前の `analyzed.paths` を、既存の浅いパスと新たに報告された浅いパスとともに `shallow.paths` に格下げする。
  - **UNKNOWN_SCOPE:** 旧形式のストアには、和集合をとれる以前のスコープブロックがない。その散文はできる範囲でマージするが、新しいブロックにはこの実行だけを記録する。
- **フル再スキャン:** 9 つの成果物すべてを丸ごと置き換え、スコープブロックはこの実行だけから作る。既存のフル再スキャンの動作から変更はない。
- **NO_STORE:** 9 つの成果物すべてをこの実行から作る。フォーカスした初回スキャンは `kind: partial` であり、`kind: full` が有効なのは `analyzed.paths` に `./` が含まれる場合だけである。

アーキテクトは候補を、共有 CodeKB ではなく `<record>/.aidlc-engine/codekb-stage-<repo>/` に必ず書く。ステージングディレクトリには、上記の 9 つのファイル名だけがちょうど含まれ、他のエントリはない。これは一時的なトランザクションの入力であり、永続的なステージの成果物ではない。`{{INVOKE}} engine workspace codekb --repo <repo> --json` は、そのディレクトリを `stage` として、すでにその中にあるファイルを `staged` として、アクティブな記録とそのインテントの slug を `record` と `intent` として出力する。候補を書いたら、もう一度実行する: `staged` にはちょうど 9 つの名前が並んでいなければならない。

ブロックの `fingerprint:` 行には、マージ後または置き換え後のブロックの最終的な `analyzed.paths`（カンマ区切り）を指定して mint コマンドを実行し、その出力をそのまま貼り付ける:

   ```
   {{INVOKE}} engine workspace codekb-scope-diff --repo <repo> --mint --paths <analyzed paths>
   ```

その後、アーキテクトにステージング済みのタイムスタンプを確認させ、`VALID` と出力されるまでブロックを修正させる。フィンガープリントが現在のソースと一致しないと言われた場合は、もう一度 mint する。不明なフィンガープリント（git の外）は、公開時のチェックに任せる:

   ```
   {{INVOKE}} engine workspace codekb-scope-diff --repo <repo> --check <record>/.aidlc-engine/codekb-stage-<repo>/reverse-engineering-timestamp.md
   ```

Minimal の深さでも、9 つの成果物すべてと上記のすべての必須セクションは存在する。各インベントリや所見は、それを持つ成果物に一度だけ記録し、他の場所からは相互参照することで簡潔に保つ。同じソース一覧、依存関係の表、永続化に関する所見を複数のファイルにわたって繰り返さない。これは方法論の既存の深さの契約であり、出力の長さの上限ではない。

**最終的な公開ディレクトリはエンジンで解決し、パスを自分で組み立ててはならない。** 読み取り専用のツールを実行する

```
{{INVOKE}} engine workspace codekb --repo <repo>
```

（`--repo` を省くのは、記録されていないプロジェクトルートのリポジトリの場合だけである。リポジトリがちょうど1つの集合を含め、登録済みのリポジトリの識別子にはすべて渡す）。
このツールは1行だけを出力する: 正確な最終ディレクトリで、たとえば `aidlc/spaces/<active-space>/codekb/<repo>/` である。`--json` を付けると `{space, repo, dir, record, intent, stage, staged}` を出力し、パスはすべてプロジェクトからの相対パスである。マージのために既存のストアをこのディレクトリから読むのはよいが、候補をそこに直接書いてはならない。

**カバレッジの安全網 - 書き込みの前に実行する（比較には変更前の以前のストアが必要）。** Step 1 のガードが既存のストアを見つけた場合（NO_STORE 以外のすべての判定）は、新しい、またはマージしたタイムスタンプの内容をファイル書き込みツールで `<record>/inception/reverse-engineering/scope-draft-<repo>.md` に書き（リポジトリごとに1つの下書き。タイムスタンプのファイル名にはしない - 記録ディレクトリの配置チェックは成果物の語幹をキーにしている）、単独のコマンドとして次を実行する

```
{{INVOKE}} engine workspace codekb-scope-diff --repo <repo> --compare <record>/inception/reverse-engineering/scope-draft-<repo>.md
```

Step 5 の完了サマリーのために、出力を `<repo>` ごとに保持する。これは、要求された範囲とフォーカスのマージのルールに対する決定論的な安全網である: COVERS は、新しいブロックが以前の検証済みカバレッジを保持したことを意味し、NARROWER は格下げされた、または失われたカバレッジを示す。"Full rescan" を選んだ後のフォーカスした実行も、承認の前にここで NARROWER として表に出る。比較は、そのリポジトリの `scope-draft-<repo>.md` を読み終えると削除する（"The scope draft has been removed."）。スコープの下書きは一時的なもので、インテントの記録に残ることはないので、自分で削除してはならない。

Step 2 の直前に取得したスナップショットの値をそのまま使い、完全な候補を compare-and-swap のユーティリティで公開する:

```
bun {{HARNESS_DIR}}/tools/aidlc-utility.ts codekb-publish \
  --repo <repo> \
  --staged <record>/.aidlc-engine/codekb-stage-<repo>/ \
  --paths <snapshot paths> \
  --expect-store <snapshot store_generation> \
  --expect-source <snapshot source_fingerprint> \
  --json
```

このユーティリティは、9 つのファイルすべてと候補のタイムスタンプを検証し、スペース+リポジトリのロックを取得し、ソースと共有ストアの世代を再確認してから、ステージング済みのディレクトリ全体を、ロールバック/リカバリ付きで最終的な `codekb-path` の場所に入れ替える。他のどの手順も、それら 9 つの共有ファイルに書き込んではならない。

- `CODEKB_STORE_CHANGED`: このリポジトリのスナップショットの後に、別のインテントが公開した。Step 1 の状態チェックを再実行し、新しいストアを読み、フォーカスのマージとスコープの和集合/格下げを計算し直し、新しい候補のパス集合に対して新しいスナップショットを取り、公開を再試行する。既存の開発者のスキャンを再利用してよいのは、同じパスに対する新しいスナップショットが同じ `source_fingerprint` を返す場合だけである。
- `CODEKB_SOURCE_CHANGED`: スキャン前のスナップショットの後にソースのバイト列が変わった。ステージング済みの候補は古い: 新しいスナップショットを取り、そのリポジトリについて Step 2 と統合をやり直し、ステージング済みの 9 つのファイルを上書きしてから再試行する。
- `CODEKB_CANDIDATE_STALE`: タイムスタンプのフィンガープリントが、現在公開しようとしているソースから mint されたものではない。候補を作り直して再試行する。

拒否を、直接の書き込みや、新たに観測した世代を古い候補に差し込むことで回避してはならない。公開が成功すると、そのリポジトリの `.aidlc-engine/codekb-stage-<repo>/` ディレクトリは自動的に削除される。決して手で削除してはならない。結果が `"staged_removed": false` を報告した場合は、ステージング済みのファイルが読まれた後に変更されている: そのディレクトリは次の公開で上書きされるよう残しておく。最終ディレクトリは、スペース内のすべてのインテントで共有される、リポジトリごとの永続的なコードナレッジベースであり続ける。

アーキテクトの返答を読み、そのリポジトリの 9 つの成果物がすべてそろったら、`directive.pipeline.record` にあるそのコマンドでアーキテクトのリンクを記録する（全メンバーがそろっている場合の経路のみ: 開発者が単独で作業する場合は、開発者がすでに成果物を書いており、そのリンクが唯一のリンクである）。

選ばれたすべてのリポジトリのチェーンが、`directive.pipeline.links` の各リンクについてレシートを持つまで、完了を report してはならない — 全メンバーがそろっている場合は両方のリンク、開発者が単独で作業する場合は開発者のリンクだけである。

### Step 4: Completion Handoff

選ばれたすべてのリポジトリのスキャンが完了したら、`stage-protocol.md` の完了シーケンスに次の順序で従う:

1. Step 5 の告知とリポジトリごとのサマリーを、NARROWER の警告があればそれも含めて提示する。
2. `directive.protocol_modules` に `learnings` が挙げられている場合は、学びのステップを行う: 利用者への質問がある場合は、それを尋ねてターンを終える。回答が記録されたら続行する。
3. `{{INVOKE}} engine orchestrate report --stage reverse-engineering --result awaiting-approval` で承認ゲートをちょうど1回開く。
4. Step 5 の承認の質問を尋ねる。

利用者の回答は、その後 `approved` または `rejected` として report される。通常のワークフロー実行では、このステージを `completed` として report することはない。単独実行（`directive.single === true`）では、代わりにオーケストレーターの単独ステージランナーの分岐に戻る。`report --single --stage "reverse-engineering" --result completed` はその分岐が担う。
その `report` 呼び出しが、ライフサイクルの遷移と前進をすべて担う。散文の中で遷移を実行してはならず、この記録処理をユーザーに説明してもならない。

### Step 5: Present Completion & Request Approval

stage-protocol.md の完了テンプレートを使う:
- 完了サマリーを伴う告知
- **リポジトリごと**に作成された 9 つの成果物すべてのサマリー（複数リポジトリのインテントでは、各リポジトリの `aidlc/spaces/<active-space>/codekb/<repo>/` の一式を列挙する — Step 3 で `codekb-path --repo <repo>` が出力したディレクトリ）。既存のストアを変更せずに残した、再利用したリポジトリを明示する
- **Step 3 の比較が NARROWER を返したすべてのリポジトリについて**、サマリーには質問の前にリポジトリ名を付けた警告を必ず含め、そのリポジトリのツールのカバレッジ一覧をそのまま引用する:

  ```
  WARNING for <repo>: this scan's verified scope is narrower than the previous
  store. On a focused merge, prior prose is preserved, but deep coverage for
  the following paths and components was demoted (affected paths remain
  recorded as shallow):
  <paths and components from the compare output>
  Choose Request Changes to widen the scan instead.
  ```

  （COVERS、または以前のストアがない場合は、警告の行は不要である。）
- ナレッジベースの鮮度チェック（そのフィンガープリントが何をカバーするか、何があれば古くなるか）はサマリーに含めない: 利用者がそれについてすることは何もない。
- レビュー対象のパス: 集合内の各リポジトリについて `aidlc/spaces/<active-space>/codekb/<repo>/`
- 選択肢付きの構造化された承認の質問: Approve（Requirements Analysis へ進む）/ Request Changes。NARROWER を返したリポジトリが1つでもある場合、Approve の選択肢の説明には、どのストアの検証済みカバレッジが狭くなったかを書かなければならない（例: "Accept the narrower verified coverage for <repos>; continue to Requirements Analysis"）。

## Sensors

このステージの出力は、`aidlc/spaces/<active-space>/codekb/<repo>/`（`codekb-path --repo <repo>` が解決するディレクトリ）配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: なし。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。モジュールがない場合は、日誌とステップの両方を省略する。
