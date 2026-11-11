# Skills and Runner Commands

**AI-DLC はコマンドのファミリーです。** `/aidlc` オーケストレーターに加えて、1 語で入力できるランナーコマンドのセットが用意されています。スコープごとに 1 つ、ステージごとに 1 つ、そしてセットアップ用に 1 つです。これらはオーケストレーターがすでに公開している機能の一部への便利な入り口なので、`/aidlc` だけでフレームワーク全体を利用することも、フラグを省いて目的の入り口を直接入力することもできます。

> **ハーネスに関する注記。** この章では Claude Code の操作面を使います — スキルは
> `.claude/skills/` 配下にあり、ピッカーから先頭の `/` を付けて入力します。Kiro は同じ
> ランナーのセットを `.kiro/skills/` 配下に提供します（こちらも `/` で入力）。Codex はそれらを
> `.agents/skills/` に配置し、`$` を付けて入力します（`$aidlc-bugfix`）。ランナーの*セット*と
> それぞれの動作はハーネス間で同一で、異なるのはディレクトリとプレフィックスだけです。
> [他のハーネスでの実行](harnesses/README.md)を参照してください。

---

## Many skills, one engine

この実装が提供するコマンドはすべて、`.claude/skills/` 配下のスキルです。いずれも同じ決定論的なエンジンを動かし、違いは開始前に何を組み込んでいるかだけです:

