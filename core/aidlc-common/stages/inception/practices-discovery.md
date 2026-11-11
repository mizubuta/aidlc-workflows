---
slug: practices-discovery
phase: inception
execution: CONDITIONAL
condition: Always rerun for freshness. Brownfield discovers from evidence + reverse-engineering artifacts. Greenfield prompts user via structured questions using org.md defaults.
lead_agent: aidlc-pipeline-deploy-agent
support_agents:
  - aidlc-quality-agent
  - aidlc-developer-agent
  - aidlc-devsecops-agent
mode: subagent
summary_confirmation: required
produces:
  - team-practices
  - discovered-rules
  - evidence
  - practices-discovery-timestamp
consumes:
  - artifact: code-structure
    required: false
    conditional_on: brownfield
  - artifact: technology-stack
    required: false
    conditional_on: brownfield
  - artifact: dependencies
    required: false
    conditional_on: brownfield
  - artifact: code-quality-assessment
    required: false
    conditional_on: brownfield
  - artifact: architecture
    required: false
    conditional_on: brownfield
  - artifact: business-overview
    required: false
    conditional_on: brownfield
requires_stage:
  - state-init
  - reverse-engineering
sensors:
  - required-sections
  - upstream-coverage
scopes:
  - enterprise
  - feature
  - mvp
  - infra
  - classic
  - workshop
inputs: <record>/aidlc-state.md + (brownfield) reverse-engineering evidence
outputs: "team-practices.md, discovered-rules.md, evidence.md, practices-discovery-timestamp.md, plus one contribution file per support agent. On affirmation, content is promoted to aidlc/spaces/<active-space>/memory/team.md and project.md."
---

# Practices Discovery

このステージでは、チームの働き方を明らかにする: 作業の進め方（way of working）、ウォーキングスケルトンに
対する方針、テストの姿勢、デプロイ、コードスタイル。これはハブアンドスポーク型の
アンサンブルである。pipeline-deploy のリードが草案を作り、quality、developer、devsecops が
それぞれ独立に草案を精査し、人間がプラクティスの選択を確定させ、
リードが結果を統合する。

確定（affirmation）ゲートでは、決定的なツールが、確定した内容を
アクティブなスペースの `memory/team.md` と `memory/project.md` に昇格させる。その昇格が成功するまで、
人間の承認はコミットされない。

## Steps

### Step 1: Check Conditions

`<record>/aidlc-state.md` を読み込み、プロジェクトの種類とアクティブなスペースを判定する:

- **Brownfield:** 利用可能なリバースエンジニアリングの成果物とワークスペースの
  設定をエビデンスとして使う。
- **Greenfield:**
  `aidlc/spaces/<active-space>/memory/org.md` をデフォルトのプラクティスの情報源として使う。

`aidlc/spaces/<active-space>/memory/team.md` にすでに確定済みの内容がある場合は、
どちらのプロジェクト種類でも、それを再実行時のコンテキストとして使う。Step 2〜8 は
両方のプロジェクト種類で実行する。

プロジェクトの種類を理由にこのステージをスキップしてはならない。スキップするのは、アクティブな
スコープのコンパイル済み計画で `practices-discovery` が `SKIP` とされている場合だけである。

### Step 2: Lead Draft (Always)

最初のターンを `aidlc-pipeline-deploy-agent` に委任する。リードは自身の
ペルソナとナレッジを自分で読み込む。ペルソナの文章を貼り付けるのではなく、パスを渡す。

- **Brownfield:** git の履歴、CI/デプロイの設定、利用可能な
  リバースエンジニアリング成果物のパスを調べる。ブランチ戦略、
  デプロイの頻度、環境のトポロジー、目に見えるチームの慣習を推測する。
- **Greenfield:** `aidlc/spaces/<active-space>/memory/org.md` から対応する 5 つのセクションを読み込み、
  確立されたチームの事実としてではなく、提案される
  デフォルトとして扱う。
- **Re-run:** `aidlc/spaces/<active-space>/memory/team.md` 内の対応する空でないセクションを、
  現在の確定済みベースラインとして読み込む。

