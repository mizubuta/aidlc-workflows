# Writing a Technical Environment Document

技術環境ドキュメントは、**プロジェクトをどう構築するか**を示します。使用しなければならない
言語、フレームワーク、クラウドサービス、セキュリティルール、テスト、そして使用してはならないものです。
これは [ビジョンドキュメント](vision-document-guide.md) の技術面における対となる文書であり、
Construction のステージが作業の拠り所とするリファレンスです。

これがないと、設計や構築のステージは選択のたびにあなたに質問するか、
後で変更が必要になるデフォルトを選んでしまいます。許可リストと禁止リストに
それぞれ理由を添えておけば、エージェントが推測で動くことを防げます。

---

## Where it goes in AI-DLC

ビジョンドキュメントは最初のリクエストに含めます。技術環境ドキュメントは、
すべてのステージが読み込む場所、つまりスペースのチームナレッジに置きます。

1. 次の場所に保存します:
   `aidlc/spaces/<space>/knowledge/aidlc-shared/technical-environment.md`
   （他のスペースを作成していなければ、スペース名は `default` です）。`aidlc-shared/` 内の
   すべての Markdown ファイルは、すべてのステージのコンテキストに含まれます。
   [ナレッジ](../08-knowledge.md) の章では、このフォルダーを
   `aidlc/knowledge/aidlc-shared/` と呼んでいます。
