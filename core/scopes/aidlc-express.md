---
name: express
depth: Minimal
keywords:
  - express
  - lightweight
description: "Lightest run: requirements to deploy, no design pass, no reviewers"
skeleton: off
runner: true
review_cap: none
guard_policy: off
sensors: off
learnings: off
summary_confirmation: off
plan_approval: off
collaborators: off
---

# express scope

`express` は、軽量な実行を求めるコミュニティの要望に応えるスコープである。
要件からコード、テスト、デプロイまで一直線に進み、設計パスもレビュアーの
ディスパッチも行わない。

Guard Policy の既定値は off である。変更された入力は承認をやり直すのではなく、記録して通知する。プラン承認、レビューフリーズ、状態遷移、レビュアーの読み取り範囲は、指示のない作業向けに引き下げられる。人間の関与（Human presence）は高いまま維持する。

センサー、学び、サマリー確認もオフである。インテントごとに
`/aidlc --sensors on|off`、`/aidlc --learnings on|off`、
`/aidlc --summary-confirmation on|off` で上書きできる。

プラン承認もオフである。コードプランが書かれると、それを示す1行が表示され、
コード生成が始まる。その行には、いつでもプランを見直すよう依頼できることが
記されている。プランを構築前に確認したい、あるいはすべてのプランについて
確認してほしいと依頼することもできる。

## Why these stages, why skip those

Requirements Analysis が契約を確立し、Code Generation がそれを実装し、
Build and Test がそれを検証し、Operation の末尾ステージ群が結果をデプロイして
監視できる。レビュアーは `review_cap: none` によって無効化される。Minimal の
テストでも、要件に基づくユニットテストと、コンポーネントごとのハッピーパスの
最低ラインは必須である。

`express` は Units Generation をスキップするため Unit DAG が存在し得ず、
スウォームの経路には構造上到達できない。Reverse Engineering は CONDITIONAL の
ままであり、既存コードがある場合にブラウンフィールドの理解を提供する。デプロイの
末尾ステージ群も CONDITIONAL で、デプロイ対象がなければ自らスキップする。これらの
ステージは、完全な設計の生成ステージが意図的に存在しない場合、承認済みの要件、
ワークスペースのデプロイ設定、ビルド結果、および先行する末尾ステージの成果物を使用する。

## Membership

グリッドには、Initialization の3ステージ、Reverse Engineering、
Requirements Analysis、Code Generation、Build and Test、Deployment Pipeline、
Deployment Execution、Observability Setup が含まれる。それ以外のステージはすべて SKIP である。