リードは、宣言された 4 つの成果物すべての初版を
`<record>/inception/practices-discovery/` 配下に書く。タイムスタンプの成果物は、最終統合までは
草案のままである。これらの宣言済み成果物を編集するのはリードだけである。

### Step 3: Blind Support Review

ディレクティブの `support_agents` に列挙されたコラボレーターだけを、過不足なくディスパッチする。
ハーネスが並列委任をサポートしている場合は、1 つの並列バッチとして行う。各ブリーフには、
ステージのパス、リードの草案のパス、関連するエビデンスのパスだけを含める。
どのブリーフやコンテキストにも、他のスポークの貢献内容を含めてはならない: スポーク同士は
互いの内容を見ない（mutually blind）。

**`directive.support_agents` が空の場合、このステップは丸ごとスキップする** — この
スコープでは collaborators のスイッチが off であり、リードが単独で作業する
（`stage-protocol-ensemble.md` §5）: リードの草案がそのままプラクティスとなり、
インタビューでそれらを確定させ、スポークによるレビューは行わない。全員がそろう場合の
顔ぶれは次のとおり:

- **aidlc-quality-agent** - テストの姿勢、カバレッジツール、CI の
  品質ゲート、テスト/コードのパターン、インタビューで解決すべきギャップを評価する。
- **aidlc-developer-agent** - 命名、レイヤー境界、エラー処理、
  ファイル構成、コードスタイルの慣習を評価する。
- **aidlc-devsecops-agent** - lint/フォーマットのルール、SAST/DAST、シークレットと
  依存関係のスキャン、サプライチェーンのコントロールを評価する。

ディスパッチされた各支援エージェントは、次のファイルを書く:

`<record>/inception/practices-discovery/contributions/<agent-slug>.md`

1 行目は必ず `**Collaborator:** <agent-slug>` とし、その後に
`stage-protocol-collaborators.md` の定義どおり `## Contribution` と `## Positions` を続ける。
インタビューの前に、ディスパッチしたすべてのコラボレーターのファイルを回収する。
ファイルの存在と識別マーカーは、エンジンが検査する決定的な完了エビデンスである。

### Step 4: Interview (Always)

`<record>/inception/practices-discovery/practices-discovery-questions.md` を
作成し、`memory/team.md` の 5 つのセクション（Way of
Working、Walking Skeleton、Testing Posture、Deployment、Code Style）について構造化された質問を提示する。

- **Brownfield:** リードの草案と独立したレビューで確定できなかった点だけを
  質問する。エビデンスは回答を示唆できるが、チームの意図は
  あくまで人間が判断する。
- **Greenfield:** 5 つの領域すべてを質問し、対応する `memory/org.md` の
  セクションを回答の候補として使う。
- **Re-run:** 対応する `memory/team.md` の内容をデフォルトとして示す。

Way of Working の回答によって、チームの作業がマージされる 1 本のブランチが決まる
（Step 5）。どこにも示されていない場合は、`memory/org.md` が示すブランチとする。
どちらかを質問するのは、1 つの回答が 2 本のブランチを挙げていて、作業がどちらにマージされるのか
判断できない場合だけである。

回答の候補を引き出す元の `memory/*.md` のセクションは、
このフレームワーク独自の解決ルールのために書かれているため、回答する利用者が
知っているはずのない用語を含んでいる。そこから 2 つの義務が生じ、それは選択肢だけでなく
質問文にも同様に適用される:

- **セクションの言葉ではなく、利用者の言葉で質問する。** セクションの言い回しは
  質問を作るための入力であり、決して質問そのものではない。"Walking Skeleton" はあるプラクティスの
  名前であり、"Should we build a thin end-to-end slice first?"（まずエンドツーエンドの薄いスライスを作りましょうか？）は
  誰でも答えられる質問である。提示する内容からフレームワークのプロセス用語を取り除く。
