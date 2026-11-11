---
name: aidlc-replay
description: >
  Print a structured session narrative for stakeholders who weren't in
  the room. Numbers (stage counts, phase rollup, duration) come from
  `{{INVOKE}} engine runtime summary`; prose comes from the audit trail and
  artefacts. Renders to the terminal only — writes no file, never
  mutates workflow state, never emits audit events.
argument-hint: ""
user-invocable: true
classification: read-only
---

# AI-DLC Session Replay

## Purpose

ワークフローの監査証跡と成果物を、読みやすいストーリーに変換する。何が、
どの順序で、なぜ決定されたのかを示す。非同期レビュー、その場にいなかった
ステークホルダー向け、あるいはセッション後の記録として使う。生ログのダンプではない。

## Classification

読み取り専用。このスキルはナラティブをターミナルに出力し、ファイルは
**一切書き込まない**。ワークフローのステージポインタを進めることはなく、
監査イベントを発行することも決してない。

## The counting rule

すべての件数と集計値（ステージ数、フェーズ別の内訳、所要時間、
承認済み/失敗/保留の集計、記録された学び）は、ファイルを目で数えるのではなく
ツールから取得する:

```bash
{{INVOKE}} engine runtime summary --json
```

ナラティブの散文（何が起きたか、主要な決定、その理由）は、アクティブな記録の
監査シャードと成果物から、あなたが組み立てる。骨格となる数値はツールのものを使う。
ツールがすでに数値を報告している場合は、ステージや成果物を決して手で数えてはならない。

## Steps

### Step 1: Read the aggregates

`{{INVOKE}} engine runtime summary --json` を実行する。

非ゼロで終了した場合（まだ `runtime-graph.json` がない場合）は、次を出力する:

```
No session data yet — start a workflow with /aidlc before running
/aidlc-replay.
```

そして停止する（STOP）。それ以外の場合は、パースした JSON を保持しておく。レポート内の
すべての数値について、そのフィールドを引用する。

### Step 2: Read the narrative sources

`aidlc/active-space` から `<active-space>` を解決し（デフォルトは `default`）、
`aidlc/spaces/<active-space>/intents/active-intent` から
`<active-intent>` を解決する。アクティブなワークフローの
記録は
`aidlc/spaces/<active-space>/intents/<active-intent>` である。

- `<record>/audit/` 配下のすべての `.md` シャード — 完全なイベント証跡。ナラティブ用に、
  イベントを `Timestamp` フィールドの順に並べる。
- `<record>/aidlc-state.md` — アクティブステージのカーソル。
- `<record>/<phase>/` 配下を再帰的にたどった成果物 — 各ステージが生成したもの
  （ユニットごとの Construction の出力を含む）。

これらは *散文* のための情報源である。Step 1 の JSON がすでに件数を持っている場合は、
これらから件数を導き出してはならない。

### Step 3: Render the replay

次の形式でナラティブをターミナルに出力する（ファイルは書き込まない）:

```markdown
# Session Replay
**Workflow**: {summary.workflow_id}
**Scope**: {summary.scope}
**Duration**: {summary.duration_minutes} min   (or "in progress")
**Stages**: {summary.stages.approved} approved / {summary.stages.total} total

## Executive Summary
{3-5 sentences: what was built or decided, key choices, constraints, outcome}

## Timeline
{For each phase in summary.by_phase, in workflow order:}

### {Phase} Phase  —  {by_phase[phase].approved}/{by_phase[phase].total} stages approved

#### {Stage Name}
**What happened**: {1-2 sentences from the audit trail}
**Key decisions**: {bullets, with reasoning drawn from the audit shards}
**Artefacts produced**: {list with one-line descriptions}

{...repeat per stage that executed...}

## Decisions Register Summary
{Table: decision | alternatives considered | chosen option | rationale}

## Learnings Captured
From orchestrator: {summary.learnings.from_orchestrator}
From user additions: {summary.learnings.from_user_addition}
{Then narrate the notable ones from the stage memory.md diaries.}

## What's Next
{Outstanding open threads from the last audit entries / open questions}
```

### Step 4: Offer adjustments

次のように提案してもよい: "Want a different tone (more technical / more executive)
or an added section?"（別のトーン（より技術的 / より経営層向け）にするか、セクションを追加しますか？）
— 求められたらターミナルに再出力する。この提案を繰り返してはならない。提案は1回で十分である。
ユーザーがリプレイの保存を望む場合は、ここでファイルを書き込むのではなく、
`/aidlc-outcomes-pack`（ファイルを書き込むスキル）を案内する。
