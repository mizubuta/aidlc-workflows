# NFR Reliability and Observability Guide

> このガイドは、Security Engineer（NFR Requirements ステージのリードエージェント）が保持する NFR Requirements Guide の完全版を補足するものである。NFR Requirements ステージと Build and Test ステージで品質に焦点を当てた貢献を行うために必要な、信頼性とオブザーバビリティのセクションを QA Engineer に提供する。

## Reliability Target-Setting

### SLA/SLO/SLI Hierarchy

| 用語 | 定義 | 例 | 定義する人 |
|------|-----------|---------|-------------|
| **SLI**（Indicator：指標） | 計測対象のメトリクス | 成功したリクエスト数 / 総リクエスト数 | エンジニアリング |
| **SLO**（Objective：目標） | SLI に対する社内目標 | 30 日間で成功率 99.95% | エンジニアリング + プロダクト |
| **SLA**（Agreement：合意） | 外部に対する契約上のコミットメント | 稼働率 99.9%（違約金あり） | ビジネス + 法務 |

**ルール**: SLO は SLA より厳しくしなければならない。SLA が 99.9% なら、社内バッファを確保するために SLO を 99.95% に設定する。

### Availability Targets and Their Implications

| 可用性 | 年間ダウンタイム | 月間ダウンタイム | 必要なもの |
|-------------|-----------------|-------------------|----------|
| 99%（ツーナイン） | 3.65 日 | 7.3 時間 | 基本的な監視、手動復旧 |
| 99.9%（スリーナイン） | 8.76 時間 | 43.8 分 | 自動再起動、ヘルスチェック、アラート |
| 99.95% | 4.38 時間 | 21.9 分 | マルチ AZ、自動フェイルオーバー、ロードバランシング |
| 99.99%（フォーナイン） | 52.6 分 | 4.38 分 | マルチリージョンのアクティブ-アクティブ構成、ゼロダウンタイムデプロイ |
| 99.999%（ファイブナイン） | 5.26 分 | 26.3 秒 | あらゆるものの完全自動化、広範な冗長化 |

### Recovery Objectives

| 目標 | 定義 | 設定方法 |
|-----------|-----------|------------|
| **RTO**（Recovery Time Objective：目標復旧時間） | サービス復旧までに許容できる最大時間 | ダウンタイム 1 時間あたりのビジネス影響に基づく |
| **RPO**（Recovery Point Objective：目標復旧時点） | 許容できるデータ損失の最大期間 | データを再作成または喪失するコストに基づく |
| **MTTR**（Mean Time to Recovery：平均復旧時間） | 障害から復旧するまでの平均時間 | 運用の中で計測する。目標は RTO 未満にする |
| **MTBF**（Mean Time Between Failures：平均故障間隔） | 障害と障害の間の平均時間 | 運用の中で計測する。信頼性への投資判断の根拠となる |

## Observability Requirements

### Three Pillars Specification

#### Metrics Requirements
コンポーネントごとに次を定義する:
- **ビジネスメトリクス**: コンバージョン率、処理された売上額、アクティブユーザー数
- **アプリケーションメトリクス**: リクエストレート、エラーレート、レイテンシ（RED メソッド）
- **インフラストラクチャメトリクス**: CPU、メモリ、ディスク、ネットワーク（USE メソッド）
- **保持期間**: 各メトリクス階層をどれだけ保持するか（1 分粒度は 7 日間、5 分粒度は 30 日間、1 時間粒度は 1 年間）

#### Logging Requirements
| ログレベル | 使用する場面 | 保持期間 | インデックス |
|-----------|------------|-----------|---------|
| ERROR | 対応が必要な、回復不能な障害 | 90 日 | 全文インデックス |
| WARN | 回復可能な問題、機能低下 | 30 日 | 全文インデックス |
| INFO | 重要なビジネスイベント、状態遷移 | 30 日 | 構造化フィールドのみ |
| DEBUG | トラブルシューティング用の診断詳細 | 7 日 | インデックスなし（保存のみ） |

#### Tracing Requirements
- すべてのサービス境界をまたぐ分散トレース
- W3C Trace Context ヘッダーによるトレースコンテキストの伝播
- サンプリング戦略: エラーは 100%、通常トラフィックは 10%（量に応じて調整する）
- トレースの保持期間: 完全な詳細は 7 日間、トレースのメタデータは 30 日間

### Alerting Requirements Template
アラートごとに:
```
Alert: [descriptive name]
SLI: [which service level indicator]
Threshold: [when to fire -- e.g., error rate > 1% for 5 minutes]
Severity: [page (wake someone up) | ticket (next business day) | log (informational)]
Runbook: [link to response procedure]
Notification: [who gets notified via which channel]
Auto-remediation: [if any automated response is triggered]
```

### Observability Anti-Patterns to Avoid
- 症状ではなく原因に対してアラートを出す（CPU 使用率ではなく、エラーレートに対してアラートを出す）
- サービス境界をまたぐ相関 ID が欠けている
- 機密データ（PII、認証情報、トークン）をログに記録する
- ノイズの多いしきい値によるアラート疲れ（デプロイ前に調整する）
- 明確なオーナーのいないダッシュボードの乱立（すべてのダッシュボードにオーナーが必要）
