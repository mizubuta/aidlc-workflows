# Workflow Profiles

AI-DLC は、すべてのタスクに同じライフサイクルを強制するわけではありません。よくある種類の作業に向けて、
一連の **ワークフロープロファイル** を提供しています。完全な機能開発、素早いバグ修正、
インフラ変更、軽量な Express 実行などです。

エンジンはワークフロープロファイルを **スコープ** と呼びます。この 2 つの用語は、同じ
選択を異なる視点から表したものです:

- **ワークフロープロファイル** は利用者から見た体験です。どのような種類の作業を
  行っていて、どの程度の手続きが必要かを表します。
- **スコープ** は `aidlc-state.md` に保存されるエンジンの設定です。そのワークフローで使われる、正確なステージの
  ルート、深さ、テスト戦略、レビューの上限、手続きのスイッチを表します。

`/aidlc <profile>` でプロファイルを明示的に選ぶか、作業内容を説明して
AI-DLC に提案させます。アシスタントはあなたの説明を読み、それに合う
プロファイルを提案します。どれも合わない場合はアダプティブコンポーザーを提案します。
何かが始まる前に、あなたがルートを確認します。明示的に指定したプロファイルは、
指定したとおりに使われます。その後に何を構築するかを続けて書くと、すぐに開始してステージ数と
ゲート数を表示します。プロファイル名だけを指定すると、まず何のための作業かを尋ねます。

## Quick chooser

| ワークフロープロファイル | 最適な用途 | ステージ数 | 深さ | テスト戦略 | 開始方法 |
|------------------|----------|--------|-------|---------------|------------|
| **Classic** | Inception と Construction を通した V1 スタイルの手続きで、Build and Test で終了する | 15 / 30 | Standard | Standard | `/aidlc classic` |
| **Express** | 要件からコードとテストまでの最も軽量な経路 | 7 / 30 | Minimal | Minimal | `/aidlc express` |
| **Feature** | 完全なライフサイクルを使う本番機能 | 30 / 30 | Standard | Standard | `/aidlc feature` |
| **Enterprise** | 完全なトレーサビリティを備えた、規制対象または高保証の作業 | 30 / 30 | Comprehensive | Comprehensive | `/aidlc enterprise` |
| **MVP** | Operation フェーズを除いた、実際の最初の製品インクリメント | 20 / 30 | Standard | Standard | `/aidlc mvp` |
| **Proof of concept** | 最小限の有用な実装経路で実現可能性を検証する | 5 / 30 | Minimal | Minimal | `/aidlc poc` |
| **Bugfix** | 焦点を絞った修正と回帰テストが必要な既知の不具合 | 6 / 30 | Minimal | Minimal | `/aidlc bugfix` |
| **Refactor** | 製品の振る舞いを変えずに既存コードを改善する | 7 / 30 | Minimal | Minimal | `/aidlc refactor` |
| **Infrastructure** | 環境、IaC、デプロイ基盤、またはコストに関する作業 | 12 / 30 | Standard | Standard | `/aidlc infra` |
| **Security patch** | CVE または焦点を絞った脆弱性への対応 | 7 / 30 | Minimal | Minimal | `/aidlc security-patch` |
| **Workshop** | ファシリテーター主導のトレーニングやグループでの実施セッション | 23 / 30 | Standard | Minimal | `/aidlc workshop` |

