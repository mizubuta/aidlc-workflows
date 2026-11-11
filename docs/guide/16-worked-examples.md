# Worked Examples

AI-DLC の動作を示す 2 つの完全なウォークスルーです: バグ修正と機能開発です。それぞれ、コマンドの呼び出し、ステージの進行、承認ゲート、成果物の出力を示します。

> **ハーネスに関する注記。** これらの記録は **Claude Code** で取得したものなので、
> その画面要素 — `/aidlc` や、`Task` 呼び出しでディスパッチされるサブエージェントのステージ — が表示されています。
> ステージの流れ、ゲート、成果物はどのハーネスでも同じで、違うのは
> ディスパッチの仕組みだけです（Kiro は `subagent` ツール、Codex は `spawn_agent`
> サブエージェントを使います）。[他のハーネスでの実行](harnesses/README.md) を参照してください。

---

## Bugfix Walkthrough

この例では、ユーザープロフィール API の null ポインター例外を修正します。**bugfix** スコープは、Minimal の深さで 9 つのステージ（Initialization 3 つ + ドメイン 6 つ）を実行します。

### Invocation

```
/aidlc bugfix
```

コンダクターが、何を修正したいかを尋ねます:

> **What would you like to build?**（何を作りたいですか？）

あなたはこう答えます:

> `display_name` フィールドが null のとき、ユーザープロフィール API が HTTP 500 を返します。`GET /api/v1/users/:id/profile` エンドポイントが `ProfileSerializer.serialize()` で NullPointerException を起こしてクラッシュします。display_name が必須化される前に作成されたユーザープロフィールの約 12% が影響を受けています。

### Stages executed

| # | ステージ | フェーズ | リードエージェント | モード |
|---|-------|-------|------------|------|
| 0.1 | Workspace Scaffold | Initialization | orchestrator | inline (auto-proceed) |
| 0.2 | Workspace Detection | Initialization | orchestrator | inline (auto-proceed) |
| 0.3 | State Init | Initialization | orchestrator | inline (auto-proceed) |
| 2.1 | Reverse Engineering | Inception | aidlc-developer-agent + aidlc-architect-agent | pipeline |
| 2.3 | Requirements Analysis | Inception | aidlc-product-agent | inline |
| 3.5 | Code Generation | Construction | aidlc-developer-agent | subagent |
| 3.6 | Build and Test | Construction | aidlc-quality-agent | inline |
| 4.1 | Deployment Pipeline | Operation | aidlc-pipeline-deploy-agent | inline |
| 4.3 | Deployment Execution | Operation | aidlc-pipeline-deploy-agent + aidlc-developer-agent | inline |

### Initialization (stages 0.1-0.3) — auto-proceed

3 つの Initialization ステージは、1 回の決定的なツール呼び出し（`aidlc-utility intent-create`）として、ユーザーとの対話なしに 1 秒もかからずに実行されます:

- **0.1 Workspace Scaffold** - 最初のインテントを自動作成し、その記録ディレクトリを `aidlc/spaces/<space>/intents/<YYMMDD>-<label>/`（以下 `<record>/` と表記）に作成します - `<YYMMDD>` は記録が時系列順に並ぶようにするための簡潔な UTC 日付のプレフィックスで、`<label>` はコンダクターがリクエストの要点を短いケバブケースにしたものです。正式な ID は、`intents.json` レジストリの行に保持される UUIDv7 です
- **0.2 Workspace Detection** — ルールベースのスキャンで、Java 17、Spring Boot 3.2、Maven、ブラウンフィールドのプロジェクトであることを特定します
- **0.3 State Init** — スコープ `bugfix`、深さ `Minimal`、実行対象としてマークしたドメインステージで `aidlc-state.md` を初期化します

> Setting up the bugfix workflow for this: 6 stages, 6 approval gates.（このための bugfix ワークフローを設定しています: 6 ステージ、6 承認ゲート。）

### Stage 2.1 — Reverse Engineering

