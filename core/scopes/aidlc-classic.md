---
name: classic
depth: Standard
keywords: []
description: "V1-style ceremony through Inception and Construction - the implicit default"
skeleton: off
review_cap: advisory
guard_policy: off
sensors: on
learnings: on
summary_confirmation: off
plan_approval: on
collaborators: off
---

# classic scope

`classic` は暗黙の既定スコープである - 利用者も
`AWS_AIDLC_DEFAULT_SCOPE` もスコープを指定しないときに使われ、Inception と Construction を通じて
v1 スタイルのセレモニーを復元する: Inception のステージごとに人間の承認が1回ある。
Construction ではユニットごとにビルドし、人間は各ユニットのコードプランと、
"Review each checkpoint" の下ではそのチェックポイントを承認するため、ユニットの設計ステージは
それ自体では停止せずに実行される。チーム所有のユニットは、ステージごとまたは
Unit の終わりの独自のゲートを保持する。Ideation はスキップされ、Operation はプレースホルダのままである。ステージで宣言された実行モードは
変わらない。コラボレーターはオフなので、各ステージはリードエージェント
だけで実行される（`/aidlc --collaborators on` でサポートエージェントが加わる）。

Guard Policy の既定はオフである: 変更された入力は記録され、1行で告知される。また、指示のない作業については、プラン承認、レビューの凍結、状態遷移、レビュアーの読み取りスコープが引き下げられる。人間の関与は維持される。

レビューは助言的である: ステージごとに1パスで、その指摘は
承認ゲートで人間に届き、反駁と修復のループはない。明示的な自律モードでは、
マージ前の単一のレビューが維持される。ウォーキングスケルトンのセレモニーとサマリー確認は
オフである。センサーは実行され、学びの儀式も実行される。オフの下では、Plan Approval、
レビューの凍結、状態遷移、レビュアーの読み取りスコープは、指示のない作業については脇に退き、
そのたびに `GUARD_STOOD_ASIDE` 行を記録する。承認の
質問は引き続きコンダクターが尋ねる。人間のターンの権限と監査は引き続き
有効である。

インテントごとのセレモニーは `/aidlc --sensors on|off`、
`/aidlc --learnings on|off`、`/aidlc --summary-confirmation on|off`、
`/aidlc --collaborators on|off` で上書きする。グローバルなキルスイッチ
`AIDLC_DISABLE_SENSORS=1`、`AIDLC_DISABLE_LEARNINGS=1`、
`AIDLC_DISABLE_SUMMARY_CONFIRMATION=1`、`AIDLC_DISABLE_COLLABORATORS=1` は、
インテントがオンと言っていても、それぞれのセレモニーを強制的にオフにする。これらは
`aidlc config flags --bypass <NAME>` で記録することもできる。

## Why these stages, why skip those

AI-DLC v1 には Ideation フェーズがなかったため、`classic` は Ideation の7ステージをすべてスキップする。
Inception のステージはすべて保持し、Construction のステージは Build and
Test まで保持する。CI Pipeline と Operation の7ステージはすべてスキップする: 顧客は
独自の CI と下流の仕組みを持ち込み、Build and Test がすべてのユニットにまたがる唯一の統合ビルドであるためである。無条件のステージは8つだけである: Initialization の3ステージ、
Requirements Analysis、Units Generation、Delivery Planning、Code Generation、
Build and Test。残りの Inception と Construction の作業は
CONDITIONAL であり、プロジェクトのコンテキストから自ら選択されるため、v1 の適応的な振る舞いが保たれる。

テスト戦略は depth から Standard を継承するため、本番相当のテストへの
期待は引き続き有効である。別の `workshop` スコープは、既存のワークショップのワークフロー向けに、
教育目的の Minimal テストの上書きを保持している。

## Membership

Initialization、Inception のすべてのステージ、そして Build and Test までの Construction ステージが
グリッドに含まれる: 33ステージ中18ステージ。Ideation の7ステージすべて、
CI Pipeline、Operation の7ステージすべては SKIP である。このスコープには意図的にキーワードがない。明示的に名前を指定するか、
暗黙の既定として使う。
