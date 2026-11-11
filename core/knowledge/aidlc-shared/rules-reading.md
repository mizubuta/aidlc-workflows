# Reading Active-Space Rule Files

> **対象読者**: `aidlc/spaces/<active-space>/memory/` からチームが承認したプラクティスを
> 読む必要があるすべてのエージェント。
> **このファイルの所有者**: フレームワーク。
> `aidlc-pipeline-deploy-agent/branching-strategies.md` と、プラクティスを考慮した振る舞いを
> 採用するほかのエージェントから参照される。

ルールの名前空間は、ワークフロー開始時に厳密加算型の 5 層チェーン
`org → team → project → phase → stage` を通じて解決される。最初の 3 つの
ファイルは `aidlc/spaces/<active-space>/memory/{org,team,project}.md` である。
`phases/<phase>.md` は、ステージの frontmatter の `phase: <name>` が
それを選択することで付加される。ステージのルールは将来の利用のために予約されている。コンパイル時に、
解決済みのチェーンが各ステージノードの `rules_in_context` に焼き込まれる。適用可能な
ルールはすべてチェーンに現れ、実行時に脱落するものはない。ステージ開始時に、
エンジンは実質的な内容を持つ各ルールファイルを、`run-stage` の前に、
上限付きの `load-steering` ディレクティブの順序付きシーケンスとして配信する（空のテンプレートは
§ 1 に従って除外される。サイズに基づくパスのフォールバックはない）。このファイルは、
エージェントがその配信シーケンスの外でプラクティスを必要とするとき（ツールの形成、質問プロンプト）に、
それらの層を安全に読む方法を説明する。

---

## 1. Empty-template detection

ルールのセクションは、本文の空白でないすべての行が `<!--` で始まるか空白のみである場合に
**空** とみなす。テンプレートには、承認済み状態の例を示す HTML コメントの
プレースホルダが同梱されている。チームがプラクティスを承認するまでは、
このルールにより、すべてのセクションが空となる。

セクションを読んで空だとわかった場合は、次の層にフォールバックする
（§ 3 を参照）。例示の散文を解析してはならない。コメントは人間の読者のために存在し、
エージェントの推論のためのものではない。

空のセクションの例:

```
## Way of Working

<!-- Populated by practices-discovery affirmation. Example after affirmation:
"We squash-merge to `main`. Trunk-based; feature branches resolve in 1-2 days." -->
```

内容が記入されたセクションの例:

```
## Way of Working

We squash-merge to `main`. Trunk-based; feature branches resolve in 1-2 days.
```

本文の 1 行目が `<!--` であることが空であることのシグナルであり、本文の 2 行目が
散文であることが記入済みであることのシグナルである。

---

## 2. Semantic-topic matching

見出しの形は、`team.md`、`org.md`、`project.md`、そしてそれらを利用する
エージェントごとの KB の間でずれることがある。見出しの文字列の完全一致ではなく、
**トピック** で照合する。

**way of working / branching / merge** のトピックの場合:
- 完全一致の `## Way of Working` 見出しを優先する。
- なければ、`branch`、`merge`、`way` のいずれかを含む任意の `## ` 見出しにフォールバックする
  （大文字小文字を区別しない）。

**walking skeleton** のトピックの場合:
- `## Walking Skeleton` を優先する。
- なければ、`skeleton` を含む任意の見出しにフォールバックする（大文字小文字を区別しない）。

**testing** のトピックの場合:
- `## Testing Posture` を優先する。
- なければ、`test` を含む任意の見出しにフォールバックする（大文字小文字を区別しない）。
- Code Generation では、明示的な `Methodology` と `Ordering` の
  記述を、カバレッジ、ツール、統合、スコープに関する
  注記とは独立して解決する。ケイデンスについて何も述べていない狭い層の注記は加算的なままであり、
  広い層で承認された手法を消すことはない。狭い層の手法が矛盾する場合は、
  厳密加算モデルのもとではエラーとなる。

**deployment** のトピックの場合:
- `## Deployment` を優先する。
- なければ、`deploy` または `release` を含む任意の見出しにフォールバックする
  （大文字小文字を区別しない）。

**code style** のトピックの場合:
- `## Code Style` を優先する。
- なければ、`style` または `format` を含む任意の見出しにフォールバックする
  （大文字小文字を区別しない）。