2 段のパイプラインがコードベースをスキャンします。まず aidlc-developer-agent がコードをスキャンし、次に aidlc-architect-agent が統合して成果物を書き出します。リポジトリについて 9 つの永続的な成果物を `aidlc/spaces/default/codekb/user-service/` に作成します:

| 成果物 | 内容 |
|----------|----------|
| `business-overview.md` | ユーザーサービス — プロフィール、設定、認証トークン |
| `architecture.md` | Spring Boot のモノリス、3 層設計 |
| `code-structure.md` | 6 つのパッケージ: controller、service、model、repository、serializer、config |
| `api-documentation.md` | `/api/v1/users/` 配下の 8 つの REST エンドポイント |
| `component-inventory.md` | コントローラー、サービス、リポジトリ、シリアライザーの一覧 |
| `technology-stack.md` | Java 17、Spring Boot 3.2、PostgreSQL 15、Jackson 2.15 |
| `dependencies.md` | Maven の依存関係ツリー、サードパーティライブラリ、バージョン制約 |
| `code-quality-assessment.md` | テストカバレッジ 62%、基本的な CI |
| `reverse-engineering-timestamp.md` | スキャンをいつ、どのコミットに対して実行したか |

**承認ゲート:**

```
Reverse Engineering complete. How would you like to proceed?
- Approve        -> Continue to Requirements Analysis
- Request Changes -> Provide revision feedback
```

**Approve** を選びます。

### Stage 2.3 — Requirements Analysis

aidlc-product-agent のペルソナが読み込まれ、`<record>/inception/requirements-analysis/requirements-analysis-questions.md` に確認の質問を作成します:

```markdown
## Q1. [Product owner] Bug Severity Classification
How severe is this bug for your users?
A. Critical — causes data loss or security exposure
B. High — blocks a core workflow for affected users
C. Medium — degraded experience but workaround exists
D. Low — cosmetic or minor inconvenience
X. Other (please specify)

[Answer]:
```

コンダクターが対話モードを提示します:

```
How would you like to answer these questions?
- Guide me        -> Walk through each question interactively
- I'll edit the file -> Fill in answers directly
- Chat            -> Discuss freely
```

**Guide me** を選び、次のように答えます: Q1 = High、Q2 = フォールバックとしてユーザー名を使う、Q3 = null を適切に処理する（マイグレーションなし）。

コンダクターは、3 つの機能要件（null の処理、シリアライザーの修正、フォールバックのロジック）と 1 つの非機能要件（応答時間の劣化なし）を含む `requirements.md` を生成します。

**承認ゲート:** **Approve** を選びます。

### Stage 3.5 — Code Generation

コンダクターはコード生成の計画を作成し、aidlc-developer-agent サブエージェントに委譲します:

**計画:**
1. null の `display_name` を処理できるよう `ProfileSerializer.serialize()` を修正する
2. null／非 null のケースのユニットテストを追加する
3. `ProfileService.getProfile()` の防御的チェックを修正する
4. API エンドポイントの統合テストを追加する

計画を承認します。サブエージェントは 4 つのステップすべてを実装します:

- **変更**: `ProfileSerializer.java`（ユーザー名へのフォールバック付きで null セーフに）
- **変更**: `ProfileService.java`（防御的な null 処理）
- **作成**: `ProfileSerializerTest.java`（ユニットテスト 2 件）
- **作成**: `ProfileControllerIntegrationTest.java`（統合テスト 2 件）

**承認ゲート:** **Approve** を選びます。

### Stage 3.6 — Build and Test

aidlc-quality-agent がビルドとテストを実行します:

```
mvn clean compile        # BUILD SUCCESS
mvn test                 # 89 tests, 0 failures
mvn verify               # Integration tests pass
```

結果は `<record>/construction/build-and-test/test-results.md` に記録されます: 89 件のテストが成功、失敗 0 件、カバレッジは 62% から 64% に向上しました。

**承認ゲート:** **Approve** を選びます。

