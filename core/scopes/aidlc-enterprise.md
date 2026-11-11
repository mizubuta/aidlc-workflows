---
name: enterprise
depth: Comprehensive
keywords: []
description: Regulated enterprise feature, full audit trail
skeleton: on
guard_policy: strict
sensors: on
learnings: on
summary_confirmation: on
plan_approval: on
collaborators: on
---

# enterprise scope

監査証跡を完全に残す必要がある規制対象の作業向けの、Comprehensive の深さ。
グラフ内のすべてのステージを実行し、近道はしない。Ideation、Inception、
Construction、Operation のすべてを最初から最後まで実行するため、成果物の連鎖
（インテントからデプロイ、フィードバックまで）が完全で、追跡可能になる。

ガードポリシーの既定値は strict である。入力が変更されると承認が再び開かれ、フェンスは一切下げられない。監査証跡こそがこのスコープの目的だからである。

## Why every stage

エンタープライズの作業には、コンプライアンス、セキュリティレビュー、サインオフの
義務が伴う。ステージを省略するスコープ（`mvp`、`poc`、`bugfix`）は、速度と引き換えに
これらを手放すことをいとわない。ここでは、文書化されていない判断のコストが
ステージを実行するコストよりも高いため、このスコープは背骨全体を維持する。
すなわち、前段の market-research と team-formation、完全な設計パス、
そして後段の Operation ステージ（observability、incident-response、
performance-validation）である。

## Membership

`enterprise` は、33 個のステージすべてを EXECUTE とする唯一のスコープである。
EXECUTE/SKIP グリッドの基準となる列であり、他のすべてのスコープはこのスコープの
部分集合である。キーワードトリガーはなく、このスコープは説明文から推定されるのではなく、
意図的に選択される。