フォールバックで複数の候補見出しが一致した場合は、文書順で最初に
現れるものを優先する。

---

## 3. Fallback chain

1 つの有効なプラクティス記述を必要とする判断では、アクティブなスペースの
記述的な層を狭いものから広いものへ順に調べ、空でなく矛盾のない
最初のセクションを返す:

1. **`aidlc/spaces/<active-space>/memory/project.md`** — プロジェクト固有の
   特化。
2. **`aidlc/spaces/<active-space>/memory/team.md`** — チームが承認したプラクティス。
3. **`aidlc/spaces/<active-space>/memory/org.md`** — チームの口調で書かれた
   フレームワークのデフォルト。
4. **ハードコードされたデフォルト** — 3 つの層がすべて空の場合にのみ使う
   （practices-discovery が実行される前のグリーンフィールドの初回実行時、または
   プロジェクトが practices-discovery を SKIP した場合）。

このトピック選択は、より広いルールを消すものではない。ランタイムは引き続き、適用可能な
すべての層を読み込む。より広いポリシーと矛盾する狭い層の記述は、上書きではなく
受け入れ時のエラーである。

ハードコードされたデフォルトは次のとおり:

| トピック | デフォルト |
|---|---|
| Way of Working | トランクベース開発。ベースは `main`、ターゲットは `main`。squash-merge |
| Walking Skeleton | スコープに依存する。アクティブなスコープファイルの `skeleton:` フィールドが、デフォルトのセレモニーの方針を与える |
| Testing Posture | 手法が承認されていない場合はテストを後に書く順序。量はテスト戦略の軸が決める |
| Deployment | トランクベースで、マージ時に staging へデプロイする。production のゲートは人間が承認する |
| Code Style | プロジェクトの linter/formatter の構成に従う |

フォールバックチェーンが層 4 まで下りる必要がある場合は、
`PRACTICES_SECTION_EMPTY`（助言のみ）を出力する。これにより、doctor と下流の
可観測性が、承認済みのチームプラクティスではなくフレームワークのデフォルトで動いている
プロジェクトを検出できる。

---

## 4. Read protocol (pseudocode)

```
def read_practice(topic):
  for layer in [
    aidlc/spaces/<active-space>/memory/project.md,
    aidlc/spaces/<active-space>/memory/team.md,
    aidlc/spaces/<active-space>/memory/org.md,
  ]:
    section = match_section(layer, topic)  # § 2
    if section and not is_empty(section):  # § 1
      return section
  return hardcoded_default(topic)          # § 3
```

このプロトコルは意図的に同期的で、副作用を持たない。エージェントは
ツールの呼び出しを形成するときにこれを呼び出し、オーケストレーターは
質問プロンプトを形成するときにこれを呼び出す。どちらも同じフォールバックチェーンを使うため、
ユーザーに見える振る舞いはディスパッチ全体で一貫する。

---

## 5. Worked example — aidlc-pipeline-deploy-agent reading way-of-working

オーケストレーターは、Bolt の作成時に `aidlc-pipeline-deploy-agent` をディスパッチする。
このエージェントの役割は、チームの意図を `aidlc-worktree create --slug
<slug> --base <branch>` に対応づけることである。エージェントは次のように読む:

1. `project.md` の `## Way of Working` → 空（新規のテンプレート）。
2. `team.md` の `## Way of Working` → 空（新規のテンプレート）。
3. `org.md` の `## Way of Working` → "trunk-based; base `main`, target
   `main`; squash-merge"。
4. エージェントによる `aidlc-worktree` の呼び出しとあわせて、
   `{base: "main", strategy: "squash"}` をオーケストレーターに返す。

もし `team.md` の `## Way of Working` が "We use GitFlow with
`develop` as the integration branch"（統合ブランチとして develop を使う GitFlow を採用している）と書かれていれば、エージェントは代わりにそれを
`--base develop` に対応づける。フォールバックチェーンは同じで、記入済みの層が優先される。

3 つの層がすべて空だった場合（practices-discovery を SKIP したプロジェクト）、
エージェントは `PRACTICES_SECTION_EMPTY` を出力し、ハードコードされた
デフォルト `{base: "main", strategy: "squash"}` を適用する。doctor がこれを表面化するので、
チームは自分たちのプラクティスを記録したければ、後から practices-discovery を実行できる。