2. すべてを `aidlc-shared/` に置いてください。セキュリティ、テスト、クラウドの標準は、
   異なるエージェントがリードするステージで読み込まれます（たとえば Code Generation は
   developer エージェントとして実行されます）。また、エージェント専用のフォルダーは、そのエージェントが
   作業している間しか読み込まれません。エージェント専用のフォルダーは、1 つのエージェントだけが
   必要とする資料にのみ使ってください。フォルダー名については
   [会社の標準を追加する](../08-knowledge.md#adding-company-standards)
   を参照してください。
3. 決して破ってはならない少数のルール（禁止ライブラリ、必須バージョンなど）は、
   `aidlc/spaces/<space>/memory/project.md` にも記載してください。チームが手がけるすべての
   プロジェクトに適用される場合は `team.md` に記載します。メモリファイルは
   すべてのステージが適用するルールです。
   [ルールと学習ループ](../09-rules-and-the-learning-loop.md) を参照してください。

これは、その作業で最初に `/aidlc` を実行する前に行ってください。計画に
Practices Discovery（Inception の序盤）が含まれる場合、チームの働き方について質問されます。新規
プロジェクトでは、このドキュメントにすでに記載されている回答をすばやく確認するだけで済みます。

---

## The short version

```text
1. Language and version
2. Package manager
3. Web framework (if any)
4. Cloud provider and deployment model, or "local only"
5. Test framework
6. Prohibited libraries and services, as a table: prohibited | reason | use instead
7. Security basics: how users sign in, how input is validated, where secrets live
8. Example code: one short example each of a typical endpoint, function and test
```

項目 6 では、理由と代替手段が重要です。それがないと、エージェントは禁止には
従いますが、適切な代替手段を判断できません。

項目 8 では、短い例が 1〜2 個あるだけでも、Code Generation はパターンを発明する代わりに
それに従うことができます。基本項目以外で追加するなら、これが最も役に立ちます。

### For work on an existing codebase

```text
1. Existing stack: language, framework, database, infrastructure, with versions
2. What never changes: services, schemas, contracts, configuration no piece of work may touch
3. Prohibited patterns: libraries or approaches that conflict with the existing code
4. Security basics: how sign-in and secrets work in the existing system
5. Example code taken from the existing codebase
```

例は実際のファイルから取ってください。そうすれば、新しいコードが既存のコードになじみます。
Reverse Engineering もコードを読み込みますが、どのパターンを残したいのか、
どのパターンから移行しようとしているのかまでは判断できません。

この作業で追加・変更する内容や、この作業だけが手を付けてはならないものは、
[ビジョンノート](vision-document-guide.md#for-work-on-an-existing-codebase) に記載します。
このドキュメントはスペースに残り、以降のすべての作業がこれを読み込みます。

プロジェクトが新規か既存かは、このドキュメントではなくプロジェクトフォルダー内のコードから
判断されます。既存のプロジェクトを AI-DLC が新規と判断した場合は、リクエストでその旨を伝えるか、
`/aidlc --project-type brownfield` で開始してください。

---

## The full structure

各セクションには対象となるプロジェクトの種類を示しています。**(New)** は新規プロジェクト、
**(Existing)** は既存のコードベースに対する作業、**(Both)** は両方です。

### 1. Project Technical Summary (Both)

```markdown
## Project Technical Summary

- **Project name**: [Name]
- **Project type**: [New / Existing codebase]
- **Runtime environment**: [Cloud / On-premises / Hybrid]
- **Cloud provider**: [AWS / Azure / GCP / Multi-cloud / None]
- **Deployment model**: [Serverless / Containers / VMs / Hybrid]
- **Team size and key skills**: [Number, experience relevant to the choices]
```

### 2. Programming Languages (Both)

```markdown
## Programming Languages

### Required
| Language | Version | Purpose | Reason |
|----------|---------|---------|--------|
| TypeScript | 5.x | Backend services, infrastructure code | Team expertise, type safety |

### Permitted with justification
| Language | When it may be used |
|----------|---------------------|
| Go | High-throughput services where latency is critical |

### Prohibited
| Language | Reason |
|----------|--------|
| PHP | No team expertise, not the platform direction |
```

既存のコードベースの場合は、すでに使用している言語と、それぞれの方針
（維持、アップグレード、移行）を追加してください。

### 3. Frameworks and Libraries (Both)

```markdown
## Frameworks and Libraries

### Required
| Framework or library | Version | Area | Reason |
|----------------------|---------|------|--------|
| React | 18.x | Frontend UI | Organisational standard |
| Jest | 29.x | Unit tests | One test runner across projects |

### Preferred when the need arises
| Library | Purpose | Use when |
|---------|---------|----------|
| Zod | Runtime validation | Any external input or API payload |
| Pino | Structured logging | Every service that logs |

### Prohibited
| Library | Reason | Use instead |
|---------|--------|-------------|
| Moment.js | Deprecated, large | date-fns or Luxon |
| request | Deprecated | Native fetch |

### Getting a new library approved
[Who approves a library that is not listed, and what they need to see.]
```

### 4. Cloud Environment and Services (Both)

```markdown
## Cloud Environment

- **Provider**: [AWS / Azure / GCP]
- **Account structure**: [Single account / Multi-account]
- **Regions**: [Primary, and disaster recovery if any]

### Allowed services
| Service | Approved use | Constraints |
|---------|--------------|-------------|
| AWS Lambda | Event-driven compute, API handlers | 15 min timeout |
| Amazon DynamoDB | Key-value and document storage | On-demand for dev |
| Amazon S3 | Objects, static assets | Versioning and encryption on |

### Disallowed services
| Service | Reason | Use instead |
|---------|--------|-------------|
| Amazon EC2 (direct) | Prefer managed compute | Lambda or ECS Fargate |

### Getting a new service approved
[Who approves a service that is not listed, and what they need to see.]
```

### 5. Architecture and Patterns (Both)

```markdown
## Architecture and Patterns

| Pattern | When to use | When not to use |
|---------|-------------|-----------------|
| Serverless-first | Default for new services | Long-running or connection-heavy work |
| Modular monolith | Single-team projects, first releases | Independently scaled domains |

### API standards
- **Style**: [REST / GraphQL / gRPC]
- **Versioning**: [URL path / header]
- **Documentation**: [OpenAPI 3.x for every REST API]
- **Naming**: [kebab-case URLs, camelCase JSON fields]
- **Errors**: [The standard error response shape]

### Data
- **Primary store**: [e.g. DynamoDB for service-owned data]
- **Relational data**: [e.g. PostgreSQL when queries need joins]
- **Ownership**: [e.g. each service owns its data; no shared databases]

### Messaging
- **Synchronous**: [e.g. HTTP between services]
- **Asynchronous**: [e.g. a queue for tasks, an event bus for events]

### Frontend (if any)
- **Components, state, routing, build tool**: [Your choices]
```

### 6. Security (Both)

```markdown
## Security

### Sign-in and access
- **Authentication**: [e.g. Amazon Cognito, OIDC, SAML]
- **Authorisation model**: [e.g. role-based]
- **Tokens and sessions**: [Format, expiry, refresh]

### Data protection
- **At rest**: [Encryption required, key management]
- **In transit**: [TLS 1.2 or later]
- **Personal data**: [Fields, masking, retention]

### Secrets
- **Storage**: [e.g. AWS Secrets Manager]
- **Rotation**: [Every N days]
- **Never**: secrets in source code, in config files, or shared between services

### Compliance
- **Standards**: [SOC 2, HIPAA, PCI-DSS, GDPR, or "none specific"]
- **Audit logging**: [What is logged, and for how long]
- **Scanning**: [Dependency and image scanning tools]
- **Licences**: [Allowed: MIT, Apache 2.0, BSD. Prohibited: GPL, AGPL]
```

#### Choose a security framework

フレームワークを 1 つ選び、そのカテゴリごとに、プロジェクトがどう対処するかを記載してください。
該当しないカテゴリにはその理由を、後で導入予定の統制にはそのギャップと
対応するフェーズを明記します。

| プロジェクトの種類 | よく選ばれるもの |
|-----------------|----------------|
| Web アプリケーションと API | OWASP Top 10、OWASP API Security Top 10 |
| クラウドインフラ | クラウドプロバイダーの Well-Architected セキュリティの柱、CIS Benchmarks |
| 政府機関または規制対象 | NIST 800-53、FedRAMP、ISO 27001 |
| 一般的なソフトウェア | CIS Controls v8、SANS Top 25 |

カテゴリが 10 個以下のフレームワークであれば、表全体をこのドキュメントに記載します。
大規模なフレームワークの場合は、このドキュメントの隣に専用のファイルを置き
（例: `aidlc-shared/nist-800-53-compliance.md`）、ここからリンクしてください。

### 7. Testing (Both)

```markdown
## Testing

| Test type | Required | Target | Tooling |
|-----------|----------|--------|---------|
| Unit | Yes | 80% line coverage | Jest / pytest |
| Integration | Yes | Every service-to-service call | Testcontainers |
| End-to-end | For key journeys | Critical user journeys | Playwright |
| Performance | When there are targets | The stated response times | k6 |
| Security | Yes | Every public endpoint | OWASP ZAP |

- **Mocking**: [Mock external dependencies, never internal business logic]
- **Test location**: [Next to the source, or a separate tests/ tree]
- **Naming**: [e.g. describe / it]

### Pipeline gates
| Pipeline step | Tests that must pass | On failure |
|---------------|----------------------|------------|
| Pull request | Unit and integration | Block the merge |
| Before staging deploy | End-to-end | Block the deploy |
```

ワークフローが書くテストの量は、その作業に対して選んだテスト戦略にも従います。
[スコープ、深さ、テスト戦略](../05-scopes-and-depth.md) を参照してください。

### 8. Example Code (Both)

サンプルコードは、よくある処理の標準的なやり方を示します。これにより、生成されるコードは
新しいスタイルを発明するのではなく、それに従います。

プロジェクトにあるものについて、次の例を用意してください: プロジェクトのレイアウト、
API エンドポイント、データベースアクセス、エラー処理、認可の適用、
ユニットテストと統合テスト、ログ出力、設定の読み込み、インフラモジュール。

例はリポジトリ内に置きます。たとえば次のようにします:

```text
project-root/
  examples/
    api-endpoint/
      handler.ts          # working code, not pseudocode
      handler.test.ts     # how to test it
      README.md           # what it shows, when to use it, what to change
    database-access/
      repository.ts
      repository.test.ts
      README.md
```

各例は実行可能で、テストが付属し、短い README を備えているべきです。README には、
どのパターンを示しているか、いつ使い、いつ使わないか、どの部分をカスタマイズし、
どの部分をそのまま維持すべきかを記載します。

examples フォルダーは単独では読み込まれません。このドキュメントの中で、どの例が何に
適用されるかを記載してください。たとえば「すべての API エンドポイントは
`examples/api-endpoint/` に従う」と書いておけば、Code Generation はそのステップを計画するときに
それを開きます。標準が変わったら例も最新の状態に保ち、置き換えられた例は
`deprecated-` 接頭辞を付けて名前を変え、後継を指す注記を添えてください。

### 9. Existing Codebase Only

```markdown
## Existing Technical Inventory

- **Languages and frameworks, with versions**: [List]
- **Infrastructure and deployment**: [Services, model]
- **Test coverage today**: [Figure or a short assessment]
- **Known technical debt**: [Key items]

### Keep
| Technology | Reason to keep |
|------------|----------------|

### Migrate
| Today | Target | Priority | Approach |
|-------|--------|----------|----------|
| JavaScript | TypeScript | High | File by file |

### Remove
| Item | Reason | When |
|------|--------|------|

### While old and new coexist
- **API versions**: [How v1 and v2 run side by side]
- **Schema changes**: [How migrations run alongside existing data]
- **Feature flags**: [How new behaviour is switched on]
```

---

## Where each section is used

| セクション | ステージ | 使われ方 |
|---------|-------|----------------|
| Project Technical Summary | Practices Discovery | チームのプラクティスとスキル |
| Languages, Frameworks and Libraries | NFR Design, Code Generation | 何をどのバージョンで使うか、そして何を決して追加しないか |
| Cloud Services | Feasibility & Constraints, Infrastructure Design | 設計で使用できるサービス |
| Architecture and Patterns | Domain Design, Contract Design, Functional Design | 構造、API、データに関する意思決定 |
| Security | NFR Requirements, NFR Design, Build and Test | セキュリティ統制とそのチェック |
| Testing | Code Generation, Build and Test, CI Pipeline | テストの種類、カバレッジ、ゲート |
| Example Code | Code Generation | 各コードが従うパターン |
| Existing Technical Inventory | Reverse Engineering, Practices Discovery | 維持、移行、削除するもの |

どのステージが実行されるかは、承認した計画によって決まります。
[フェーズとステージ](../04-phases-and-stages.md) を参照してください。
