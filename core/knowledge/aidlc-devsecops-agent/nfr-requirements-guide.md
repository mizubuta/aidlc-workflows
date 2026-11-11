# NFR Requirements Guide

## Performance Requirement Benchmarking

### How to Define Performance Requirements
すべてのパフォーマンス要件は、次の項目を必ず指定しなければならない:
- **メトリクス**: 何を測定するか（応答時間、スループット、リソース使用量）
- **目標値**: 定量的なしきい値
- **パーセンタイル**: 目標値をどのパーセンタイルに適用するか（p50、p95、p99）
- **負荷条件**: どの程度の同時実行数/トラフィックのもとで目標値を満たす必要があるか
- **測定方法**: メトリクスをどのように、どこで取得するか

### Common Performance Targets by Application Type

| アプリケーションの種類 | メトリクス | 目標値 | パーセンタイル | 備考 |
|-----------------|--------|--------|------------|-------|
| Web アプリケーション（対話型） | ページ読み込み時間 | < 2s | p95 | First Contentful Paint |
| REST API（同期） | 応答時間 | < 200ms | p99 | ネットワーク転送時間を除く |
| 検索クエリ | 結果の返却 | < 500ms | p95 | ランキング処理を含む |
| バッチ処理 | スループット | N レコード/分 | 平均 | 許容できる最低レートを定義する |
| リアルタイムメッセージング | エンドツーエンドのレイテンシ | < 100ms | p99 | 送信から配信まで |
| ファイルアップロード | 処理時間 | 1MB あたり < 5s | p95 | アップロード後の処理 |

### Performance Anti-Requirements
不合理な期待は明示的に除外する:
- 「システムは高速であるべき」（測定できない）
- 「すべてのページが瞬時に読み込まれる」（目標値もパーセンタイルもない）
- 「無制限のユーザーに対応する」（キャパシティの境界がない）

次のように置き換える: 「ダッシュボードページは、同時接続ユーザー 500 人の条件下で、p95 で 3 秒未満で読み込まれる。」

## Scalability Assessment Frameworks

### Capacity Planning Template

| 観点 | 現在 | 6か月後の目標 | 12か月後の目標 | スケーリングの仕組み |
|-----------|---------|---------------|-----------------|-------------------|
| 同時接続ユーザー数 | N | N * X | N * Y | 水平オートスケーリング |
| データ量 | N GB | N * X GB | N * Y GB | パーティショニング戦略 |
| 秒間リクエスト数 | N | N * X | N * Y | ロードバランサー + レプリカ |
| ストレージ増加率 | N GB/月 | N * X GB/月 | N * Y GB/月 | 階層型ストレージポリシー |

### Scalability Requirement Format
```
NFR-SCALE-NNN: [Scalability Requirement]
Current baseline: [measured current capacity]
Target capacity: [required capacity at specific timeline]
Growth model: [linear, exponential, seasonal, step-function]
Scaling approach: [horizontal, vertical, partitioning, caching]
Cost constraint: [infrastructure cost must remain below $X/month at target scale]
Degradation policy: [what degrades first when approaching capacity limits]
```

### Scaling Decision Matrix

| シグナル | スケールアップ | スケールアウト | まず最適化 |
|--------|----------|-----------|----------------|
| CPU が常時 > 70% | 単一インスタンスのワークロード | ステートレスなサービス | 非効率なアルゴリズムがないか確認する |
| メモリ逼迫 | インメモリ処理 | 分散キャッシュ | メモリリークがないか確認する |
| I/O 待ち > 30% | より高速なストレージ階層 | リードレプリカ、シャーディング | クエリ最適化、インデックス作成 |
| キュー深度が増加 | より高速なコンシューマー | コンシューマーインスタンスの追加 | バッチサイズの調整 |
| コネクションプールの枯渇 | プールサイズの拡大 | コネクションの多重化 | コネクションリークの調査 |

## Reliability Target-Setting

### SLA/SLO/SLI Hierarchy

| 用語 | 定義 | 例 | 定義する人 |
|------|-----------|---------|-------------|
| **SLI**（指標） | 測定対象のメトリクス | 成功したリクエスト数 / 全リクエスト数 | エンジニアリング |
| **SLO**（目標） | SLI に対する内部目標 | 30 日間で成功率 99.95% | エンジニアリング + プロダクト |
| **SLA**（合意） | 外部への契約上のコミットメント | 金銭的ペナルティ付きの稼働率 99.9% | ビジネス + 法務 |

**ルール**: SLO は必ず SLA より厳しくしなければならない。SLA が 99.9% なら、内部のバッファを確保するため SLO を 99.95% に設定する。

### Availability Targets and Their Implications

| 可用性 | 年間ダウンタイム | 月間ダウンタイム | 必要なもの |
|-------------|-----------------|-------------------|----------|
| 99%（ツーナイン） | 3.65 日 | 7.3 時間 | 基本的な監視、手動での復旧 |
| 99.9%（スリーナイン） | 8.76 時間 | 43.8 分 | 自動再起動、ヘルスチェック、アラート |
| 99.95% | 4.38 時間 | 21.9 分 | マルチ AZ、自動フェイルオーバー、負荷分散 |
| 99.99%（フォーナイン） | 52.6 分 | 4.38 分 | マルチリージョンのアクティブ-アクティブ構成、ゼロダウンタイムデプロイ |
| 99.999%（ファイブナイン） | 5.26 分 | 26.3 秒 | あらゆるものの完全自動化、広範な冗長性 |

