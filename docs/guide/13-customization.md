# Customization

AI-DLC は、チームのニーズに合わせて調整できるように設計されています。この章では、設定の上書き、スコープの構成、ステージのカスタマイズ、ステータスライン、ツールの権限について説明します。

> **ハーネス固有の設定。** ハーネスに依存しないカスタマイズ（スコープの構成、
> ステージの深さ、ナレッジ、ルール）は、どのハーネスにも適用されます。この章で扱う
> 仕組みレベルの設定（`settings.json` / `settings.local.json`、
> ステータスラインのコマンド、`$CLAUDE_PROJECT_DIR`、ツール権限のブロック）は
> **Claude Code 固有** です。Kiro CLI では同等の設定を
> `.kiro/settings/cli.json` とそのエージェント設定で行い、Kiro IDE ではエージェントの Markdown の
> `tools:` と `permissions.rules` を使います。Codex は `.codex/config.toml`
> と Starlark ルール、Cursor は `.cursor/hooks.json` と `.cursor/cli.json`
> （権限のみ）、opencode はプロジェクトルートの `opencode.json`、Copilot は
> `.github/hooks/aidlc.json`（フックの配線）と `~/.copilot/config.json`
> （フォルダーの信頼）を使います。各ハーネスの設定箇所については、
> [Kiro で実行する](harnesses/kiro.md)、
> [Kiro CLI (V2) で実行する](harnesses/kiro.md#kiro-cli-v2)、
> [Codex CLI で実行する](harnesses/codex-cli.md)、
> [Cursor での AI-DLC](harnesses/cursor.md)、
> [opencode での AI-DLC](harnesses/opencode.md)、
> [GitHub Copilot での AI-DLC](harnesses/copilot.md) を参照してください。

---

## Settings Overrides (`settings.local.json`)

共有の `.claude/settings.json` はプロジェクトに属し、バージョン管理にコミットされます。AI-DLC は、あなたのフック、deny ルール、独自の `statusLine`、環境変数、その他の設定を保ったまま、自身のフック登録とコマンドの allow エントリを更新します。何を復元し何を残したかは注記で説明されます。このファイルに `--force` は不要です。同梱版で廃止された allow エントリは自動的には削除されません。チームに影響を与えずにローカル環境の設定を上書きするには、個人用の上書きファイルを作成します（たとえば AI-DLC のステータスラインをオンにしているなどの理由で `.claude/settings.local.json` がすでに存在する場合は、上書きコピーせずにそのファイルにキーを追加してください）:

```bash
cp .claude/settings.local.json.example .claude/settings.local.json
```

このファイルは `.gitignore` に記載されているため、個人的な変更がコミットされることはありません。次の用途に使います:

- モデルの選択を上書きする（例: 別の Opus や Sonnet のモデル ID に切り替える）
- ローカル環境用の環境変数を設定する
- セキュリティ要件に合わせてツールの権限を調整する

---

## Agent Models and Effort (Tiers)

同梱のエージェントは `tier:`（`judgment` | `balanced` | `templated`）付きで記述されており、ビルド時に各ハーネスのネイティブなモデル／effort のキーに変換されます。モデルポリシーが記録されていない場合、judgment と templated のエージェントはセッションのモデルと effort を引き継ぎます。balanced のレビュアーは、Claude Code では Sonnet を medium の effort で使い、Codex と opencode ではセッションのモデルを引き継いだうえで medium の推論設定を適用します。Kiro、Cursor、Copilot では、すべてのティアがセッションのモデルと effort を引き継ぎます。変換の全体表は [エージェントシステム](../reference/05-agent-system.md) を参照してください。

初回実行のウィザードは既定で `balanced` の **プリセット** を選びます。これはレビュアーの
**ティア** とは別物で、3つのグループすべてに medium の effort を記録します。
Kiro IDE、Cursor、Copilot ではプリセットを記録しません。これらではエージェントが
セッションのモデルと effort を保つためです（Kiro では、そのエージェントにモデルや effort が
設定されていない限り）。
プリセットは `aidlc config models --preset balanced --project --yes` で選択します:

| プリセット | 判断 | レビュー | 書き上げ |
|--------|----------|-----------|------------|
| `thorough` | セッションの effort | `xhigh` | セッションの effort |
| `balanced` | `medium` | `medium` | `medium` |
| `minimal` | `medium` | `medium` | `low` |

プリセットが設定するのは effort だけで、モデル ID は決して設定しません。エージェントごとの例外はグループの
ダイヤルより優先され、グループのダイヤルは同梱のティアの既定値より優先されます。Kiro IDE、Cursor、Copilot
ではこれらのグループの effort ダイヤルを表現できません。それでもそこに記録したポリシーは、
効果のないキーとして書き込まれるのではなく、保持されたうえで表現されていないと報告されます。Kiro CLI は各セッションを
1つのモデルで実行するため、そこではプリセットが対話セッション全体に1つの effort を設定します
（`minimal` は low、`balanced` は medium、`thorough` は extra-high）。これは
セッションのモデルとともに個人用の Kiro 設定に保存されます。
[Kiro CLI (V2) のセッションのモデルと effort](harnesses/kiro.md#kiro-cli-v2-session-model-and-effort) を参照してください。
明示的なグループのダイヤルには、引き続き Kiro CLI 上の設定箇所がありません。プロファイル、上書き、
アップグレードの手順については [モデルポリシー](18-install-and-lifecycle.md#model-policy) を参照してください。

1つのエージェントの effort を変えるには、`aidlc config models --agent <name> --effort <low|medium|high|xhigh|max> --project --yes` を実行します。モデルを固定するには、`--effort` の代わりに `--model <id>`（インストールで有効なモデル ID）を使うか、両方を渡します。`--effort default`（または `--model default`）はそのエージェントのその設定だけを取り除き、プリセットやグループが再び適用されるようにします。コマンドはこれを変更の取り消し手順として表示します。`--project` の代わりに `--local` を使うと、変更を自分だけに留められます。config は変更を設定ファイルに記録し、そこからそのエージェントのファイルを書き直します。config が書き込む `aidlc-*-agent` ファイルを手で編集しないでください。`aidlc config models --check` は手による編集をドリフトとして報告し、次の `aidlc config` で記録済みの値に戻されます。Claude Code、Codex、opencode は、1つのエージェントに対してモデルと effort の両方を受け付けます（Codex は `max` を `xhigh` として、opencode は `xhigh` を `high` として実行します）。Kiro CLI (V2) はモデルを受け付け、effort はモデルと一緒の場合にのみ受け付けます。Kiro IDE（および v3 エンジン上の Kiro CLI）は、1つのエージェントに対してモデルと effort の両方を受け付け、そのエージェントの `.kiro/agents/aidlc-<name>-agent.md` に書き込みます（[1つのエージェントを独自のモデルと effort で動かす](harnesses/kiro.md#one-agent-on-its-own-model-and-effort)）。そこではプリセットやグループの effort は何も変えません。Cursor と Copilot では、すべてのエージェントがセッションのモデルと effort を保ち、コマンドはそこで適用できなかった内容を伝えます。ソースから独自の配布物をビルドする際に**すべての**エージェントに上限を設けるには、`core/memory/org.md`/`project.md` の frontmatter に `tier_cap:` を設定するか、`AIDLC_TIER_CAP=<tier>` を付けてパッケージャーを実行します。どちらも `bun scripts/package.ts` に対するパッケージ時のつまみであり、実行時の設定ではありません。

---

## Per-Project Default Scope

プロジェクト内のすべてのワークフローを同じスコープで開始したい場合は、`.claude/settings.json` の `env` ブロックに `AWS_AIDLC_DEFAULT_SCOPE` を設定します（同梱のファイルではすでに `classic` に設定されており、フレームワークにハードコードされたフォールバックと一致しています。既定でライフサイクル全体を実行するには `feature` に設定してください）:

```json
{
  "env": {
    "AWS_AIDLC_DEFAULT_SCOPE": "feature"
  }
}
```

同梱の `env` ブロックに含まれるのは、AI-DLC のスコープの既定値だけです。プロバイダーと
モデルの設定は、引き続き Claude Code と利用者が管理します。

これを設定すると、暗黙のスコープ解決には `feature` が使われます。代わりに、`aidlc config flags --default-scope feature --project --yes`（このチェックアウトに限るなら `--local`）でプロジェクトの既定値を記録することもできます。実際の `AWS_AIDLC_DEFAULT_SCOPE` 環境変数は記録されたフラグより優先されるため、同梱の設定の env エントリを変更または削除するまでは、そちらが優先されたままです。インテントの `aidlc-state.md` が（その記録ディレクトリの下に）作成されると、そのスコープが正となり、暗黙の既定値を変更しても進行中のワークフローは変わりません。

**優先順位（高い順）:**

1. 明示的な CLI フラグ: `/aidlc feature` や `/aidlc --scope bugfix` が優先されます。
2. 自由記述に対してエージェントが提案する計画: `/aidlc fix the login bug` には引き続き `bugfix` が提案され、`/aidlc-init "fix the login bug"` も同様です。利用者は既存の確認プロンプトで提案された計画を上書きできます。
3. `.claude/settings.json` が与える値を含む、実際の `AWS_AIDLC_DEFAULT_SCOPE` 環境変数。
4. 記録された `aidlc config flags --default-scope` の値（ローカル設定は共有のプロジェクト設定より優先されます）。
5. `classic` - 一致しない自由記述の解決や、`--scope` なしで直接
   `intent-create` を呼び出した場合に使われるフレームワークのフォールバックです。説明付きで
   `--scope` なしの `/aidlc-init` は、代わりに `/aidlc` と同じ計画の提案を表示します。

**有効な値:** `enterprise`、`feature`、`mvp`、`poc`、`bugfix`、`refactor`、`infra`、`security-patch`、`classic`、`workshop`、`express`。無効な値は、呼び出し時にわかりやすいメッセージとともにエラーになります。チームは、`.claude/scopes/aidlc-<name>.md` ファイルを置いて、対象ステージの `scopes:` リストにタグを付けることで、スコープを追加定義できます。[コントリビュート: スコープの追加](../reference/11-contributing.md#adding-a-scope) を参照してください。チームは `.claude/agents/` にエージェントを追加定義することもできます。[コントリビュート: エージェントの追加](../reference/11-contributing.md#adding-an-agent) を参照してください。

**設定の確認:** `/aidlc --doctor` を実行して、構成された既定のスコープが有効であることを確認します（環境変数と記録された既定値はこのチェックを共有します）。同梱の値の場合、詳細な一覧には次の行が含まれます:

```text
  ok    AWS_AIDLC_DEFAULT_SCOPE=classic (valid)
```

Claude Code 外のターミナルで実行する `aidlc doctor` は `.claude/settings.json` の `env` ブロックを受け取らないため、そこでこの行が表示されるのは、シェルで `AWS_AIDLC_DEFAULT_SCOPE` を設定している場合か、`aidlc config flags --default-scope` で既定値が記録されている場合だけです。作業がどのスコープで実行されているかは、`/aidlc --status` の `Scope:` 行に表示されます。

なぜスコープだけで、深さやテスト戦略は設定しないのでしょうか。各スコープは深さを宣言しており、スコープが上書きしない限りテスト戦略はその深さを引き継ぎます。そのため `classic` は Standard/Standard、`workshop` は Standard/Minimal、`express` は Minimal/Minimal で始まります。どちらかを上書きする必要がある場合は、CLI で `--depth` または `--test-strategy` を渡してください。

**機密性のある値:** `.claude/settings.json` はバージョン管理にコミットされます。シークレット、認証情報、個人的な上書きをここに置かないでください。機密性のあるものには `.claude/settings.local.json`（gitignore 対象）を使ってください。

---

## Scope Configuration

スコープは、どのステージをどの深さとテスト戦略で実行するかを制御します。AI-DLC には 11 個の名前付きスコープがあります。完全な表（EXECUTE／総ステージ数、既定の深さ、テスト戦略、それぞれの用途）は [Scopes, Depth, and Test Strategy § The 11 Core Scopes](05-scopes-and-depth.md#the-11-core-scopes) に一元化されています。このセクションでは、それらの*構成*と上書きについて説明します。

### Choosing a scope

明示的に指定するか、あなたの言葉からエージェントに提案させます:

```
/aidlc enterprise       # Explicit scope
/aidlc Build a payments API  # The agent proposes a plan, or offers to tailor one
/aidlc Fix the login bug     # The agent proposes "bugfix"; you confirm
```

### Overriding at runtime

ワークフローの途中でいつでもスコープを上書きできます:

- **任意の承認ゲートで**: 別のスコープや深さを要求します
- **ユーティリティコマンドで**: `/aidlc --scope enterprise` がアクティブなスコープを変更します
- **ステージの追加**: Ideation と Inception の承認ゲートで、以前スキップしたステージをワークフローに戻せます

---

## Intent Configuration

インテントの9つの設定は、`depth`、`test-strategy`、`review`、
`guard-policy`、`sensors`、`learnings`、`summary-confirmation`、
`plan-approval`、`collaborators` の順です。さらに3つのキー、`guard.review-freeze`、
`guard.state-transition`、`guard.reviewer-scope` は、1つの作業に限って1つのガードをオフにしたり
オンに戻したりします。`guard.plan-approval` は `plan-approval` の別名です。
CLI の経路は、1つのアトミックなセッター
`config-change` を共有します。設定を含む `/aidlc` 行を入力すると、ヒューマンターンフックが
それを丸ごと記録しますが、何も適用しません。エージェントがその行に対してセッターを
1回実行し、セッターが行内のすべての設定を1つのトランザクションで検証・適用します。
個別の更新を連ねるのではなく、1つのコマンドに設定をまとめてください:

```
/aidlc --depth standard --test-strategy minimal --review advisory --guard-policy relaxed --sensors off --learnings on --summary-confirmation off
/aidlc config set guard-policy strict --sensors on --learnings on
/aidlc --scope bugfix --review none --guard-policy relaxed --sensors off
```

ネイティブの同等コマンドは、`aidlc engine config set <key> <value>` に続けて
残りの `--key value` フラグを付けたものです。`config get <key>` は 13 個すべてのキーを受け付け、
`config list`（必要に応じて `--json`）は 13 個すべてを返します。Guard Policy、切り替え可能なフェンス、
セレモニーについては、実効値とその出所も含みます
（`guard.plan-approval` は `plan-approval` の別名で、同じ値を読み取ります）:

```
/aidlc config get guard-policy
/aidlc config get plan-approval
/aidlc config get summary-confirmation
/aidlc config list --json
```

入力された `config set`、`get`、`list` の要求はディスパッチ中に実行され、
ターンが終わる前にコマンドの実際の出力を返します。拒否された設定は
そのエラーを返します。成功した設定はワークフローを前進させません。入力された `/aidlc`
行で下げたガードも同じセッターを通り、エージェントが行ごとに1回実行します。
フックは何も適用しません。

読み取り専用の追加の `guard.human-presence` 参照は、`on (default)` または `off (env AIDLC_SKIP_HUMAN_PRESENCE_GUARD)` を報告します。これは作業ごとの設定ではなく、`config list` には含まれません。

`config-change` が受け付けるのは、これらの設定フラグと、`--intent`、`--space`、
`--project-dir` のセレクターだけで、少なくとも1つの設定が必要です。例:

```bash
bun .claude/tools/aidlc-utility.ts config-change --guard-policy relaxed --sensors off --intent login-fix --space platform --project-dir /work/shop
```

入力形式の `/aidlc config set guard-policy relaxed --intent <name> --space <name>`
と `/aidlc --guard-policy relaxed --intent <name> --space <name> ...` は、エージェントのセッターが
スイッチを適用するインテントとスペースを指定します。
フラグ形式の末尾の `...` は、任意のタスク説明を表します。
存在しない名前のインテントは拒否されます。状態ファイルがない場合、その行のすべての設定
（Guard Policy、計画の承認、フェンスのスイッチ、セレモニー、
collaborators、深さ、テスト戦略、レビュー）は、このチャットで次に開始する作業のために
保持され、エージェントはその旨を1行で伝えます。作業がまだ存在しない状態で Claude
Code のチャットに `/aidlc --test-strategy minimal` と入力すると、次の出力が得られました:

```text
Test strategy will be minimal for the piece of work you start now (set by you).
```

セレクターは、アクティブなカーソルを切り替えずに、状態、メモリポリシー、監査について同じインテントを対象にします。
変更の前に、与えられたすべての値が検証されます。
無効な値や未知のフラグがあると、問題のフラグを名指しして更新全体が拒否されます。
メモリで強制された strict ポリシーは、明示的な `relaxed` や `off` の
Guard Policy 設定を、それに付随するすべての設定やスコープ変更とともに拒否します。
明示的な strict や無関係な設定は引き続き許可されます。1つのロックが、状態の読み取り、共有の
適用処理、完全な監査バッチ、1回の状態書き込みをまとめて保護します。監査に失敗すると、
状態は変更されません。`Last Updated` が変わるのは、出所の変更を含む、保存内容が実際に変わった場合だけです。
保存済みの同じ選択を繰り返しても何も起きません。

スコープの変更も同じフラグを受け付け、同じ適用処理を使います。
現在と同じスコープでも、与えられた設定は適用されます。スコープが所有する Guard Policy は、
新しい既定値がより厳しい場合はそれに従います。新しい既定値がより低い場合は、スコープの変更を
あなたが求めたときにのみそれに従い、そうでなければ現在の値が残り、出力がその旨を
1行で伝えます。セレモニーの行は引き続き新しいスコープの既定値に従います。明示的な
上書き（`set by you` または `set by a command`）と、存在しない旧形式の行は
保持されます。実効ポリシーは引き続きメモリが制御します。明示的なフラグは
明示的な出所を記録し、`config-change` と同じ引き下げの要件に
従います。新しいスコープ自身のレベルと等しいレビューレベルを指定すると、`Review Override` フィールドが空文字列にクリアされ、
ステージの宣言とそのスコープのレビュー上限が適用されます。それ以外のレベルは、
この作業に限ってその上限を置き換えます。

### Ceremony Switches

スコープは、互いに独立した5つのセレモニーの既定値を所有します。それぞれ `on` または `off` を受け付けます。
同梱のすべてのスコープは、既定値に頼らず5つすべてを明示的に宣言するようになりました。
いずれかを省略したスコープファイルは、引き続き `on` にフォールバックします。Classic は
sensors、learnings、計画の承認を `on` に、サマリー確認を `off` に設定します。
Bugfix は learnings とサマリー確認を `off` にし、sensors と
計画の承認は `on` のままにします。Express は最初の4つをオフにし、poc は計画の
承認もオフにします。`collaborators` は「既定でオン」の例外です。同梱で有効なのは
`enterprise` だけで、それ以外のすべてのスコープではオフです。そのため初回の実行は軽量で、
チームは意図的に collaborators を再び有効にします。

| スコープのキー | インテントごとのフラグ | グローバルなキルスイッチ | オフにすると無くなるもの |
|-----------|-----------------|--------------------|------------------|
| `sensors` | `/aidlc --sensors on\|off` | `AIDLC_DISABLE_SENSORS=1` | センサーの実行とそのゲートチェック |
| `learnings` | `/aidlc --learnings on\|off` | `AIDLC_DISABLE_LEARNINGS=1` | ステージの学びの読み書きの手順 |
| `summary_confirmation` | `/aidlc --summary-confirmation on\|off` | `AIDLC_DISABLE_SUMMARY_CONFIRMATION=1` | 出力前の独立したサマリー確認のチェックポイント |
| `plan_approval` | `/aidlc --plan-approval on\|off` | `AIDLC_DISABLE_PLAN_APPROVAL_GUARD=1` | 各コード計画を構築前に承認するよう求める停止（[プラン承認](#plan-approval)） |
| `collaborators` | `/aidlc --collaborators on\|off` | `AIDLC_DISABLE_COLLABORATORS=1` | ステージのサポートエージェント。オフにすると、すべてのステージをリードのみで実行します（ブラインドスポーク、モブラウンド、パイプラインのサポートリンクなし）。レビュアーには影響しません |

優先順位は、グローバルなキルスイッチ（`1`）→ 有効なインテントのフィールド → スコープの既定値 →
`on` です。キルスイッチは `aidlc config flags --bypass <NAME>` で記録することもできます。
新しいインテントは、`aidlc-state.md` の `Guard Policy` の後に `Sensors`、`Learnings`、`Summary Confirmation`、
`Plan Approval`、`Collaborators` を保存し、それぞれに
`on (from scope classic)` のような出所のラベルを付けます。この作業のために組み立てた計画の場合、`--status` はそのラベルを `(from the approved plan)` と表示します。状態ファイルにはスコープが保持されます。記録上あなたが求めたことが示されている場合、ラベルは `set by you` になります。
つまり、自分の行にフラグとしてスイッチを入力した場合
（横に入力した設定も含む）、自分のターミナルでコマンドを実行した場合、
あるいは自分の言葉で依頼し、エージェントがセッターでその依頼のメッセージを
指定した場合（`--message <id>`）です。エンジンはそのメッセージがあなたのもので、このチャットのものであり、
後の判断でまだ使われていないことを確かめます。あなたのメッセージ全体が監査の
行に記録されます。それを求めていない返信（"looks good, carry on"）の後にエージェントが
自分の判断で実行した設定も効力を持ちますが、ラベルは `set by a command` になります。依頼と一緒に入力した sensors、learnings、サマリー確認
（`/aidlc --learnings on build the export`）や、作業が存在する前に入力したものも、
その依頼で作成される作業では `set by you` になります。新しい作業を開始するコマンドに含まれる計画の承認や
Guard Policy も、そのチャットであなたが求めていれば同様です。
エンジンは、コマンドではなくあなたの言葉をフラグの裏付けとします。
あなたのメッセージの裏付けがなく、エージェントが自分の判断で付けたフラグは、エージェントや
スクリプトが実行するコマンドによるものとなり、`set by a command` になります。進行中の作業に対する変更は、
どちらの場合も `CEREMONY_SET` を記録します。新しい作業を開始するコマンドのフラグは、
それを記録せずに新しい状態ファイルに保存されます。進行中の作業でサマリー
確認や計画の承認をオフにするには、あなた自身のターンが必要です。
エージェントが実行すると、あなたに入力を求めるメッセージとともに拒否されます。フックにメッセージのテキストを渡さない Kiro IDE のビルドでは入力しても機能しないため、拒否メッセージは Kiro IDE の更新を求めます（[Kiro ガイド](harnesses/kiro.md#whats-different-on-kiro)）。`/aidlc --status` は実効値とその出所を表示します。
スコープを変更すると、スコープ由来の設定は更新されますが、あなたの上書きは保持されます。
フィールドがない、または形式が不正な場合は、実行を止めずにスコープにフォールバックします。
設定やスコープ変更によってサマリー確認をオンにすると、それ以降に実行される作業に
適用されます。ゲートで待機中のステージや、すでに完了したユニットの作業に対して、
存在しなかったサマリーを求めることはありません。

コンポーザーは、これら5つとスコープの `review_cap` をコンポーズゲートで
提案し、あなたは承認前にどれでも変更できます。計画の承認は、コンポーザーが
自分からは決してオフにしないものです。ゲートで "skip plan approval for this work" と言えば、
作業はそれがオフの状態で、あなたによる設定として作成されます。これらの値はこの作業にのみ適用され、
同梱の計画でもカスタムの計画でも同じです。スコープとして保存した計画はそれらを frontmatter に保存するため、
そのスコープ上の新しいインテントはすべてそれらの値で始まります。それでもキルスイッチが
優先されます。ゲートはキルスイッチが強制的にオフにした `on` の値に印を付け、ワークフローの途中であれば
エージェントは、そのスイッチはワークフローの外で外す必要があることを1行で伝えます。どこで設定されているかは
探しません。

計画の承認（それが名指しする承認そのもの）を除き、これらのスイッチは
承認ゲート、ヒューマンターンの権限、監査、チームのユニット間の
書き込み保護を取り除きません。Classic は
ウォーキングスケルトンのセレモニーをオフにし、ゲート付きフローのレビューを advisory に制限します。明示的な自律モードでも、
マージ前の1回のレビューは維持されます。

隔離された `--single` の試行は、メインのインテントのセレモニーの上書きではなく、
選択したスコープのポリシーを使います。そのスコープは合成された
ステージ開始イベントに記録され、完了まで固定されます。別のスコープでの再開は
拒否されます。スコープが記録されていない旧形式の隔離開始は、
サマリー確認を維持し、このスコープの比較は行いません。

### Plan approval

コード生成がコード計画から構築を始める前に、AI-DLC はその計画の承認をあなたに
求めます。この作業で承認を求めるかどうかは `plan_approval` セレモニーが決めます。

- **オン**（express と poc を除く同梱のすべてのスコープ）: 各計画が
  サマリーとパスとともに、**Approve Plan**、**Request Changes**、**I'll
  edit the files** の選択肢付きで表示され、あなたが答えるまで何も構築されません。
- **オフ**（express と poc）: 計画が書かれると1行が表示され、
  構築が始まります:

  > Plan written: aidlc/spaces/default/intents/260820-checkout/construction/code-generation/code-generation-plan.md. Plan approval (you approve each code plan before it is built) is off for this piece of work (from scope poc). Starting code generation now. You can ask me to go over the plan at any time.

  （訳: 計画を書きました: …。この作業では計画の承認（各コード計画を構築前にあなたが承認すること）はオフです（スコープ poc による）。コード生成を今から開始します。いつでも計画の見直しを依頼できます。）

  監査証跡には、構築された計画のフィンガープリントを含む `PLAN_APPROVAL_SKIPPED` の行が
  記録され、質問ファイルには
  `[Answer]: Plan approval off` と書かれます。あなたが承認したという記録は残りません。

**まず計画を見直す。** 計画の承認がオフの場合、各計画を名指しする行には
いつでも見直しを依頼できることが書かれ、構築が始まります。自分の言葉で依頼すると、
構築を待たせたままその計画が承認のために表示されます。設定はオフのままなので、次の計画は
尋ねられずに構築されます。依頼した時点で計画がすでに構築中であれば、その構築は
最後まで進み、その後、計画が構築結果と並べて表示され、あなたが答えるまで
他には何も始まりません。承認すると構築されたものは維持されます。あなたが求めた変更は
そのユニットの承認ゲートに戻り、そこで **Request Changes** を選ぶと、あなたの言葉とともに
差し戻されます。

**オフにできるのはあなただけ。** 自分の言葉でそう伝える（"skip plan approval for
this work"）とエージェントがオフにします。あるいは `/aidlc --plan-approval off` か
`/aidlc config set plan-approval off` と入力すると、エージェントがセッターを実行します。どちらの
方法でもこの作業に適用され、`CEREMONY_SET` の行が記録されます。すでにあなたの回答を
待っている計画は、その後構築されます。計画の承認に触れただけの質問や発言では
何も変わりません。作業が存在する前、つまりコンポーズゲートやスコープの
確認の時点では、あなたの言葉はその依頼への回答になり、そこから作成される作業は
計画の承認がオフの状態で、あなたによる設定として始まります。作業内容を説明する前に言った場合は、
あなたが次に行う依頼への回答になります。あるいは、ここに他の作業があり、代わりにそれを再開する場合は、
その作業に適用され、その旨が一度だけ伝えられます。それ以外の依頼から作成された作業はスコープの値を保ち、
先に `/aidlc --plan-approval on` と入力すると、その言葉は取り消されます。エージェントは自分からオフにすることも、
それを提案することも決してありません。直前の判断以降にあなたからの返信がない状態で実行すると、そのセッターは拒否されます。オンにすること
（`/aidlc --plan-approval on`）は、エージェントを含めどこからでも可能で、
次の計画から適用されます。`guard.plan-approval` は同じスイッチの別名です:
`/aidlc config set guard.plan-approval off` は計画の承認をオフにし、
`Guards Off` の行は書きません。

**編集された計画。** 承認後に編集された計画について再度尋ねるかどうかは、
この設定ではなく Guard Policy が決めます。`strict` は再度尋ね、`relaxed`
と `off` は1行の通知とともに続行します（[Guard Policy](#guard-policy) を参照）。

**オンに保つもの、すべてでオフにするもの。**

- メモリの `## Guard Policy` セクションに `Mode: strict` がある場合（`org.md`、
  `team.md`、`project.md` のいずれか）、そのリポジトリのすべての人に対して、express と poc も含め
  計画の承認はオンに保たれます。オフにしようとすると、そのファイルを名指しする文とともに拒否され、
  `/aidlc --status` には
  `Plan Approval: on (guard policy strict (from project.md))` と表示されます。
- マシンのスイッチ `AIDLC_DISABLE_PLAN_APPROVAL_GUARD=1` は、そのマシン上のすべての作業で
  計画の承認をオフにし、メモリのロックよりも優先されます。これが有効になるのは、
  ハーネスがそれを付けて起動された場合（セッション開始フックがそのセッションについて記録します）、
  `aidlc config flags --bypass AIDLC_DISABLE_PLAN_APPROVAL_GUARD` で記録された場合、あるいは
  プロジェクトにハーネスのセッションが記録されていない場合（CI、単純な CLI 実行）です。セッション内で
  コマンドが自分自身のためにそれを設定しても、未設定として扱われます。そのようにして構築された
  各計画はスキップとして記録され、`Source` にそのスイッチが
  記されます。

**スコープの変更** では新しいスコープの値に従います。feature から express に移ると
計画の承認はオフになり、express から feature に移るとオンになります。この作業に対して
あなた自身が設定した値は残り、メモリのロックは引き続き優先されます。これは
Guard Policy とは異なります。Guard Policy では、より低いスコープの既定値に従うのは
あなた自身がスコープの変更を求めた場合だけで、そうでなければより厳しい値が残ります。

**チームモード。** 各ユニットの計画は、そのユニットの担当者が自分の
チェックアウトで承認します。計画の承認がオフの場合はそこで尋ねられずに構築され、
チームのマージは承認の代わりにスキップの記録を受け入れます。

**Kiro IDE。** フックにメッセージのテキストを渡さない Kiro IDE のビルドでは、
計画の見直しの依頼を受け取れないため、そこではすべての計画が引き続き承認のために
表示され、Kiro IDE を更新すれば計画が尋ねられずに構築されるようになることが
1行で伝えられます。

`/aidlc --status`、`/aidlc config get plan-approval`、`/aidlc config list` は、
実効値とその出所を表示します。例:
`Plan Approval: off (from scope poc)`。

### Guard Policy

Guard Policy は、`strict`、`relaxed`、`off` の3つの値を持つ1つの設定です。取り組んでいる作業について、フレームワークのガードがどこまで道を譲るかを決めます。対象は2つあります。すでに承認したものが裏で変わっていたと判明したときにどうするか、そして誰も求めていない作業に対して5つのフェンスがどれだけ強く守るかです。

**承認済みの入力が変わったとき。** レビュー済みのドキュメントがレビュー後に編集された場合や、出力が現在のサマリー確認なしに保存された場合です。

- `strict` は承認をやり直します。実行は、何が変わったかを名指しする平易な1文とともに止まり、あなたに再度尋ねます。あなたの確認が最新で、エージェントがそれ以前に保存したドキュメントが再保存されていないだけであり、まだ何もそれをレビューしていない場合は、エージェントがまずそれを再保存し、それで解消しない場合にのみあなたに尋ねます。
- `relaxed` と `off` は続行します。変更は監査証跡に `CHANGE_ACCEPTED` の行として一度記録され、あなたにはそれについて1行が伝えられ、実行は続きます。何も削除されません。承認とその証拠は元のまま残ります。

コード計画を承認した後に他のコードが動いた場合（`git pull`、別のユニットの取り込み）は、下記の1つの例外を除き、どの値でも再度尋ねることはありません。計画の承認は、計画とそのテスト手順についてのものだからです。構築は続き、どのファイルが動いたかが伝えられ（例: `2 files changed since this plan was approved: src/api.ts, src/db.ts. Building auth now.`）、`CHANGE_ACCEPTED` の行がそれを記録します。あなたが取り込んだ AI-DLC の更新は、あなたの変更ではありません。そのファイルが名指しされることはなく、動いたのがそれだけであれば何も伝えられず記録もされず、ユニットの並列構築はそれらをコミットするよう求めずに始まります。

例外は、`strict` の下でのユニットの並列構築です。Code Generation の開始後にプロジェクトで起きた変更のうち、ユニット自身の取り込み、AI-DLC の更新、Git が無視して追跡しないファイルのいずれでもないもの（あなたの編集やコミット、`git pull`）は、次のユニットの取り込みを止めます。`relaxed` と `off` の下では、ユニットは変更を保持したまま取り込まれ、1行でその旨が伝えられます。変更が計画の承認後に起きた場合、`strict` はユニットのセットアップ前に計画を再度承認するようにも求めますが、再承認しても取り込み時の停止は解除されません。先に進むには、変更を元に戻してエージェントに続行を伝えるか、この作業の Guard Policy を `relaxed` に設定します（`/aidlc --guard-policy relaxed`）。そうすると、ユニットは変更を保持したまま取り込まれます。チームがメモリで `strict` をロックしている場合、そのスイッチはファイルを名指しする行とともに拒否されるため、そこでは変更を元に戻すのが先に進む方法です。

**フェンスがどれだけ強く守るか。** `strict` は5つのフェンスすべてを立てたままにします。`relaxed` は計画の再承認とレビューフリーズを下げます。`off` はその2つに加えて、状態遷移とレビュアーの読み取り範囲も下げます。どの値でも、人間の関与と、クレーム済みチェックアウトのユニット書き込み所有権は下がりません。下げられたフェンスも、何かを通すたびに監査の行を書きます。チームのブロッキングチェックを和らげる値もありません。`blocking` に設定されたチェックは、`strict`、`relaxed`、`off` のいずれでも、そのゲートやユニットのチェックポイントを止めます。作業を先に進められるのは、そこでのあなたの Override か、その作業でチェックをオフに切り替えることだけです。

**承認後に計画自体が変わったとき。** 同じユニットまたはステージの対象と試行について、計画、テスト手順、Testing Contract への編集は、`relaxed` または `off` の下では再承認を必須とせずに続行します。`strict` の下では、それらの編集は承認をやり直させ、編集された計画について再度尋ねられます。どちらの場合も、"Your approved plan changed before the build: step 4 now says ... instead of ..." のように何が変わったかを伝える1行が表示され、エージェントは承認した計画に戻すかどうかを尋ねます。構築が始まった後は、同じ行で構築を続行することが伝えられ、代わりに承認済みの計画を構築することが提案されます。はいと答えるか、どのチャットからでも（`/aidlc` の後でも）自分の言葉でそれを求めると、あなたが与えた計画、テスト手順、承認が元に戻り、その後の構築は承認済みの計画に基づくものになります。これは Guard Policy だけが決めることです。[plan approval](#plan-approval) の設定が決めるのは、そもそも計画について尋ねるかどうかだけです。同じインテント、対象、試行の中で Testing Posture、スコープ、テスト戦略、プロジェクトの種類が変わった後の更新にも同じルールが適用されます。必要に応じて契約と手順を更新し、フェンスが下がっている間は続行します。`strict` の下での例外が1つあります。構築のレビュー後に行われた Testing Posture の変更（学びのステップで残すテストのプラクティスなど）は、その構築の承認をやり直させず、次の計画から適用されます。決めるのは実効的なフェンスの設定で、`/aidlc --status` に表示されます。計画の再度の見直しは引き続き依頼できます。

最初の Plan Approval（[plan approval](#plan-approval) がオンの間）とその他のゲートは引き続き必須です。フェンスが下がっていても、編集された内容が承認されたことにはなりません。元の回答と承認の証拠は、あなたが実際に承認したものの記録として残ります。レビュアーの判定が変えられることはなく、証拠が削除されることもなく、エージェントがあなたの代わりに答えることも決してできません。

AI-DLC は Plan Approval を自ら求めます。計画のサマリーとパスを、**Approve Plan**、**Request Changes**、**I'll edit the files** とともに表示します。Code Generation の開始以降、または前の計画の承認以降に、ワークスペース内のファイルがすでに変わっている場合（計画より先に進んだエージェントや、あなた自身の編集）、サマリーの末尾にそれらを名指しする1行が付きます: "Already changed before you approved: src/app/page.tsx."（訳: 承認前にすでに変更されています: src/app/page.tsx。）それ以降に承認された計画が名指しするファイルはその計画自身の構築によるものなので列挙されず、AI-DLC の更新が変更した AI-DLC 自身のファイルも列挙されません。エージェントは、この作業に関するどのチャットからでも、あなたの選択をあなたの言葉そのままとともに記録します。あなたの承認がコミットされプッシュされると、同じ作業を続けるチームメイトのクローンやあなたの2台目のノートパソコンは、再度尋ねることなくその計画を構築し、あなたの回答は質問ファイルに残ります。そこで計画が変更された場合や Code Generation をやり直した場合は、尋ねられます。"Approve, but add a test for the empty cart" は承認と指示の組み合わせです。エージェントはそれを計画に加え、承認はその時点の計画全体に及びます。計画を編集したり、`code-generation-questions.md` に回答を書いたりしてから、終わったことをエージェントに伝えることもできます。下げられた計画承認フェンスが許す内容変更に対して、再承認の停止は追加されません。

既存の委任されたワーカーは、検証済みの親インテントの現在の
計画承認の設定に従います。下げても上げても、次のチェックで適用されます。
その設定を適用するためにワーカーを作り直す必要はありません。

#### Defaults per scope

| スコープ | 既定値 |
|-------|---------|
| enterprise | strict |
| poc, express, classic, bugfix, feature, mvp, refactor, workshop, security-patch, infra | off |

`bugfix`、`classic`、`express`、`feature`、`infra`、`mvp`、`poc`、`refactor`、`security-patch`、`workshop` は `off` で同梱されています。`classic` は暗黙の既定スコープなので、スコープを指定しない作業も `off` で始まります。`enterprise` では、`off` はあなたが求めるものです。組み立てた計画はスコープファイルを書きません。一致した計画は、元の同梱スコープの既定値か、あなたが求めたより厳しい値を持ちます。カスタムの計画は、ゲートであなたが承認した値かそれより低い値を既定値とする同梱スコープ上で実行されます。スコープとして保存した計画は、その値を `guard_policy: <value>` として保存します。

インテントの作成時には、計画が実行されるスコープから Guard Policy を読み取ります。コンダクターは `strict` または `relaxed` の場合に `--guard-policy` を渡し、より低いスコープの既定値を引き上げます。`off` の場合は決して渡しません。コンポーズゲートで一致した計画の Guard Policy を下げると、その値を持つ同梱スコープ上のカスタム計画になります。引き上げた場合、計画は一致したままです。どちらの場合も、インテントは作成時にその値を受け取ります。コンポーザーが進行中のインテントの値を変えることは決してありません。

#### The three places to set it

1. **スコープファイル。** `scopes/aidlc-<name>.md` の `guard_policy: strict | relaxed | off` は、そのスコープ上のすべての新しいインテントの初期値です。同梱のすべてのスコープがこれを宣言しています。何も宣言しないスコープファイルは、`enterprise` 以外の同梱スコープと同じく off で始まるため、strict にしたいプラグインや組み立てたスコープは `guard_policy: strict` と記述します。
2. **メモリ。** `aidlc/spaces/<space>/memory/org.md`、`team.md`、`project.md` のいずれかにある、`Mode: strict` の1行だけを持つ `## Guard Policy` セクションは、リポジトリのすべての人に対して strict を保持します。これはスコープの既定値やインテントごとの値より優先され、[plan approval](#plan-approval) をオンに保ちます。明示的な `--guard-policy relaxed`、`--guard-policy off`、`/aidlc config set guard.<fence> off`、または計画の承認をオフにする操作は、メモリファイルを名指しする文とともに拒否され、そのコマンドに付随する設定やスコープ変更も一切適用されません。フェンスを `on` にすることは引き続き許可されます。`Mode: relaxed` や `Mode: off` は、値がスコープ由来であるすべての作業のポリシーを設定します（status に表示されます。例: `Guard Policy: off (from team.md)`）。宣言しているうち最も狭いレイヤーが優先されます（`project.md`、次に `team.md`、次に `org.md`）。1つの作業に対してあなたが設定した値はそのまま残り、いずれかのレイヤーの `Mode: strict` は引き続きすべてに優先します。空のセクションは何も変えません。それ以外の値は、ファイルと許可された3つの値を名指しする検証エラーになります。
3. **インテント。** `/aidlc --guard-policy relaxed|off`、`/aidlc config set guard-policy relaxed|off`、または確認の言葉 `guard policy relaxed|off` を入力すると、エージェントは同じターンで、選択されている作業に対してセッターを実行し、監査の行を記録します（`/aidlc --status` には `Guard Policy: relaxed (set by you)` と表示されます）。他のインテント設定を含む入力コマンドは、まずすべての値を検証し、同じロックの下ですべてを適用します。不正な形式のコマンドや無効な付随設定がある場合は何も変わりません。スイッチを入力した後、コンダクターは `next` を実行し、それを注入するハーネスでは、道を譲った旨の行や `AIDLC Guard Policy: ...` のフックコンテキストを伝えます。平易な言葉による依頼（"stop asking me to re-approve when files change"、"be strict about changes"）はコンダクターが実行します。コンダクターはあなたが求めた値で `config-change --guard-policy <strict|relaxed|off>` を実行し（直前の判断以降にあなたからの返信が記録されていれば引き下げます）、その出力をそのまま表示して止まります。この Kiro IDE のビルドがプロンプトのテキストを渡さない場合、進行中の作業は引き下げられません。Kiro IDE を更新するか、より低い既定値のスコープで新しい作業を始めてください。スコープを変更すると、スコープが所有するポリシーが自動的に引き上げられることがあります。より低いスコープの既定値に従うのは、あなたがスコープの変更を求めた場合です。それ以外のスコープ変更では、より厳しい値がそのまま残り、その旨が1行で伝えられます。作成時にスコープ自身の既定値を指定すると、スコープの値が記録されます。廃止された `Change Control: relaxed|off` フィールドだけがある場合は、値を変えず、ポリシーの監査の行も書かずに自動的に正規化されます。

#### Where the value lives

解決された値は、作成時にインテントの `aidlc-state.md` に `- **Guard Policy**: <value> (from scope <name>)` として書き込まれ、フラグ、入力された確認の言葉、またはコンダクターがセッターを通じて実行する平易な言葉による依頼によって書き換えられ、値としてのみ読み取られます。状態ファイルはインテントとともにコミットされるため、値はセッションをまたいで残り、チームメイトも同じ値を見ます。実行中のインテントの実効値を変えるメモリの編集は、次に統制されたチェックが実行されたときに、メモリファイルを名指しする `GUARD_POLICY_SET` の行として記録されます。このフィールドが存在する前に作成されたインテントは、あなたが設定するまで `strict (not set)` のままです。ただし、メモリファイルが `relaxed` または `off` を宣言していれば、それにも適用されます。無効なフィールドは、3つの値のいずれかを付けた `/aidlc --guard-policy` で修復されるまで使用できません。次のインテントは、再びそのスコープの既定値から始まります。

状態ファイルに `Guard Policy` と廃止された `Change Control` の両方があり、
ポリシーの語が異なる場合（状態ファイルは git で受け渡されるため、名前変更前のリリースを使うチームメイトが
両方を残すことがあります）、次の `next` はどちらか1つを残し、それを
唯一の行として書きます。残すのは、監査証跡上で最後に設定されたもの、なければあなた自身が
設定したもの、なければスコープの初期値、なければより厳しいものです。
`Source: command` と、両方の行を名指しする `Settled` フィールドを持つ
`GUARD_POLICY_SET` の行を記録し、何も尋ねません。あなた自身の最後の選択を残した場合は何も伝えません。
記録から判断できなかった場合は、どちらを残したかを一度だけ伝えます。例:

> This work had two settings for how closely AI-DLC checks changes, and they disagreed. The record does not say which was set last, so AI-DLC keeps the stricter one: it checks everything. You can change it any time.

（訳: この作業には、AI-DLC が変更をどれだけ厳密にチェックするかについて 2 つの設定があり、食い違っていました。記録からはどちらが最後に設定されたかわからないため、AI-DLC はより厳しいほうを維持します。つまり、すべてをチェックします。これはいつでも変更できます。）

それまでの間、status には `strict (from conflicting state lines)` と表示されます。メモリで
strict を保持しているチームでは strict が保たれ、2つの行はそのまま残ります。両方が一致している場合は、
`Guard Policy` の行が使われます。ポリシーの行が書き込まれると、廃止された
行は削除され、設定は1つになります。

#### This setting used to be called Change Control

古い表記はすべてこのリリースでも機能し、次のマイナーバージョンで削除されます。スコープのキー `change_control`、状態フィールド `Change Control`、メモリの見出し `## Change Control`、フラグ `--change-control`、設定キー `change-control` です。廃止されたフラグや設定キーを入力すると、新しい表記を名指しする1行が表示されます。廃止されたスコープのキーやメモリの見出しは、何も伝えずに読み取られます。古い名前が再び書き込まれることはありません。ポリシーの行が書き込まれると、それが唯一の行だったか `Guard Policy` と並んでいたかにかかわらず、廃止された `Change Control` の行は削除されます。`CHANGE_CONTROL_SET` の監査イベントは古い台帳でも引き続き読み取れます。新しい行は `GUARD_POLICY_SET` です。

廃止された `Change Control: relaxed` または `Change Control: off` の行だけを持つ作業は、あなたの次のメッセージの時点で名前が変更されます。AI-DLC は値を保ち、行を `Guard Policy` として書き、ポリシーの監査の行は書きません。名前の変更がまだ行われていない場合、すべての `/aidlc` の実行で、ディレクティブの `change_notices` に通知が含まれます。通知を表示しても行は書き換えられません。例: `This work still has an old setting that lets AI-DLC skip some of its checks (it asks you to confirm less often). Do you want to keep that, or have AI-DLC check everything again? I'll ask again until you choose.` `/aidlc config set guard-policy relaxed` または `guard policy relaxed` と入力すると、エージェントのセッターがその値を保ったまま行を `Guard Policy` として書き直します。あるいは strict を求めると、コンダクターがセッターを通じてフェンスを引き上げます。どちらの書き込みでも通知は止まります。廃止された strict の行だけの場合は通知されず、次にいずれかの Guard Policy の設定が書き込まれたときに書き直されます。

### The five fences

フェンスとは、何も求めていない操作、つまりワークフローが現在実行中のどのステップも求めていない操作を拒否するガードです。3つのフェンスは、1つの作業に限ってオフにし、再びオンに戻すことができます。計画承認のフェンスは代わりに [plan approval](#plan-approval) の設定に従い、編集された計画について再度尋ねるかどうかは Guard Policy が決めます。人間の関与は鍵の保持者であり、作業ごとのスイッチはありません。

| フェンス | 拒否するもの | 作業ごとのスイッチ | マシン全体のキルスイッチ |
|-------|-----------------|------------|--------------------------|
| 計画の承認 | 計画が承認される前のコード、または計画の承認がオフの場合は、尋ねずに構築することをエンジンが記録する前のコード | なし: `guard.plan-approval` は [plan approval](#plan-approval) の設定の別名 | `AIDLC_DISABLE_PLAN_APPROVAL_GUARD=1`（計画の承認をオフにする） |
| レビューフリーズ | レビューのレシート後のレビュー済み内容への編集 | `guard.review-freeze` | `AIDLC_DISABLE_REVIEW_FREEZE_HOOK=1` |
| 状態遷移 | ワークフロー自身のものではなく直接実行されるライフサイクルコマンド | `guard.state-transition` | なし |
| レビュアーの読み取り範囲 | ディスパッチされたレビュアーによる兄弟ユニットの内容の読み取りや検索 | `guard.reviewer-scope` | `AIDLC_DISABLE_REVIEWER_SCOPE_HOOK=1` |
| 人間の関与 | 実際の人間のターンを伴わない承認や回答 | なし: 鍵の保持者にはインバンドのスイッチがない | `AIDLC_SKIP_HUMAN_PRESENCE_GUARD=1` |

```
/aidlc config set guard.review-freeze off
/aidlc config set guard.review-freeze on
```

フェンスやポリシーの語を下げるかどうかは、利用者が決めることです。自分の言葉で依頼するとエージェントがセッターを実行します（"turn the guards off" のようにガード全体をオフにするよう求めると、Guard Policy が `off` に設定されます）。あるいは `/aidlc config set guard.<fence> off`、`/aidlc --guard-policy relaxed|off`、または確認の言葉 `guard policy relaxed|off` のいずれかを、値を1つ選んで入力します。エージェントのセッターは、入力されたスイッチを `--intent <name>` と `--space <name>` で選択された作業に、これらのセレクターが省略された場合はチャットのワークフロー選択で選ばれた作業に適用し、状態と監査の行を書きます。それを注入するハーネスでは、`AIDLC Guard Policy: ...` をフックコンテキストとして報告します。存在しない名前のインテントは拒否されます。作業が存在する前に入力された Guard Policy の `relaxed` または `off` のスイッチは、このチャットで次に開始する作業のためのものです（`Guard Policy relaxed for the piece of work you start now (set by you).`）。依頼と同じメッセージで入力した場合は、その依頼に伴います（`Guard Policy relaxed for the work you are asking for (set by you).`）。新しい作業は作成時にそれを受け取り、代わりに開いている作業を続けることを選んだ場合はそこに適用されます。メッセージだけで開いている作業が変わることは決してありません。`/aidlc --guard.review-freeze off fix the parser` のようなフェンスのスイッチも同様に機能します（`The review freeze check is off for the work you are asking for (set by you).`）。唯一の例外は、コード計画の質問が開いていて、あなたが計画ファイルを編集していない間です。このとき、言葉とともに入力した設定（`/aidlc --guard-policy off approve the plan`、またはあなたの選択を伴う `strict`）はこの作業のためのものとなり、その言葉はその質問へのあなたの返信になります。引用されていない `--` の後の言葉は、引き続き新しい作業の説明になります。作業が存在する前に入力した計画の承認の `off`、フェンスのスイッチ、その他すべての設定（セレモニー、collaborators、深さ、テスト戦略、レビュー）も同じように、このチャットで次に開始する作業のために保持されます。無関係な返信では何も開かれません。

CLI のセッターが引き下げるのは、直前の判断以降にあなたからの返信が届いている場合だけで、スイッチ権限のためのセッション参照は行いません。承認と同じメッセージで求めた設定は、その承認に便乗します。"approve, and turn plan approval off" は、1ステップでオフにして承認します。"Approve the plan, and run Construction on its own from here" も両方を行います。どちらの順序でも、どの質問でも設定は有効です。エージェントは、その返信を扱っている間であれば、まずあなたの回答（承認、またはサマリー、コード計画、Construction の質問への回答）を記録し、その後に設定を記録できます。すでに承認や別の判断を与えたあなたのメッセージは、あなたが別のメッセージを送った後、チェックがエージェントを止めた後、あるいは次の質問が表示された後には、設定を追加で変えることはありません。そのため、エージェントがあなたの以前の "Approve" を根拠にチェックをオフにすることはできません。単独のメッセージで依頼すれば、すぐに行われます。承認対象のコード計画が書かれた後は、計画の承認がオフになるのは、その質問が表示された後にあなたが送ったメッセージによる場合だけです。それより前に依頼していた場合は、その計画について理由を述べる1行とともに一度尋ねられ、再度そう伝えるとオフになります。そのメッセージでエージェント自身の質問に与えた回答も、依頼を有効なままにします（エージェントが変更を提案したときに選ぶ選択肢は、その変更の依頼です）。あなたのメッセージの後に記録されたその他の判断はそれを使い切り、承認には常にそれ専用の返信が必要です。フックは Windows でも動作するため、入力されたスイッチはプロンプトを転送するすべてのハーネスで機能します。すでにオフのフェンスや、すでに `set by you` と記されている同一のポリシーの語は、CLI の更新が何もしないため、鍵を必要としません。

ガードの質問が "turn the check off for this piece of work" を提示した場合、それを選ぶだけで十分です。エージェントがあなたに代わってオフにし、スイッチが表示する1行で、この作業ではオフになっていること、いつでもオンに戻せることを一度だけ伝えます。Codex ではスキルは `$aidlc` です。

スイッチの入力とみなされるもの: `/aidlc`（または `$aidlc`、`aidlc`）で始まり、先頭にフラグを持つメッセージ。たとえば `/aidlc --guard-policy relaxed`、`/aidlc --guard-policy off --guard.state-transition off`、`/aidlc --guard-policy relaxed build the auth service`（説明はフラグの後に続き、読み取られません）。同じコマンドの先頭の後に続く `config set guard-policy relaxed|off` または `config set guard.<fence> off`。必要に応じて他の設定（`--depth minimal --sensors off`）や `--intent <name>` と `--space <name>` の組を続けられます。あるいは確認の言葉 `guard policy relaxed|off` 単独。未知のフラグや無効な値は、それを名指しして行全体を拒否します。大文字小文字や末尾のピリオドは問いません。スイッチに触れただけの質問や発言はスイッチではありません。`/aidlc why was config set guard.plan-approval off suggested?` は何も変えず、説明の後に置いたフラグも同様です。

チャットからの直接の `intent create --guard-policy relaxed|off` は、値が選択したスコープの既定値より低い場合は拒否されます（`off` のスコープでの `relaxed` は引き上げなので適用されます）。作業を作成し、あなたがより低い値を求めたときにエージェントがセッターを実行します。作業が存在する前、または新しい作業とともに入力した値は、あなたが求めた作業について `set by you` として記録されます。直接の `scope change --guard-policy relaxed|off` は `config-change` と同じ引き下げのルールに従います。より低いスコープの既定値は、利用者がスコープの変更を求めた場合に適用されます。それ以外のスコープ変更では、実行中のワークフローのより厳しい値が保たれ、その旨が1行で伝えられます。`AIDLC_UNATTENDED=1` はプロンプト時の適用を抑止し、CLI による引き下げを拒否します。メモリの strict と無人実行のチェックの後、CLI のセッターが利用者のプロンプトなしに引き下げるのは、あなたが自分のターミナルで入力した場合（あなた自身の行為）か、`fenceKeyBypassed` がフィクスチャやハーネス起動時の関与バイパスを認識した場合だけで、インラインの環境変数の代入は対象外です。`AIDLC_SKIP_HUMAN_PRESENCE_GUARD=1` を付けて起動された有人のハーネスについて、セッション開始フックは Plan Approval のランタイムディレクトリに `presence-bypass-<session>` のスタンプを保持します。

モデルのツールは、フックを呼び出すことも、`aidlc/.aidlc-sessions/` や、いずれかの `.aidlc-plan-approval/` または `<record>/.aidlc-engine/gate-words/` ディレクトリに書き込むこともできません。これは [状態遷移ガード](../reference/06-hooks-and-tools.md#pretooluse-aidlc-state-transition-guardts) によって強制されます。

メモリファイルが Guard Policy の strict を保持している場合、`/aidlc config set guard.<fence> off` はそのファイルを名指しする文とともに拒否されます。リポジトリのすべての人に対して変更するには、その `Mode: strict` の行を編集します。フェンスを `on` にすることは引き続き許可されます。

メモリで保持された strict は、以前に下げたフェンスにも優先します。永続化された `Guards Off` のエントリはインテントに残りますが、`/aidlc --status` はそのフェンスを `Checks off:` の行に含めません。エントリが再び有効になるのは、メモリの行が strict を保持しなくなった後だけです。マシン全体のキルスイッチは引き続き最優先です。

1つをオフにすると、`aidlc-state.md` に `- **Guards Off**: review-freeze (set by you)` が書き込まれ、`GUARD_DISABLED` の監査の行が1つ記録されます（この行のラベルは1つです。そこにあるすべてのチェックがあなたのものである間は `set by you`、1つでもそうでなくなると `set by a command` になります）。再びオンにすると、そのリストから削除され、`GUARD_RESTORED` が書き込まれます。`on` を設定すると、ポリシーによって下げられたフェンスも引き上げられ、`- **Guards On**: review-freeze (set by you)` が記録され、`GUARD_RESTORED` が書き込まれます。これらの行が名指しするのは切り替え可能なフェンスだけです（計画の承認が独立した設定になる前に書かれた `plan-approval` のエントリも引き続き読み取られます）。永続化された人間の関与のエントリは無視されます。`/aidlc --status` は、誰かがオフにした各チェック（この作業についてのあなた、またはこのマシンの環境）とその出所を名指しする `Checks off:` の行を表示します。計画承認のフェンスはそこでは `plan re-approval`、つまり承認後に編集された計画に対するチェックとして表示されます。より低い Guard Policy によってオフになるチェックはそこには列挙されません。それらは `Guard Policy:` の行に含まれ、その行がポリシーの出所を伝えます。優先順位は、マシン全体のキルスイッチ、次にメモリが strict を保持していない場合の作業ごとのオフ、次に作業ごとのオン、次に Guard Policy の語、最後に既定のオンです。Guard Policy の語を自分で設定すると、それぞれの方向ですべてのチェックに及びます（各変更は `GUARD_DISABLED` または `GUARD_RESTORED` として書かれます）。`off` はこの作業でオンのままにしていたチェックをクリアするため、オンのものは残りません。`strict` はオフにしていたチェックをクリアするため、オフのものは残りません。`relaxed` はどちらもクリアしないため、チェックをオンやオフに戻すことは決してありません。メモリで保持された strict は引き続き優先され、その後に切り替えた個々のチェックも引き続き適用されます。

レビュアーの読み取り範囲の設定は、ディスパッチされたレビュアーの読み取り／検索の範囲を統制します。チームの1つのユニットを所有するとスタンプされたチェックアウトは、引き続き別のユニットの `construction/` サブツリーに書き込めません。その所有権の境界は、Guard Policy、作業ごとのフェンス設定、`AIDLC_DISABLE_REVIEWER_SCOPE_HOOK` のいずれでも切り替えられません。

自分の言葉で依頼すればエージェントが実行します。入力されたスイッチも同じセッターで、エージェントが最初の `next` から実行します。切り替え可能なフェンスのメインセッションでの拒否メッセージは、そのスイッチを名指しします。依頼したメッセージを指定せずに、フェンスをオフにするセッターを実行すると、次のように表示されます:

> Turning the review-freeze check off is the person's call. Run it with --message <id>, the message they asked in, when they ask for it.

（訳: レビューフリーズのチェックをオフにするかどうかは利用者が決めることです。利用者が求めたときに、--message にその依頼のメッセージの ID を指定して実行してください。）

計画の承認をオフにする CLI のセッターは、次のように表示します:

> Turning plan approval off lets code generation start without the person approving the plan, so it is their call. Run it with --message <id>, the message they asked in, when they ask for it.

（訳: 計画の承認をオフにすると、利用者が計画を承認しないままコード生成が始まるため、利用者が決めることです。利用者が求めたときに、--message にその依頼のメッセージの ID を指定して実行してください。）

依頼したメッセージを指定せずに、ポリシーを `relaxed` に変えるセッターを実行すると、次のように表示されます:

> Setting Guard Policy relaxed lowers fences, which is the person's call. Run it with --message <id>, the message they asked in, when they ask for it.

（訳: Guard Policy を relaxed に設定するとフェンスが下がるため、利用者が決めることです。利用者が求めたときに、--message にその依頼のメッセージの ID を指定して実行してください。）

スコープの既定値より低い明示的な `relaxed` フラグを付けた作成（`enterprise` 上）を、そのチャットにあなたのメッセージが記録されていない状態で実行した場合も、同じ表示になります:

> Setting Guard Policy relaxed lowers fences, which is the person's call. Run it with --message <id>, the message they asked in, when they ask for it.

他のフェンス名や `off` の場合は、対応する名前や値が使われます。無人実行では、ドライバー向けの案内が付け加えられます。メモリで保持された strict は、スイッチの適用やバイパスの確認より前に拒否し、代わりに編集すべきメモリファイルを名指しします。人間の関与による拒否はスイッチを名指ししません。利用者からの返信が記録されていないこと、そしてすでに送られた返信がどうなったか（ハーネスのフックの手順、または `/aidlc --doctor`）を伝え、再度の返信を求めることは決してありません。

人間の関与は5つの中で最も厳格です。それはあなたの承認をあなたのものにするものなので、Guard Policy も作業ごとの設定もそれを下げません。下げられるのは、マシン全体に設定されたか `aidlc config flags --bypass` で記録された `AIDLC_SKIP_HUMAN_PRESENCE_GUARD=1` だけで、それがオフの間は AI-DLC がその旨を伝えます。`AIDLC_UNATTENDED=1` は別途ヒューマンターンの発行を差し止めますが、フェンスを下げるものではありません。

### Who asked for this: the authority chain

ガードが目にするすべての操作は、何かが決まる前に分類されます。それは記録された人間のターンでカバーされているか、ワークフロー自身の指示でカバーされているか、どちらでもないか、です。この分類は誰が作業していたかを記録するもので、利用者が入力したスイッチに対してエージェントが実行するセッターとは別のものです。

- **あなたの付与。** ワークフローが最後にエージェントに指示を出した後にあなたが送ったメッセージ。ワークフローが次の指示を出すまで、そのターンのためにディスパッチされたコンダクターとエージェントを分類します。フェンスやポリシーの変更を認可するものではありません。
- **ワークフローの指示。** エンジンが現在有効にしているステージ。誰が行うかにかかわらず、その指示が求める作業をカバーし、その中でまだ保留中の承認も含みます。ループが自身のステップの1つを飛ばすことはカバーしません。それこそがフェンスが気付くものです。
- **どちらでもない。** 指示の範囲外で、それ以降あなたから何もないもの。最も狭いカバー範囲であり、読み取れないシグナルはこれにフォールバックします。

問題は誰が入力しているかではありません。開発者エージェントはコンダクターの言葉に基づいて動き、コンダクターはあなたの言葉に基づいて動くため、権限はチェーンを下っていきます。コンダクターがエージェントをディスパッチすると、その時点で有効な権限がディスパッチにスタンプされ、エージェントはそれを引き継ぎます。エージェントが自分自身に付与を発行することは決してできず、読み取れないシグナルはカバー範囲を広げるのではなく狭めます。シグナルはフレームワークがすでに保持しているものです。あなたの最後のプロンプトをワークフローの最後の前進コマンドと照らして記録する `.aidlc-engine/` 配下のターンマーカー、アクティブなディレクティブのマーカーにあるカウンター、そして実行中のエージェント台帳にあるディスパッチのスタンプです。

この分類がフェンスを下げることも、利用者が入力したスイッチの代わりになることも決してありません。その役割は証拠の記録です。下げられたフェンスが何かを通すたびに、監査の行が有効な権限を名指しするため、読み手はそれが起きたときに誰が作業していたかを確認できます。`strict` の下での変更された入力は、この分類にかかわらず、統制された境界で尋ねられます。

### What you see when a guard decides

- **道を譲る。** フェンスを下げたものを名指しする1行が表示され、作業は続きます: `Continuing past the plan re-approval check because it is off for this piece of work (guard policy relaxed (set by you)). Recorded in the audit trail: dispatch of aidlc-developer-agent`。`GUARD_STOOD_ASIDE` の行が1つ、フェンス、有効な権限、付与がどのように証明されたか、実行者がメインセッションかディスパッチされたエージェントかを記録します。"are you sure" と尋ねられることは決してありません。フェンスはすでにオフだからです。

  Claude Code では、フックは JSON の `systemMessage` を1つ出力し、Claude Code はそれをフックのメッセージとしてあなたに表示します。モデルはそれを見ず、`GUARD_STOOD_ASIDE` の行が記録となります。Codex、opencode、Kiro CLI では、プレーンなフックの行が表示されます。Kiro IDE では表示されません。IDE はフックの出力をセッション開始時とプロンプト送信時にのみエージェントに渡すため、そこでの道を譲る動作は表示されず、監査の行が唯一の記録になります。拒否は別です。フックがツール呼び出しをブロックすると、エージェントはその理由を目にします。行が書かれるのはインテントにすでに監査証跡がある場合だけなので、台帳がまだない新規プロジェクトの Kiro IDE では、道を譲っても行も記録も残りません。下げられたフェンスが何を通したかを知りたい場合は、表示された行を見たことに頼らず、インテントの `audit/` シャードにある `GUARD_STOOD_ASIDE` の行を読んでください。
- **守る。** メモリが Guard Policy の strict を保持していない場合、切り替え可能なフェンスのメインセッションでの拒否は、何が欠けているかを伝え、通り抜ける方法を提示するようエージェントに伝える1文を加えます。あなたが今それを行うつもりだったなら、エージェントはこの作業でレビューフリーズのチェックをオフにするかどうかを尋ね、あなたがそう答えるとオフにして、その旨を1行で伝えます。メモリが strict を保持している場合、拒否はスイッチを提示する代わりにメモリファイルを名指しします。人間の関与の場合は、利用者からの返信が記録されていないことを伝え、繰り返しを求めることも、スイッチを案内することも決してありません。
  計画の承認がスイッチを名指しすることは決してありません。それをオフにするのは、常にあなたの発案だけです。まだ承認していない計画や、`strict` の下で承認後に編集された計画については、`next` を実行することで尋ねられます。
  ディスパッチされたエージェントがスイッチの文を目にすることは決してありません。その拒否はメインセッションへ誘導します。
- **尋ねる。** `strict` の下では、あなたが何かを承認した後に変わった入力について、何が変わったかを名指しして一度尋ねます。

---

## Stage Customization

各ステージは、`.claude/aidlc-common/stages/[phase]/` にある自己完結した `.md` ファイルです。ステージファイルは次のものを定めます:

- **メタデータ** — ステージ番号、フェーズ、実行モード、リード／サポートエージェント
- **入力** — 読み込む先行の成果物
- **ステップ** — 番号付きの実行手順
- **出力** — 生成する成果物
- **完了** — 承認ゲートのパターン

ステージの動作を変更するには、そのステージファイルを直接編集します。すべてのステージは、共通のパターン（承認ゲート、質問の形式、状態の追跡）についてステージプロトコルを参照しています。

### Depth levels

各スコープには、成果物の詳細度を制御する既定の深さがあります:

| 深さ | 説明 |
|-------|-------------|
| **Minimal** | 簡潔な成果物、対象を絞った分析、任意の内容なし |
| **Standard** | バランスの取れた詳細度、主要および副次的な懸念事項をカバー |
| **Comprehensive** | 完全な詳細度、広範な分析、任意の内容をすべて含む |

任意の承認ゲートで別のレベルを要求することで、深さを上書きできます。

---

## Status line (Claude Code and Codex)

**Claude Code** と **Codex CLI** では、AI-DLC はターミナル下部のステータスラインに作業の状況を表示できます。これはあなたが自分で設定したステータスラインの代わりに表示されるため、使うかどうかはあなたが選びます。初回セットアップで一度尋ねられ、いつでも切り替えられます。他のハーネスにはステータスラインがなく、`/aidlc --status` と各ゲートでの進捗行でワークフローの位置を表示します。

Claude Code では、`mvp` の計画で実行した例は次のようになります（罫線文字はプレーンテキストで表示。右側にはモデル、コンテキスト、コストが続きます）:

```text
[AIDLC] inventory-api - IDEATION [##--------] 1/4 > Feasibility -- Architect Agent
```

これは順に、作業、現在のフェーズ、フェーズの進捗（バーと比率で、どちらも現在のフェーズが対象）、ステージの表示名、リードエージェントを示します。Reverse Engineering が 10 個以上のリポジトリを分析している間は、`> Reverse Engineering 37/190` のように、ステージ名の後にそのうち完了した数が続きます。作業が選択される前は `[AIDLC] ready` と表示されます。右側には Claude Code のモデル、次に `ctx:N%`（コンテキストウィンドウの使用率）が続きます。Claude の使用量台帳にデータがある場合は、入出力のトークン数と `$<usd>` が、アクティブなワークフローと現在のトランスクリプト／セッションについてのみ続きます。以前のワークフローやセッションは含まれません。`AIDLC_DISABLE_USAGE_TRACKING=1` を設定すると、使用量の追跡が完全にオフになり、このセグメントが取り除かれます。各セグメントの説明は [最初のワークフロー](02-your-first-workflow.md) にあります。

`$<usd>` の値は **公開されている定価** で計算したローカルの見積もりであり、請求額ではありません。記録されたプロバイダーが Amazon Bedrock の場合（`CLAUDE_CODE_USE_BEDROCK=1`）、Bedrock の実際の請求額は推論プロファイル、リージョン、サービス階層、および交渉済みやサブスクリプションの価格によって決まるため、この数値は請求書と一致しないことがあります。長いワークフローのトークン量の大部分はキャッシュの読み取り（割引されたキャッシュ読み取り料金で課金されます）なので、カウントと見積もりは着実に増えていきます。これは実際の使用量であり、水増しではありません。新しい使用量を独自の料金で計算するには、`AIDLC_MODEL_RATES` に料金ファイルを指定してください（[料金表と上書き](../reference/06-hooks-and-tools.md#rate-table-and-overrides) を参照）。すでに記録された合計は、計算時の料金のままです。発表、画面共有、録画の最中などに見積もりを表示したくない場合は、`AIDLC_DISABLE_USAGE_TRACKING=1` を設定してください（[トラブルシューティング](15-troubleshooting.md#statusline-shows-a-cost-segment-you-dont-want-or-usage-tracking-concerns) を参照）。

### Turning it on or off

- 初回セットアップ時（まだ AI-DLC がないプロジェクトでの `aidlc config`）に、AI-DLC は "Show AI-DLC's progress in your status line? It takes the place of your own while you work in this project."（AI-DLC の進捗をステータスラインに表示しますか？ このプロジェクトで作業している間は、あなた自身のステータスラインの代わりに表示されます。）と尋ねます。すでに自分のステータスラインがある場合の推奨回答は "No, keep mine" で、ない場合は "Yes" です。
- その後はいつでも、チャットで `/aidlc --statusline on` または `/aidlc --statusline off`（Codex では `$aidlc --statusline on|off`）、あるいはターミナルで `aidlc config --statusline on|off` を実行します。何も尋ねず、何が変わったかと元に戻す方法を1行で表示します。例:

  ```text
  $ aidlc config --statusline on
  AI-DLC's status line shows in this project; /aidlc --statusline off turns it off.
  ```
- 何も尋ねないセットアップ（`--yes`、CI、スクリプト、コピーチャネル、チームメイトのクローンでの最初のチャット）では、ステータスラインは追加されません。
- このリリースより前から AI-DLC のステータスラインを表示していたプロジェクトは、尋ねられることなくそれを保持します。それがあなた自身のステータスラインを覆っている場合は、次の更新時（またはチームメイトのクローンでの最初のチャット時）に一度だけ "AI-DLC's status line replaces yours here; /aidlc --statusline off gives yours back."（ここでは AI-DLC のステータスラインがあなたのものを置き換えています。/aidlc --statusline off で元に戻せます。）と伝えられます。

**Claude Code: あなた自身の設定。** あなたの選択は git が無視する自分の `.claude/settings.local.json` に保存されるため、チームメイトはそれぞれ自分で選び、共有の `.claude/settings.json` は何も変わりません。あなたの回答は `aidlc.settings.local.json`（これも無視対象）に保持されるため、セットアップが再度尋ねることはありません。AI-DLC があなた自身の `~/.claude/settings.json`（または `$CLAUDE_CONFIG_DIR/settings.json`）を読むのは回答を推奨するためだけで、決して書き込みません。チームが共有の `.claude/settings.json` で設定したステータスラインはそのまま残ります。組織の管理された Claude Code 設定がステータスラインを指定している場合は、あなたが何を選んでもそちらが表示されます。

AI-DLC の値は次のとおりです:

```json
"statusLine": {
  "type": "command",
  "command": "bun \"$CLAUDE_PROJECT_DIR/.claude/tools/aidlc.ts\" engine statusline"
}
```

（ネイティブインストールでは `aidlc engine statusline`）。これは `.claude/tools/data/harness.json` に同梱され、オンにしたときにだけ `.claude/settings.local.json` に書き込まれます。

**Codex CLI: プロジェクトの設定。** Codex にはプロジェクト内に個人ごとの設定ファイルがないため、Codex では選択はプロジェクトのものになります。`on` は、共有の `.codex/config.toml` の `[tui]` テーブルに AI-DLC の `status_line` の行を書き込み、プロジェクトで作業するすべての人に適用されます。`off` はその1行を取り除きます。ファイル内のそれ以外は何も変わらず、あなた自身の `~/.codex/config.toml` は回答を推奨するために読まれるだけで、決して書き込まれません。代わりに、各自が Codex の `/statusline` で自分の Codex ステータスラインにタスクの進捗を追加することもできます（AI-DLC が最新に保つのは `task-progress` の項目です）。AI-DLC が書き込む行は次のとおりです:

```toml
[tui]
status_line = ["model-with-reasoning", "git-branch", "task-progress", "context-used"]
```

### Customizing the format

`.claude/hooks/aidlc-statusline.ts` を直接編集します。ステータスラインはネイティブインストールでもプロジェクトのコピーを実行します。出力形式はファイル末尾付近の `main()` 関数で定義されています。フックは `aidlc-state.md` からフェーズ、ステージ、エージェントを読み取り、ステージの slug を表示名に対応付け、同じフェーズ内のチェックボックスの解析結果から Unicode のプログレスバーと `n/m` の比率の両方を組み立てます。

このファイルは AI-DLC のものなので、プロジェクトで次に `aidlc config` を実行すると、置き換えずにあなたの編集の箇所で止まります:

```text
$ aidlc config --harness claude --yes
error: 1 config conflict(s): .claude/hooks/aidlc-statusline.ts (locally modified or unowned)
fix: to keep your version, move .claude/hooks/aidlc-statusline.ts somewhere else and run the same command again; to take the shipped version over it, run it again with --force.
```

`--force` で同梱版を取り込む前に、自分のバージョンのコピーを保存しておいてください。

---

## Tool Permissions

`.claude/settings.json` の `permissions.allow` リストは Claude Code のツールを事前承認し、ワークフローが呼び出しごとの権限プロンプトなしに実行されるようにします。ネイティブインストールでセットアップしたプロジェクトでは、次のように始まります（全部で 45 エントリ）:

```json
"permissions": {
  "allow": [
    "Edit(/**)",
    "Task",
    "WebSearch",
    "Bash(aidlc engine *)",
    "Bash(aidlc doctor)",
    "Bash(aidlc doctor --verbose)",
    "Bash(aidlc version)",
    "Bash(aidlc --doctor)",
    "..."
  ]
}
```

Claude Code 2.1.296 は、フォルダーを信頼するかどうかを最初に尋ねるときにこれらを列挙します（省略あり。罫線文字はプレーンテキストで表示）:

```text
 Accessing workspace:
 /home/you/inventory-api
 ...
 ! This folder pre-approves 45 tool permissions in .claude/settings.json:
   WebSearch, Edit(/**), Bash(aidlc engine *), Bash(aidlc doctor), Bash(aidlc doctor --verbose),
 Bash(aidlc version), Bash(aidlc --doctor), Bash(aidlc --doctor --verbose), and 37 more
 These will apply without asking. Only proceed if you trust this configuration.
 Security guide
 > No, exit
   Yes, I trust this folder
```

`Edit(/**)` は、プロジェクト内のどこでもファイルの作成と変更をカバーします。プロジェクト設定では、Claude Code は先頭の `/` をプロジェクトルートに固定します。プロジェクト外への書き込みは、Claude Code の既定と同様に確認を求めます。プロジェクト内の読み取りと検索にはエントリは不要です。コピーチャネルが事前承認するのは AI-DLC 自身のワークフローコマンドだけで、それぞれ AI-DLC が実行する形で列挙されています。エンジンコマンド、`doctor`、`version`、`--doctor`、`status`、読み取り専用の `config <section> --show --json` と `--help` の形式、そして `aidlc-*.ts` ツールです。ネイティブリリースはこれらを `Bash(aidlc engine *)` に書き換えます。`config` による変更、マシンの AI-DLC インストールを変更するコマンド（`use`、`update`、`rollback`、`uninstall`、`system`）、およびその背後のツールスクリプトはどのエントリにも一致しないため、Claude Code 自身のプロンプトで承認します。素の `Bash` はありません。Claude Code は複合コマンドのサブコマンドをそれぞれ個別に照合し、既知の安全な環境変数の固定セットだけを取り除くため、エンジンコマンドが事前承認されたままになるのは、それ単体で実行された場合だけです。`cd ... &&` の接頭辞、絶対パスの `$CLAUDE_PROJECT_DIR`、`VAR=1` の接頭辞、`jq` へのパイプ、`$(...)` による取り込みは、いずれもプロンプトを表示します。プロジェクト自身のビルドやテストのコマンドはリストの外にあり、一度だけプロンプトが表示されます。"Yes, and don't ask again" と答えると、それらのルールが `.claude/settings.local.json` に保存されます。

### How permissions work

- **上限ではなく事前承認**: `settings.json` の allow リストは、確認なしに実行されるものを決めます。リストにないツールやコマンドも、Claude Code のプロンプトを承認すれば実行されます
- **Claude Code のエージェントは既定でセッションのツールセット全体を引き継ぎます**。このハーネスでは `disallowedTools: Task` がサブエージェントの入れ子の起動をブロックします
- **任意のエージェントごとの絞り込み**: エージェントの frontmatter に `tools:` の allowlist を追加すると、そのエージェントを絞り込めます。省略するとすべてを引き継ぎます。`tools:` を列挙すると、完全修飾の `mcp__<server>__<tool>` の ID も列挙しない限り、引き継がれた MCP ツールは除外されます

### Expanding permissions

追加の機能を必要とするカスタムステージを作成する場合にのみ、allow リストにツールを追加してください。

### Narrowing permissions

使用のたびに手動の承認を求めるには、allow リストからツールを削除します。`Task` を削除すると、ディスパッチされる4つのステージ（2.1 Reverse Engineering のパイプライン、2.2 Practices Discovery のサブエージェント、2.4 User Stories のモブ、3.5 Code Generation のサブエージェント）が委任のたびに権限を求めるようになることに注意してください。ワークスペースの検出（0.2）は `aidlc-utility intent-create` の内部で決定論的に実行されるため、`Task` は使いません。

---

## Extending AI-DLC

上記の設定、スコープ、深さ、ステージの編集は、実行するワークフローの日々の調整をカバーします。ステージの追加、エージェントの追加、スコープの定義、恒常的なルールの教え込み、決定論的なチェックの組み込み、ドメインナレッジの追加など、チームのためにフレームワーク自体を作り変えたい場合、それは独自のガイドを持つ別の仕事です: **[ハーネスエンジニアガイド](../harness-engineering/00-overview.md)**。

境界線はデータかコードかです。そのガイドのすべては、YAML frontmatter 付きの Markdown ファイルか、フレームワークが読み取る JSON の設定であり、TypeScript の編集はありません。拡張の種類ごとの参照先:

| やりたいこと | 参照先 |
|--------------|----------|
| ステージの処理内容を編集する、または新しいステージを追加する | [ステージの構造](../harness-engineering/01-anatomy-of-a-stage.md), [ステージの追加](../harness-engineering/02-adding-a-stage.md) |
| エージェントを追加または変更する | [エージェントの追加](../harness-engineering/03-adding-an-agent.md) |
| スコープを定義または調整する | [スコープ](../harness-engineering/04-scopes.md) |
| 恒常的なルールを教える、または学習ループを運用する | [ルールと学習ループ](../harness-engineering/05-rules-and-the-loop.md) |
| 決定論的なチェック（センサー）をステージに組み込む | [センサー](../harness-engineering/06-sensors.md) |
| チームのドメインナレッジを追加する | [チームのナレッジ](../harness-engineering/07-team-knowledge.md) |

変更がフレームワークの*コード*（オーケストレーター、フック、CLI ツール、コンパイルパイプライン）に対するものであれば、[開発者リファレンス](../reference/00-overview.md) を参照してください。

---

## Knowledge and Rules

2層のナレッジシステムと、ルール／学習ループのシステムの詳細については、次を参照してください:

- [ナレッジ](08-knowledge.md) — チームのナレッジディレクトリと方法論の参照ファイル
- [ルールと学習ループ](09-rules-and-the-learning-loop.md) — 振る舞いのルールと自己学習の流れ

---

## Next Steps

- [スコープ、深さ、テスト戦略](05-scopes-and-depth.md) — スコープとステージの完全な対応表
- [エージェント](06-agents.md) — エージェントの権限と機能
- [トラブルシューティング](15-troubleshooting.md) — ステータスラインの問題、フックの構成
- [用語集](glossary.md) — スコープ、深さ、ガードレール、ナレッジの定義
