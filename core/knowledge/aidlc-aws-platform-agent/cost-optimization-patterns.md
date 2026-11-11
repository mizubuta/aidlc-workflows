# AWS Cost Optimization Patterns

## Purpose
信頼性やパフォーマンスを犠牲にせずに AWS のコストを削減するための実践的な戦略。コスト最適化は一度きりの作業ではなく、継続的な取り組みである。

## Compute: Rightsizing

### Process
1. AWS Compute Optimizer を有効にする（無料、アカウント全体）
2. 14 日間のデータ収集後に推奨事項を確認する
3. 過剰にプロビジョニングされたインスタンス（平均 CPU < 20%、メモリ < 30%）を特定する
4. まず非本番環境でサイズを変更し、その後メンテナンスウィンドウ中に本番環境で変更する

### Common Findings
- ほとんどのチームは、初回デプロイ時に 30〜50% 過剰にプロビジョニングしている
- Graviton（ARM）インスタンスは、同等の x86 インスタンスより 20〜40% 優れたコストパフォーマンスを提供する
- CPU 使用パターンが変動するワークロードには、バースト可能なインスタンス（t3/t4g）を検討する

### Action Items
- Cost Explorer または Compute Optimizer でインスタンスの使用率を毎月確認する
- 低使用率が続く場合（7 日間 CPU < 10%）の CloudWatch アラームを設定する
- 非本番インスタンスのスケジュールを自動化する（午後 7 時に停止、午前 7 時に起動）

## Pricing Models

### On-Demand
- コミットメント不要、時間あたりのコストが最も高い
- 用途: 予測できないワークロード、短期的なスパイク、使用パターンが確立する前の新規アプリケーション

### Savings Plans
- 一定量のコンピュート使用量（$/時間で測定）に対する 1 年または 3 年のコミットメント
- **Compute Savings Plans**: EC2、Lambda、Fargate にまたがって適用される（最も柔軟）
- **EC2 Instance Savings Plans**: インスタンスファミリーとリージョンに固定される（割引率が高い）
- 一般的な削減率: 30〜40%（1 年、前払いなし）から 60〜72%（3 年、全額前払い）
- まずベースラインの使用量をカバーする Compute Savings Plans から始め、ピークには On-Demand を使う

### Reserved Instances
- 特定のインスタンスタイプ、リージョン、OS に対する 1 年または 3 年のコミットメント
- Savings Plans より柔軟性は低いが、割引率は同程度
- インスタンスタイプが非常に予測しやすい定常的なワークロードに限って検討する

### Spot Instances
- 最大 90% の割引だが、2 分前の通知で中断される可能性がある
- 用途: バッチ処理、CI/CD ワーカー、データ処理、Auto Scaling グループ配下のステートレスな Web サーバー
- ベストプラクティス: 中断頻度を下げるため、複数のインスタンスタイプと AZ に分散する
- 決して使わない用途: データベース、単一インスタンスのワークロード、中断を許容できないもの全般

## Storage: S3 Lifecycle Policies

### Recommended Transitions
```
S3 Standard (active data, frequent access)
  → 30 days → S3 Intelligent-Tiering (variable access patterns)
  → 90 days → S3 Infrequent Access (known infrequent access)
  → 180 days → S3 Glacier Instant Retrieval (rare access, millisecond retrieval)
  → 365 days → S3 Glacier Deep Archive (archive, 12-hour retrieval)
```

### Rules
- ライフサイクルルールを設定する前に、S3 Storage Lens でアクセスパターンを分析する
- アクセスパターンが予測できない場合は S3 Intelligent-Tiering を使う（移行を自動化できる）
- 未完了のマルチパートアップロードは 7 日後に削除する（気づかないうちに蓄積する）
- S3 analytics を有効にして、ライフサイクルポリシーの効果を検証する

### Quick Wins
- コンプライアンス上の保持期間を過ぎた S3 上の古い CloudTrail ログを削除する
- ELB/CloudFront のアクセスログを 90 日後に Glacier へ移行する
- アップロード前にオブジェクトを圧縮する（gzip、zstd）— ストレージと転送のコストを削減できる

## Database: DynamoDB

