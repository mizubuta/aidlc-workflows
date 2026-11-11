---
slug: workspace-detection
phase: initialization
execution: ALWAYS
condition: Scans and classifies workspace — auto-proceeds (no approval gate)
lead_agent: orchestrator
support_agents: []
mode: inline
produces: []
consumes: []
requires_stage:
  - workspace-scaffold
sensors: []
scopes:
  - enterprise
  - feature
  - mvp
  - poc
  - bugfix
  - refactor
  - infra
  - security-patch
  - classic
  - workshop
  - express
inputs: none (scans filesystem)
outputs: workspace classification (greenfield/brownfield), technology stack detection
---

# Workspace Detection

`aidlc-utility init` の中で決定的に実行される。以下の Step 3 の検出ルールが、スキャナーの分類ロジックにおける信頼できる情報源である。

## Steps

### Step 1: Update State

1. `<record>/aidlc-state.md` を更新する: `Current Stage` を `detecting workspace` に設定する
2. workspace-detection を `[-]`（進行中）としてマークする

### Step 2: Scan Workspace

スキャナーは、トップレベルのファイルと既知のソースディレクトリ（`src/`、`app/`、`lib/`、`pages/`、`components/`、`tests/`）を調べる。ただし、ハーネスのディレクトリ（`.claude/`、`.kiro/`、`.codex/`、`.opencode/`、`.aidlc/`、`.cursor/`）、`aidlc/`、`node_modules/`、`.git/`、`dist/`、`build/`、`.next/`、`target/`、`vendor/` は除外する。

また、AI-DLC がフォルダーに丸ごと書き込んだファイル（Cursor のルートにある `install.ts` など）も決して数えない。インストールされた各ハーネスは、これらをファイル単位のルート統合として `<harness directory>/tools/data/aidlc-projection.json` に列挙しており、スキャナーはその一覧を読み込む。ルートにある `install.ts` のうち、インストール済みのどのハーネスも自分のものとして主張していないものは、プロジェクト自身のコードであり、引き続き数える。

ネストしたプロジェクトのフォールバック: トップレベルのシグナルが**1 つも**検出されない場合（そのままなら greenfield と分類されるレイアウト）、スキャナーは任意の名前のコンテナディレクトリを決定的に再帰走査する。走査はワークスペースルートから 3 階層下までに制限される。各階層で、上記の除外ディレクトリ、サンプル/ドキュメント用ディレクトリ、既知のソースディレクトリ名、隠しディレクトリ、シンボリックリンク、ディレクトリ以外のものをスキップし、訪問した各ディレクトリで同じシグナル群を再適用する（そのディレクトリ自身の既知ソースディレクトリへの再帰も含む）。上限内で brownfield と判定されたヒットはすべて、その言語/フレームワーク/ビルドシステムが結果にマージされ、スラッシュで連結した相対パスがネストしたルートとして記録される。ウォーカーはヒットより下には降りない。少なくとも 1 つのヒットが見つかった場合、訪問した git リポジトリ（`.git` を持つディレクトリ）のうち、その位置または配下にヒットがないものもすべてネストしたルートとして記録される。これにより、複数のリポジトリをまとめた親フォルダーでは、言語リスト外のファイル（`index.html` など）しか持たないリポジトリがあっても、すべてのリポジトリが挙げられる。これが分類を変えることはない。この仕組みにより、ファイルの重複カウントを避けつつ、`services/api/src/main.py` のようなレイアウトを捉えられる。ルートにすでにソースのシグナルがある場合、フォールバックは決して実行されない。

スキャンのシグナル:
- ディレクトリ構造（トップレベルと主要なサブディレクトリ）
- 設定ファイル（package.json、pom.xml、build.gradle、Cargo.toml、pyproject.toml など）
- ビルドシステムのファイル（Makefile、Dockerfile、docker-compose、CI/CD の設定）
- パッケージ/依存関係のファイル（ロックファイル、vendor ディレクトリ）
- ソースコードのディレクトリとその言語
- リポジトリのメタデータ（`.gitmodules` によるサブモジュール宣言）
- テスト基盤（テストディレクトリ、テスト設定ファイル、カバレッジ設定）
- ドキュメント（README、docs/、wiki/）

**分析から除外するもの**（アプリケーションコードではなく、フレームワークの足場）:
- ハーネスのディレクトリ（`.claude/`、`.kiro/`、`.codex/`、`.opencode/`、`.aidlc/`、または `.cursor/`）— AI-DLC フレームワークのファイル（スキル、エージェント、フック、ツール、ナレッジ）
- `aidlc/` — AI-DLC のワークスペースルート（`aidlc/spaces/<space>/...` にあるスペースツリー）
- AI-DLC が丸ごと書き込んだルートのファイル（Cursor の `install.ts`）。インストール済みのハーネスが列挙しているもの（上記参照）
- `node_modules/`、`.git/`

### Step 3: Detect Project Type

スキャナーが集めた根拠に基づいて分類する:

シグナルはまずルートで評価される。どれも検出されない場合、ネストしたプロジェクトのフォールバックが、ルートから 3 階層下までの候補コンテナディレクトリで同じシグナルを再評価する（Step 2 を参照）。

