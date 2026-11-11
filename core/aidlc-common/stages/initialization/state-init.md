---
slug: state-init
name: State Initialization
phase: initialization
execution: ALWAYS
condition: Creates full populated state file and determines routing — auto-proceeds
lead_agent: orchestrator
support_agents: []
mode: inline
produces: []
consumes: []
requires_stage:
  - workspace-detection
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
inputs: workspace classification from workspace-detection, scope from orchestrator
outputs: <record>/aidlc-state.md (full populated version, engine-resolved)
---

# State Initialization

`aidlc-utility init` の中で決定論的に実行される。状態ファイルの契約の参照用として残している。

## Steps

### Step 1: Update State

1. `<record>/aidlc-state.md` を更新する: `Current Stage` を `initializing state` に設定する
2. state-init を `[-]`（進行中）としてマークする

### Step 2: Create Full State File

`{{HARNESS_DIR}}/knowledge/aidlc-shared/state-template.md` から状態の契約を読み込む。
コンパイル済みのステージグラフとスコープグリッドから生成した完全な記入済みバージョンで、
`<record>/aidlc-state.md` を上書きする:
- プロジェクトの説明: 正確なテキストを 1 つの JSON 文字列として
  `<record>/project-description.json` に保存する。状態の `Project` フィールドには、
  安全な 1 行のプレビューだけを書き込む
- プロジェクトの種類（workspace-detection による greenfield/brownfield）
- ワークスペースの状態（workspace-detection による言語、フレームワーク、ビルドシステム）
- 開始日: `{{INVOKE}} engine now` が出力する時刻
- スコープ構成（スコープのルーティングに従って実行／スキップするステージ）
- ステージ進捗のチェックボックス一式（全ステージ。INITIALIZATION ステージのうち workspace-scaffold、workspace-detection は [x] とマークする）
- state-init を `[-]`（進行中）としてマークする
- Total Stages: EXECUTE のステージだけを数える（SKIP は数えない）。正式な数は
  コンパイル済みのスコープグリッド（`{{HARNESS_DIR}}/tools/data/scope-grid.json`）にあり、
  各ステージの `scopes:` frontmatter を転置したものである。現在のスコープごとの数は
  `{{INVOKE}} engine gen scope-table` を、現在のコンパイル済みステージ一覧は
  `{{INVOKE}} engine gen stage-table` を
  実行して確認する。
- Completed: 完了した INITIALIZATION ステージの数を設定する（通常は 3）
- In Progress: 初期化後の最初のステージ名を設定する
- Active Agent: 初期化後の最初のステージのリードエージェントを設定する（Stage Graph から）

### Step 3: Determine Routing

プロジェクトの種類に応じて:
- **Brownfield** → 初期化後の最初のステージ: reverse-engineering（Inception）
- **Greenfield** → 初期化後の最初のステージ: requirements-analysis（Inception）。reverse-engineering はスキップする

ルーティングの決定を aidlc-state.md に反映する:
- スコープとプロジェクトの種類に基づいて `Stages to Execute` と `Stages to Skip` を設定する
- greenfield プロジェクトでは reverse-engineering を SKIP としてマークする

### Step 4: Finalize State

**`--init` から呼び出された場合:**
- Lifecycle Phase を READY に設定する
- Current Stage を `workspace initialized — run /aidlc [scope] to start` に設定する
- Ideation フェーズへは進んではならない

**ワークフローの開始から呼び出された場合:**
- Lifecycle Phase を初期化後の最初のフェーズ（スコープに応じて IDEATION または INCEPTION）に設定する
- Current Stage を初期化後の最初のステージに設定する

### Step 5: Update State and Audit

1. `<record>/aidlc-state.md` で state-init を `[x]`（完了）としてマークする
2. エンジンが、プロジェクトの種類と技術スタックの概要とともに WORKSPACE_INITIALISED を監査証跡に記録する。自分で追記してはならない

### Step 6: Auto-Proceed

このステージには承認ゲートが一切ない。初期化後の最初のステージへ自動的に進む（--init から呼び出された場合は停止する）。

## Sensors

このステージは `aidlc-state.ts` を通じて `<record>/aidlc-state.md` を
決定論的に書き込む。状態ファイルは構造化されたマニフェストであり、
markdown の形状センサーが対象とする自由形式の成果物ではない。そのため
frontmatter の `sensors:` リストは空である。

インポート: なし。

将来の状態の形状チェックは、ここでインポートする専用のマニフェストにするべきである。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを行う。利用者に何かを尋ねるタイミングは
そのモジュールが定める。モジュールがない場合は、日誌と学びのステップの両方を
スキップする。