- **専門用語は、初出時に質問文そのものの中で説明する。** ユーザーが知らない可能性のある名前を
  持つプラクティスには、それを定義する 1 文節を付ける。説明は選択肢の 1 つに埋め込むのではなく
  質問の行に置き、選択の前に定義が読まれるようにする。Walking Skeleton の領域では、
  **"Build a thin end-to-end slice first? A walking skeleton is a minimal
  version that runs the whole way through, built first to prove the pieces
  connect before the real features go in."**（まずエンドツーエンドの薄いスライスを作りますか？
  ウォーキングスケルトンとは、端から端まで一通り動く最小限のバージョンで、本来の機能を入れる前に
  各部品がつながることを証明するために最初に作るものです。）のように質問し、その下に yes/no の選択肢を
  示す。2 回目以降の言及では説明は不要である。

標準の（ゲート以外の）質問フローに従い、インタビューの各質問は提示する前に `aidlc-log.ts decision` で、
各回答は応答を受け取った後に `aidlc-log.ts answer` で記録する。

### Step 5: Lead Integration

最終統合のターンを `aidlc-pipeline-deploy-agent` に委任する。リードの
草案のパス、Step 3 で生成されたすべての貢献ファイルのパス（リードが単独で作業する場合はなし）、
回答済みのインタビューファイルを渡す。宣言された 4 つの成果物を更新するのはリードだけである:

1. **team-practices.md** - `memory/team.md` に対応する 5 つのセクション
   （`## Way of Working`, `## Walking Skeleton`, `## Testing Posture`,
   `## Deployment`, `## Code Style`）を、チームの視点の文体で書く。`## Testing Posture` には
   必ず次を含める:
   - `- **Methodology**: tdd | bdd | atdd | test-after | custom`（これらの値のいずれか
     1 つだけを書き、ほかには何も書かない。理由は
     `- **Methodology evidence**: ...` の箇条書きに書く）
   - `- **Ordering**: <the affirmed ordering in one explicit sentence>`

   回答が複数の進め方を組み合わせている場合（たとえば、実装前に BDD シナリオを書き、
   実装後に下位レベルの単体テストを書く場合）は、必ず `custom` を使う。
   カバレッジ、ツール、テストの種類、範囲に関するメモは追加の箇条書きとして残す。
   それらは 2 つの構造化フィールドの代わりにはならない。

   `## Way of Working` には必ず `- **Integration branch**: <branch>` を含める: これは
   作業のマージ先として利用者が確定した 1 本のブランチであり、git での名前だけを書き、
   ほかには何も書かない（たとえば `main`）。再実行時に `memory/team.md` が
   それを文章の中でしか述べていない場合は、その文章が意味するブランチを書く。ツールが
   読むのはこの行であり、その横の散文ではない。
2. **discovered-rules.md** - 人間が明言した厳格な制約についてのみ、`## Mandated` のルールを
   `ALWAYS ...` 形式で、`## Forbidden` のルールを `NEVER ...` 形式で書く。
   各ルールは 1 つのリスト項目（`- ALWAYS ...`）とする。ルールのない見出しには
   リスト項目を置かず、`None.` だけを書く。
3. **evidence.md** - 各参加者が調べたことや推測したこと、
   インタビューでの決定事項、未解決の不確実性。
4. **practices-discovery-timestamp.md** - 次の 1 行:
   `Discovered: <ISO-8601 timestamp> at commit <hash>`.

統合後、`PRACTICES_DISCOVERED` を発行する:

```bash
{{INVOKE}} engine state practices-event \
  --type discovered \
  --field 'Sources Scanned: <list>' \
  --field "Drafts: team-practices.md, discovered-rules.md"
```

### Step 6: Learnings + Affirmation Gate

`directive.protocol_modules` に `learnings` が含まれている場合にのみ、セクション 13 の学びの手順を実行し、その後で以下の確定ゲートに従う。モジュールがない場合は、直接そのゲートに進む:

1. 質問の前にゲートを開く:
   `{{INVOKE}} engine orchestrate report --stage
   practices-discovery --result awaiting-approval`.
2. 確定ゲートを `aidlc-log.ts decision` や
   `aidlc-log.ts answer` で記録しない。その監査イベントはライフサイクルの `report` 呼び出しが担う。