**Brownfield** — 次の指標の**いずれか**が存在する:
- ソースコードファイルが存在する（`.js`、`.ts`、`.jsx`、`.tsx`、`.py`、`.java`、`.go`、`.rs`、`.rb`、`.cs`、`.cpp`、`.c`、`.kt`、`.swift`、`.php`）。インフラストラクチャのコードも含む（`.tf`、`.hcl`、`.bicep`。Terraform の `.terraform.lock.hcl` は数えない）
- アプリケーションフレームワークの設定が検出される（next.config、vite.config、angular.json など）
- アプリケーションの依存関係を含むパッケージマニフェストがある（dev 以外の依存を持つ package.json、requirements.txt、Cargo.toml、go.mod、pom.xml など）
- アプリケーションのソースディレクトリが存在する（src/、app/、lib/、pages/、components/）
- ワークスペースルートに解析可能な `.gitmodules` があり、少なくとも 1 つのサブモジュールパスのエントリを含む（サブモジュールのディレクトリがまだ初期化されていなくても、リポジトリのメタデータがコードの存在を宣言している）

**Greenfield** — 次の条件が**すべて**成り立たなければならない:
- 認識対象のどの言語のソースコードファイルもない
- アプリケーションフレームワークの設定がない
- パッケージマニフェストがない、**または**マニフェストが足場/開発用ツールのみを含む
- アプリケーションのソースディレクトリがない

プロジェクトを brownfield に**しない**もの: README、.gitignore、LICENSE、エディターの設定、空のディレクトリ、アプリケーションコードを伴わない CI/CD の定型ファイル、ハーネスのディレクトリ（`.claude/`、`.kiro/`、`.codex/`、`.opencode/`、`.aidlc/`、または `.cursor/`。AI-DLC フレームワーク）、`aidlc/` ディレクトリ（AI-DLC のワークスペース成果物）、AI-DLC が丸ごと書き込んだルートのファイル（Cursor の `install.ts`、opencode の `opencode.json`）。

### Step 4: Verify Classification

決定的なスキャナーが Step 3 のルールを直接適用する。作業の性質について利用者が述べたことはスキャン結果より優先され、`Project Type Source` に誰が決めたか（`workspace scan` または `you`）が記録される:

- 開始時に、リクエストとともに `/aidlc --project-type brownfield`（または `greenfield`）を指定すると、種別が設定される。その場合でも、言語、フレームワーク、ビルドシステムはスキャンによって埋められる。
- 後になって、利用者が自分の言葉で、作業が既存コードに対するものである、あるいは新規プロジェクトである（たとえば、新規として扱うつもりの `create-next-app` の足場）と述べた場合は、`{{INVOKE}} engine orchestrate next --project-type brownfield`（または `greenfield`）を実行する。エンジンはフォルダーを再スキャンし、種別を利用者が決めたものとして記録し、Workspace State を更新する。既存コードの場合は、作成以降に追加されたリポジトリを記録し、Reverse Engineering を再びプランに組み込む。ワークフローがすでにそのステージを過ぎている場合は、次に Reverse Engineering が実行され、その後ワークフローは利用者がいたステージに戻る。`Project Type` を決して手で編集しない。
- スキャンによって新規プロジェクトとして設定され、Construction の前にフォルダーにコードが追加された場合、`next` は利用者にどちらなのかを一度だけ尋ねる。どちらの回答も利用者が決めたものとして記録されるため、再び尋ねることはない。

### Step 5: Identify Technology Stack

スキャン結果から次を特定する:
- **言語**: 検出された主要言語と副次的な言語
- **フレームワーク**: Web フレームワーク、ライブラリ、UI ツールキット
- **ビルドシステム**: ビルドツール、タスクランナー、パッケージマネージャー
- **テスト基盤**: テストフレームワーク、カバレッジツール、テストランナー

### Step 6: Update State and Audit

1. `<record>/aidlc-state.md` で workspace-detection を `[x]`（完了）としてマークする
2. 検出した言語、フレームワーク、ビルドシステムで Workspace State セクションを更新する
3. エンジンが、スキャン結果と分類を含めて WORKSPACE_SCANNED を監査証跡に記録する。自分で追記しては決してならない

### Step 6a: Relay the Submodule Warning (if present)

作成時の出力に未初期化サブモジュールの警告が含まれている場合（スキャナーが、サブモジュールのパスが空/未初期化の
`.gitmodules` を見つけた場合）は、その警告をそのままユーザーに伝え、
作業を進める前に `git submodule update --init --recursive` を実行するよう伝える。reverse-engineering はディスク上のコードを必要とするためである。
コマンドを自分で実行すると申し出ては**ならない**。また、自動進行をブロックしては**ならない**。これは助言としての
伝達にすぎない。

### Step 7: Auto-Proceed

このステージには承認ゲートが**ない**。次のステージ（state-init）へ自動的に進む。

## Sensors

このステージは `aidlc-utility init` の中でワークスペーススキャナーを実行する。
出力するのは分類の状態であり、エージェントが作成する Markdown ではない。そのため、
frontmatter の `sensors:` リストは空である。

インポート: なし。

カスタマイズした検出レポートでは、関連するマニフェストをここでインポートすること。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
