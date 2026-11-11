---
name: aidlc-outcomes-pack
description: >
  Generate a comprehensive handover document at workflow close so the
  team can own, operate, and continue the system without re-running the
  workflow. Stage/phase/learning counts come from
  `{{INVOKE}} engine runtime summary`; prose comes from the artefacts. Writes
  OUTCOMES.md but never mutates workflow state or emits audit events.
argument-hint: ""
user-invocable: true
classification: read-only
---

# AI-DLC Outcomes Pack

## Purpose

ワークフローの終了時に、納品したシステムをチームが所有し、運用し、
開発を続けるために必要なすべてを、コンテキストを取り戻すために
ワークフローを再実行することなく得られる、単一の引き継ぎ文書を作成する。

## Classification

ワークフローの状態に対しては読み取り専用である。このスキルはステージの
ポインタを決して進めず、監査イベントも決して出力しない。ただし、レポートの
成果物を 1 つ（ワークスペースルートの `OUTCOMES.md`）**書き込む**。それが
このスキルの出力である。それ以外には何も書き込まない。

## The counting rule

ステージの集計、フェーズごとの集約、メモリーエントリの件数、記録された学びの件数は、
成果物ツリーを目で数えるのではなく、ツールから取得する:

```bash
{{INVOKE}} engine runtime summary --json
```

セクションの*内容*（何を作ったか、セットアップ手順、意思決定）は、成果物と
納品されたコードからあなたがまとめる。パックに現れる*件数*はすべて、
ツールの出力にたどれなければならない。

## Steps

### Step 1: Read the aggregates

`{{INVOKE}} engine runtime summary --json` を実行する。

0 以外で終了した場合（`runtime-graph.json` がまだない）は、次を出力する:

```
No session data yet — an outcomes pack is generated at the close of a
workflow. Run /aidlc to completion first.
```

そして停止する。そうでなければ、解析した JSON を以下の件数フィールドのために保持しておく。

### Step 2: Read the content sources

`<active-space>` を `aidlc/active-space`（既定値は `default`）から、
`<active-intent>` を
`aidlc/spaces/<active-space>/intents/active-intent` から解決する。アクティブなワークフローの
記録は
`aidlc/spaces/<active-space>/intents/<active-intent>` である。

- `<record>/<phase>/` 配下を再帰的にたどったすべての成果物 — 要件、
  設計上の意思決定、NFR、インフラストラクチャ、ユニットごとの出力。
- ワークスペースルートにある、納品されたアプリケーションとインフラストラクチャの
  コード。

### Step 3: Write OUTCOMES.md

ワークスペースルート（`<record>/` の外）に `OUTCOMES.md` を書き込む:

```markdown
# Outcomes Pack
**Scope**: {summary.scope}
**Stages delivered**: {summary.stages.approved} approved / {summary.stages.total} total
**Duration**: {summary.duration_minutes} min

## 1. What Was Built
- Project name and description (from requirements)
- Scope the workflow ran at
- Units of work delivered and what each contains
- Key architectural decisions and why (from design artefacts)
- Tech stack with version pins

## 2. Repository Structure
- Annotated directory tree of the delivered code
- What lives where and why

## 3. Setup Guide
- Prerequisites (runtimes, tools, cloud CLI versions)
- Local development setup, step by step
- Required environment variables
- How to run tests

## 4. Build and Deploy
- Build steps
- Full test-suite run
- Infrastructure deployment (from Build and Test artefacts)
- IaC deployment commands with expected outputs, if generated

## 5. Architecture Decisions
- Every significant decision made during the workflow
- Alternatives considered and why rejected
- Constraints that shaped the design (from rules and practices)

## 6. What to Commit vs Archive
| Artifact | Action | Destination |
|----------|--------|-------------|
| `decisions.md` (per stage) | Commit | `docs/decisions/` |
| Architecture summary (1 page) | Write + commit | `docs/architecture.md` |
| NFR summary table | Write + commit | `docs/nfr-summary.md` |
| `<record>/audit/*.md` shards | Archive — do NOT commit to app repo | Compliance archive |
| Stage question files | Discard | — |
| `<record>/aidlc-state.md` | Discard | — |
| Application / infrastructure code | Already committed | — |

## 7. Workflow Footprint
- Stages: {summary.stages.approved} approved, {summary.stages.failed} failed, {summary.stages.pending} pending
- Memory entries captured: {summary.memory.total}
  ({summary.memory.interpretations} interpretations, {summary.memory.deviations} deviations, {summary.memory.tradeoffs} trade-offs, {summary.memory.open_questions} open questions)
- Learnings captured: {summary.learnings.from_orchestrator} from orchestrator, {summary.learnings.from_user_addition} from user additions

## 8. Known Limitations and What to Tackle Next
- Scope items explicitly deferred during the workflow
  (cross-reference the {summary.memory.open_questions} open questions above)
- Technical debt identified but not resolved
- Recommended next steps
```

### Step 4: Confirm

短いサマリーを出力する: 書き込んだセクション、元資料がないためにスキップしたセクション、
推奨する追記。"want to add or adjust a section?"（セクションの追加や調整をしますか？）と
1 回だけ尋ねてもよい — それを繰り返してはならない。