### Stages 4.1 and 4.3 — Deploy

Deployment Pipeline は既存のデリバリー設定を調べ、デプロイ戦略、
CD 設定、ロールバック用ランブックを記録します。このバグ修正では既存のデプロイ先環境を
使うため、Environment Provisioning はスキップされたままです。

承認後、Deployment Execution はテスト済みの成果物をそのパイプラインを通じてデプロイし、
スモークテストとヘルスチェックを実行して、デプロイログを記録します。
最後のゲートを承認すると、ワークフローが完了します。

### End state

```
aidlc/spaces/default/
  codekb/
    user-service/             # 9 space-level RE artifacts
  intents/260624-null-display-fix/
    aidlc-state.md            # All 9 stages marked [x]
    audit/                    # Full decision trail (per-clone shards)
    inception/
      requirements-analysis/ # requirements.md + questions
    construction/
      bugfix-null-display-name/
        code-generation/     # plan + summary
      build-and-test/        # instructions + test results
    operation/
      deployment-pipeline/   # CD config + strategy + rollback runbook
      deployment-execution/  # deployment log + smoke tests + health checks
```

ワークスペースのルートにあるアプリケーションコード:
- `ProfileSerializer.java`（変更）
- `ProfileService.java`（変更）
- `ProfileSerializerTest.java`（作成）
- `ProfileControllerIntegrationTest.java`（作成）

### Key observations

1. **すべてのドメインステージに承認ゲート** — 各判断をあなたがコントロールします
2. **Minimal の深さ** — 簡潔で的を絞った成果物。修正内容の定義に必要な質問だけが出されます
3. **サブエージェントへの委譲** — 重い作業（RE、コード生成）は、あなたが承認している間にサブプロセスで実行されます
4. **完全な監査証跡** — すべての判断が ISO タイムスタンプ付きで記録されます
5. **セッションの再開** — どの時点で中断しても、`/aidlc` が進行中の状態を検出します

---

## Feature Walkthrough

この例では、タスク管理アプリケーション向けの通知サービスを構築します。**feature** スコープは、Standard の深さで 33 のステージすべてを実行します。このウォークスルーでは、すべてのフェーズにわたる主要なステージを取り上げます。コラボレーターをオンにして（`/aidlc --collaborators on`）実行しているため、以下の Inception のアンサンブルではメンバー全員が登場します。feature スコープの出荷時設定ではオフになっており、その場合それらのステージはリードエージェントだけで実行されます。

### Invocation

```
/aidlc feature
```

> **What would you like to build?**（何を作りたいですか？）

> タスク管理アプリ向けの通知サービスです。タスクが割り当てられたとき、期日が近づいたとき、コメントが投稿されたときに、ユーザーがアプリ内通知と、任意でメールのダイジェストを受け取れるようにしたいです。ユーザーごとの通知設定もサポートしてください。

### Initialization (stages 0.1-0.3) — auto-proceed

3 つの Initialization ステージは `aidlc-utility intent-create` の中で自動的に実行されます。Workspace Detection は次を特定します: TypeScript、Node.js 20、Express、PostgreSQL、既存のタスクサービスとユーザーサービスを持つブラウンフィールドのプロジェクト。

> Setting up the feature workflow for this: 30 stages, 30 approval gates, 5 stages repeat per unit of work in Construction.（このための feature ワークフローを設定しています: 30 ステージ、30 承認ゲート、Construction では 5 ステージを作業ユニットごとに繰り返します。）

### Ideation Phase (stages 1.1-1.7)

**Stage 1.1 — Intent Capture**（aidlc-product-agent）

aidlc-product-agent はまず、参照してよい情報源の範囲を
`intent-capture-questions.md` に記録し、次に課題、対象ユーザー、
ステークホルダー、意思決定の権限、コミュニケーションのニーズ、スコープについて尋ねます:

