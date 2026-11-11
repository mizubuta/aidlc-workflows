# Infrastructure Guide

## IaC Tool Selection

| ツール | 最適な用途 | 考慮事項 |
|------|----------|---------------|
| AWS CDK | AWS ネイティブ、TypeScript/Python のチーム、複雑なコンストラクト | ベンダーロックイン、学習曲線が急 |
| Terraform | マルチクラウド、チームでの標準化、成熟したエコシステム | HCL 構文、状態管理の複雑さ |
| CloudFormation | AWS のみ、シンプルなスタック、CDK では過剰な場合 | 冗長な YAML/JSON、ドリフト検出が遅い |
| Pulumi | 汎用言語を使いたい多言語のチーム | コミュニティが小さい、状態バックエンドの選択が必要 |
| Docker Compose | ローカル開発、シンプルなマルチコンテナーのアプリ | 本番のオーケストレーション向けではない |

## CI/CD Pipeline Design

### Standard Pipeline Stages
```
[Source] -> [Lint] -> [Build] -> [Unit Test] -> [SAST] -> [Package] ->
[Integration Test] -> [Deploy Staging] -> [E2E Test] -> [Security Scan] ->
[Approval Gate] -> [Deploy Production] -> [Smoke Test] -> [Monitor]
```

### Stage Requirements
- **Lint**: フォーマットと静的解析の違反で早期に失敗させる。30 秒未満。
- **Build**: コンパイル/トランスパイルし、依存関係を解決する。積極的にキャッシュする。2 分未満。
- **Unit Test**: スイート全体を実行する。1 つでも失敗したらパイプラインを失敗させる。3 分未満。
- **SAST**: 静的なセキュリティスキャン。high/critical の検出結果でブロックする。5 分未満。
- **Package**: コンテナーイメージまたはデプロイ用の成果物をビルドする。コミット SHA でタグ付けする。
- **Integration Test**: テスト用データベースとモック化した外部サービスに対して実行する。10 分未満。
- **Deploy Staging**: 自動化する。縮小した規模で本番のトポロジーを再現する。
- **E2E Test**: 重要な経路のみ。15 分未満。不安定なテストはスキップせず、隔離する。
- **Security Scan**: ステージングに対する DAST。依存関係の脆弱性チェック。
- **Approval Gate**: 本番への手動承認（任意。リスク許容度に応じて）。
- **Deploy Production**: 選択したデプロイ戦略で自動化する。
- **Smoke Test**: デプロイ後にコアのエンドポイントが正しく応答することを確認する。2 分未満。

## Deployment Strategies

### Blue-Green
- 2 つの同一の環境を用意し、トラフィックをアトミックに切り替える
- **長所**: 即時のロールバック、ダウンタイムなし
- **短所**: インフラのコストが 2 倍、データベーススキーマの同期が複雑
- **使用する場面**: ダウンタイムなしが必須で、データベースの変更が後方互換である場合

### Canary
- トラフィックのごく一部（1〜5%）を新しいバージョンに振り向け、徐々に増やす
- **長所**: 影響範囲が限定される、実ユーザーでの検証
- **短所**: トラフィックの分割とメトリクス比較の自動化が必要
- **使用する場面**: トラフィックの多いシステムで、実際の負荷のもとで検証する必要がある場合

### Rolling
- インスタンスを段階的に置き換える（1 つずつ、または N 個ずつ）
- **長所**: 追加のインフラが不要、段階的なロールアウト
- **短所**: 複数のバージョンが同時に稼働する、ロールバックが遅い
- **使用する場面**: ステートレスなサービス、後方互換の変更

### Recreate
- 古いインスタンスをすべて停止し、新しいインスタンスをすべて起動する
- **長所**: シンプル、バージョンが混在しない
- **短所**: 切り替え中にダウンタイムが発生する
- **使用する場面**: メンテナンスウィンドウが許容される場合、破壊的な変更

## Monitoring & Observability Stack

### The Four Pillars
1. **メトリクス**: 時間とともに変化する数値の計測（CPU、レイテンシー、エラー数）
   - ツールの例: CloudWatch、Prometheus + Grafana、Datadog
   - 主要なメトリクス: サービスには RED（Rate、Errors、Duration）、リソースには USE（Utilization、Saturation、Errors）

2. **ログ**: アプリケーションとインフラからの構造化されたイベントの記録
   - 形式: timestamp、level、service、traceId、message、context を含む JSON
   - ツールの例: CloudWatch Logs、ELK スタック、Loki
   - 保持期間: ホット 30 日、ウォーム 90 日、コールド 1 年

3. **トレース**: サービスをまたぐリクエストの流れ
   - ツールの例: X-Ray、Jaeger、Zipkin、OpenTelemetry
   - 計装の対象: HTTP ハンドラー、データベース呼び出し、外部 API 呼び出し、キュー操作

4. **アラート**: 異常の自動通知
   - 原因（CPU 80% 超）ではなく、症状（エラー率 1% 超）でアラートを出す
   - 重大度レベル: P1（呼び出し）、P2（チケット）、P3（ダッシュボード）
   - すべてのアラートに Runbook へのリンクを含める

## Container Orchestration Checklist

コンテナー化したデプロイでは、次のものを定義する。
- [ ] ベースイメージの選択（最小限、セキュリティパッチ適用済み、バージョン固定）
- [ ] 本番イメージを小さくするためのマルチステージビルド
- [ ] ヘルスチェックのエンドポイント（`/health` または `/readyz`）
- [ ] グレースフルシャットダウンの処理（SIGTERM、接続のドレイン）
- [ ] noisy neighbor の問題を防ぐためのリソース制限（CPU、メモリ）
- [ ] シークレットの管理（イメージ内にも環境変数にも置かない -- シークレットマネージャーを使う）
- [ ] ログを stdout/stderr に出力する（ファイルベースにしない）
- [ ] コンテナー内では root 以外のユーザーを使う
- [ ] 可能な場合は読み取り専用のファイルシステムにする

## Environment Strategy

```
Local Dev    -> Developer laptop, docker-compose, hot reload
CI           -> Ephemeral, created per pipeline run, destroyed after
Staging      -> Persistent, mirrors production topology, reduced scale
Production   -> Full scale, multi-AZ, monitoring and alerting active
```

同等性のルール:
- ステージングは本番と同じ IaC テンプレートを必ず使う（規模はパラメーター化する）
- ステージングは本番と同じデータベースエンジンとバージョンを必ず使う
- ステージングには代表的な（匿名化した）データ量を持たせるべきである