### Recovery Objectives

| 目標 | 定義 | 設定方法 |
|-----------|-----------|------------|
| **RTO**（Recovery Time Objective、目標復旧時間） | サービス復旧までに許容できる最大時間 | ダウンタイム 1 時間あたりのビジネス影響に基づく |
| **RPO**（Recovery Point Objective、目標復旧時点） | 許容できるデータ損失の最大期間 | データの再作成または損失のコストに基づく |
| **MTTR**（Mean Time to Recovery、平均復旧時間） | 障害から復旧するまでの平均時間 | 運用の中で測定する。目標値は RTO 未満であるべき |
| **MTBF**（Mean Time Between Failures、平均故障間隔） | 障害と障害の間の平均時間 | 運用の中で測定する。信頼性への投資判断の根拠となる |

## Observability Requirements

### Three Pillars Specification

#### Metrics Requirements
コンポーネントごとに次を定義する:
- **ビジネスメトリクス**: コンバージョン率、処理された売上、アクティブユーザー数
- **アプリケーションメトリクス**: リクエストレート、エラーレート、レイテンシ（RED メソッド）
- **インフラストラクチャメトリクス**: CPU、メモリ、ディスク、ネットワーク（USE メソッド）
- **保持期間**: メトリクスの各階層をどれだけ保持するか（1 分粒度は 7 日間、5 分粒度は 30 日間、1 時間粒度は 1 年間）

#### Logging Requirements
| ログレベル | 使用する場面 | 保持期間 | インデックス |
|-----------|------------|-----------|---------|
| ERROR | 対応が必要な回復不能の障害 | 90 日 | 全文インデックス |
| WARN | 回復可能な問題、劣化した振る舞い | 30 日 | 全文インデックス |
| INFO | 重要なビジネスイベント、状態遷移 | 30 日 | 構造化フィールドのみ |
| DEBUG | トラブルシューティング用の診断詳細 | 7 日 | インデックスなし（保存のみ） |

#### Tracing Requirements
- すべてのサービス境界をまたぐ分散トレース
- W3C Trace Context ヘッダーによるトレースコンテキストの伝播
- サンプリング戦略: エラーは 100%、通常のトラフィックは 10%（量に応じて調整する）
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
- 症状ではなく原因に対してアラートを出す（CPU 使用率ではなく、エラーレートでアラートを出す）
- サービス境界をまたぐ相関 ID が欠落している
- 機密データ（PII、認証情報、トークン）をログに記録する
- ノイズの多いしきい値によるアラート疲れ（デプロイ前に調整する）
- 所有者が明確でないダッシュボードの乱立（すべてのダッシュボードに所有者が必要）

## Security Requirement Templates

### Authentication Requirements
```
NFR-AUTH-NNN: [Authentication Requirement]
Method: [OAuth 2.0, JWT, session-based, API key, mTLS]
Token lifetime: [access token TTL, refresh token TTL]
MFA requirement: [none, optional, required for admin, required for all]
Session management: [max concurrent sessions, idle timeout, absolute timeout]
Password policy: [min length, complexity, rotation, breach detection]
```

### Authorization Requirements
```
NFR-AUTHZ-NNN: [Authorization Requirement]
Model: [RBAC, ABAC, ACL, policy-based]
Roles: [list of roles and their permission boundaries]
Resource granularity: [organization, team, user, resource-level]
Delegation: [can users delegate permissions, under what constraints]
Audit: [which authorization decisions are logged]
```

### Data Protection Requirements
```
NFR-DATA-NNN: [Data Protection Requirement]
Classification: [public, internal, confidential, restricted]
Encryption at rest: [algorithm, key management, rotation schedule]
Encryption in transit: [TLS version, cipher suites, certificate management]
PII handling: [identification, masking, pseudonymization, retention limits]
Data residency: [geographic constraints, cross-border transfer rules]
Backup encryption: [separate key, recovery testing frequency]
```

### Compliance Requirements
```
NFR-COMP-NNN: [Compliance Requirement]
Framework: [GDPR, SOC 2, HIPAA, PCI-DSS, ISO 27001]
Scope: [which components/data are in scope]
Controls: [specific controls required — access logging, encryption, retention]
Audit frequency: [continuous, quarterly, annual]
Evidence requirements: [what documentation/logs must be maintained]
```

### Security Anti-Requirements
不合理な期待は明示的に除外する:
- 「システムは安全であるべき」（測定できない）
- 「脆弱性ゼロ」（不可能 — 許容できるリスクを定義する）
- 「軍用レベルの暗号化」（定義されていない — アルゴリズムと鍵長を指定する）

次のように置き換える: 「すべての API エンドポイントで、RS256 署名による JWT 認証を必須とする。アクセストークンは 15 分で失効する。リフレッシュトークンは 7 日で失効し、使い捨てである。」