```markdown
## Sources

- [desc] Initial description: "A notification service for our task management app..."
- [scope] Workflow-selected scope: `feature`.

## Q1. [Product owner] Which notification channels are in scope?
A. In-app only
B. In-app + email
C. In-app + email + push
D. In-app + email + push + SMS
X. Other

[Answer]: B. In-app + email
```

作成される成果物では、各主張がその情報源の一覧または確認済みの回答に
結び付けられています:

```markdown
## Target Customer

Task-management users receiving assignment, due-date, or comment events. [desc]

## Notification Channels

In-app notifications and optional email digests are in scope. [Q1]

## Assumptions & Open Questions

None.
```

ステークホルダーマップでは、`Source` 列で同じタグを使います。根拠のない内容は
追加の質問として尋ねられるか、`## Assumptions & Open
Questions` の下に残ります。残された前提にはあなたの明示的な承諾が必要で、引き続き
前提としてラベル付けされます。その後、aidlc-product-lead-agent が、通常の承認ゲートの前に
情報源との結び付きをレビューします。

**Stage 1.4 — Scope Definition**（aidlc-product-agent）

スコープの境界を定義します: スコープ内（3 種類のトリガー、ユーザー設定、メールダイジェスト）、スコープ外（プッシュ通知、SMS、リアルタイムの WebSocket）。優先順位付きの項目を含む `scope-document.md` と `intent-backlog.md` を作成します。

**Stage 1.7 — Approval & Handoff**（aidlc-delivery-agent）

Ideation のすべての出力を集約したイニシアチブブリーフをまとめます。フェーズ境界の検証で、インテントからスコープへのトレーサビリティを確認します。

> Progress: 7 of 30 stages done. Next: Reverse Engineering.（進捗: 30 ステージ中 7 完了。次: Reverse Engineering。）

### Inception Phase (stages 2.1-2.9)

**Stage 2.1 — Reverse Engineering**（pipeline）

既存のコードベースを 2 段でスキャンします。9 つの成果物を、リポジトリのスペースレベルのストア `aidlc/spaces/<active-space>/codekb/<repo>/` に書き出し、通知サービスが統合する必要のある既存のサービス構成、データベーススキーマ、API パターンを特定します。

**Stage 2.2 — Practices Discovery**（aidlc-pipeline-deploy-agent）

これはサブエージェントのハブ&スポークです。aidlc-pipeline-deploy-agent が Reverse Engineering の証拠からドラフトを作成し、その後 aidlc-quality-agent、aidlc-developer-agent、aidlc-devsecops-agent が、互いの貢献を見ずに並行してそのドラフトを確認します。人間へのインタビューで証拠のギャップとポリシー上の判断を解消し、その後リードが 3 つの貢献すべてを `team-practices.md`、`discovered-rules.md`、`evidence.md` に統合します。ゲートでは **Approve** / **Request Changes** が提示されます。Approve の後、`practices-promote` が `aidlc/spaces/<active-space>/memory/team.md` と `project.md` を書き出し、承認のタイムスタンプと対応する `PRACTICES_AFFIRMED` レシートをアトミックに記録します。その後で初めて、コンダクターはステージが承認されたと報告します。昇格が欠けている、古い、または失敗した場合は、ゲートは開いたままで、ステージは未完了のままです。

**Stage 2.3 — Requirements Analysis**（aidlc-product-agent）

12 の機能要件（通知のトリガー、設定の CRUD、メールのレンダリング、ダイジェストのスケジューリング）と 5 つの非機能要件（配信レイテンシー < 5 秒、メールのリトライ、設定の保存）を作成します。質問はエッジケースを掘り下げます: メール配信が失敗したらどうなるか？ダイジェストはどのくらいの頻度で実行すべきか？

**Stage 2.4 — User Stories**（mob）

aidlc-product-agent がまずペルソナとストーリーのドラフトを作成します。その後、aidlc-design-agent、aidlc-developer-agent、aidlc-quality-agent が、互いの内容を見ないコラボレーターとしてそのドラフトを確認し、それぞれが識別子付きの貢献ファイルを書きます。リードの aidlc-product-agent は、**Approve** / **Request Changes** のゲートを提示する前に、3 つの貢献すべてを `personas.md` と `stories.md` に統合します。

