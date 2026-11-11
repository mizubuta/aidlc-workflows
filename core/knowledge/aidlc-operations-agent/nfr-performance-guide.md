# NFR Performance and Scalability Guide

> このガイドは、Security Engineer（NFR Requirements ステージのリードエージェント）が持つ完全な NFR Requirements Guide を補足するものである。NFR Requirements と NFR Design のステージで、インフラに焦点を当てた貢献を行うために必要なパフォーマンスとスケーラビリティのセクションを DevOps Engineer に提供する。

## Performance Requirement Benchmarking

### How to Define Performance Requirements
すべてのパフォーマンス要件には、次のものを必ず明記する。
- **メトリクス**: 何を計測するか（応答時間、スループット、リソース使用量）
- **目標**: 定量的なしきい値
- **パーセンタイル**: 目標がどのパーセンタイルに適用されるか（p50、p95、p99）
- **負荷条件**: どの程度の並行数/トラフィックのもとで目標を満たさなければならないか
- **計測方法**: メトリクスをどのように、どこで取得するか

### Common Performance Targets by Application Type

| アプリケーションの種類 | メトリクス | 目標 | パーセンタイル | 備考 |
|-----------------|--------|--------|------------|-------|
| Web アプリケーション（対話型） | ページの読み込み時間 | < 2s | p95 | First Contentful Paint |
| REST API（同期） | 応答時間 | < 200ms | p99 | ネットワークの転送時間を除く |
| 検索クエリ | 結果の提供 | < 500ms | p95 | ランキングを含む |
| バッチ処理 | スループット | N レコード/分 | 平均 | 許容できる最低の速度を定義する |
| リアルタイムメッセージング | エンドツーエンドのレイテンシー | < 100ms | p99 | 送信から配信まで |
| ファイルのアップロード | 処理時間 | MB あたり < 5s | p95 | アップロード後の処理 |

### Performance Anti-Requirements
不合理な期待は明示的に除外する。
- "The system should be fast"（システムは高速であるべき）（計測できない）
- "All pages load instantly"（すべてのページが即座に読み込まれる）（目標もパーセンタイルもない）
- "Handle unlimited users"（無制限のユーザーに対応する）（容量の上限がない）

次のように置き換える: "The dashboard page loads in < 3 seconds at p95 under 500 concurrent users."（同時ユーザー 500 人のもとで、ダッシュボードページが p95 で 3 秒未満で読み込まれる。）

## Scalability Assessment Frameworks

### Capacity Planning Template

| 観点 | 現在 | 6 か月後の目標 | 12 か月後の目標 | スケーリングの仕組み |
|-----------|---------|---------------|-----------------|-------------------|
| 同時ユーザー数 | N | N * X | N * Y | 水平オートスケーリング |
| データ量 | N GB | N * X GB | N * Y GB | パーティショニング戦略 |
| 1 秒あたりのリクエスト数 | N | N * X | N * Y | ロードバランサー + レプリカ |
| ストレージの増加率 | N GB/月 | N * X GB/月 | N * Y GB/月 | 階層型ストレージのポリシー |

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
| CPU が常に 70% 超 | 単一インスタンスのワークロード | ステートレスなサービス | 非効率なアルゴリズムを確認する |
| メモリの逼迫 | インメモリ処理 | 分散キャッシュ | メモリリークを確認する |
| I/O 待ちが 30% 超 | より高速なストレージ階層 | リードレプリカ、シャーディング | クエリの最適化、インデックス作成 |
| キューの深さが増加 | より高速なコンシューマー | コンシューマーのインスタンスを増やす | バッチサイズの調整 |
| コネクションプールの枯渇 | プールサイズの拡大 | コネクションの多重化 | コネクションリークの調査 |
