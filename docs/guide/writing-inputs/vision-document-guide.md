# Writing a Vision Document

ビジョンドキュメントは、**何を、誰のために、なぜ構築するのか**、そして
最初のリリースがどこまでかを示すものです。新しいワークフローへの主要な入力です。Intent Capture、
Scope Definition、Requirements Analysis はこれをもとに計画を立てるため、各ステージが
尋ねる質問は、あなたがすでに書いた内容の簡単な確認になります。

開始するのにビジョンドキュメントは必須ではありません。`/aidlc Build a REST API for inventory
management` だけでも完全なリクエストです。ビジョンドキュメントが効果を発揮するのは、複数の
人がまず目標について合意する必要がある場合、スコープが数文に収まらないほど大きい場合、
あるいはワークショップを運営していて、最初の 1 時間を質問ではなく
意思決定に使いたい場合です。

技術面の対になるドキュメント（技術スタック、ツール、禁止事項）は、
[技術環境ドキュメント](technical-environment-guide.md) です。

---

## Giving it to AI-DLC

ドキュメントをプロジェクト内に保存し（Markdown またはプレーンテキストが最適です）、
最初のリクエストでその名前を指定します:

```text
/aidlc Read ./vision.md and build what it describes
```

Codex CLI では `/aidlc` の代わりに `$aidlc` を使います。相対パスは
プロジェクトルートから解決されます。内容をリクエストに貼り付けて
`<document>` と `</document>` で囲むことも、PDF や Word ファイルを指定することもできます。それぞれの読み込み方法と、名前が複数の
ファイルに一致した場合の動作については、
[既存のドキュメントから始める](../02-your-first-workflow.md#starting-from-an-existing-document)
を参照してください。

ワークフローは、ドキュメントを指示ではなくデータとして扱います。ドキュメントをもとに計画を立てますが、
すべての承認を与えるのは引き続きあなたです。Intent Capture はいずれにしても質問を行います。
ドキュメントですでに回答されている質問はすぐに確認でき、ドキュメント内の
未解決の質問は早い段階で取り上げられます。短いビジョンドキュメントを使った Claude Code での実行で、
唯一の未解決の質問が倉庫ごとの在庫数に関するものだったとき、Intent Capture はそれについて
独自の質問を追加しました（短縮し、罫線文字はプレーンテキストで表示し、
ラベルが入る位置を示すために質問文の前に `[Product owner]` を加えています）:

```text
 Warehouses
 Q9. [Product owner] Do the internal tools need stock counts per warehouse, or one total per SKU?
 The vision doc leaves this open, and it changes what a "count" means for every SKU.
 > 1. One total per SKU
   2. Per warehouse per SKU
   3. One total now; per-warehouse later
   4. Not yet defined
```

---

## The short version

ほかに何も書かないとしても、次の 4 つは書いてください:

```text
1. One paragraph saying what you are building and for whom
2. The features in the first release (what is IN scope)
3. What is NOT in the first release
4. Open questions: things you already know are uncertain or undecided
```

未解決の質問は任意ですが、価値があります。Intent Capture と Requirements
Analysis はそれらを早い段階で質問にします。そうしなければ、設計中に
抜けが想定外の問題として表面化することになります。

### For work on an existing codebase

```text
1. Current state: one paragraph on what the system does today
2. What we are adding or changing
3. Features IN scope for this piece of work
4. Features OUT of scope for this piece of work
5. What must NOT change: components, APIs or data the new work must not touch
6. Open questions
```

最も重要なのは「変更してはならないもの」です。Reverse Engineering は既存の
コードを読みますが、どの部分に手を触れてはならないかを知っているのはあなただけです。それを明記することで、計画が
それらに近づかないようになります。

---

## The full structure

### 1. Executive Summary

3〜5 文で書きます。このセクションだけを読んだ人でも、
プロジェクトが何であり、誰のためのもので、なぜ存在するのかがわかるようにします。

```markdown
## Executive Summary

[Project Name] is a [type of system or product] that lets [target users]
[core capability]. It addresses [business problem or opportunity] by
[approach or difference]. The expected outcome is [measurable result].
```

例:

```markdown
## Executive Summary

OrderFlow is a web-based order management platform that lets mid-size
retailers track inventory, process customer orders, and manage suppliers in
one place. It addresses the scattered tooling that causes fulfilment delays
and stock mismatches. The expected outcome is a 30% cut in order processing
time and no more manual stock reconciliation.
```

### 2. Business Context

```markdown
## Business Context

### Problem Statement
[The specific business problem. Be concrete: "orders wait two days for a
stock check" says more than "improve efficiency".]

### Business Drivers
[Why now: market pressure, a regulation, a contract, an internal need.]

### Target Users and Stakeholders
| User Type | Description | Primary Need |
|-----------|-------------|--------------|
| [Role] | [Who they are] | [What they need from the system] |

### Business Constraints
[Budget, regulation, organisational policy, deadlines, anything that is not
negotiable.]

### Success Metrics
| Metric | Today | Target | How it is measured |
|--------|-------|--------|--------------------|
| [Metric] | [Baseline] | [Goal] | [Method] |
```

### 3. Full Scope Vision

製品が成熟したときになりうるすべての姿です。この部分は意図的に
理想を描くものです。構築を約束することなく、最初のリリースがどこへ向かっているのかを
ワークフローが把握する助けになります。

```markdown
## Full Scope Vision

### Product Vision Statement
[One sentence or a short paragraph: what the world looks like when the
product is fully realised.]

### Feature Areas
#### Feature Area 1: [Name]
- **Description**: [What this area covers]
- **Key capabilities**: [Capability 1], [Capability 2], [Capability 3]
- **User value**: [Why it matters to users]

### Integration Points
- [System or service]: [purpose of the integration]

### User Journeys
#### Journey 1: [Name]
1. [Step]
2. [Step]
**Outcome**: [What the user achieves]

### Growth and Roadmap (optional)
| Phase | Focus | Timeframe (if known) |
|-------|-------|----------------------|
| First release | [Core scope] | [Target] |
| Phase 2 | [Expansion] | [Target] |
```

### 4. First Release (MVP) Scope

測定可能な価値を提供する、最小限の機能のセットです。ここに記載されていない機能は、
最初のリリースには含まれません。

```markdown
## MVP Scope

### Objective
[The one thing the first release must prove or deliver, in one or two
sentences.]

### Success Criteria
- [ ] [Testable criterion]
- [ ] [Testable criterion]

### Features In Scope
| Feature | Description | Why it cannot wait |
|---------|-------------|--------------------|
| [Feature] | [Short description] | [Reason] |

### Features Out of Scope
| Feature | Why it can wait | Target phase |
|---------|-----------------|--------------|
| [Feature] | [Reason] | [Phase 2, 3 or later] |

### Journeys the First Release Supports
#### Journey 1: [Name]
1. [Step]
2. [Step]
**Outcome**: [What the user achieves]
**Simplified compared with the full vision**: [What is missing or reduced]

### Assumptions and Accepted Limits
- **Assumption**: [Statement]. **Risk if wrong**: [Consequence]
- **Accepted limit**: [What is limited on purpose, and why]

### Definition of Done
- [ ] Every in-scope feature built and tested
- [ ] [Project-specific criterion]
- [ ] [Sign-off needed]
```

### 5. Risks, Dependencies and Open Questions

```markdown
## Risks and Dependencies

### Key Risks
| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| [Risk] | High / Medium / Low | High / Medium / Low | [Plan] |

### External Dependencies
- [Dependency]: [owner], [status]

### Open Questions
- [ ] [Question]
- [ ] [Question]
```

---

## Writing guidelines

すべきこと:

- 具体的かつ測定可能にする。「注文処理時間を 30% 短縮する」は「もっと
  速くする」より優れています。
- 完全なビジョンと最初のリリースを分けておく。両者を混ぜることが、スコープが
  膨らむ原因です。
- スコープ外のリストを書く。スコープ内のリストと同じだけの価値があります。
- 営業資料のためではなく、チームのために書く。
- 異議を唱えられるよう、前提を明記する。
- 実際にテストできる成功基準を示す。

すべきでないこと:

- 「世界クラスの」「シームレスな」「直感的な」といった言葉を、裏付けとなる数値なしに
  使う。
- 技術や実装の詳細を列挙する。それらは
  [技術環境ドキュメント](technical-environment-guide.md) に書くべきものです。
- 最初のリリースのセクションを省略する。どのプロジェクトにも開始時の境界が必要です。
- 読み手がビジネスを知っていると想定する。自明に思えるときでも、
  問題記述は書いてください。

---

## Where each section is used

| セクション | ステージ | 使われ方 |
|---------|-------|----------------|
| Executive Summary | Intent Capture & Framing | 問題、ユーザー、期待される成果 |
| Business Context | Intent Capture & Framing, Requirements Analysis | 尋ねる質問と、要件をどこまで深く掘り下げるか |
| Full Scope Vision | Scope Definition, User Stories, Domain Design | 製品の向かう先、ペルソナとコンポーネント |
| First Release Scope | Scope Definition, Requirements Analysis | 今回の作業の境界 |
| Features In / Out | Requirements Analysis, Units Generation, Code Generation | 今回の作業で何を構築するか |
| Risks and Dependencies | Feasibility & Constraints, NFR Requirements | リスクと制約の評価 |
| Open Questions | Intent Capture & Framing, Requirements Analysis | 明確化のための質問として早い段階で尋ねられる |

これらのステージのうちどれが実行されるかは、あなたが承認する計画によって決まります。Express や
Bugfix の計画では、そのうちのごく一部しか実行されません。すべてのステージについては
[フェーズとステージ](../04-phases-and-stages.md) を、計画については
[ワークフロープロファイル](../workflow-profiles.md) を参照してください。