**Stage 2.6 — Domain Design**（aidlc-architect-agent）

aidlc-architect-agent が通知サービスのアーキテクチャを設計します:

- **コンポーネント**: NotificationService、PreferenceService、EmailRenderer、DigestScheduler — それぞれの振る舞い、依存先／依存元、所有するエンティティ
- **エンティティの所有権**: NotificationService は Notification と NotificationEvent を所有し、PreferenceService は Preference を所有します
- **根拠**: イベント駆動のトリガーパターン（ポーリングとの比較）、メールキューに SQS を使用（直接送信との比較）を Rationale セクションに記録

統合された `components.md`（フェンス付きの `yaml` カタログ + mermaid 図、概要、所有権、根拠の表）と、`decisions.md`（ADR ログ）を作成します。

**Stage 2.7 — Units Generation**（aidlc-architect-agent）

3 つの作業ユニットに分解します:

1. **notification-core** — イベントハンドラー、通知の保存、アプリ内配信
2. **notification-preferences** — 設定の CRUD API、デフォルト設定
3. **notification-email** — メールレンダラー、SQS 連携、ダイジェストのスケジューラー

`unit-of-work.md` と `unit-of-work-dependency.md` を作成します: まず notification-core、次に notification-preferences、そして notification-email の順です。notification-email の設定参照は preferences の API に依存します。

**Stage 2.8 — Contract Design**（aidlc-architect-agent）

システムが互いに連携する 3 つのユニットに分かれるため、Contract Design はユニット間の境界を正式に定めます: notification-core が呼び出し元に公開する内部イベントの契約と、notification-email が notification-preferences から利用する設定参照の契約です。`contract-summary.md`（境界ごとに 1 行、それぞれインラインの仕様ブロック付き）を作成します。

**Stage 2.9 — Delivery Planning**（aidlc-delivery-agent）

Delivery Planning は notification-core を最初の統合スライスと特定します: イベントが、保存された通知データとアプリ内配信に到達しなければなりません。後のデリバリーグループには preferences と email が含まれますが、実際の順序はそれらの依存関係に従います。Bolt ごとの DoD は `bolt-plan.md` に、根拠は `risk-and-sequencing-rationale.md` に、SES/SQS への依存は `external-dependency-map.md` に記録されます。エンジンは 2.7 の DAG と記録された反復の選択に従います。フェーズ境界の検証で、要件とアーキテクチャの整合性を確認します。

この例では、プロジェクトのチェックは `bun run verify:notifications` です。イベントを送信し、
保存とアプリ内配信を検証します。コンダクターは、提案するコマンドをファイル書き込みツールで
`<record>/verification-command.txt` に書き出し、
その後、検証コマンドの決定を開始します。`decision` ツールの JSON 出力から正式な
`command` 全体を、質問内のコードスパンにそのままコピーします:
"Use this command to verify each completed Unit?
`bun run verify:notifications`"（完了した各ユニットの検証にこのコマンドを使いますか？）。コマンドが省略されることは決してありません。人間は
`<record>/verification-command.txt` を開いて確認することもできます。正式なコマンドは、
空白でない 1 行で、最大 1024 文字でなければなりません。制御文字と
表示を偽装する文字（ゼロ幅文字や双方向制御文字を含む Unicode の書式文字、
行／段落の区切り文字、ノーブレークスペース U+00A0）は拒否されます。
**Approve** を選びます（そのチャットで、またはそれが最新の質問である間は新しいチャットで）。
その正確な回答が記録されて初めて、型付きのセッターがインテントのチェックポイントに対して
このコマンドを認可します。

> Progress: 16 of 30 stages done. Next: Functional Design.（進捗: 30 ステージ中 16 完了。次: Functional Design。）