ステージ数は静的なルートを表します。すべてのプロファイルが最初に実行する
Initialization の 3 つのセットアップステージを除いた後のステージを数えており、これは
計画の提案で示される数です。条件付きとマークされたステージは、その条件が当てはまらない場合に
自ら省略されることがあります。たとえば、グリーンフィールドプロジェクトにおける Reverse Engineering などです。正確なステージ×プロファイルのマトリクスについては、
[スコープ、深さ、テスト戦略](05-scopes-and-depth.md#stage-by-scope-matrix) を参照してください。

## `classic`

**Classic を選ぶ場面:** v1 スタイルの手続きを望む場合です。Inception ではステージごとに 1 回の人間による
承認があり、Construction はユニットごとに構築されます。各ユニットのコード計画を承認し、
**Review each checkpoint** を選んだ場合はそのチェックポイントも承認します（
ユニットの設計ステージには独自の停止点はありません。チームが所有するユニットは、独自の
ステージごとまたはユニット終了時のゲートを維持します）。Ideation は省略され、Operation は
プレースホルダーとして残されます。ステージで宣言された実行モードとサポートエージェントは変わりません。

あなたも
`AWS_AIDLC_DEFAULT_SCOPE` も別のプロファイルを指定していない場合、Classic がエンジンの暗黙の既定値です。会話型のコールドスタート
フローでは、詳細なタスク説明があると、何かが作成される前にアダプティブな構成の提案を受けることがあります。
Classic は Standard の成果物とテストを使います。ウォーキングスケルトンの
手続きとサマリー確認はオフです。センサーは実行され、学びの儀式も実行されます。
レビューは助言的です（ステージごとに 1 回のパスで、指摘は承認ゲートで示されます）。
明示的な自律モードでは、マージ前の 1 回のレビューが維持されます。ガードポリシーの既定値は off です。
計画の再承認、レビューの凍結、状態遷移、レビュアーの読み取り範囲は、
指示のない作業では適用を見送り、そのたびに `GUARD_STOOD_ASIDE` 行を記録します。Plan Approval は
引き続きエンジンによって求められます。人間のターンの権限と監査は引き続き有効です。

`/aidlc --sensors on|off`、`/aidlc --learnings on|off`、
`/aidlc --summary-confirmation on|off`、または `/aidlc --plan-approval on|off` を使うと、
インテント単位でスコープの設定を上書きできます（計画の承認をオフにできるのはあなただけです）。
`AIDLC_DISABLE_SENSORS=1`、`AIDLC_DISABLE_LEARNINGS=1`、
`AIDLC_DISABLE_SUMMARY_CONFIRMATION=1`、`AIDLC_DISABLE_PLAN_APPROVAL_GUARD=1` は、
インテントがオンと指定している場合でも、それぞれの手続きを強制的にオフにします。

問題そのものがまだ明確でなく、
市場調査、実現可能性の分析、明示的なスコープの発見が役立つ場合は、Classic を選ばないでください。代わりに
Feature または Enterprise を選んでください。

## `express`

**Express を選ぶ場面:** 要件がすでに理解されており、要件、実装、テスト、そして
任意のデプロイ工程を通る、サポートされた最短の経路を望む場合です。

Express は、Ideation、設計パス、ユニット分解、Delivery Planning、
CI Pipeline を省略します。ステージのレビュアーのディスパッチを無効にし、Minimal の成果物と
要件駆動のテストを使います。Reverse Engineering とデプロイのステージは
条件付きのままです。

Express は、センサー、学び、サマリー確認もオフにします。
これらはインテントごとに [`/aidlc --sensors on|off`](12-cli-commands.md#aidlc-sensors-learnings-summary-confirmation-ceremony-controls)、
[`/aidlc --learnings on|off`](12-cli-commands.md#aidlc-sensors-learnings-summary-confirmation-ceremony-controls)、
または [`/aidlc --summary-confirmation on|off`](12-cli-commands.md#aidlc-sensors-learnings-summary-confirmation-ceremony-controls) で上書きできます。

Express の [ガードポリシー](13-customization.md#guard-policy) の既定値は off です。計画の
承認、レビューの凍結、状態遷移、レビュアーの読み取り範囲は、
指示のない作業では適用を見送り、そのたびに `GUARD_STOOD_ASIDE` 行を記録します。人間の在席確認は
オンのままです。1 つのインテントについて引き上げるには、`/aidlc --guard-policy strict` または `relaxed` と入力します。

曖昧な作業、チームをまたぐ作業、規制対象の作業、アーキテクチャの比重が大きい作業には
Express を選ばないでください。Express の速さは、そうした意思決定の場を意図的に取り除くことで得られています。

## `feature`

**Feature を選ぶ場面:** 本番機能を構築しており、実用的な深さで
完全なライフサイクルを望む場合です。

Feature は、セットアップ後の 30 ステージすべてを Standard の深さで実行します。インテントの発見から
設計、実装、デプロイ、フィードバックまでを含みます。Enterprise のような重いドキュメント要件を課すことなく、
未知の事項を浮かび上がらせる機会を、ワークフローに最も広く与えます。

## `enterprise`

**Enterprise を選ぶ場面:** 作業が規制対象、高リスク、監査上センシティブである場合、
または正式なコンプライアンスや運用上の証跡が必要な場合です。

Enterprise は、セットアップ後の 30 ステージすべてを Comprehensive の深さで実行します。市場、
コンプライアンス、セキュリティ、設計、オブザーバビリティ、インシデント対応、パフォーマンスの
作業を維持します。文書化されていない判断のコストが、追加の手続きのコストよりも
高いためです。

## `mvp`

**MVP を選ぶ場面:** アイデアが実現可能かどうかを試すだけでなく、
実際の最初の製品インクリメントをリリースする場合です。

MVP は、Inception と Construction の設計/構築の経路全体を Standard の
深さで維持します。Ideation の手続きの一部を削り、Operation フェーズを省略します。製品に本番運用と
完全なフィードバックのライフサイクルが必要になったら、Feature または Enterprise に移行してください。

## `poc`

**Proof of Concept を選ぶ場面:** 主な問いが、あるアプローチが機能するかどうかである
場合です。

PoC は Minimal の深さで 5 ステージを使います。製品、設計、運用の手続きの大半を省きながら、
要件、コード、テストまで素早く到達します。PoC は後の意思決定のための
証拠であり、本番対応のためのプロファイルではありません。

## `bugfix`

**Bugfix を選ぶ場面:** 不具合が既知であり、望む成果が、検証を伴う焦点を絞った
修正である場合です。

Bugfix は Minimal の深さで 6 ステージを使います。ワークスペースの理解、
要件、Code Generation、Build and Test、デプロイの経路を維持し、
発見、広範な設計、無関係な Operation の作業を省きます。

Bugfix は学びとサマリー確認もオフにします。そのため、どのステージも日誌を付けず、
修正の後に "Keep that for next time?"（次回のために残しますか？）と尋ねることもありません。センサー、ステージの承認、計画の
承認はオンのままです。

## `refactor`

**Refactor を選ぶ場面:** 振る舞いを安定させたまま、内部構造、
保守性、技術的負債を改善する場合です。

Refactor は Minimal の深さで 7 ステージを使います。既存コードの理解、
内部的な変更の定義、その実装、振る舞いが
退行していないことの証明、そして検証済みの結果をデプロイまで運ぶことに重点を置きます。作業がユーザーから見える振る舞いを変える場合は、
代わりに Feature を使ってください。

## `infra`

**Infrastructure を選ぶ場面:** 成果物が環境、IaC の変更、
デプロイ基盤、プラットフォームの機能、またはコスト最適化である場合です。

Infrastructure は Standard の深さで 12 ステージを使います。ユーザー向けの製品の
手続きを取り除き、要件、NFR、インフラ設計、CI/CD、
デプロイ、オブザーバビリティに集中します。Code Generation は Plan
Approval のもとで IaC を書き、Environment Provisioning がそれを適用する前に、Build and Test がチェック（たとえば `terraform validate`
や `cdk synth`）を実行します。

## `security-patch`

**Security Patch を選ぶ場面:** 既知の CVE、脆弱性、
または範囲の狭いセキュリティ上の不具合に対応する場合です。

Security Patch は Minimal の深さで 7 ステージを使います。セキュリティに関係する
要件、検証、実装、デプロイの経路を維持し、
無関係な製品の手続きは避けます。焦点を絞ったパッチではなく、より広範なセキュリティやコンプライアンスの
プログラムには Enterprise を使ってください。

## `workshop`

**Workshop を選ぶ場面:** ファシリテーターがトレーニングラボや、協調して進める
グループセッションを主導する場合です。

Workshop は、ファシリテーターが演習を用意するため Ideation を省略し、
Inception から Operation までのライフサイクルを Standard の深さで実行します。教育セッションを止めずに進めるため、
意図的に Minimal のテスト戦略を使い、通常の
レビューを 1 回の助言的なパスに制限します。複数参加者での運用手順は
[Workshop モード](workshop-mode.md) にあります。

## Let AI-DLC choose or compose

プロファイルを覚える必要はありません。作業内容を説明してください:

```
/aidlc Fix the login timeout bug
/aidlc Build a regulated payment approval service
/aidlc Create a lightweight prototype for the new search flow
```

既製のプロファイルが説明に合う場合、アシスタントはそれを提案し、
確認のためにステージ数とゲート数を表示します。詳細な作業や曖昧な作業には、
アダプティブコンポーザーを使う提案が示されます。アダプティブコンポーザーは目的に合わせたステージの
ルートを提案し、作成する前に承認を待ちます。

次のようにして、その経路を強制することもできます:

```
/aidlc compose "harden the deployment pipeline and add observability"
```

## Construction approvals and execution

ソースを生成する新しい単独ユニットのワークフローは、既定で、検証済みの完了チェックポイントを伴いながら、ユニットを 1 つずつ
順番に構築します。これにはユニット
分解と、ソースを生成するユニット単位のステージが含まれていることが必要です。設計のみの作業や、
Express のようにユニット分解を省略するプロファイルは、既存の
ステージフローを維持します。チームが所有するユニットのゲートは、独自のポリシーを維持します。既存のワークフローや
明示的なイテレーションの選択は、新しい既定値によって変換されません。

skeleton-on の場合、最初のユニットは、動作する最小の統合された
スライスとして計画されます。その該当する設計作業と Code Generation が完了し、実際の統合
チェックに合格し、あなたがスケルトンを承認してから、以降のユニットが始まります。最初の
設計ドキュメントは、動作するスケルトンではありません。skeleton-off では、Construction の開始時に **Continue
automatically** / **Review each checkpoint** が提示されます。skeleton-on では、
スケルトンのチェックポイントの後に提示されます。記録済みの選択が繰り返し尋ねられることはなく、
Construction 中に明示的に自律を許可または取り消すことができます。

承認ポリシーと実行は別のものです。対象となる Code Generation の
バッチをファンアウトするには、stage-major の順序とスウォーム実行を明示的に選びます。ガイド付きと
自動のバッチ完了の両方がサポートされています。unit-major は直列のままです。
すべてのユニットの Plan Approval、検証コマンドの選択、生成前の
サマリー確認には、引き続きあなたの回答が必要です。対象となるスウォームの計画は、個別の受領記録を伴う 1 回の
**Approve Plans** の提示を共有できます。記録された、
人間が承認した検証コマンドは、すべてのユニット/バッチのチェックポイントで再利用されます。
変更するには、新たな人間の受領記録が必要です。失敗した場合は引き続き停止します。既存のワークフローは、
新しいチェックポイント/実行のフィールドがない場合、記録済みのイテレーションと従来の振る舞いを維持します。
[Construction コマンド](12-cli-commands.md#construction-order-and-execution) を参照してください。

## Related controls

ワークフロープロファイルは、ルートと既定値を選びます。次の項目は個別に調整できます:

- **深さ**: `--depth minimal|standard|comprehensive` で調整します。
- **テスト戦略**: `--test-strategy minimal|standard|comprehensive` で調整します。
- **レビューの上限**: `--review adversarial|advisory|none` で調整します。実際に適用されるクラスは、
  ステージ自身が宣言したクラスを、設定されていればこの実行単位の上限で、
  設定されていなければプロファイルの上限で引き下げたものです。どちらの上限も、ステージを
  宣言されたクラスより上に引き上げることはできず、実行単位の上限を設定するとプロファイルの上限は置き換えられます。

これらの上書きは、あるプロファイルを別のプロファイルに変えるものではありません。選択した
プロファイルを調整するものです。規範となるルーティングマトリクスと上書きのセマンティクスについては、
[スコープ、深さ、テスト戦略](05-scopes-and-depth.md) を参照してください。
