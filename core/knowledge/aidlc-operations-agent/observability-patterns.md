# Observability Patterns

メトリクス、ログ、トレースによってシステムの挙動を可視化し、迅速な診断とプロアクティブな検知を可能にする。

## The Three Pillars of Observability

### 1. Metrics
時間ごとに集計される数値計測値。カーディナリティが低く、コストも低いため、アラートやダッシュボードに最適である。
- **種類**: カウンター（常に増加する）、ゲージ（増減しうる）、ヒストグラム（値の分布）。
- **AWS**: CloudWatch Metrics。Lambda/ECS からのカスタムメトリクスには CloudWatch Embedded Metric Format（EMF）を使う。
- カスタムのビジネスメトリクスを出力する: 1 分あたりの注文数、1 時間あたりの決済失敗数、1 日あたりのサインアップ数など。

### 2. Logs
個々のイベントを記録した、タイムスタンプ付きのレコード。カーディナリティが高く、量も多い。
- 一貫したフィールド（`timestamp`、`level`、`requestId`、`service`、`message`）を持つ構造化ログ（JSON）を使う。
- ヘッダーを通じて伝播される共通の `requestId` または `traceId` を使い、サービス間でログを相関させる。
- **AWS**: CloudWatch Logs。サービスと環境ごとに Log Group を分ける。

### 3. Traces
複数のサービスを通過するリクエストのエンドツーエンドの経路。レイテンシの内訳と依存関係を示す。
- **AWS**: 分散トレーシングには X-Ray を使う。AWS サービスへの SDK 呼び出しは自動で計装される。
- ビジネスロジック、データベースクエリ、外部 HTTP 呼び出しには、カスタムセグメントを計装する。
- トレースマップを使ってサービスの依存関係を可視化し、レイテンシのボトルネックを特定する。

## The Four Golden Signals (Google SRE)

すべてのサービスについて、次の 4 つのシグナルを監視する:

1. **レイテンシ（Latency）**: リクエストの処理にかかる時間。パーセンタイル（p50、p95、p99）で計測する。成功したリクエストと失敗したリクエストは分けて追跡する。
2. **トラフィック（Traffic）**: システムにかかる需要。API なら 1 秒あたりのリクエスト数、キューなら 1 秒あたりのメッセージ数、WebSocket ならアクティブな接続数。
3. **エラー（Errors）**: 失敗したリクエストの割合。明示的なエラー（5xx）と暗黙的なエラー（誤った応答、タイムアウト、リトライ）の両方を含める。
4. **飽和度（Saturation）**: システムがどれだけ逼迫しているか。CPU 使用率、メモリ使用量、ディスク I/O、コネクションプールの使用率、予約済み同時実行数に対する Lambda の同時実行数。

## CloudWatch Dashboards and Alarms

### Dashboard Design
- サービスごとに 1 つのダッシュボードを用意し、ゴールデンシグナルを最上部に配置する。
- 依存先の健全性を含める: 下流サービスのレイテンシ、データベース接続数、キューの深さ。
- 派生メトリクスには CloudWatch Metrics Math を使う: エラーレート = errors / (errors + successes) * 100。
- 一貫性のため、サービス間でダッシュボードのレイアウトを標準化する。

### Alarm Configuration
- 原因ではなく症状に対してアラームを設定する。"CPU > 80%" ではなく "error rate > 1%" でアラートを出す。
- 複合アラームを使ってノイズを減らす: 複数の条件が同時に成立したときだけアラートを出す。
- 適切な評価期間を設定する: 一時的なスパイクで発火する単一データポイントのアラームは避ける。安定性のために `3 out of 5 datapoints` を使う。
- アラームのアクションを定義する: SNS 通知、Lambda による自動修復、OpsCenter アイテムの作成。
- 重大度の階層: P1（オンコール担当者を呼び出す）、P2（Slack でアラートし、数時間以内に修正）、P3（チケット化し、スプリント内に修正）。

## X-Ray Distributed Tracing

- API Gateway、Lambda、ECS、SQS で X-Ray トレーシングを有効にする。
- X-Ray SDK を使って、アプリケーションロジック用のカスタムサブセグメントを作成する。
- 主要なディメンションにはアノテーション（インデックス化され、検索可能）を付ける: `customerId`、`orderStatus`、`region`。
- デバッグ用の詳細にはメタデータ（インデックス化されない）を付ける: リクエスト/レスポンスのボディ、クエリパラメータ。
- X-Ray グループとフィルター式を使って、特定の条件に一致するトレースを見つける: `service("payment-service") AND responsetime > 3`。

## CloudWatch Logs Insights Queries

運用上のトラブルシューティングに役立つクエリパターン:

```
# Error rate over time
filter @message like /ERROR/
| stats count() as errors by bin(5m)

# Slowest requests
filter @duration > 1000
| sort @duration desc
| limit 20

# Request count by status code
parse @message '"statusCode":*,' as statusCode
| stats count() by statusCode

# Cold start impact (Lambda)
filter @type = "REPORT"
| stats avg(@duration), max(@duration), avg(@initDuration) by bin(10m)
```

## Anomaly Detection

- CloudWatch Anomaly Detection は ML を使ってベースラインを確立し、そこからの逸脱に対してアラートを出す。
- 安定したパターンを持つレイテンシやエラーレートのメトリクスで有効にする。
- 検知バンドの幅は、許容できるばらつきに基づいて設定する（標準偏差 2 つ分が一般的な出発点）。
- 異常検知と静的しきい値を組み合わせる: 異常検知は緩やかなドリフトを捉え、静的しきい値は急性の障害を捉える。

## Log Aggregation and Retention

- すべてのログを一貫した命名で CloudWatch Logs にルーティングする: `/aws/lambda/{service}`、`/ecs/{cluster}/{service}`。
- ロググループごとに保持ポリシーを設定する: 開発は 30 日、ステージングは 90 日、本番は 1〜3 年（またはコンプライアンス要件に従う）。
- 長期保持とコスト最適化のために S3 にアーカイブする。90 日より古いログには Glacier を使う。
- マルチアカウント構成では、CloudWatch のクロスアカウントログ集約を検討する。

## Synthetic Monitoring

- CloudWatch Synthetics の canary: スクリプト化されたチェックを定期的に（1〜5 分ごとに）実行する。
- 重要なユーザージャーニーをテストする: ログイン、注文、ダッシュボードの表示。
- canary は AWS が管理するインフラストラクチャから実行され、実際のユーザーから報告される前に問題を検知する。
- canary の失敗には P1 の重大度でアラートを出す。それはユーザー向けの経路が壊れていることを意味するからである。
- UI の比重が大きいアプリケーションには、ビジュアルモニタリング（スクリーンショットの比較）を使う。
