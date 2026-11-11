# Architecture Decision Records (ADRs)

## Purpose
ADR は、重要なアーキテクチャ上の決定、その決定を促した背景、そして代替案ではなくある選択肢を選んだことによる結果を記録する。ADR は、アーキテクチャが今の形になっている「なぜ（WHY）」を説明する決定の記録簿となる。

## When to Create an ADR

次のような決定の場合に ADR を作成する:
- 一度実装すると元に戻すのが難しい（データベースの選定、API コントラクト、フレームワーク）
- 複数のチームやコンポーネントに影響する
- コスト、パフォーマンス、セキュリティに大きな影響がある
- 議論があった — 意見の相違があった場合、その理由を文書化する必要がある
- 以前のアーキテクチャの方向性を変更する

次の場合は ADR を作成**しない**:
- 日常的な実装上の選択（変数名、コードのフォーマット）
- 容易に元に戻せる決定
- チームのガイドラインにすでに文書化されている標準的なプラクティス

## ADR Template

```markdown
# ADR-[NUMBER]: [Title - Short Descriptive Name]

## Status
[Proposed | Accepted | Deprecated | Superseded by ADR-XXX]

## Date
[YYYY-MM-DD]

## Context
What is the issue or situation that motivates this decision? Describe the forces
at play: technical constraints, business requirements, team capabilities, timeline
pressures, and any other factors influencing the decision.

Be specific. Include:
- What problem are we solving?
- What constraints exist (budget, timeline, team skill, compliance)?
- What quality attributes matter most (performance, security, maintainability)?
- What existing decisions or systems does this interact with?

## Decision
State the decision clearly and concisely. Use active voice:
"We will use PostgreSQL as the primary data store for the order management service."

Not: "It was decided that PostgreSQL might be a good option."

## Consequences

### Positive
- What becomes easier, faster, or better as a result of this decision?
- What risks are mitigated?

### Negative
- What becomes harder or more expensive?
- What new risks are introduced?
- What capabilities are foreclosed?

### Neutral
- What trade-offs are we accepting?
- What follow-up decisions will be needed?

## Alternatives Considered

### Alternative 1: [Name]
- Description: Brief explanation of this option
- Pros: What would this have given us?
- Cons: Why did we reject it?

### Alternative 2: [Name]
- Description: Brief explanation of this option
- Pros: What would this have given us?
- Cons: Why did we reject it?

## References
- Links to relevant RFCs, design documents, benchmark results, or discussions
```

## Numbering Convention

ゼロ埋めした連番を使う:
```
ADR-001-use-postgresql-for-orders.md
ADR-002-adopt-event-driven-integration.md
ADR-003-select-react-for-frontend.md
```

### Naming Rules
- 連番は決して再利用しない（非推奨になった場合でも）
- 番号の後ろに、内容を表すケバブケースのサフィックスを付ける
- 専用の `/docs/adr/` または `/architecture/decisions/` ディレクトリに保存する
- すべての ADR をステータスと1行の概要付きで一覧にした `index.md` を用意する

## ADR Lifecycle

### Proposed
決定は議論中である。ADR はレビューとフィードバックを受け付けるドラフトである。プルリクエストやアーキテクチャレビュー会議に含める。

### Accepted
決定は承認済みであり、従うべきものである。実装を進めてよい。承認日と承認者を記録する。

### Deprecated
決定はもはや意味を持たない — 対象のシステムや機能は削除されている。歴史的な経緯のために ADR は残すが、そのことを明確に示す。

### Superseded
新しい決定がこの決定を置き換えた。新しい ADR へリンクする:
```
## Status
Superseded by [ADR-015](./ADR-015-migrate-to-dynamodb.md)
```
置き換える側の ADR は、元の ADR を参照する:
```
## Context
This decision supersedes [ADR-003](./ADR-003-select-postgresql.md) because...
```

## Best Practices

### Writing Quality
- 背景を何も知らない将来の読み手に向けて書く — 決定後にチームに加わった人を想定する
- 「何を（WHAT）」だけでなく「なぜ（WHY）」に焦点を当てる — 何が作られたかはコードが示し、なぜかは ADR が説明する
- トレードオフについて正直に書く — どの決定にも欠点があり、それを文書化することで信頼が生まれる
- 利用できる場合は定量的なデータ（ベンチマーク結果、コスト見積もり、負荷テストの数値）を含める

### Process
- ADR は実装の後ではなく、実装の**前**に作成する — ADR はドキュメントではなく、意思決定のためのツールである
- ADR は、それが影響するコードと一緒にプルリクエストでレビューする
- ADR を四半期ごとに見直す — 状況の変化に応じて更新が必要なものもある
- ADR は簡潔に保つ — 1〜2ページが理想であり、それより長い場合はスコープが広すぎることを示唆する

### Common Mistakes
- 事後にドキュメントとして ADR を書く（ADR は決定を導くものであるべき）
- 代替案を省略する（決定が十分に検討されていないように見える）
- 負の結果を記録しない（誤った安心感を生む）
- ADR を細かくしすぎる（実装の詳細に ADR は不要）
- ステータスを更新せずに ADR を放置して古くする

## Example ADR Index

| ADR | タイトル | ステータス | 日付 |
|-----|-------|--------|------|
| 001 | 注文管理に PostgreSQL を使用する | Accepted | 2024-01-15 |
| 002 | サービス間でイベント駆動型の連携を採用する | Accepted | 2024-02-01 |
| 003 | フロントエンドに React と TypeScript を選定する | Accepted | 2024-02-10 |
| 004 | API 認証に JWT を使用する | Superseded by 012 | 2024-03-01 |
| 005 | AWS ECS Fargate にデプロイする | Accepted | 2024-03-15 |