- **`/aidlc`** - 完全なオーケストレーターです。フラグは何も組み込まれていません。やりたいことを説明すると、エージェントがそのための計画を提案し、あなたが確認すると、計画内のすべてのステージを完了まで進めます。最もよく使うのはこれです。
- **スコープランナー** - `/aidlc-bugfix`、`/aidlc-express`、`/aidlc-feature`、`/aidlc-mvp`、`/aidlc-security-patch`。同じ完全なワークフローを、スコープを固定し、計画の提案なしで実行します。
- **ステージランナー** - `/aidlc-domain-design`、`/aidlc-code-generation` ほか 28 個。1 つのステージを単独で実行し、メインのワークフローには一切触れません。プラグインが所有するステージは、`/test-pro-integration` のように、プラグインのプレフィックスが付いたそのままのコマンド名を使います。
- **`/aidlc-init`** - 作業を開始します。`--scope <name>` と説明を指定すると、1 ステップでインテントを作成し（Initialization フェーズ全体を実行し）、停止します。`--scope <name>` だけを指定すると、まだ何も作成しません。作業の目的を尋ね、その回答で作業を開始します。説明だけを指定すると、まず `/aidlc` と同じ計画の提案を表示し、その後は `/aidlc` と同じように続行します。`/aidlc` と異なり、スコープ名で始まる説明（"feature flags for billing"）も説明として読み取られます。すでに進行中の作業はそのまま残され、新しい作業はその横で開始されます。
- **セッションスキル** — `/aidlc-session-cost`、`/aidlc-replay`、`/aidlc-outcomes-pack`。ワークフローに対する読み取り専用のビューです。[セッション管理](11-session-management.md)で説明しています。
- **`/aidlc-compose`** - 適応型コンポーザーです。タスクを読み取り、それに合うステージを提案し、承認後はそれを独自の計画として実行します。`/aidlc compose` と同じです。[適応型コンポーザー](05-scopes-and-depth.md#the-adaptive-composer)を参照してください。
- **`/aidlc-knowledge`** - DocumentKB です。チーム独自のドキュメント（PDF、Word ファイル、Markdown、プレーンテキスト）を、エージェントが引用できるスペースごとのカタログにインデックスします。セッションスキルと同様に単独で動作しますが、読み書き可能です。カタログを変更し、ドキュメントの監査イベントを出力します（ワークフローの状態は決して変更しません）。`/aidlc knowledge <verb>` と同じ操作面です。動詞については [CLI コマンド](12-cli-commands.md)を参照してください。
- **`/aidlc-issue-report`** - 実行中に問題が発生したものの、何が悪かったのかをうまく説明できない場合にも使えます。AI-DLC と AI ツールがこのマシンに残したもの（監査記録のすべての行、状態、フックの健全性、doctor の検出結果、このプロジェクトに関するツール自身のチャットログのイベント）を 1 つのフォルダーに集め、分析を行う AI-DLC チームに送れるようにします。あなたの名前、会社やプロジェクトの詳細、あなたやエージェントが書いた内容はラベルに置き換えられます（同じものが繰り返し現れても同じラベルが保たれるため、チームはそれを把握できます）。何かを診断することはありません。引き渡す前にフォルダーを確認し（人物、会社、マシンを特定できそうな値が残っていれば拒否します）、カバーを一度読んで送ってよいと伝えるよう求め、自分から何かを送信することは決してありません。Codex では `$aidlc-issue-report` と入力します。それ以外の環境では、単に issue レポートを依頼することもできます。[トラブルシューティング](15-troubleshooting.md)を参照してください。

ランナーが行うことはすべて、`/aidlc` にフラグを付ければ実行できます。ランナーはパッケージングにすぎません — `/aidlc-bugfix` と入力でき、`/` メニューに表示されるのは使い勝手が良いというだけで、それ以上のものではありません。ランナーをすべて削除するとショートカットはなくなりますが、機能は残り、`/aidlc` のフラグで利用できます。

生成されたランナーは、ホストが許す限り**明示的な呼び出し専用**です。Claude Code、Cursor、GitHub Copilot（VS Code と CLI）では、各ランナーに `disable-model-invocation: true` が付いています（Codex では、その `agents/openai.yaml` 内の `allow_implicit_invocation: false`）。そのため、エージェントが自分からランナーを開始することはなく、その説明はモデルが毎ターン読むスキル一覧に含まれません。ランナーは入力することで開始します。ヘッドレスの `copilot -p "/aidlc-bugfix ..."` でもランナーは実行されます。Copilot は入力された行をテキストとしてエージェントに渡し、同梱の `AGENTS.md` がそのランナーのファイルを読むようエージェントに指示するためです。Kiro CLI、Kiro IDE、opencode にはこのような設定がないため、これらの環境ではエージェントが自らランナーを開始することがあります。`/aidlc` はランナーではなく、エージェントからも引き続き利用できます。

---

## Scope-runners — a named door per problem class

スコープランナーは、1 つのスコープを固定して完全なワークフローを実行します。どのような種類の作業を行うかがすでに分かっていて、計画の提案を省略したいときに使います。

```
/aidlc-bugfix          Fix a specific bug
/aidlc-express         Lightest run: requirements to deploy, no design pass, no reviewers
/aidlc-feature         Full lifecycle for new features, practical depth
/aidlc-mvp             Skip operations, ship the core
/aidlc-security-patch  CVE response
```

それぞれ、オーケストレーターに `--scope` を渡すのと同じです:

```
/aidlc-bugfix          ==  /aidlc --scope bugfix
/aidlc-express         ==  /aidlc --scope express
/aidlc-feature         ==  /aidlc --scope feature
```

`/aidlc` に渡すのとまったく同じように、説明やフラグをそのまま渡せます:

```
/aidlc-bugfix The profile API returns 500 when display_name is null
/aidlc-feature --status
```

**ランナーが用意されているのは 5 つのコアスコープです** — スコープファイルで `runner: true` が付いている、よく使われるものです。フレームワークは全部で 11 のスコープを定義しています（[スコープ、深さ、テスト戦略](05-scopes-and-depth.md)を参照）。それ以外のスコープ — `enterprise`、`poc`、`infra`、`refactor`、`classic`、`workshop` — は、常にオーケストレーター経由で利用できます。プラグインが所有するスコープも `runner: true` を設定できます。そのランナーは、`/test-pro-validation` のように、プラグインのプレフィックスが付いたそのままのスコープ名を使います。

```
/aidlc --scope enterprise
/aidlc --scope poc
```

ワークフローが開始されると、そのスコープは `aidlc-state.md` に固定されるため、同じランナーを再実行するとワークフローを最初からやり直すのではなく再開します。別のスコープで実行するには、`/aidlc --scope <name>` を使います。

---

## Stage-runners — run one stage, leave your workflow alone

ステージランナーは、**1 つのステージを単独で**実行します。そのステージに設定されたエージェントとレビュアーを使い、合成の完了を記録し、ワークフローの学びも承認ゲートもなしに停止します。メインのワークフローの `Current Stage` を進めることは決してありません。この分離はツール自体が強制します。

```
/aidlc-domain-design
/aidlc-code-generation
/aidlc-requirements-analysis
/aidlc-reverse-engineering
```

それぞれが `/aidlc --stage <slug> --single` をパッケージ化したものです:

```
/aidlc-code-generation    ==  /aidlc --stage code-generation --single
```

### When you'd use one

- **ワークフローに乗らずに、方法論の一部だけを適用する。** ある問題について要件分析をしたいが、ライフサイクル全体を進める準備はできていない。`/aidlc-requirements-analysis` を実行し、成果物を受け取ったら終わりです。
- **あなた自身がオーケストレーターになる。** 作業の順序を手作業で決めていて、目の前のステージだけをフレームワークに実行させたい場合です — 人間が主導し、フレームワークは方法論の 1 ステージ分を提供します。
- **ステージを単独で再実行する。** メインのワークフローは別の地点で止めたままにしておけます。この実行はワークフローを動かしません。そこにすでにあるファイルのコピーを保存したうえで、作業がドキュメントを保管する場所にそのステージのドキュメントを書き込みます。

### Why it's safe

`--single` の不変条件はツールによって強制されます。`next --single` は作業開始前に
分離された開始を記録し、`report --single` はその同じ合成の試行だけを閉じることができ、
開始のない直接の完了は拒否します。どちらの操作も、
メインのワークフローの `Current Stage` には書き込みません。ランナーがメインの
ポインターを進めようとした場合、エンジンは代わりにエラーを返します。作業の記録に
そのステージのファイルがすでにある場合、`next --single` は実行開始前にそれらを
その記録内の `archive/<time>-<stage>/` にコピーし、1 行で
そのフォルダーを示します。

ブートストラップ用の 3 つの **initialization** ステージにはステージランナーがありません — インテントを中途半端に作成しても単独では意味がないためです。代わりに、initialization フェーズ全体が 1 つのコマンドとしてパッケージ化されています:

```
/aidlc-init --scope <name> <description>   create the intent in one step, then stop
/aidlc-init --scope <name>                 ask what the work is for, then start it
/aidlc-init <description>                  plan offer first, then continue (a leading scope name is part of the description)
```

---

## The runner families at a glance

| ファミリー | 例 | 動作 | 同等のオーケストレーター操作 |
|---|---|---|---|
| オーケストレーター | `/aidlc` | 完全なワークフロー。計画を提案し、あなたが確認する | - |
| スコープランナー | `/aidlc-bugfix`, `/aidlc-express`, `/aidlc-feature`, `/aidlc-mvp`, `/aidlc-security-patch` | 完全なワークフロー。スコープは固定で、確認する計画はない | `/aidlc --scope <name>` |
| ステージランナー | `/aidlc-domain-design`, `/aidlc-code-generation`, ...（全 30 個） | 1 つのステージを単独で実行し、ワークフローを決して進めない | `/aidlc --stage <slug> --single` |
| Init ラッパー | `/aidlc-init` | 作業を開始する。`--scope` と説明があれば 1 ステップで作成し、`--scope` だけなら先に目的を尋ね、説明だけなら先に計画を提案する | `/aidlc --scope <name>`、または新しいワークスペースでの `/aidlc <description>`（先頭のスコープ名を除く） |
| セッションビュー | `/aidlc-session-cost`, `/aidlc-replay`, `/aidlc-outcomes-pack` | 読み取り専用のワークフローレポート | [セッション管理](11-session-management.md)を参照 |
| コンポーザー | `/aidlc-compose` | タスクに合わせた計画を提案し、承認後に実行する | `/aidlc compose` |
| ドキュメントナレッジ | `/aidlc-knowledge` | チーム独自のドキュメントをインデックスし、読み取る（スペースごとの DocumentKB） | `/aidlc knowledge <verb>` |
| Issue レポート | `/aidlc-issue-report` | 機密情報を取り除いた実行記録を、AI-DLC チームに送るためにまとめる | `/aidlc --doctor --export` を基盤とする |

ライフサイクル内の実行可能なステージごとに、ステージランナーが 1 つあります。全体を確認するには、スキルディレクトリを一覧表示してください。`aidlc config` で Claude Code 用にセットアップしたプロジェクトでは次のようになります（一部省略）:

```text
$ ls .claude/skills/
aidlc
aidlc-approval-handoff
aidlc-bugfix
aidlc-build-and-test
aidlc-ci-pipeline
aidlc-code-generation
aidlc-compose
...
aidlc-session-cost
aidlc-team-formation
aidlc-units-generation
aidlc-user-stories
```

合計 43 エントリーです: `/aidlc`、30 個のステージランナー、5 個のスコープランナー、`/aidlc-init`、`/aidlc-compose`、`/aidlc-knowledge`、`/aidlc-issue-report`、3 つのセッションスキル。

---

## Author your own runner — write a stage file

フレームワークをカスタマイズする場合に重要なのはここです: **ランナーを手で書く必要はありません。** ランナーは、コンパイルされたステージグラフとスコープファイルから生成されます。

ステージランナーを追加するには、ステージを追加します。ステージファイルを書き、グラフを再コンパイルし、再生成します:

```bash
aidlc engine gen runners
```

ジェネレーターはコンパイルされたステージ一覧（唯一の信頼できる情報源）を読み取り、実行可能なステージごとにランナーの殻を出力します。新しいステージの `/aidlc-<your-stage>` コマンドは自動的に現れます — 書くべきランナーファイルも、コピーすべき定型コードもありません。スコープランナーも同じ仕組みで、frontmatter に `runner: true` を宣言したスコープに対して生成されます。`scopes --all` はすべてのスコープファイルに対してランナーを出力します。

```bash
aidlc engine gen runner-scopes      # generate scope-runners
```

ランナーのセットは手作業で保守するのではなく導出されるため、対象のステージやスコープとずれることがありません。ディスク上のセットが信頼できる情報源から乖離した時点で、2 つのチェックが CI を失敗させます:

```bash
aidlc engine gen runners --check             # stage-runner drift
aidlc engine gen runner-scopes --check       # scope-runner drift
```

何もずれていないプロジェクトでは、ステージランナーのチェックは次のように出力します:

```text
$ aidlc engine gen runners --check
stage-runner set is in sync with the compiled stage graph (30 runners).
```

ランナーを再生成せずにグラフにステージを追加した場合や、なくなったステージの孤立したランナーが残っている場合は、差分とともに明確に失敗します。ステージファイルを追加して再生成することが作成手順のすべてです。ランナーはその結果として生まれ、ジェネレーターが保守してくれます。

ステージファイルの書き方については、[カスタマイズ](13-customization.md)と[フェーズとステージ](04-phases-and-stages.md)を参照してください。エンジン、ディレクティブの契約、ランナーの殻が内部で `next`/`report` をどう動かすかについては、リファレンスの [スキルシステム](../reference/17-skill-system.md) の章を参照してください。

---

## Quick reference

```
# Full workflow
/aidlc                              propose a plan, run everything
/aidlc --scope enterprise           any of the 11 scopes

# Scope-runners (the 5 high-traffic doors)
/aidlc-bugfix · /aidlc-express · /aidlc-feature · /aidlc-mvp · /aidlc-security-patch

# One stage, isolated (never advances your workflow)
/aidlc-code-generation              == /aidlc --stage code-generation --single

# Start a piece of work (Initialization phase)
/aidlc-init --scope <name> <desc>   create it in one step, then stop
/aidlc-init --scope <name>          ask what the work is for, then start it
/aidlc-init <description>           plan offer first, then continue

# Add your own: write a stage/scope file, then
aidlc engine gen runners
aidlc engine gen runner-scopes
```

関連項目: [CLI コマンド](12-cli-commands.md) · [スコープ、深さ、テスト戦略](05-scopes-and-depth.md) · [カスタマイズ](13-customization.md)