### On-Demand vs Provisioned Capacity
| 要素 | On-Demand | Provisioned |
|--------|-----------|-------------|
| トラフィックパターン | 予測不能、スパイクが多い | 安定、予測可能 |
| 料金 | リクエスト単位 | キャパシティユニット時間単位 |
| スケーリング | 即時（上限内） | 遅延を伴う Auto Scaling |
| 最適な用途 | 新規テーブル、開発／テスト、イベント駆動 | パターンが判明している本番環境 |

### Cost Reduction Strategies
- 定常的な本番テーブルには、Auto Scaling 付きのプロビジョンドキャパシティを使う
- 予測可能なベース負荷にはリザーブドキャパシティを有効にする（プロビジョンドに対する追加割引）
- TTL を使って期限切れの項目を自動削除する（TTL による削除には書き込みコストがかからない）
- 負荷が均等に分散するようにパーティションキーを設計する（ホットパーティションはプロビジョンドキャパシティを無駄にする）

## Lambda Cost Optimization

### Memory and Duration
- Lambda の料金 = （割り当てメモリ）×（実行時間）×（呼び出し回数）
- メモリを増やすと CPU も増える — メモリを増やすことで実行時間と総コストを削減できる場合がある
- AWS Lambda Power Tuning を使って、関数ごとに最適なメモリ設定を見つける

### Strategies
- コールドスタートを最小化する: デプロイパッケージを小さく保ち、共有依存関係にはレイヤーを使う
- ARM/Graviton ランタイム（`arm64`）を使うと、コストが 20% 削減され、パフォーマンスも向上する
- 適切なタイムアウトを設定する（3 秒で終わる関数に最大の 15 分を設定しない）
- SQS メッセージをバッチ処理する（1 回の呼び出しで最大 10 メッセージを受信）
- レイテンシー要件で必要な場合を除き、プロビジョニングされた同時実行は避ける（高価である）

### Invocation Reduction
- SQS のバッチウィンドウを使い、Lambda を呼び出す前にメッセージをためる
- コンテンツフィルタリング付きの EventBridge ルールを使い、関連するイベントの場合だけ呼び出す
- 結果を DynamoDB または ElastiCache にキャッシュし、重複するコンピュートを削減する

## Cost Allocation Tagging

### Required Tags
すべてのリソースに適用する最小限のタグセットを定義し、強制する:
```
Project:     project-name
Environment: dev | staging | prod
Team:        team-name
CostCenter:  cost-center-code
Service:     service-name
```

### Enforcement
- AWS Organizations の SCP を使い、必須タグのないリソース作成を拒否する
- CDK Aspects を使い、タグを自動的に付与し、タグの有無を検証する
- Billing コンソールでコスト配分タグを有効にする（Cost Explorer に表示するにはタグの有効化が必要）

## Cost Explorer Queries

### Monthly Cost Reviews
1. **サービス別コスト**: コスト要因の上位 5 つを特定する
2. **タグ（チーム）別コスト**: コストを担当チームに帰属させる
3. **日次コストの推移**: 想定外のコストスパイクを検出する
4. **使用タイプ別コスト**: 具体的なリソース（データ転送、API 呼び出し、ストレージ）を特定する

### Anomaly Detection
- AWS Cost Anomaly Detection を有効にし、想定外のコスト増加を自動通知する
- 想定される月間支出の 50%、80%、100% で予算アラートを設定する
- 環境ごと（本番と非本番）に別々の予算を作成する

### Monthly Cost Optimization Checklist
- [ ] 適正サイズ化のため Compute Optimizer の推奨事項を確認する
- [ ] アイドル状態のリソース（未使用の EIP、アタッチされていない EBS ボリューム、アイドル状態のロードバランサー）を確認する
- [ ] Savings Plans の使用率とカバレッジを確認する
- [ ] S3 ストレージのティア間の分布を確認する
- [ ] データ転送コスト（クロスリージョン、インターネットへの送信）を確認する
- [ ] 非本番環境が業務時間外に停止するようスケジュールされていることを確認する
- [ ] Power Tuning の結果を踏まえて Lambda 関数のメモリ設定を確認する
