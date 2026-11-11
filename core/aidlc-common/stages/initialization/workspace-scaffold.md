---
slug: workspace-scaffold
phase: initialization
execution: ALWAYS
condition: Ensure-exists the per-intent record and in-scope phase dirs, idempotent (creates on demand, skips existing)
lead_agent: orchestrator
support_agents: []
mode: inline
produces: []
consumes: []
requires_stage: []
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
inputs: none (first stage after session start)
outputs: the per-intent record tree (one dir per in-scope phase + verification dir) and the space-level knowledge/ dir
---

# Workspace Scaffold

`aidlc-utility intent-create` の内部で決定的に実行される。ワークスペースの外枠は `dist/`（SEED）に同梱されており、インテント作成時にはインテントごとの記録と、スコープ内フェーズのディレクトリが存在することを保証するだけである（必要時に冪等に作成）。監査イベントの意味を示す参照資料として残している。

## Steps

### Step 1: Update State

1. `<record>/aidlc-state.md` を更新する: `Current Stage` を `scaffolding workspace` に設定する
2. workspace-scaffold を `[-]`（進行中）にする

### Step 2: Ensure the Space Shared Directories

スペースレベルの空の CodeKB 親ディレクトリ
`aidlc/spaces/<space>/codekb/` が存在することを保証する。これにより、Reverse Engineering の
実行前でも共有ストアを安全に参照できる。リポジトリごとのディレクトリは遅延作成のままであり、
`codekb/<repo>/` は Reverse Engineering がそのリポジトリの成果物を書き込んだときにのみ
現れる。

スペースレベルのドメインナレッジディレクトリ
`aidlc/spaces/<space>/knowledge/`（略記 `aidlc/knowledge/`）が存在することを保証する。これは
**自由形式で、ブートストラップ時には空** である — 固定のファイル構成も、エージェントごとの
サブディレクトリも、初期配置の README もない。チームが時間をかけて独自の Markdown をここに追加していく。
このディレクトリは `memory/`、`codekb/`、`intents/` と同階層にあるため、ドメイン
ナレッジは 1 つのインテントの記録に閉じ込められることなく、スペース内のすべてのインテントにわたって蓄積される。
エージェントのペルソナは、`aidlc/knowledge/aidlc-shared/` と `aidlc/knowledge/<agent>/` が存在すれば
そこからチームのナレッジを読み込む。
これらを作成するのはチームであり、インテント作成ステップではない。（エンジンのエージェントごとの方法論（METHODOLOGY）
ナレッジは、別途 `{{HARNESS_DIR}}/knowledge/` 配下に読み取り専用で同梱される。）

### Step 3: Ensure Phase Artifact Directories

アクティブなインテントの記録ディレクトリ `aidlc/spaces/<space>/intents/<YYMMDD>-<label>/` 配下に、
インテントごとのフェーズ成果物用の空ディレクトリ（README なし）が存在することを保証する。
冪等に（必要時に）作成する:

- スコープが実行する（SCOPE RUNS）フェーズごとに 1 ディレクトリ: `<record>/initialization/` と、
  `ideation/`、`inception/`、`construction/`、`operation/` のうち、アクティブなスコープで
  EXECUTE となるステージを 1 つ以上含むもの
- `<record>/verification/`（スコープに依存しない）

スコープが完全に除外するフェーズにはディレクトリを作成しない（NO）。bugfix の記録に空の `operation/` があると、
そのフェーズが最初から計画になかったにもかかわらず、計画されたうえでスキップされた作業のように読めてしまう。
現れるフェーズはワークフローが実行するフェーズとちょうど一致し、
それ以外は監査証跡の `PHASE_SKIPPED` イベントが示す。

ステージごとのディレクトリはここでは作成しない（NOT）。ステージのディレクトリ
（`<record>/<phase>/<slug>/`）は、そのステージが初めて成果物を書き込んだときに現れる。そのため
記録には、何かを生み出したステージだけが表示される。
`reverse-engineering/` が最初から現れないのも同じ理由である: このステージは 9 つの
成果物を、インテントの記録ではなく、スペースレベルのリポジトリ別ストア `aidlc/spaces/<space>/codekb/<repo>/`
（リポジトリごとに 1 つの共有ビューで、brownfield の再実行のたびに書き直される）に書き込む。
ステージ実行時に `<record>/inception/reverse-engineering/` に置かれるのは、自身の `memory.md` 日誌だけである。
書き込み先のパスはステージファイルを参照すること。

### Step 4: Display Confirmation

ワークスペースの準備ができたことを平易な 1 行で伝え、ユーザーの作業が置かれる
単一のディレクトリ名を示す。ディレクトリツリーは表示しない: フォルダ構成は
フレームワーク内部の管理事項であり、ユーザーが読む必要のあるものではない。

### Step 5: Update State and Audit

1. `<record>/aidlc-state.md` で workspace-scaffold を `[x]`（完了）にする
2. 監査証跡への WORKSPACE_SCAFFOLDED の記録はエンジンが行う。自分で追記してはならない

### Step 6: Auto-Proceed

このステージには承認ゲートが**ない**（NO）— 次のステージ（workspace-detection）へ自動的に進む。

## Sensors

このステージは `aidlc-utility intent-create` の内部で決定的なセットアップ処理を実行する —
インテントごとの記録とスコープ内フェーズのディレクトリが存在することを保証し、状態イベントを発行する。
エージェントが書く Markdown はここには置かれないため、frontmatter の `sensors:` リストは
空である。

インポート: なし。

セットアップレポートをカスタマイズする場合は、関連するマニフェストをここでインポートする。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学習ステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