### Construction Phase (stages 3.1-3.7)

ソースを生成するこの新しい単独ワークフローは、**Unit-major・直列チェックポイントの
デフォルト**の対象になります。2.9 の Bolt プランは計画の内容のままで、エンジンは
`unit-of-work-dependency.md` に従ってユニットを順にたどります。

**最初のユニット: notification-core — 動作する統合スライス**

コンダクターは、後のどちらのユニットにも着手する前に、notification-core に該当する設計ステージと
Code Generation を実行します。Functional Design は Notification と NotificationEvent の
エンティティ、重複排除、レート制限を扱い、NFR とインフラの作業は、該当する範囲で
最初のスライスを扱います。あなたは要約を確認し、生成の前に Code Generation の計画を承認します。
その後、ユニットはイベントハンドラー、通知リポジトリ、アプリ内配信の
エンドポイントを作成します。この例では、ソースファイル 3 つとテストファイル 4 つです。

必須のレビューと完了レシートの後、コンダクターは
記録済みのコマンドを検証を通じて実行します:

```bash
aidlc engine bolt checkpoint --action verify --unit "notification-core" --kind skeleton
```

プロジェクトのチェックに時間がかかる場合は、`--action verify` に `--timeout-ms 3600000` を
追加します（1 時間）。デフォルトは 30 分で、値は 2,147,483,647 ミリ秒以下の正の整数でなければなりません。
タイムアウトすると、所有するプロセスグループ／ツリーが終了され、
検証の失敗が記録されます。終了を確認できない場合は、後片付けが不完全であることを報告します。
[タイムアウトの契約](../reference/12-state-machine.md#construction-checkpoint-verification-timeout) を参照してください。

`verify` が `verified: true` を報告し、現在のチェックポイントが
`ready: true` になって初めて、コンダクターはセッションに結び付いた承認の質問を開始します:

```bash
aidlc engine bolt checkpoint --action ask --unit "notification-core" --kind skeleton
```

"Verified with `bun run verify:notifications` (exit 0). Approve this
completed notification-core?"（bun run verify:notifications で検証済み（exit 0）。完了した notification-core を承認しますか？）を **Approve** / **Request Changes** とともに提示し、待機します。
コードスパンには、要約ではなく記録されたコマンド全体が表示されます。あなたは
**Approve** を選びます。承認を認可するのは、このチェックポイントの質問に対するその正確な返答だけです。
そのチャットでも、それが最新の質問である間は新しいチャットでもかまいません。
無関係な返答、エージェント自身の言葉、別の質問への返答では
認可されません。コンダクターは、あなたが返答したチャットで、あなたが実際に選んだものを記録し、
あなたが選んでいない `--user-input` を渡すことは決してありません:

```bash
aidlc engine bolt checkpoint --action approve --unit "notification-core" --kind skeleton --user-input 'Approve'
```

それ以前の Functional Design のレビューだけでは、統合が動作することは
確認できず、その回答でこのチェックポイントを認可することもできません。
`verify` を再実行すると、このインテントについて開いているチェックポイントの質問と記録済みの応答が、
どのセッションであっても取り消されます。コマンドと成果物が変わっていなくても、コンダクターは再検証し、
`verified: true` の後に改めて尋ねなければなりません。スウォームの
バッチでは、同じルールが `finalize` に適用されます: 新たな検証とソースの
取り込みの後、改めて尋ねる前に `ready: true` を確認します。

自律実行の選択がまだ記録されていない場合、ワークフローは次を提示します:

```
How should I continue building the remaining work?
  ▸ Continue automatically
  ▸ Review each checkpoint
```

**Continue automatically** を選びます。コンダクターは `autonomous` を記録し、
直列のまま続行します。この承認の選択によってスウォーム実行が有効になることはありません。

**残りのユニット: notification-preferences、次に notification-email**

各ユニットは、次のユニットが始まる前に、該当する設計ステージ、有効になっている要約確認、
Plan Approval、コード、チェック、レビューをそれぞれ通過します:

- **notification-preferences** — Preference エンティティ、デフォルト、チャネルの切り替え、CRUD API、リポジトリ、検証。ソースファイル 2 つとテストファイル 3 つ。
- **notification-email** — 配信ルール、レンダラー、SQS コンシューマー、承認済みの設定参照の契約を使うダイジェストの cron ジョブ。ソースファイル 4 つとテストファイル 5 つ。

コンダクターは、記録された付与に基づき、`ask` や `--user-input` なしで、検証済みの通常のユニットチェックポイントを
それぞれ自動承認できます。Plan Approval と検証コマンドの選択は引き続き
あなたを待ちます。`directive.ceremony.summary_confirmation === "on"` の場合は
要約確認も同様です。すべての
ユニットが承認されると、完了専用のステージディレクティブが、ステージ本体やレビュアーの作業を
もう一巡させることなく、記録処理を整合させます。

**失敗した場合の様子。** notification-email のチェックが、SES のモックを
構築できないために失敗した場合、ワークフローは停止して失敗を説明します。
すでに完了している preferences ユニットは完了のままです。必要な修復計画の
承認は引き続き人間の判断であり、自動完了の付与が失敗したチェックの検証として
扱われることは決してありません。

**Stage 3.6 — Build and Test**（aidlc-quality-agent、すべてのユニットの後に一度実行）

ビルド手順を生成し、3 つのユニットすべてにわたってテストスイート全体を実行します: 47 件のテストが成功、失敗 0 件、カバレッジ 78%。

**Stage 3.7 — CI Pipeline**（aidlc-pipeline-deploy-agent）

lint、ビルド、テスト、セキュリティスキャンのステージを持つ CI パイプラインを設定します。品質ゲート: カバレッジ >= 75%、重大な脆弱性なし。

> Progress: 23 of 30 stages done. Next: Deployment Pipeline.（進捗: 30 ステージ中 23 完了。次: Deployment Pipeline。）

### Operation Phase (stages 4.1-4.7)

**Stage 4.1 — Deployment Pipeline** — ヘルスチェックのゲートを備えたブルー／グリーンデプロイ戦略

**Stage 4.2 — Environment Provisioning** — SQS キュー、SES の設定、通知保存用の DynamoDB テーブル

**Stage 4.4 — Observability Setup** — 通知配信のレイテンシー、メール送信レート、デッドレターキューの深さを表示する CloudWatch ダッシュボード。配信失敗のアラーム。

**Stage 4.7 — Feedback & Optimization** — SLO 目標（アプリ内配信 99.9%、30 秒以内のメール配信 99%）、コスト分析、フィードバックループのドキュメント。

### Key differences from bugfix

| 観点 | Bugfix | Feature |
|--------|--------|---------|
| 実行したステージ | 9 | 33 |
| 深さ | Minimal | Standard |
| フェーズ | Initialization + Inception + Construction + Operation | 5 つすべて |
| 作業ユニット | ユニットの DAG なし。ステージ単位の作業 | 3 |
| Construction の進め方 | 既存のステージ単位のフロー | 新しい Unit-major・直列チェックポイントのフロー（2.9 は引き続きデリバリーのグループ分けを計画） |
| 条件付きステージ | ほとんどスキップ | ほとんど実行 |
| 承認ゲート | 通常のステージ承認 | 検証済みスケルトンを人間がチェックポイントで承認。以降の完了は記録された選択に従う。Plan Approval、検証コマンドの選択、要約は人間が行う |

---

## Next Steps

- [スコープ、深さ、テスト戦略](05-scopes-and-depth.md) — スコープがどのステージを実行するかをどのように決めるか
- [ステージの実行のしくみ](04-phases-and-stages.md) — ステージのプロトコルの詳細
- [エージェント](06-agents.md) — エージェントのペルソナと責務
- [成果物リファレンス](14-artifacts-reference.md) — 成果物のディレクトリツリー全体
