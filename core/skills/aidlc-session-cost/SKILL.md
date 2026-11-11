---
name: aidlc-session-cost
description: >
  Read-only session cost view. Prints deterministic aggregates for the
  current workflow — duration, stage outcomes, memory entries, sensor
  firings, learnings captured — sourced entirely from
  `{{INVOKE}} engine runtime summary`. Never mutates workflow state,
  never emits audit events, never writes files.
argument-hint: ""
user-invocable: true
classification: read-only
---

# AI-DLC Session Cost

## Purpose

現在のワークフローが何を消費したかについて、透明で決定論的なビューをチームに提供する:
どれだけの時間実行されたか、いくつのステージがゲートを通過したか、
オーケストレーターが観察の日誌にどれだけ書き込んだか、センサーが何回発火したか、
そして学びがいくつ記録されたか。

このスキルが出力する数値はすべて
`{{INVOKE}} engine runtime summary --json` から得る。これは `runtime-graph.json` を
イベントソーシングで集計して実体化したビューである。このスキルは**独自の集計を
一切行わない**。トークンを見積もらず、成果物のツリーを走査せず、
`audit.md` も読まない。ツールの出力にない数値を、
このスキルが作り出すことはない。

## Classification

読み取り専用。このスキルは、ワークフローのステージポインタを決して進めず、
監査イベントを決して発行せず、ファイルを決して書き込まない。ステージの途中を含め、
ワークフローのどの時点でも安全に実行できる。

## Steps

### Step 1: Read the aggregates

次を実行する:

```bash
{{INVOKE}} engine runtime summary --json
```

コマンドが 0 以外で終了した場合（`runtime-graph.json` がまだない — ワークフローが
グラフをまだコンパイルしていない）、次を出力する:

```
No session data yet.

Session cost becomes available once a workflow has started and its
first stage transition has compiled runtime-graph.json. Run /aidlc to
begin, then re-run /aidlc-session-cost.
```

そして停止する。

それ以外の場合は JSON を解析する。形式は次のとおり:

```jsonc
{
  "workflow_id": "...",          // ISO timestamp of the live workflow
  "scope": "...",
  "started_at": "...",
  "duration_minutes": 40,         // null when nothing has completed yet
  "stages":   { "total": N, "approved": N, "failed": N, "pending": N },
  "by_phase": { "<phase>": { "total": N, "approved": N, "failed": N, "pending": N }, ... },
  "memory":   { "total": N, "interpretations": N, "deviations": N, "tradeoffs": N, "open_questions": N },
  "sensors":  { "total": N, "passed": N, "failed": N, "could_not_run": N, "budget_override": N, "incomplete": N },
  "learnings":{ "from_orchestrator": N, "from_user_addition": N }
}
```

### Step 2: Render the report

各フィールドをそのまま出力する — どの値も再計算、丸め、再見積もりしてはならない。
`duration_minutes` が `null` の場合は `in progress` を使う。

```
Session Cost
============

Workflow:   {workflow_id}
Scope:      {scope}
Duration:   {duration_minutes} min   (or "in progress")

Stages
  Total:      {stages.total}
  Approved:   {stages.approved}
  Failed:     {stages.failed}
  Pending:    {stages.pending}

By phase
  {phase}    {approved}/{total} approved[, {failed} failed][, {pending} pending]
  ...

Memory entries
  Total:            {memory.total}
  Interpretations:  {memory.interpretations}
  Deviations:       {memory.deviations}
  Trade-offs:       {memory.tradeoffs}
  Open questions:   {memory.open_questions}

Sensors
  Fired:            {sensors.total}
  Passed:           {sensors.passed}
  Failed:           {sensors.failed}
  Could not run:    {sensors.could_not_run}
  Budget-override:  {sensors.budget_override}
  Incomplete:       {sensors.incomplete}

Learnings captured
  From orchestrator:    {learnings.from_orchestrator}
  From user additions:  {learnings.from_user_addition}
```

### Step 3: Surface advisory notes (optional, narrative only)

表の後に、短い説明の注記を加えてもよい — たとえば、多くのステージがまだ保留中である
ことや、センサーが頻繁に `incomplete` で発火していることを指摘する。注記は 1〜2 文にとどめ、
上記の数値だけに基づかせる。ツールが報告していない指標を作り出してはならない。

> トークンに関する注記: このスキルは意図的にトークンの見積もりを**出力しない**。
> 廃止された、ファイルサイズからトークン数を推定するヒューリスティックは、データの体裁をとった
> 当て推量にすぎなかった。実際のトークン集計が必要な場合は、ファイルサイズからの概算ではなく、
> Claude Code のセッションから読み取ること。
