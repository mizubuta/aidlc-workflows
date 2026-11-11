# Branching Strategies

よく使われるブランチ戦略のメニューであり、それぞれの形、使いどころ、そして AIDLC の Construction ワークツリーが各戦略にどう対応するかを示す。オーケストレーターが Bolt の境界で aidlc-pipeline-deploy-agent をディスパッチするとき、このファイルは、チームが確認したブランチ戦略を `aidlc-worktree` ツールのフラグに対応付けるためにエージェントが参照するメニューである。

> **プラクティスの読み方:** 空テンプレートの検出、意味的なトピックの照合、アクティブなスペースの `project.md → team.md → org.md → hardcoded defaults` フォールバックチェーンについては `knowledge/aidlc-shared/rules-reading.md` を参照。以下のランブックでは、それらのファイルは `aidlc/spaces/<active-space>/memory/` 配下にある。
>
> より上位の CI フローのコンテキストについては、`cicd-patterns.md` の § "Branch Strategies" も参照。

新しいワークツリーのディレクトリとブランチには `bolt-<id8>_<slug>` を使う。`<id8>` は
選択されたインテントのレジストリ UUID の接尾辞で、ユニットの claim と共有される。保持される
ソースと退避された ref は、同じインテントにスコープされる。命名と、
アップグレード前のレガシー Bolt の出自に基づく完了については、
[Bolt の識別](../aidlc-shared/worktree-info-schema.md#bolt-identity) を参照。常に
ツールが返したパスとブランチを返し、`--slug` から再構成してはならない。

---

## Trunk-Based Development (default)

```
main ────────────────────────────────────────►
  ▲   ▲   ▲   ▲   ▲   ▲
  │   │   │   │   │   │   short-lived feature branches
  │   │   │   │   │   │   (1-2 days max), squash-merge to main
  unit-a  unit-c      unit-e
      unit-b  unit-d
```

**形。** すべての作業は、短命のフィーチャーブランチを経由して `main` にマージされる。長命のブランチは存在しない。未完成の作業は、本番環境でフィーチャーフラグによって制御される。

**使いどころ。** ほとんどのチームの既定。特に、CI パイプラインの所要時間が短く（30 分未満）、本番環境の問題をすばやく検出できるだけのオブザーバビリティがある場合に適する。

**よくある問題。**
- リリース頻度の低いチームは、リリースウィンドウまで機能を「保留」しておくのが難しいと感じる。答えはブランチではなくフィーチャーフラグである。
- テストカバレッジが十分でないチームはトランクベースにすべきではない。すべてのコミットが本番相当のパイプラインに流れるため、不安定なテストが全員を止めてしまう。

**ワークツリーの対応付け。** 作成: `{{INVOKE}} engine worktree create --slug <bolt-slug> --base main`。マージ: `--target main --strategy squash`。各 Bolt = `main` 上の 1 つの squash コミット。

**並列 Bolt。** 最もきれいに適合する。複数の Bolt を同時に進行できる。それぞれが現在の `main` から分岐し、squash がマージ時に履歴を平坦化するため、rebase の競合なしにそれぞれマージし戻せる。

### Execution runbook

トランクベースでディスパッチされた場合:

1. `shared/rules-reading.md` に従い、アクティブなスペースの `project.md`、`team.md`、`org.md` の順に `## Way of Working` を読む。3 つすべてが空の場合にのみ、ハードコードされた既定値を使う。
2. フラグを決定する: 既定は `--base main --target main --strategy squash`。`team.md` が明示的に別のことを言っている場合にのみ外れる。
3. **作成**: `{{INVOKE}} engine worktree create --slug <bolt-slug> --base main` を呼び出す。
4. **マージ**（Bolt のゲート承認後）: 呼び出し元はメインのチェックアウトで `main` 上にいなければならない。`{{INVOKE}} engine worktree merge --slug <bolt-slug> --target main --strategy squash --message '<commit message>'` を呼び出す。
5. § Response contract に従い、JSON エンベロープをオーケストレーターに返す。

### Failure modes

- **マージ時のダーティなツリー。** 取り込みが変更するファイルについて、`main` にコミットされていないローカルの変更がある場合: git は取り込みを開始せず、ツールはそれらのファイルと手順（それらをコミットしてから、同じマージを再実行する）を示す。それ以外の git の失敗は、git のメッセージをそのまま含む。オーケストレーターの halt-and-ask が再試行/中止を提示する。再試行ではワークツリーは保持される。中止時の明示的な discard は、ライブのチェックアウトとブランチを削除する前に作業を退避させる。
- **squash 時の競合。** squash が並行する `main` の動き（例: 別の Bolt が先に取り込まれた）と競合する。ツールは `{status: "conflict", conflict_files, detail, next}` とともに非ゼロで終了し、取り込みをメインのチェックアウトにステージしたままにする。そこでそれらのファイルを統合し、`git add` して（削除されたままのものは `git rm`）同じマージを再実行すると完了する。`git reset --merge` で取り込みを破棄できる。
- **ブランチがすでに存在する。** 監査前のエラーであり、ツールは上書きを拒否する。オーケストレーターは、許可された場合にのみ先に `worktree discard` で古い試行を退避させるか、別の slug を選ぶべきである。そのファイルは、中止結果または doctor の `restore_operation` を使い、そのエンジンのルートを、列挙された引数をそのまま argv として、別の復元用名前空間で呼び出して回復する。レンダリングされたヒントやコマンドは、人間向けの表示テキストにすぎない。
- **インテントにレジストリ UUID がない。** 作成は拒否される: `Intent record <relative record dir> has no registry identity (uuid); adopt or re-create the intent before Construction. Bolt worktrees are named by intent so parallel intents cannot collide.` slug のみのブランチで再試行してはならない。
- **ブランチが別の場所でチェックアウトされている。** Bolt のブランチや、その保持/退避された ref を削除する前に、クリーンアップでは、そのブランチが自分の Bolt ディレクトリでチェックアウトされているか、どこでもチェックアウトされていないことが必要である。他の所有者による拒否では、stderr に所有者のパスが示され、監査に `(checked out in another worktree of this repository)` が記録される。そのチェックアウトは保持し、拒否を表に出す。

---

## GitHub Flow

```
main ────────────────────────────────────────►
  ▲       ▲       ▲       ▲
  │       │       │       │   feature branches with PRs
  │       │       │       │   (no time limit; can live longer than 1-2 days)
  feat-A  feat-B  feat-C  feat-D
```

**形。** `main` + 期限のないフィーチャーブランチ。PR が `main` にマージされる。ブランチはトランクベースより長く存続できる。大きな機能では数日から数週間になる。`develop` や `release` ブランチはない。

**使いどころ。** トランクベースの規律は欲しいが、より長命のフィーチャーブランチが必要なチーム。オープンソースプロジェクトでよく使われる。

**よくある問題。**
- 長く存続しすぎるブランチは、マージの負債を蓄積する。取り込むかクローズするかの規律が必要である。
- フィーチャーフラグがないと、進行中の機能が無関係な作業のリリースを妨げる。

**ワークツリーの対応付け。** base/target はトランクベースと同じ: `--base main --target main`。戦略は通常 `squash` だが、ブランチを履歴に残したいチームは `merge` を使う。チームが確認時に選び、エージェントは `team.md` を読む。

### Execution runbook

GitHub Flow でディスパッチされた場合:

1. アクティブなスペースの `project.md`、`team.md`、`org.md` の順に `## Way of Working` を読む（マージ方式についての記述を含む）。トランクベースとの違いは、マージ戦略の選択（squash か merge か）である。
2. フラグを決定する: 確認内容に従い `--base main --target main --strategy <squash|merge>`。既定は `squash`。
3. **作成**: `{{INVOKE}} engine worktree create --slug <bolt-slug> --base main`。
4. **マージ**: `{{INVOKE}} engine worktree merge --slug <bolt-slug> --target main --strategy <squash|merge> [--message '<msg>']`。`--strategy merge` では、no-fast-forward のマージコミットによって bolt ブランチの個々のコミットが保持される。
5. § Response contract に従って返す。

### Failure modes

- **トランクベースと同じ**。加えて:
- **`--strategy merge` での古いベース。** 動き続ける `main` に対する長命の bolt ブランチは競合を生む。ツールは競合のエンベロープを報告し、ユーザーがワークツリー（競合時は保持される）で解決して、マージを再度呼び出す。

---

## GitFlow

```
main         ────────────────────────────────►
              ▲           ▲
              │           │   release/v1.0   release/v1.1
develop ─────┴───────────┴────────────────────►
  ▲   ▲   ▲       ▲   ▲
  │   │   │       │   │       feature branches off develop
  │   │   │       │   │
  feat-A feat-B   feat-C feat-D
                    ▲
                    │ hotfix/v1.0.1 (off main, merged to both)
```

**形。** 2 つの長命ブランチ（`main` = 本番、`develop` = 統合）に加え、`feature/*`、`release/*`、`hotfix/*` の短命ブランチ。リリースは `develop` → `release/*` → `main` の順に、バージョンタグ付きで切り出される。

**使いどころ。** 厳格なリリース管理を行うチーム。四半期ごとのリリース、規制下のデプロイ、統合を続けながらの安定した本番環境など。エンタープライズや金融サービスでよく見られる。

**よくある問題。**
- 長命の `develop` は、リリースサイクルの間に `main` に対するマージの負債を蓄積する。リリースの切り出し時のマージが苦痛になる。
- ホットフィックスには二重マージ（`main` と `develop` の両方へ）が必要であり、2 つ目のマージを忘れやすい。

**ワークツリーの対応付け。** フィーチャーの Bolt: `--base develop --target develop`。ホットフィックスの Bolt: `--base main --target main`（そしてオペレーターが別途 `develop` にマージする。これは `aidlc-worktree` の範囲外）。戦略はブランチの履歴を保持するために通常 `merge` だが、`squash` も有効である。

### Execution runbook

GitFlow でディスパッチされた場合:

1. アクティブなスペースの `## Way of Working` を読む。統合ブランチの名前を探す（慣例は `develop` だが、`integration` や `next` を使うチームもある）。
2. フィーチャーの Bolt の場合: `--base <integration> --target <integration> --strategy <merge|squash>`。既定は `merge`。
3. ホットフィックスの Bolt の場合（Construction ではまれで、通常は帯域外のステージから起動される）: `--base main --target main --strategy merge`。`aidlc-worktree merge` が成功した後、オペレーターが別途ホットフィックスを `<integration>` にマージし戻す。ツールの範囲外である。
4. **作成**: `{{INVOKE}} engine worktree create --slug <bolt-slug> --base <integration>`。
5. **マージ**: 呼び出し元はメインのチェックアウトで `<integration>` 上にいなければならない。`{{INVOKE}} engine worktree merge --slug <bolt-slug> --target <integration> --strategy <merge|squash>`。
6. § Response contract に従って返す。ホットフィックスの場合は、オーケストレーターがフォローアップを表に出せるように `notes: "manual merge to <integration> required"` を含める。

### Failure modes

- **`<integration>` ブランチがローカルにない。** 監査前のエラーであり、ツールはブランチの捏造を拒否する。
- **マージ時の cwd が誤っている。** 防御的な HEAD チェックが失敗する: `expected branch <integration>, found <actual>`。呼び出し元は先にメインのチェックアウトに `cd` し、`git checkout <integration>` しなければならない。
- **2 つ目のターゲットへのホットフィックスのマージを忘れる。** `aidlc-worktree` の範囲外である。オーケストレーターによる aidlc-pipeline-deploy-agent のディスパッチでは、常に `notes` に 2 つ目のターゲットのリマインダーを含めるべきである。

---

## Release Branches

```
main         ────────────────────────────────►
              ▲                       ▲
              │                       │
release/v1.0 ┴──── (frozen for stabilisation) ──────►
  ▲   ▲
  │   │   bug fixes only on release branch
  │   │
  fix-A fix-B
              │
              └──► merge release/v1.0 → main + tag v1.0.0
```

**形。** `main` 上ではトランクベースまたは GitHub Flow で、コードフリーズ時にリリースブランチを切り出す。安定化の作業（バグ修正のみ）はリリースブランチで行い、新機能は `main` で継続する。

**使いどころ。** 継続的デプロイよりもリリースの安定性が重要な、バージョン付きのソフトウェアを出荷するチーム。デスクトップアプリ、組み込みソフトウェア、リリース日が固定されたエンタープライズ製品など。

**よくある問題。**
- リリースブランチでのバグ修正は、次のリリースで退行しないように、cherry-pick するか `main` にマージし戻さなければならない。
- 安定化の期間が長いと、リリースブランチのマージバックを待つ機能開発が止まることがある。

**ワークツリーの対応付け。** `main` 上の Bolt は `--base main --target main` を使う。リリースブランチの修正の Bolt は `--base release/vX.Y --target release/vX.Y` を使う。戦略は通常 `merge`（トレーサビリティのために修正ブランチを履歴に残す）。

### Execution runbook

Release Branches でディスパッチされた場合:

1. アクティブなスペースの `## Way of Working` を読む。リリースブランチのパターンを探す（慣例は `release/vX.Y`）。
2. Bolt のメタデータから、その Bolt がどのラインに属するかを判断する（オーケストレーターが `target_line: main | release/vX.Y` のヒントを渡す）。あいまいな場合の既定は `main`。
3. **作成**: `{{INVOKE}} engine worktree create --slug <bolt-slug> --base <line>`。
4. **マージ**: 呼び出し元はメインのチェックアウトで `<line>` 上にいる。`{{INVOKE}} engine worktree merge --slug <bolt-slug> --target <line> --strategy merge`。
5. Bolt がリリースブランチの修正だった場合は、応答に `notes: "consider cherry-pick to main"` を含める。相互のマージはオペレーターが扱う。
6. § Response contract に従って返す。

### Failure modes

- **リリースブランチがローカルにない。** 監査前のエラー。
- **Bolt がリリースブランチをターゲットにしたが、main が分岐している。** Bolt は完了する。オーケストレーターが `notes` フィールドを通じて cherry-pick のリマインダーを表に出す。
- 誤った cwd / ダーティなツリー / 競合のケースは **GitFlow と同じ**。

---

## Monorepo

```
main ────────────────────────────────────────►
  ▲   ▲   ▲   ▲
  │   │   │   │   feature branches with path-based scope
  │   │   │   │
  pkg-a/feat-1  pkg-b/feat-2  pkg-c/refactor  shared/lib-update
```

**形。** 複数のパッケージ/サービスを保持する単一のリポジトリ。ブランチはパスでスコープされる（`packages/auth/` 内の変更は 1 つの Bolt であり、パッケージをまたぐ変更には明示的なパッケージ間の調整が必要）。その上で、トランクベース、GitHub Flow、または上記のいずれでも運用できる。

**使いどころ。** サービスをまたぐアトミックな変更の恩恵を受ける、密結合した複数のサービスを持つチーム。ツールのサポートが必要: Nx、Turborepo、Pants。

**よくある問題。**
- CI はパスを認識しなければならない（変更のあるパッケージだけをテストする）。モノリシックな CI では意味がなくなる。
- パッケージをまたぐ変更はきれいに並列化できない。協調したマージが必要である。

**ワークツリーの対応付け。** 基盤となる戦略と同じ（既定はトランクベース）。パスの認識はワークツリー層ではなく CI/テスト層にある。戦略は通常、パッケージの変更ごとに `squash`。

### Execution runbook

Monorepo でディスパッチされた場合:

1. 基盤となる戦略（既定はトランクベース）を、上記のそのランブックに従って決定する。
2. `git worktree list` の出力が診断しやすいままになるように、Bolt の slug にはパッケージのスコープを含めるべきである（例: `feature-1` ではなく `auth-token-rotation`）。
3. 作成 + マージは基盤となる戦略と同一。
4. § Response contract に従って返す。

### Failure modes

- **パッケージをまたぐ Bolt。** Bolt のユニットが 2 つのパッケージにまたがる場合、マージは成功するが、cherry-pick や他パッケージとの調整のリマインドはオペレーターの仕事である。わかっていれば `notes` で表に出す。
- **基盤となる戦略と同じ。**

---

## Response contract

オーケストレーターがワークツリーの作成またはマージのために aidlc-pipeline-deploy-agent をディスパッチすると、エージェントは `aidlc-worktree` を直接呼び出し、下記の JSON エンベロープをオーケストレーターに報告する。その後、SKILL.md の Step 0.5 / Step 6.75 が、監査イベントが記録されたことを確認する決定的な安全策として `aidlc-worktree verify` を呼び出す。

### Create response (success)

```json
{
  "emitted": "WORKTREE_CREATED",
  "slug": "payments",
  "worktree_path": "/Users/dev/project/.aidlc/worktrees/bolt-7c31e9a0_payments",
  "branch": "bolt-7c31e9a0_payments",
  "base": "main",
  "audit_timestamp": "2026-05-18T12:34:56Z",
  "notes": "<optional follow-up reminders for the orchestrator>"
}
```

### Merge response (success)

```json
{
  "emitted": "WORKTREE_MERGED",
  "slug": "payments",
  "worktree_path": "/Users/dev/project/.aidlc/worktrees/bolt-7c31e9a0_payments",
  "target": "main",
  "strategy": "squash",
  "commit_sha": "<sha>",
  "audit_timestamp": "2026-05-18T12:34:56Z",
  "notices": ["<one line per thing the person should hear, when there is one>"]
}
```

取り込みは利用者自身の git で実行される。利用者のフック、フィルター、マージ
ドライバーは、利用者が行う任意のコミットと同様に実行される。取り込まれた内容が
レビュー済みのソースと異なる場合（pre-commit フック内のフォーマッター、smudge フィルター、設定された
マージドライバー）、取り込みはその結果を維持し、`notices` は利用者向けに 1 行を含み
（"Your pre-commit hook changed 3 files while landing Unit auth-mfa: ..."）、
`SWARM_SOURCE_MERGED` の行は、両方のコミットと並べて `Landed changes` と
`Driver merges` の下にファイルを記録する。その行は一度だけ伝える。拒否したり元に戻したりしてはならない。

マージのエラーが `[merge-succeeded:<sha>]` を含み、
`SWARM_SOURCE_MERGED` の結果後の監査行が失敗したと述べている場合、Git のマージはすでに
取り込まれているが、集約されたソースの権限は存在しない。ワークツリーを保持し、
同じマージコマンドを再試行してはならない。ステージの試行をやり直すか、
人間の明示的な承認を得た後にのみ `AIDLC_SKIP_SOURCE_FRESHNESS=1` を使う。

### Merge response (the person's hook refused the commit)

```json
{
  "status": "commit-refused",
  "slug": "payments",
  "worktree_path": "/Users/dev/project/.aidlc/worktrees/bolt-7c31e9a0_payments",
  "target": "main",
  "strategy": "squash",
  "staged": ["src/foo.ts"],
  "output": "lint: src/foo.ts has a console.log",
  "detail": "`git commit` refused the landing of Unit payments in /Users/dev/project; the landing is still staged there.",
  "next": "Fix what it named (edit and `git add` in /Users/dev/project, or pass --message for a commit-message rule), then run this same merge again: it finishes the staged landing. To drop the staged landing instead: `git reset --merge` in /Users/dev/project."
}
```

メインのチェックアウトは、取り込みがステージされた状態で git が残したとおりに保たれ、
ワークツリーは保持される。利用者のフックが取り込みを拒否したことと、その理由（`output` フィールドの言葉）を
1 行で利用者に伝え、ステージされた取り込みの中で指摘されたものを修正し、
同じマージを再実行する。それによって取り込みが完了し、
修正が `Landed changes` の下に記録される。修正できない場合は、停止して
`staged` に列挙されているものを伝える。

### Merge response (conflict)

```json
{
  "status": "conflict",
  "slug": "payments",
  "worktree_path": "/Users/dev/project/.aidlc/worktrees/bolt-7c31e9a0_payments",
  "conflict_files": ["src/foo.ts", "src/bar.ts"],
  "detail": "Unit payments's landing clashes with what is already on main: it is staged in /Users/dev/project, and git could not merge 2 files: src/foo.ts (changed on both sides), src/bar.ts (changed on both sides).",
  "next": "Combine both sides in those files and `git add` them in /Users/dev/project, then run this same merge again: it finishes the landing. To drop the landing instead: `git reset --merge` in /Users/dev/project."
}
```

取り込みは、git が残したとおりメインのチェックアウトにステージされたままで、ワークツリー
は保持される。各ユニットがそのプランで求めたものを保持できるように、列挙された各ファイルで両側を統合し、
`git add` して同じマージを再実行する。それによって
取り込みが完了し、統合したファイルが `Landed changes` の下に記録され、それらを示す
通知が 1 つ返される。統合する前に実行すると、この応答が
再び返される。rebase による取り込みの競合はワークツリー内にあり、その `detail` がそう述べる。完全なプロンプトの形と保持の不変条件については、`aidlc-common/protocols/stage-protocol-construction.md` の § "Halt-and-ask on failure" と `skills/aidlc/SKILL.md` の § "Halt-and-ask failure handling" を参照。

### Discard response

```json
{
  "emitted": "WORKTREE_DISCARDED",
  "slug": "payments",
  "worktree_path": "/Users/dev/project/.aidlc/worktrees/bolt-7c31e9a0_payments",
  "reason": "agent-discard",
  "parked_ref": "refs/aidlc/parked/7c31e9a0/payments/20260518T123456Z",
  "parked_commit": "<snapshot-commit>",
  "parked_stamp": "20260518T123456Z",
  "parked_mode": "snapshot",
  "parked_repo": null,
  "audit_timestamp": "2026-05-18T12:34:56Z"
}
```

チェックアウト、ブランチ、レビュー済みのソースの ref がすでに存在しない場合（冪等な経路）、`emitted` は `null`、`reason` は `already-discarded` となり、監査イベントは再発行されない。
その already-discarded の応答には `parked_*` フィールドがない。新しい discard では
`parked_ref` と `parked_commit` を保持し、正確な `parked_stamp`、`parked_mode`
（`snapshot`、`branch-tip`、`evidence-only` のいずれか）、`parked_repo`（プロジェクトルートでは
`null`、それ以外では兄弟リポジトリの名前）を追加する。

discard は、一時的なインデックスと `commit-tree` を使って、追跡ファイルと無視されていない未追跡ファイルのスナップショットを取る。
無視されている未追跡ファイルはバックアップされない。監査の
発行の前に、`parked_ref` の名前空間の下に、`/head`、同じコミットを指す `/snapshot` マーカー、
`/reviewed-source/<commit>` の ref を退避させる。
チェックアウトがすでに存在しないがブランチは残っている場合は、`/head` は代わりにそのブランチの先端を保持し、
`/snapshot` ではなく対応する `/branch-tip` マーカーを付ける。
その後にのみ、ライブのチェックアウトと
ブランチを削除し、元のレビュー済みソースの ref を比較削除する。レビュー済みの
ref だけが残っている場合、`parked_mode` は `evidence-only` で、`parked_commit` は文字列 `"-"` である。
`parked_ref`、`parked_stamp`、`parked_repo` は引き続き保存された証跡を識別するが、
復元する `/head` はない。

### Abort response (success)

`bolt abort` の成功時の JSON には、`emitted: "BOLT_FAILED"`、`reason: "aborted"`、
追加の `abort_reason` フィールドに入る指定された `--reason` のテキスト、`failed_bolt`、
`slug`、`discarded`、`parked_ref` が含まれる。何も退避されなかった場合（
`--discard` なしの場合を含む）、`parked_ref` は `null` である。null でない `parked_ref` の場合にのみ、返される
`parked_stamp`、`parked_mode`、`parked_repo` が追加される。
復元可能な試行では、`restore_operation` はルート `worktree` と次の引数を持つ:
`["restore", "--slug", slug, "--parked", stamp, "--repo", repo ?? ".", "--intent", recordDirName, "--space", space]`。
リポジトリのセレクターは常に存在する。兄弟リポジトリの名前、または
`parked_repo` が `null` の場合は `.` である。`recordDirName` と `space` は所有するインテントを識別するため、
アクティブなインテントが変わっても保存された操作がずれることはない。
`restore_hint` は任意の人間向け表示テキストで、
その操作から `renderEngineInvocation` によってレンダリングされる。インストール済みのネイティブまたは
ソースのプレフィックス、ハーネスの検証、シェルセーフな引数のクォートを使う。レンダリングが
例外を投げた場合、ヒントは省略され、`restore_hint_error` がその理由を持つ。
型付きの操作は引き続き利用できる。ヒントがないことは evidence-only の状態を意味しない。
スナップショットモードは
`parked_excludes: ["ignored files", "eol/text=auto normalization"]` を報告する。branch-tip
モードは代わりに `["uncommitted files (no working tree existed)"]` を報告する。
evidence-only モードは 4 つの記述子フィールドを保持するが、
復元可能なファイルが保存されていないため、`restore_operation`、`restore_hint`、`restore_hint_error`、`parked_excludes` を省略する。

保存された名前空間はわかっているが、その discard の記述子が欠けている場合、
フォールバックは `parked_ref` を保持し、スタンプが厳密に解析できる場合にのみ、
その名前空間から `parked_stamp` を導出する。そうでない場合、`parked_stamp` は `null` である。
`parked_mode: null` と `parked_repo: null` を報告する。どちらもレガシーな出力からは
推測できないためである。また `restore_operation`、`restore_hint`、
`restore_hint_error`、`parked_excludes` を省略する。代わりに、`recovery_hint` が
人間に doctor を実行して退避された試行とその正確な復元コマンドを一覧するよう求める。
このヒントは単なるガイダンスであり、実行可能な操作ではない。モードが不明であることは、
ファイルが保存されたという主張や復元の提案の根拠にならない。名前空間が保存されなかった場合、
`parked_ref` は `null` で、`parked_stamp`、`parked_mode`、`parked_repo`、
`restore_operation`、`restore_hint`、`restore_hint_error`、`parked_excludes`、
`recovery_hint` はいずれもない。監査の行は
引き続き `Reason: aborted` を使い、成功時の JSON は呼び出し元のテキストを
`abort_reason` に保持する。
中止コマンドや、それに必要な人間の同意の要件を変更してはならない。人間には、
試行が **set aside**（退避）されたと伝える。branch-tip モードでは次のように言う: "I kept its committed
work; there were no uncommitted files to save."（コミット済みの作業は保持しました。保存すべき未コミットのファイルはありませんでした。）evidence-only モードでは次のように言う:
"Nothing of its working files remained to save; only its review evidence was
kept."（保存すべき作業ファイルは何も残っていませんでした。レビューの証跡だけを保持しました。）最後の復元の提案を省略するのは、`restore_operation` がない場合だけである。
後で人間が復元を求めたら、その `worktree` ルートを
`{{INVOKE}} engine worktree <args...>` を通じて呼び出し、保存された各引数をそのまま
別々の argv 引数として渡す。引数を結合してシェルコマンドにしたり、
表示専用のヒントを実行したり、slug のみの選択を作り直したりしてはならない。レンダリングのエラーは
復元の提案を取り下げる理由にならない。返された `worktree_path` をそのまま伝える。

### Restore or purge set-aside work

`{{INVOKE}} engine worktree restore --slug <bolt-slug> [--parked <stamp>]
[--raw] [--repo <name|.>] [--intent <intent>] [--space <space>]` は、選択された
試行、または `--parked` を省略した場合は最新の退避された `/head` を復元する。スタンプは UTC の
`YYYYMMDDTHHMMSSZ` を使い、任意で数値の `-N` 衝突接尾辞を持つ。最新の選択では、
それらの接尾辞を数値順に並べる。復元は
`.aidlc/restored/bolt-<id8>_<slug>-<stamp>` を
`restore/bolt-<id8>_<slug>-<stamp>` 上に作成し、ライブの
`.aidlc/worktrees/bolt-<id8>_<slug>` パスや `bolt-<id8>_<slug>` ブランチには決して触れない。レガシーな
復元はレガシーな名前を保持し、選択されたインテントに記録された
discard の出自を必要とする。復元は、中止されたライフサイクルを再開したり、
アクティブなレビューの権限を復元したりしない。
その JSON は `{restored: true, slug, parked_ref, worktree_path, branch,
reviewed_source_refs, raw_bytes, restore_mode}` で、raw モードでのみ `materialized` が追加される。
`reviewed_source_refs` は、保持された退避済みのレビュー済み ref の数であり、それらはアクティブな名前空間にはコピーされない。
1 つのリポジトリにのみ存在する正確なスタンプは、
一般的な slug のあいまいさより先にそのリポジトリを選択する。evidence-only のストレージを選択すると、`--raw` を付けても、
`no restorable files were parked for <slug> <stamp>; only review evidence was kept` で拒否される。
分類では、まず単独の `--raw` フラグ（`restore_mode: "raw-requested"`）、
次に `/snapshot`（`"snapshot"`）、次に `/branch-tip`（`"branch-tip"`）を確認する。レガシーな退避は、
どちらの形でもいずれのマーカーも持たない。マーカーのない head がスナップショットとみなされるのは、
コミットの作成者が正確に `AI-DLC`、メールアドレスが `aidlc@localhost`、件名が
`aidlc: parked bolt-<slug> at ` で始まる場合だけである（`"legacy-snapshot"`）。そうでなければ
ブランチの先端である（`"legacy-branch-tip"`）。この識別チェックは Git の置換オブジェクトを無視する。
スナップショットと明示的な `--raw` は `raw_bytes: true` を報告し、smudge/process
フィルターと working-tree-encoding の変換をバイパスし、通常ファイルの blob を
直接ディスクにストリームする。バッファされるのはシンボリックリンクのターゲットだけである。
`materialized` は、書き込まれた通常ファイルとシンボリックリンクの数で、サブモジュールの gitlink は含まない。
どちらの branch-tip モードも Git の通常のチェックアウトを使い、フィルターとエンコーディングの
変換を適用して、`raw_bytes: false` を報告する。必須のフィルターが失敗すると、
復元は Git のメッセージとともに失敗する。`--raw` は、残っている
復元用のチェックアウトとブランチが削除された後に変換をバイパスする。

シンボリックリンクは、`core.symlinks` が未設定または true の場合はシンボリックリンクとして実体化される。
`core.symlinks=false` の場合、mode-120000 のエントリは、Git がチェックアウトするのとまったく同じく、
リンクターゲットをバイト列とする通常ファイルとして書き込まれる。
raw で復元されたフィルター対象のパスは、自身のフィルターの下で変更済みに見えることがある。サブモジュールの
gitlink は空のディレクトリになり、復元されたサブモジュールのチェックアウトにはならない。退避時の Git の
eol/`text=auto` の正規化は制約として残る。正規化された CRLF の
バイト列は回復できない。無視された未追跡ファイルと合わせて、これがスナップショットモードの
明示的な `parked_excludes` の境界である。ignore パターンに一致する追跡ファイルは
含まれたままになる。UTF-8 でない名前を持ち、`filter`、`text`、
`eol`、`ident`、`working-tree-encoding` のいずれかの属性（未指定でも
unset でもないもの）を持つ通常ファイルは、現在退避できない。discard は削除処理の前に次のメッセージで拒否する:

```text
cannot park file with a non-UTF-8 name and a content-transforming attribute (<attr>=<value>): <name>; rename the file or unset its <attr> attribute
```

`{{INVOKE}} engine worktree purge --slug <bolt-slug> [--parked <stamp> |
--older-than <days>] [--repo <name|.>] [--intent <intent>] [--space <space>]`
は、一致するすべての退避済み ref を比較削除する。セレクターがない場合は、
選択されたインテントの Bolt のすべてのスタンプを削除する。`--parked` は 1 つの正確なスタンプを選択する。
`--older-than` は、小数を含む非負の有限の日数を受け付け、
しきい値より厳密に古い試行だけを選択する。経過日数は、スタンプの
UTC の `YYYYMMDDTHHMMSSZ` 部分から求め、数値の `-N` 衝突接尾辞や
コミット日付は使わない。共有の厳格なカレンダーパーサーは、ありえない日付や
時刻を正規化せずに却下する。経過日数で絞り込む purge は、解析できない
スタンプを保持し、`skipped_unparseable: [stamps]` で報告する。2 つのセレクターは
相互に排他的である。purge は、対応する復元済みのチェックアウトが
存在する間、または Git に登録されたままである間（移動されたチェックアウトを含む）は拒否する。その JSON は
`{purged: <number-of-refs>, slug, stamps: [...], skipped_unparseable: [...]}` である。
`skipped_unparseable` は常に存在し、`--older-than` が
解析できないスタンプを保持した場合を除いて空である。正確なスタンプの purge とすべてのスタンプの purge は、それらを削除できる。
restore と purge では、`--repo <name>` は既存の兄弟 Git リポジトリを選択し、
`--repo .` はプロジェクトルートを選択する。これは現在のインテントの
リポジトリリストとは独立している。`WORKTREE_CREATED` と `WORKTREE_DISCARDED` はどちらも `Repo` を発行する。
記録された兄弟リポジトリの名前、またはプロジェクトルートの場合は `-` である。discard の行は、
作成の行が利用できない場合でも、この出自を保持する。復旧では、同じ slug の作成または discard の監査の `Repo` フィールドに
記載された有効な Git リポジトリを、その兄弟の名前がシンボリックリンクであっても受け入れる。
フレームワークは、試行のワークツリーや退避を記録した場所でのみ、正確に
復旧できる。この受け入れは slug にスコープされる。他の
slug の記録が、この slug のリポジトリの集合を広げることは決してない。インテントへの所属だけでは、
現在のものであれ過去のものであれ、シンボリックリンクを受け入れられない。インテントのリストの候補と、
記録されていない検出された兄弟は、正規のパスが正規のワークスペースルートの直下に
とどまる、実在する直接の子ディレクトリでなければならない
（`isWorkspaceRepoDir`）。同じ slug の監査の出自を持たない任意のパスや
シンボリックリンクのエイリアスは拒否される。これらのセレクターは、ライブの
create/discard の振る舞いを変えない。restore と purge は、選択や変更の前に、不明なフラグと重複した
フラグを却下する。`--raw` は restore 専用の単独のフラグである。

doctor は、保存された `/head` のエントリと、実際の
レビュー済みソースの ref を含む evidence-only の名前空間を、警告や失敗としてではなく情報として一覧する。slug、スタンプ、
経過日数、モード（`snapshot`、`branch-tip`、`legacy`、`evidence-only` のいずれか）、
正規の復元済みチェックアウトの有無、そして正確な slug とスタンプについて、明示的なリポジトリセレクターを含み、
その後に `--intent <record-dir-name> --space <space>` が続く型付きの復旧操作である。すべてのエントリは `purge_operation` を持ち、
復元可能なエントリだけが `restore_operation` を持つ。それぞれがルート `worktree` と
正確な argv の引数を持ち、シェルのプログラムであることは決してない。任意の
`restore_command` と `purge_command` は、人間向け表示のための安全なレンダリングである。
レンダリングが例外を投げた場合、対応するコマンドは省略され、
`restore_command_error` または `purge_command_error` がその理由を持つが、
操作は残る。evidence-only のエントリは、purge の操作と、その
コマンドまたはエラーのフィールドだけを持つ。doctor は、上記と同じ slug にスコープされた監査の出自と、
実在する直接の子という境界を適用する。インテントのリポジトリリストだけでは
シンボリックリンクは受け入れられず、別の slug の記録でも受け入れられない。移動されたチェックアウトは
そこで復元済みとして表示されないことがあるが、purge は引き続きその Git の登録を確認する。doctor と
purge は厳格なスタンプパーサーを共有する。ありえない日付や時刻は、
doctor の JSON では `age_days: null`、人間が読める出力では `unknown` になる。restore と
purge は監査イベントを追加しない。JSON の例と復旧の契約については
`aidlc-shared/worktree-info-schema.md` を参照。

doctor は、名前空間付きの試行とレガシーな試行の両方を棚卸しする。名前空間付きの試行は、
レジストリ UUID の接尾辞を通じて所有するインテントを解決する。レガシーな試行は、
所有者の正確な `WORKTREE_DISCARDED` の `Parked ref` の出自を必要とする。不明または
あいまいな所有者や、帰属のないレガシーな退避は省略され、たまたまアクティブなインテントを通じて
許可されることは決してない。保存された操作は、その
所有者の記録ディレクトリの名前とスペースを持つ。
---

## How AIDLC reads strategy from team practices

このセクションで説明するディスパッチのプロトコルは、**SKILL.md Step 0**（ワークツリーの作成）と **Step 6.5**（ワークツリーのマージ）で実装されている。`aidlc-bolt complete --merge` はディスパッチの前後を調整する（forkState のマージバック、forkAudit のマージバック）が、`aidlc-worktree merge` を直接呼び出しはしない。ディスパッチは SKILL.md の散文の中にある。

Bolt が開始する（Step 0）または完了する（Step 6.5）とき、オーケストレーターは 2 つの入力とともに **aidlc-pipeline-deploy-agent** への Task 呼び出しをディスパッチする:

1. `aidlc/spaces/<active-space>/memory/{project,team,org}.md` から解決された `## Way of Working` の記述（フォールバックチェーンは `shared/rules-reading.md` にある）。
2. Bolt のメタデータ（slug、ソースブランチ、リリースブランチを使うチーム向けの任意のターゲットラインのヒント）。

エージェントはこのファイル（`branching-strategies.md`）をメニューとして読み、チームが述べた戦略を上記の 5 つのいずれかに対応付け、適切な `aidlc-worktree` のフラグを選び、ツールを呼び出し、§ Response contract に従って応答のエンベロープを返す。

チームが述べた戦略がメニューにきれいに対応しない場合（例: 「ハイブリッドを使っている」）、エージェントは最も近いものを選び、応答の `notes` フィールドに逸脱を記載する。オーケストレーターはそれを監査ログに表に出す。

`project.md`、`team.md`、`org.md` のいずれもブランチのプラクティスを提供していない場合、エージェントはハードコードされた既定値（squash を使うトランクベース、ベース `main`、ターゲット `main`）を適用し、`PRACTICES_SECTION_EMPTY`（参考情報のみ）を発行する。

---

## Quick decision matrix

| 求めるもの | 使うもの |
|---|---|
| 新規プロジェクトの既定 | Trunk-Based |
| 長命のブランチを伴う OSS スタイルの PR | GitHub Flow |
| エンタープライズのリリース管理 | GitFlow |
| 安定化期間を伴うバージョン付きのリリース | Release Branches |
| 1 つのリポジトリに複数のサービス | Monorepo（上記のいずれかの上で） |

迷ったら Trunk-Based を選ぶ。最もオーバーヘッドが小さく、CI/CD エコシステムのサポートが最も充実した戦略であり、AIDLC の Construction ワークツリーはこれを既定として設計されている。