3. `team-practices.md` と `discovered-rules.md` を、2 つの選択肢とともに提示する:
   **Approve**（昇格させてから次のステージへ進む）と
   **Request Changes**。Approve の選択肢の説明には、実際の次のステージ名を書く。これは
   ゲートを開いた応答の `next_stage` フィールドから読み取り、なければ run-stage ディレクティブのもの
   （null の場合は `Complete workflow`）を使う。フィールド名をユーザーに見せてはならない。
4. 停止（STOP）し、人間の応答を待つ。
5. 応答を読み、以下の対応する `report` または昇格の手順を実行する。
   このゲートでは決して `aidlc-log.ts answer` を呼び出さない。
6. Request Changes の場合は、`--result rejected --user-input "Request Changes"` を report する
   （利用者の言葉は記録とともに保存される。補足したい場合にのみ `--reason` を追加する）。
   リードを通じて修正し（支援エージェントの再実行は、そのエビデンスを更新する必要がある場合に限る）、
   ゲートを再提示する前に `--result revised` を report する。
   却下されると、それ以前の昇格の受領記録はすべて無効になる: 却下後に Step 7 の昇格が再実行されるまで
   エンジンは `approved` を拒否するため、後で
   Approve された場合は、必ず修正後の草案を改めて昇格させなければならない。
7. Approve の場合は、まだ `approved` を report しない。同じ応答ターンのまま
   Step 7 に進む。

### Step 7: Promote (On Approve Only)

オーケストレーターは、アクティブなスペースのメモリを直接編集しない。次を実行する:

```bash
{{INVOKE}} engine state practices-promote \
  --team-practices <record>/inception/practices-discovery/team-practices.md \
  --discovered-rules <record>/inception/practices-discovery/discovered-rules.md \
  --affirming-user '<user>'
```

このサブコマンドはアクティブなスペースを解決したうえで、次を行う:

- メモリへの書き込みの前に、有効なすべての支援エージェントの貢献内容とその識別マーカーを
  再検証する（リードが単独で作業する場合はなし）。
- 両方の草案と
  `aidlc/spaces/<active-space>/memory/{team,project}.md` を読み込む。
- `team.md` 内の対応する 5 つのセクションを置き換える。
- タイムスタンプ付きの厳格な制約を、`project.md` の `## Mandated` と
  `## Forbidden` の下に追記する。
- `project.md` を先に、`team.md` を後に書き込む。
- 成功時は `PRACTICES_AFFIRMED` を発行して `Practices Affirmed Timestamp` を
  状態に記録し、失敗時は `PRACTICES_OVERRIDE` を発行する。

コマンドが 0 以外で終了した場合は停止する。承認を report したり、先へ進んだりしてはならない。
昇格が成功するまで、ステージは開いたゲートの状態にとどまる。

### Step 8: Commit Approval

Step 7 が `{"emitted":"PRACTICES_AFFIRMED",...}` を出力して 0 で終了した後:

1. `PRACTICES_AFFIRMED` を再度発行しない。
2. 保留していた承認をコミットする:
   `{{INVOKE}} engine orchestrate report --stage
   practices-discovery --result approved --user-input "Approve"`.

stage-protocol.md の完了テンプレートを使う:

- 4 つの成果物すべて、生成された貢献ファイル（リードが単独で作業する場合はなし）、
  2 つの昇格先を要約する。
- レビュー対象のパスとして `<record>/inception/practices-discovery/` を使う。
- 次のステージ名は、ゲートを開いた応答の `next_stage` から、なければ `directive.next_stage` から示す。

## Sensors

このステージで宣言された出力は、
`<record>/inception/practices-discovery/` 配下の Markdown 成果物である。

インポート: `required-sections`, `upstream-coverage`.

上流の対象: `code-structure`, `technology-stack`, `dependencies`, `code-quality-assessment`, `architecture`, `business-overview`.

Brownfield の上流の対象は条件付きである。greenfield の
ワークスペースに存在しない入力は、カバレッジの欠落とはみなさない。

## Learn

`directive.protocol_modules` に `learnings` が含まれている場合は、
`stage-protocol-learnings.md` に従う。作業中は `directive.memory_path` に日誌を付け、
承認ゲートの前に学びのステップを実施する。そのモジュールが、利用者に何かを尋ねるタイミングを定めている。
モジュールがない場合は、日誌とステップの両方を省略する。
