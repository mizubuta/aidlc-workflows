# `aidlc-worktree info` and `list` — Output Schema

`info` と `list` の固定された出力コントラクト、および Bolt の識別とリカバリのルール。オーケストレーターの `SKILL.md` にある halt-and-ask の文章は、`info` を読み込んで、code-generation 失敗時の halt-and-ask 用の構造化質問のプロンプト本文にワークツリーのパスとブランチ名を埋め込む。

このスキーマは、ツール（決定論的）と LLM（文章の作成）の間のコントラクトである。今後 JSON の形を変更する場合は、同じコミットでこのファイルも更新しなければならない。

## Bolt identity

コミットされた監査シャードはリポジトリのコンテンツであるため、`info` はそれらを信頼できないものとして扱い、プロトコルがその出力をプロンプトに埋め込む前に Bolt の識別情報を検証する（[#1281](https://github.com/awslabs/aidlc-workflows/issues/1281)）。

新しい Bolt は、選択されたインテントのレジストリ UUID にスコープされる。`<id8>` は
その 8 文字の小文字 16 進数のサフィックス（`idSuffix(uuid)`）であり、`refs/heads/claim/<id8>/<unit>` における
Unit の claim と同じ識別情報である。Unit 名、`--slug`、監査の
`Bolt slug`、状態の `Bolt Refs`、`Unit Progress` は変わらない。

| 対象 | 新しい形 |
|---------|-----------|
| ワークツリーのディレクトリ | `<projectDir>/.aidlc/worktrees/bolt-<id8>_<slug>` |
| ブランチ（ディレクトリのベース名でもある） | `bolt-<id8>_<slug>` |
| 保持されるレビュー済みソース | `refs/aidlc/reviewed-source/<id8>/<slug>/<commit>` |
| 退避されたリカバリ用 ref | `refs/aidlc/parked/<id8>/<slug>/<stamp>/…` |

`_` 区切りは Bolt slug（`[a-z][a-z0-9-]*`）には現れないため、新しい名前が
レガシーの名前と取り違えられることはない: `bolt-abcdef01-api` はレガシーの slug であり、
`bolt-abcdef01_api` はインテントの識別情報 `abcdef01` を持つ。`bolt-` プレフィックスは
引き続き `bolt-*` のブランチルールと互換である。これにより、並行するインテント内の同名の Unit が、
1 つのチェックアウト内や 1 つのクローンのワークツリー間で衝突することを防ぐ
（[#1252](https://github.com/awslabs/aidlc-workflows/issues/1252)）。

アップグレード前のレガシーな `bolt-<slug>` の Bolt は、そのディレクトリ、ブランチ、およびレガシーな
`refs/aidlc/reviewed-source/<slug>/<commit>` と
`refs/aidlc/parked/<slug>/<stamp>/…` の名前空間を、merge、discard、purge を経て
完了するまで維持する。古い形で新しい Bolt が作られることはない。レガシーの解決には、
選択されたインテント自身の、その slug に対する `WORKTREE_CREATED`、`WORKTREE_MERGED`、
`WORKTREE_DISCARDED` の行を使う。それらの因果的フロンティアにはちょうど
1 行が含まれなければならず、その順序はシャードのファイル名ではなく、タイムスタンプと同一シャード内の追記順で決まる。
読み取れないシャードや曖昧なフロンティアは、名前空間付きの識別情報に解決される。
その行はレガシーの `Worktree path` と `Bolt slug` を指定していなければならず、
`WORKTREE_CREATED` であればレガシーの `Branch name` も指定していなければならない。

レガシーのディレクトリが存在する場合、読み取り可能な `worktree-meta.json` が
監査を裏付けなければならない。選択されたインテントと一致する `intentRecord` があれば、これら 3 つの
フロンティアイベントのいずれでも許可される: merge/discard の行は意図の監査であり、クリーンアップが
完了した証拠ではない。`intentRecord` を持たない P7 以前のメタデータは、開いている作成
（フロンティア上の `WORKTREE_CREATED`）を必要とする。後の merge/discard の行は、
再利用されたレガシー名に属する可能性がある。メタデータが欠落しているか読み取れない場合、または `intentRecord` が
別のインテントを指している場合、稼働中のレガシー Bolt が承認されることは決してない。レガシーのディレクトリがない場合は、
一致する 3 つのフロンティアイベントのいずれでも、クリーンアップのみの解決が許可される。
それでも `merge`、`discard`、`purge` は削除の前に Git の状態を検証する。それ以外の場合は
名前空間付きの識別情報が適用される。ディレクトリの存在も、スペースやワークスペース内の
インテントの数も、所有権の証明にはならない。`doctor` は両方の形を報告する。

restore と purge が、名前空間付きおよびレガシーの退避済み試行を受け入れるのは、
選択されたインテント自身の `WORKTREE_DISCARDED` 行が、その正確な `Parked ref`
とスタンプを記録している場合だけである。未知の退避は無視される。記録されていない `--parked` スタンプを要求すると
拒否される:

```text
parked attempt <stamp> is not recorded by intent <record>
```

レガシーのディレクトリが存在せず、永続的な Git の証拠も残っていない場合、`create`
は名前空間付きの識別情報を使う。ブランチまたは保持された ref が残っている場合は拒否する:

```text
Legacy Bolt <slug> left branch <bolt-slug>[ and <n> retained refs] behind; run discard --slug <slug> under its intent to park and clean them before creating a new Bolt.
```

既存の `Worktree directory already exists: <dir>` という拒否は、
そのディレクトリが存在する場合にのみ適用される。選択された識別情報について証拠のない discard は、
このチェックアウトに別の識別情報のもとで同じ slug の Bolt ディレクトリがない場合にのみ
`already-discarded` を返す。それ以外の場合は非ゼロで終了する:

```text
no Bolt <slug> belongs to intent <record>; this checkout holds <name> (intent <id8>|legacy) at <path> - select that intent to discard it
```

インテントのレジストリ UUID がない状態での作成はフェイルクローズする:

```text
Intent record <relative record dir> has no registry identity (uuid); adopt or re-create the intent before Construction. Bolt worktrees are named by intent so parallel intents cannot collide.
```

識別情報を解決するコマンド（`create`、`merge`、`discard`、`restore`、`purge`）も、
レガシーの検索の前にレジストリ UUID を必要とする。これらのコマンドを使う前に、孤立した
インテントを採用するか再作成する。ディレクトリ名から UUID を推測してはならない。
`list` は引き続きアクティブなインテントとは独立した一覧であり、`verify` は
引き続き監査の検索である。どちらもレジストリの識別情報を必要としない。`info` が
レジストリ UUID を必要とするのは、インテントにスコープされた `bolt-<id8>_<slug>` 行を受け入れる場合だけである: 監査された
id8 は、選択されたインテントの `idSuffix(uuid)` と等しくなければならない。レガシーの `bolt-<slug>` 行は
引き続き何も必要としない。レジストリの `intents.json` は、選択された
インテントの id8 の唯一の信頼できる情報源である。UUID がない場合、インテントにスコープされた行は、
識別情報を解決するコマンドと同様にフェイルクローズする:

```text
error: WORKTREE_CREATED block at <ts> names an intent-scoped Bolt, but intent <record> has no registry identity (uuid); adopt or re-create the intent before Construction
```

`<id8>` は Bolt のインテントに関する権限のすべてであるため、UUID の末尾 8 桁の 16 進文字が
一致する 2 つの登録済みインテント（ペアあたり 2^-32 の事象）は、名前とリカバリ用 ref を共有するのではなく、
識別情報を解決するすべてのコマンドを拒否する:

```text
Intent record <relative record dir> shares its eight-character uuid suffix with another registered intent, so its Bolt names would collide; re-create one of the two intents before Construction.
```

クリーンアップが Bolt のブランチやその保持済み/退避済みの ref を削除する前に、そのブランチは
自身の Bolt ディレクトリでチェックアウトされているか、どこでもチェックアウトされていない状態でなければならない。別のワークツリーが
それを所有している場合、クリーンアップは拒否し、所有者のパスを stderr に示し、監査に
`(checked out in another worktree of this repository)` を記録する。拒否を回避するために
他のチェックアウトを削除してはならない。

クリーンアップのみのスウォームのマージには、解決された Bolt のパスと一致する `WORKTREE_CREATED` 行も
必要である。レガシーのディレクトリがなくなっている場合、discard は残っているブランチを、
このインテントの最新のレガシー `WORKTREE_DISCARDED`
行が保存した先端（`Parked ref` に `/branch-tip` を付けたもの、なければ `/head`）と照合する。先端が異なる場合は拒否する:

```text
legacy branch <name> tip does not match this intent's recorded discard; leaving it for inspection
```

フロンティアが `WORKTREE_CREATED` であれば、discard はまだ開始されておらず、
通常の Git のチェックを経て進む。クリーンアップが保持済みの reviewed-source ref を受け入れるのは、
そのサフィックスが 40〜64 文字の小文字 16 進数のコミットである場合だけである。parked-source ref には
有効なスタンプと、次のいずれか 1 つのサフィックスが必要である:
`<stamp>/head`、`<stamp>/snapshot`、`<stamp>/branch-tip`、または
`<stamp>/reviewed-source/<40–64-character lowercase hex commit>`。
それ以外の ref はスキップされ、決して削除されない。

スウォームのコマンドはセッションのアクティブなワークフローに従う。`--intent`/`--space`
を渡すのは、それらがそのワークフローに解決される場合だけにする。一致しない場合は、変更や
監査の出力の前に拒否される:

```text
swarm commands follow the session's active workflow (${ambient.space}/${ambient.intent}); switch to ${explicit.space}/${explicit.intent} instead of passing --intent/--space
```

## Usage

```
{{INVOKE}} engine worktree info --slug <kebab-slug>
```

slug は、その Bolt に対するすべてのワークツリーコマンド（`create`、`verify`、`merge`、`discard`、`restore`、`purge`）に引き渡される kebab-case の Bolt 識別子である。`name → slug` の変換については、`SKILL.md` の Bolt ごとのループにある "Slug derivation" の段落を参照する。

## Exit codes

| 終了コード | 意味 | stdout | stderr |
|------|---------|--------|--------|
| 0 | ヒット — JSON を出力 | JSON オブジェクト（下記参照） | （空） |
| 1 | slug に対する `WORKTREE_CREATED` がない（または監査がない）、ブロックの形式が不正、ブロックの `Branch name`/`Worktree path` が選択されたインテントの正規の Bolt を示していない、またはその `Timestamp` が ISO 8601 の UTC 時刻でない | （空） | 1 行のエラーメッセージ |

識別情報の検証では、チェック順（タイムスタンプ、ブランチ名、レジストリの識別情報、ワークツリーのパス）に次の安定した拒否メッセージを使う。`<owner>` は `intent <record>` または `the selected workspace` として表示される。各拒否は stderr に 1 行を出力し、stdout は空のままにする。拒否された監査のバイト列はどちらのストリームにも決して現れない。後半の拒否メッセージの `<ts>` は、常に検証済みのタイムスタンプである。

```text
error: malformed WORKTREE_CREATED block for Bolt <slug>: Timestamp is not an ISO 8601 UTC instant
error: malformed WORKTREE_CREATED block at <ts>: Branch name does not name Bolt <slug> for <owner>
error: WORKTREE_CREATED block at <ts> names an intent-scoped Bolt, but <owner> has no registry identity (uuid); adopt or re-create the intent before Construction
error: malformed WORKTREE_CREATED block at <ts>: Worktree path is not the canonical directory <dir> of Bolt <name>
```

終了コードのコントラクトは `verify` のセマンティクスを踏襲している: 非ゼロが停止のシグナルである。オーケストレーターは非ゼロの終了をすべて「表示するワークツリーがない」として扱う。同じ Retry/Skip/Abort の質問を提示するが、"Worktree at [path] on branch [branch_name]." の節を、ボイスコントラクトに沿った 2 文の訳（何を表示できなかったかとその理由、次に次のステップ）に置き換える（construction プロトコルの halt-and-ask の段落を参照）。slug からブランチやパスをでっち上げるのではなく、識別情報の拒否を提示する。

## JSON output shape (exit 0)

```json
{
  "slug": "onboarding-wizard",
  "path": "/Users/dev/project/.aidlc/worktrees/bolt-7c31e9a0_onboarding-wizard",
  "branch_name": "bolt-7c31e9a0_onboarding-wizard",
  "intent_id8": "7c31e9a0",
  "audit_timestamp": "2026-05-18T12:34:56Z",
  "merge_held": false
}
```

フィールドのセマンティクス:

- **`slug`** — 入力の `--slug` フラグをそのまま返す。素の kebab-case 識別子（例: `onboarding-wizard`）は引き続き人間/監査用の識別情報であり、ディレクトリ名やブランチ名ではない。`name → slug` の変換については、SKILL.md の Bolt ごとのループにある "Slug derivation" を参照する。オーケストレーターはこのフィールドを、対応関係の確認に使うのであって、別のものを選ぶためには使わない。
- **`path`** — 検証済みの識別情報に対する正規の絶対パス: インテントにスコープされた Bolt では `<projectDir>/.aidlc/worktrees/bolt-<id8>_<slug>`、レガシーの Bolt では `<projectDir>/.aidlc/worktrees/bolt-<slug>`。`info` はこの値を再構築するのであって、決してそのまま返すことはない。監査された `**Worktree path**:`（新しい行ではプロジェクト相対、レガシーの行では絶対パス）は、正規のパス比較（realpath、区切り文字を正規化）のもとでその場所に解決されなければならず、そうでない場合 `info` は上記のワークツリーパスの拒否で拒否する。利用者は一時停止した Bolt を調べるためにここに `cd` する。
- **`branch_name`** — 検証済みの識別情報に対する正規の Bolt 名であり、自由記述ではない。監査された `**Branch name**:` は `parseBoltName` で解析できなければならず、その slug は `--slug` と等しく、インテントにスコープされた名前の id8 は選択されたインテントのレジストリ id8 と等しくなければならない。形式が不正な名前や識別情報の不一致は上記のブランチ名の拒否で拒否され、レジストリ UUID のないインテントのもとでのインテントにスコープされた名前はフェイルクローズする。新しいブランチは `bolt-<id8>_<slug>` を使い、アップグレード前のレガシーなブランチは `bolt-<slug>` を維持する。`slug` から組み立ててはならない。これを再構築するのはツールであり、オーケストレーターではない。
- **`intent_id8`** — インテントにスコープされた名前の検証済み id8。なければ正規のレガシーディレクトリのメタデータの `intentId8`、それもなければ `null`。
- **`audit_timestamp`** — 一致する `WORKTREE_CREATED` ブロックの ISO 8601 UTC タイムスタンプで、出力前に検証される（`**Timestamp**:` が `YYYY-MM-DDTHH:MM:SS[.fff]Z` でない行は拒否される）。オーケストレーターが鮮度を判断するのに役立つ。現在は AUQ のプロンプトには表示されない。
- **`merge_held`** — `<path>/<record>/aidlc-state.md` にある Bolt ごとにフォークされた状態の `Merge-Held` フィールドを反映する真偽値（ファイルが存在し、かつフィールドが `true` の場合にのみ `true`。存在しない場合は `false` に解決される）。オーケストレーターは再開時にこれを読み、`aidlc-bolt complete --merge --slug <slug>` をディスパッチしても安全かどうかを判断する。保留状態は、複数失敗の halt-and-ask のシーケンスが始まる前に `aidlc-bolt hold-merge --slug <slug>` によって設定され、兄弟の AUQ がすべて解決した時点で `aidlc-bolt release-merge --slug <slug>` によって解除される。

## Most-recent semantics

`info` は、その slug に対する**最新の** `WORKTREE_CREATED` を返す — 監査ログ上の位置で最新のもの（`findLatestEvent` による末尾から先頭への走査）を意味する。同じワークフロー内で slug が作成 → 破棄 → 再作成された場合、`info` が返すのは 2 回目の作成のパスである。これは利用者のメンタルモデル「slug X の稼働中のワークツリー」と一致する。

リトライ後に失敗するシナリオ（code-gen が失敗し、利用者が Retry を選び、code-gen が再び失敗する）では、新しい `WORKTREE_CREATED` は作成されない — SKILL.md の Bolt ごとのループに従い、Retry は既存のワークツリーで再実行する。したがって `info` の出力はリトライの試行をまたいで安定している。`tests/worktree/t11-halt-and-ask-retry-correlation.sh` で固定されている。

## `list` output

`{{INVOKE}} engine worktree list` は、プロジェクトの `.aidlc/worktrees/` 配下で解析できた
Bolt ディレクトリを報告する。同じ slug を持つ複数のインテントを含みうる。
解析できないディレクトリ名はスキップされる。

```json
{
  "worktrees": [
    {
      "slug": "payments",
      "worktree_path": "/Users/dev/project/.aidlc/worktrees/bolt-7c31e9a0_payments",
      "branch": "bolt-7c31e9a0_payments",
      "intent_id8": "7c31e9a0",
      "legacy": false
    },
    {
      "slug": "payments",
      "worktree_path": "/Users/dev/project/.aidlc/worktrees/bolt-4b829d10_payments",
      "branch": "bolt-4b829d10_payments",
      "intent_id8": "4b829d10",
      "legacy": false
    },
    {
      "slug": "onboarding-wizard",
      "worktree_path": "/Users/dev/project/.aidlc/worktrees/bolt-onboarding-wizard",
      "branch": "bolt-onboarding-wizard",
      "intent_id8": null,
      "legacy": true
    }
  ]
}
```

各行は `slug`、絶対パスの `worktree_path`、`branch` を保持する。`intent_id8` は
ディレクトリのベース名から解析され、レガシーの名前では `null` となる。`legacy` は
アップグレード前の形の場合にのみ `true` となる。一覧に表示されたレガシーのディレクトリは、
選択されたインテントがそれを所有している証拠ではない: ライフサイクルのコマンドは引き続き出所を確認する。

## Worktree metadata repository provenance

新しい `.aidlc/worktree-meta.json` ファイルは `version: 1` を維持し、`intentId8`
（例: `7c31e9a0`）と `branch`（例: `bolt-7c31e9a0_onboarding-wizard`）を追加する。どちらも、
アップグレード前のメタデータを読む際には任意である。`intentRecord` は引き続き、レガシーの所有権チェックに使われる
相対的なインテントの記録ディレクトリである。

新しい `.aidlc/worktree-meta.json` ファイルは `gitCommonDirHash` を保存する。これは
正規の Git 共通ディレクトリのパスの 64 文字の SHA-256 16 進ダイジェストである。マシン上の生の
パスは永続化されない。マージの検証では、選択されたチェックアウトと
ワークツリーの共通ディレクトリをハッシュ化し、ダイジェストを比較する。

移行は、平文の `gitCommonDir` を持つ古いメタデータとの互換性を維持する:
読み取り側は比較の前に、保存されたその値をハッシュ化する。新しいメタデータは
両方のフィールドを書いてはならない。

## Recoverable discard, restore, and purge

`{{INVOKE}} engine worktree discard --slug <slug>` は、`WORKTREE_DISCARDED` を出力する前に
作業ツリーのスナップショットとレビュー済みソースの ref を退避し、その後、
稼働中のチェックアウトとブランチを削除し、元のレビュー済みソースの ref を比較削除する。
一時的な Git インデックスと `commit-tree` が、追跡ファイルと
無視されていない未追跡ファイルを取り込む。無視されている未追跡ファイルはバックアップされない。
clean フィルターや `working-tree-encoding` が設定された通常ファイルは、それらの変換を
迂回して生のバイト列を保持する。

新しい退避用の名前空間は `refs/aidlc/parked/<id8>/<slug>/<stamp>` で、`stamp` は UTC の
`YYYYMMDDTHHMMSSZ` であり、衝突時には数値の `-N` サフィックスが付く。`/head` は
生の作業ツリーの blob を持つスナップショットコミットを指し、`/snapshot` マーカーも
同じコミットを指す。スナップショットの退避では、元のブランチの OID も
`/branch-tip` に保存されるため、クリーンアップのみのリトライで、保存された未コミットの変更を
ブランチの先端と取り違えることなく、残っているブランチを比較できる。チェックアウトが
既になくなっていてブランチだけが残っている場合、`/head` は代わりにその先端を保持し、その
blob は通常のコミット済みの形となり、`/branch-tip` は同じコミットを指す。
`/reviewed-source/<commit>` は各レビュー済みソースの ref を
保持する。discard の JSON は `parked_ref`（名前空間のプレフィックスであり、
その `/head` ref ではない）と `parked_commit`（スナップショットコミットまたはブランチの先端）を維持し、
`parked_stamp`（正確なスタンプ）、`parked_mode`（`snapshot`、`branch-tip`、または
`evidence-only`）、`parked_repo`（プロジェクトルートでは `null`、それ以外は
兄弟リポジトリの名前）を追加する。退避すべきものがレビュー済みソースの ref だけの場合、
`parked_mode` は `evidence-only`、`parked_commit` は文字列 `"-"` となり、復元すべき `/head` は
存在しない。それでも `parked_ref`、`parked_stamp`、`parked_repo` は
保存された証拠を識別する。
already-discarded の応答は変わらず、`parked_*` フィールドを持たない。

成功した `bolt abort` の JSON は `reason: "aborted"` を維持し、指定された
`--reason` のテキストを追加の `abort_reason` フィールドでそのまま返す。常に
`parked_ref` を含み、何も退避されなかった場合（`--discard` なしの場合を含む）は `null` となる。
`parked_ref` が null でない場合にのみ、discard の記述子の
`parked_stamp`、`parked_mode`、`parked_repo` が追加される。
復元可能な試行では、`restore_operation` は
`aidlc-guard-operation.ts` の `EngineInvocation`（`{ route: string; args: readonly string[] }`）である。その
route は `worktree` で、args は
`["restore", "--slug", slug, "--parked", stamp, "--repo", repo ?? ".", "--intent", recordDirName, "--space", space]` である。
リポジトリのセレクターは常に存在する: 兄弟リポジトリの名前、または
`parked_repo` が `null` の場合は `.`。`recordDirName` は所有するインテントの記録
ディレクトリ名、`space` はそのスペースである。これにより、正確な保存済みの試行、
リポジトリ、インテントが選択され、同じ slug を持つ後の試行や、ワークフローの切り替え後の別のアクティブな
インテントが選択されることはない。

`restore_hint` は任意の人間向けの表示テキストで、その操作から
`renderEngineInvocation` によって生成される。レンダラーはインストールされたネイティブまたはソースの
プレフィックスを選択し、ハーネスのディレクトリを検証し、選択されたシェルに対してすべての引数を安全に
クォートする。レンダリングが例外を投げた場合、abort はヒントを省略し、
`restore_hint_error` に理由を返す。`restore_operation` は引き続き利用できる。ヒントがないことは、
証拠だけが保存されたことを意味しない。evidence-only の abort は
4 つの記述子フィールドをすべて保持するが、`restore_operation`、`restore_hint`、
`restore_hint_error`、`parked_excludes` を省略する。除外対象はモードに従う:

| `parked_mode` | `parked_excludes` |
|---|---|
| `snapshot` | `["ignored files", "eol/text=auto normalization"]` |
| `branch-tip` | `["uncommitted files (no working tree existed)"]` |
| `evidence-only` | 存在しない。レビューの証拠だけが保持され、復元可能なファイルはない |

保存された名前空間はわかっているが discard の記述子が欠落している場合、
フォールバックは `parked_ref` を保持し、スタンプが厳密に解析できる場合にのみ、名前空間から `parked_stamp` を
導出する。それ以外の場合 `parked_stamp` は `null` となる。
どちらもレガシーの出力からは推測できないため、`parked_mode: null` と `parked_repo: null` を報告し、
`restore_operation`、`restore_hint`、
`restore_hint_error`、`parked_excludes` を省略する。代わりに、`recovery_hint` が人間に対し、
doctor を実行して退避された試行とその正確な restore コマンドの一覧を表示するよう求める。
このヒントは平易な案内であり、実行可能な操作ではない。モードが不明では
どのファイルが保存されたかが確定しないため、このフォールバックから、保存されたファイルについて主張したり、
復元を提案したりしてはならない。名前空間が保存されなかった場合、`parked_ref` は
`null` となり、`parked_stamp`、`parked_mode`、`parked_repo`、`restore_operation`、
`restore_hint`、`restore_hint_error`、`parked_excludes`、`recovery_hint` は存在しない。
`aidlc/spaces/default/intents/260918-onboarding/`（UUID サフィックス `7c31e9a0`）に対するスナップショット記述子を持つ
結果は次のようになる:

```json
{
  "emitted": "BOLT_FAILED",
  "reason": "aborted",
  "abort_reason": "stale review recovery exhausted",
  "failed_bolt": "Onboarding Wizard",
  "slug": "onboarding-wizard",
  "discarded": true,
  "parked_ref": "refs/aidlc/parked/7c31e9a0/onboarding-wizard/20260918T123456Z",
  "parked_stamp": "20260918T123456Z",
  "parked_mode": "snapshot",
  "parked_repo": null,
  "restore_operation": {
    "route": "worktree",
    "args": ["restore", "--slug", "onboarding-wizard", "--parked", "20260918T123456Z", "--repo", ".", "--intent", "260918-onboarding", "--space", "default"]
  },
  "restore_hint": "bun {{HARNESS_DIR}}/tools/aidlc-worktree.ts restore --slug onboarding-wizard --parked 20260918T123456Z --repo . --intent 260918-onboarding --space default",
  "parked_excludes": ["ignored files", "eol/text=auto normalization"]
}
```

監査の `BOLT_FAILED` フィールドは引き続き `Reason: aborted` である。成功時の JSON の
`abort_reason` が呼び出し元のテキストを保持する。abort の引数と人間の同意の
要件は変わらない。話し言葉では、この試行を削除や完了ではなく、**退避した（set aside）** と
呼ぶ。`snapshot` の場合は、保存された追跡ファイルと無視されていない
未追跡ファイル、および上記の除外対象を説明する。`branch-tip` の場合は、"I
kept its committed work; there were no uncommitted files to save."（コミット済みの作業は保持しました。保存すべき未コミットのファイルはありませんでした。）と言う。`evidence-only` の場合は、
"Nothing of its working files remained to save; only its review evidence
was kept."（保存すべき作業ファイルは何も残っていませんでした。レビューの証拠だけを保持しました。）と言う。復元を提案するのは `restore_operation` が存在する場合だけにする。
後で人間が復元を要求した場合は、その `worktree` route を
`{{INVOKE}} engine worktree <args...>` を通じて呼び出し、保存された各 `args` 要素を正確に
別々の argv 引数として渡す。args を連結してシェルコマンドにしたり、
`restore_hint` を実行したり、slug だけの選択を組み立て直したりしてはならない。ヒントは人間向けの表示
テキストにすぎない。レンダリングエラーがあっても復元の提案は取り下げられない。返された
`worktree_path` を平易に伝える。

### Restore a set-aside attempt

```
{{INVOKE}} engine worktree restore --slug <slug> [--parked <stamp>] [--raw] [--repo <name|.>] [--intent <intent>] [--space <space>]
```

`--parked` がない場合、restore は最新の退避済み `/head` を選択し、タイムスタンプの
サフィックスを数値として順序付ける（`-10` は `-2` より新しい）。指定した場合、restore はその
正確なスタンプを選択する。正確なスタンプが 1 つのリポジトリにしか存在しない場合は、
リポジトリ間で共有される slug を確認する前にそのリポジトリが選択される。それでも選択が
曖昧な場合、`--repo <name>` は既存の兄弟 Git リポジトリを選択し、
`--repo .` はプロジェクトルートを選択する。`WORKTREE_CREATED` と
`WORKTREE_DISCARDED` はどちらも `Repo` を出力する: 記録された兄弟リポジトリの名前、またはプロジェクト
ルートの場合は `-`。discard の行は、作成の行が利用できない場合でもこの出所を
保持する。リカバリは、同じ slug の
作成または discard の監査の `Repo` フィールドで指定された有効な Git リポジトリを、その兄弟リポジトリの名前がシンボリックリンクであっても受け入れる: フレームワークは、
試行のワークツリーや退避を記録した場所そのものでリカバリできる。その受け入れは
slug にスコープされる。別の slug の記録によって、この slug のリポジトリ集合が広がることは決してない。
現在または過去のインテントのリポジトリ一覧に含まれているというだけでは、
シンボリックリンクは受け入れられない。インテント一覧の候補と、記録されていない発見済みの兄弟は、
正規パスが正規のワークスペースルートの直下にとどまる、実在する
直接の子ディレクトリでなければならない（`isWorkspaceRepoDir`）。同じ slug の監査の出所を持たない
任意のパスやシンボリックリンクのエイリアスは拒否される。
これらの明示的な restore/purge のセレクターは、現在の
インテントのリポジトリ一覧とは独立して機能する。稼働中の create/discard のセレクターの挙動は変わらない。
ワークスペースのコンテキストを解決する必要がある場合は、intent/space のセレクターを使う。restore と
purge は、選択や変更の前に未知のフラグと重複したフラグを拒否する。
`--raw` は値を取らない restore 専用のフラグである。restore は
ブランチ `restore/bolt-<id8>_<slug>-<stamp>` 上に `.aidlc/restored/bolt-<id8>_<slug>-<stamp>` を
作成し、稼働中の
`.aidlc/worktrees/bolt-<id8>_<slug>` のパスや `bolt-<id8>_<slug>` のブランチを再利用したり変更したりすることは決してない。ファイルを復元しても、
中断された Bolt が再開されたり、そのレビューの権限が復活したりすることはない。退避されたレビュー済み
ソースの ref は退避用の名前空間に残り、アクティブな ref にコピーし戻されることはない。
レガシーの復元は `.aidlc/restored/bolt-<slug>-<stamp>` とレガシーの
`restore/bolt-<slug>-<stamp>` ブランチを維持する。新しい稼働中のレガシー Bolt は作成しない。

restore は次の順序でモードを決定する: 値を取らない `--raw` フラグは、どの退避に対しても保存された blob を
バイト単位で正確に書き出す。それ以外の場合、`/snapshot` があればバイト単位で正確な実体化を選択し、
次に `/branch-tip` があれば Git の通常のチェックアウトを選択する。レガシーの退避は、どちらの形についても
いずれのマーカーも持たない。マーカーのない head がスナップショットとみなされるのは、そのコミットの
author がちょうど `AI-DLC`、メールアドレスが `aidlc@localhost` で、件名が
レガシーの件名 `aidlc: parked bolt-<slug> at ` で始まる場合だけである。それ以外のマーカーのない head はすべて通常の
チェックアウトを使う。この識別チェックは元のコミットを読み、Git の置換
オブジェクトを無視する。バイト単位で正確な実体化は、smudge/process フィルターと
working-tree-encoding の変換を迂回する。通常のチェックアウトはこれらの変換を適用する。
必須のフィルターが失敗すると、restore は Git のエラーとともに失敗する。
通常ファイルの blob はメモリにバッファリングされることなくディスクに直接ストリーミングされる。
バッファリングされるのはシンボリックリンクのターゲットだけである。raw で復元されたパスは、
それ自身のフィルターのもとで変更済みとして表示される場合がある。
実行可能ファイルはモードを保持する。シンボリックリンクは、
`core.symlinks` が未設定または true の場合はシンボリックリンクとして実体化される。`core.symlinks=false` の場合、モード 120000 の
エントリは、Git がチェックアウトするのとまったく同じく、リンク先をバイト列とする通常ファイルとして書き出される。サブモジュールの gitlink は空のディレクトリになり、サブモジュールの
チェックアウトは復元されない。退避時の Git の
eol/`text=auto` 正規化が明示的な限界である: 退避時に正規化された CRLF のバイト列は
復元できない。
非 UTF-8 の名前を持ち、`filter`、`text`、`eol`、`ident`、または
`working-tree-encoding` 属性（unspecified でも unset でもないもの）を持つ通常ファイルは、現時点では
退避できない。discard は解体の前に、属性に応じたメッセージで拒否する:

```text
cannot park file with a non-UTF-8 name and a content-transforming attribute (<attr>=<value>): <name>; rename the file or unset its <attr> attribute
```

稼働中の試行はそのまま残る。これらの属性を持たない名前は通常どおり
保存できる。`ignored files` の除外は、無視されている未追跡ファイルを意味し、
たまたま ignore パターンに一致する追跡ファイルを意味するものではない。

```json
{
  "restored": true,
  "slug": "onboarding-wizard",
  "parked_ref": "refs/aidlc/parked/7c31e9a0/onboarding-wizard/20260918T123456Z",
  "worktree_path": "/Users/dev/project/.aidlc/restored/bolt-7c31e9a0_onboarding-wizard-20260918T123456Z",
  "branch": "restore/bolt-7c31e9a0_onboarding-wizard-20260918T123456Z",
  "reviewed_source_refs": 1,
  "materialized": 12,
  "raw_bytes": true,
  "restore_mode": "snapshot"
}
```

`reviewed_source_refs` は、その退避用の名前空間に保持されたレビュー済みソースの ref の数を数える。`raw_bytes` は
バイト単位で正確な実体化（スナップショットまたは明示的な `--raw`）では `true`、
Git の通常のチェックアウトでは `false` となる。`materialized` は
`raw_bytes` が `true` の場合にのみ存在し、書き出された通常ファイルとシンボリック
リンクの数を、サブモジュールの gitlink を除いて数える。evidence-only の
名前空間を選択すると、`--raw` を指定した場合でも
`no restorable files were parked for <slug> <stamp>; only review evidence was kept` で拒否される。
raw での実体化が失敗した場合は、部分的なチェックアウトをその場に残し、
そのパスを報告する。失敗した Git のチェックアウトを `--raw` でリトライする前に、
残っている restore 用のチェックアウトと、報告された `branch` を削除する。

`restore_mode` は、その挙動が選択された理由を記録する:

| 値 | 選択の条件 | `raw_bytes` |
|---|---|---|
| `raw-requested` | 明示的な `--raw`。マーカーとレガシーの識別情報より優先される | `true` |
| `snapshot` | `/snapshot` マーカーが存在する | `true` |
| `branch-tip` | `/branch-tip` マーカーが存在し、`/snapshot` がない | `false` |
| `legacy-snapshot` | どちらのマーカーもない。ツールが作成したスナップショットの識別情報が一致する | `true` |
| `legacy-branch-tip` | どちらのマーカーもない。スナップショットの識別情報が一致しない | `false` |

### Purge parked refs

```
{{INVOKE}} engine worktree purge --slug <slug> [--parked <stamp> | --older-than <days>] [--repo <name|.>] [--intent <intent>] [--space <space>]
```

purge は、選択されたインテントの Bolt について、すべての退避済み ref を比較削除する。`--parked` が指定された場合は
選択されたスタンプだけを、`--older-than <days>` の閾値が指定された場合はそれより厳密に古い試行だけを
削除する。日数は非負かつ有限でなければならず、小数も
受け付ける。タイムスタンプはスタンプの UTC `YYYYMMDDTHHMMSSZ` 部分であり、
`-N` の衝突サフィックスやコミットの日付とは無関係である。共通の厳密な
カレンダーパーサーは、存在しない日付や時刻を正規化するのではなく拒否する。
`--older-than` を指定した場合、解析できないスタンプは保持され、
`skipped_unparseable: [stamps]` で報告される。経過日数がちょうど閾値の試行は
保持される。`--older-than` と `--parked` は同時に指定できない。

対応する復元済みのチェックアウトがまだ存在するか、Git に登録されている場合（移動されたチェックアウトを含む）、
purge は拒否する。そのリカバリ用 ref を purge する前に、そのチェックアウトを明示的に削除する。
purge は稼働中の Bolt のチェックアウトや
ブランチを決して削除しない。JSON は、スナップショットの数ではなく、削除された ref の数を報告する:

```json
{
  "purged": 3,
  "slug": "onboarding-wizard",
  "stamps": ["20260918T123456Z"],
  "skipped_unparseable": []
}
```

`skipped_unparseable` は常に存在する。`--older-than` が解析できないスタンプを
保持した場合を除き、空である。経過日数のフィルターがない場合、スタンプを明示した purge やすべてのスタンプを対象とする
purge は、それらのスタンプを削除できる。`stamps` リストには、選択されたスタンプだけが含まれる。

restore と purge は新しい監査イベントを出力しない。Worktree の分類は 7 つのままである。

### Doctor inventory

doctor は、通常および詳細なレポートに **Parked attempts** という情報セクションを表示する。
保存された `/head` も実際の `/reviewed-source/<commit>`
エントリも存在しない場合は省略される。これらのエントリは警告でも失敗でもない。各エントリは、
slug、正確なスタンプ、経過日数、モード（記録された保存済みの head については `snapshot`、`branch-tip`、または `legacy`、
`/head` のない記録されたレビューの証拠については `evidence-only`、
曖昧さのない discard の出所がない場合は `unrecorded`）、所有するリポジトリが、正規の
`.aidlc/restored/bolt-<id8>_<slug>-<stamp>` のパスに、正確な
`restore/bolt-<id8>_<slug>-<stamp>` ブランチで Git のワークツリーを登録しているかどうか、そして記録された試行については、
正確な `--parked <stamp>` と明示的な `--repo <name>`
または `--repo .` の引数に続けて `--intent <record-dir-name> --space <space>` を持つ、型付きの
リカバリ操作を報告する。
記録されたエントリは `purge_operation` を持ち、記録された復元可能なエントリだけが
`restore_operation` を持つ。それらの任意のレンダリング済みコマンドは人間向けの表示テキストである。
安全なレンダリングに失敗した場合、対応するコマンドは省略され、そのエラー
フィールドが理由を説明する。操作は残る。evidence-only のエントリが公開するのは
purge の操作と、そのコマンドまたはエラーのフィールドだけである。記録されていないエントリは、
操作もコマンド/エラーのフィールドも公開しない。doctor は、上で説明したのと同じ slug にスコープされた
リカバリ用リポジトリの候補集合を使う。
`legacy` の一覧モードはコミットの識別情報を分類しない。restore が
呼び出されたときにその区別を行う。別の場所に移動されたチェックアウトは、この一覧では
復元済みとして表示されない場合があるが、purge は引き続き Git のワークツリーの
登録を確認し、その ref の削除を拒否する。

doctor は、名前空間付きとレガシーの両方の試行を一覧化する。名前空間付きの試行は、
レジストリ UUID のサフィックスを通じて所有するインテントを解決する。レガシーの試行は、
単一の所有者の正確な `WORKTREE_DISCARDED` の `Parked ref` の出所を必要とし、
レガシーの復元パスとブランチを維持する。リカバリ操作には、名前空間付きとレガシーの ref のいずれについても、
所有者自身の `WORKTREE_DISCARDED` 行が、一致する slug と正確な
`Parked ref` を記録していることが必要である。記録されていない試行は、
未知または曖昧な所有者や帰属不明のレガシーの退避を含め、
`mode: "unrecorded"` として表示されたままとなり、アクティブなインテントを通じて承認されることは決してない。
それらの JSON の `note` は `no WORKTREE_DISCARDED row records this parked attempt; inspect refs/aidlc/parked/<...> manually` である。
利用可能な各操作は、所有する記録ディレクトリの名前とスペースを持つため、
ワークフローの切り替え後もそのインテントに紐付いたままとなる。

公開されている doctor の JSON は、オブジェクトの配列である `data.parked_attempts` を公開する:

| フィールド | 意味 |
|---|---|
| `slug`, `stamp` | 正確な Bolt 識別子と、保存された名前空間のスタンプ |
| `age_days` | `-N` を無視した、スタンプからの経過 UTC 日数（整数）。未来のスタンプは `0`、無効なカレンダーのタイムスタンプは `null`（テキストでは `unknown`）を示す |
| `mode` | 記録された保存済みの head については `snapshot`、`branch-tip`、または `legacy`。`/head` のない記録されたレビュー済みソースの ref については `evidence-only`。曖昧さのない discard の出所がない場合は `unrecorded` |
| `repo` | 兄弟リポジトリの名前。プロジェクトルートの場合は `null` |
| `restored_path`, `restored_exists` | 正規の復元パスと、所有するリポジトリが正確な `restore/bolt-<id8>_<slug>-<stamp>` ブランチ（レガシー: `restore/bolt-<slug>-<stamp>`）でそのパスに解決されるチェックアウトを登録しているかどうか |
| `restore_operation` | route が `worktree`、args が `["restore", "--slug", slug, "--parked", stamp, "--repo", repo ?? ".", "--intent", recordDirName, "--space", space]` の `EngineInvocation`。`evidence-only` と `unrecorded` では存在しない |
| `purge_operation` | route が `worktree`、args が `["purge", "--slug", slug, "--parked", stamp, "--repo", repo ?? ".", "--intent", recordDirName, "--space", space]` の `EngineInvocation`。`unrecorded` では存在しない |
| `restore_command` | 人間向けの表示用に `restore_operation` を安全にレンダリングした任意の値で、実行の入力ではない。操作がない場合やレンダリングに失敗した場合は存在しない |
| `restore_command_error` | `restore_command` が省略されたが `restore_operation` が存在する場合の、レンダリング失敗の理由 |
| `purge_command` | 人間向けの表示用に `purge_operation` を安全にレンダリングした任意の値で、実行の入力ではない。操作がない場合やレンダリングに失敗した場合は存在しない |
| `purge_command_error` | `purge_command` が省略されたが `purge_operation` が存在する場合の、レンダリング失敗の理由 |
| `note` | `unrecorded` の試行に対する手動調査の案内で、正確な退避済み ref のプレフィックスを含む。記録された試行では存在しない |

コンダクターは、選択した操作のエンジン route を、列挙された各引数を
それぞれ独立した argv 引数として呼び出し、決して文字列を連結してシェルに渡さない。レンダリングは、
abort のヒントと同じネイティブ/ソースの選択、ハーネスの検証、シェルに安全な引数の
クォートを使う。無効なハーネスのディレクトリによって、型付きの
操作が抑制されたり、復元可能なエントリが evidence-only のエントリに変わったりすることはない。

doctor は経過日数でフィルタリングする purge と同じ厳密なカレンダーパーサーを使うため、
存在しない日付や時刻は `age_days: null` と人間向けの `unknown` になる。
丸められた表示上の経過日数は情報提供のためのものである。purge は、経過日数を丸めるのではなく、
タイムスタンプを正確な `--older-than` の閾値と比較する。

## Stderr error messages

監査の検索の失敗には次の安定したメッセージを使う。識別情報の拒否は上記の [Bolt identity](#bolt-identity) に従う:

```
error: no WORKTREE_CREATED audit entry for slug <slug> (audit log absent)
error: no WORKTREE_CREATED audit entry for slug <slug>
error: malformed WORKTREE_CREATED block at <timestamp> (missing Worktree path or Branch name field)
```

3 番目（不正な形式のブロック）のケースは、意図の監査の照合面である: フラグ付けと修復は doctor が扱い、`info` は推測を拒否するだけである。

## AUQ prompt rendering — long-path fallback

`path` と `branch_name` はどちらも検証済みの表示用の値であり（[Exit codes](#exit-codes) とフィールドのセマンティクスを参照）、決して生の監査テキストではない。オーケストレーターはそれらを構造化質問のプロンプト本文に埋め込む。プロンプト本文はターミナルの全幅で表示され、適切に折り返される（複数行の折り返しは macOS の Claude Code でサポートされており、各リリースの前に手動で検証している）。

将来の表示面（Windows PowerShell、mosh、幅の狭い tmux のペイン）が `question` 内の長いパスを切り詰める場合、文書化されたフォールバックは、`bolt-<id8>_<slug>` の末尾を保持しつつ、ディレクトリの境界で先頭を省略記号に置き換えて切り詰めることである:

```
.../project/.aidlc/worktrees/bolt-7c31e9a0_onboarding-wizard
```

このフォールバックは**現在は実装されていない** — 現在出荷されている挙動は、適切な折り返しを前提としている。リグレッションが表面化した場合は、`info` に `--max-path-display <chars>` フラグを追加し、オーケストレーターに上記のルールに従って切り詰めさせる。

## Related files

- 実装: `{{HARNESS_DIR}}/tools/aidlc-worktree.ts`（`handleInfo` ハンドラー）
- テスト: `tests/unit/t72-worktree-info.sh`
- 呼び出し元: `{{HARNESS_DIR}}/skills/aidlc/SKILL.md`（Bolt ごとのループの halt-and-ask フロー）
- `info` が読み込む `WORKTREE_CREATED` エントリを生成する監査の出力元: `aidlc-worktree.ts` の `handleCreate`（同じファイルの `~line 154` 付近）
- 監査フォーマットの仕様: `{{HARNESS_DIR}}/knowledge/aidlc-shared/audit-format.md` の `WORKTREE_CREATED` 行
