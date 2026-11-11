# Market Research Methods

## Purpose
エンジニアリングの労力を投じる前に、競争環境を理解し、機会の規模を見積もり、情報に基づいた build-vs-buy（自社開発か購入か）の判断を下すための構造化されたアプローチ。

## Competitive Analysis Framework

### Step 1: Identify Competitors
- **直接競合**: 同じ製品、同じ市場（例: Slack と Teams）
- **間接競合**: 異なる製品、同じ課題（例: Slack とメール）
- **潜在的な競合**: 自社の領域に参入してくる可能性のある隣接分野のプレイヤー

### Step 2: Feature Comparison Matrix

| 機能 | 自社製品 | 競合 A | 競合 B | 競合 C |
|------------|-------------|-------------|-------------|-------------|
| コア機能 1 | | | | |
| コア機能 2 | | | | |
| 価格モデル | | | | |
| 対象セグメント | | | | |
| 連携エコシステム | | | | |
| 主要な差別化要因 | | | | |

それぞれを評価する: Strong / Adequate / Weak / Absent

### Step 3: Positioning Map
対象ユーザーにとって最も重要な次元を表す 2 つの軸（例: 使いやすさ対機能の強力さ、価格対網羅性）の上に競合をプロットする。十分にカバーされていない象限を特定する。

## SWOT Analysis Template

| | 好影響 | 悪影響 |
|---|---------|---------|
| **内部** | **Strengths（強み）**: 自社が得意なことは何か？ 自社だけが持つリソースは何か？ | **Weaknesses（弱み）**: 能力が不足しているのはどこか？ 競合の方が優れている点は何か？ |
| **外部** | **Opportunities（機会）**: 自社に有利な市場トレンドは何か？ 満たされていないニーズは何か？ | **Threats（脅威）**: 自社を揺るがしうるものは何か？ 競合は何を計画しているか？ |

### Making SWOT Actionable
- **Strengths + Opportunities** = 積極的に追求すべき戦略
- **Weaknesses + Threats** = 軽減または防御すべきリスク
- **Strengths + Threats** = 優位性を活かした防御的な戦略
- **Weaknesses + Opportunities** = 機会を活かすために必要な投資

## Porter's Five Forces

次の観点を分析して、業界の魅力度を評価する:

1. **新規参入の脅威**: この市場への参入はどれくらい容易か？（参入障壁が低い = 脅威が高い）
2. **供給者の交渉力**: 主要な供給者やプラットフォームにどれくらい依存しているか？（単一のクラウドプロバイダー = 交渉力が高い）
3. **買い手の交渉力**: 顧客は容易に乗り換えられるか？（スイッチングコストが低い = 交渉力が高い）
4. **代替品の脅威**: その課題は別の方法で解決できるか？（手作業のプロセス、異なる技術）
5. **既存競合との敵対関係**: 既存の競争はどれくらい激しいか？（類似製品が多い = 敵対関係が激しい）

各要因を High/Medium/Low で評価する。高い要因は利益率を圧迫し、魅力度を下げる。

## Market Sizing: TAM / SAM / SOM

### Definitions
- **TAM（Total Addressable Market）**: 世界中の市場の 100% を獲得した場合の総収益
- **SAM（Serviceable Addressable Market）**: TAM のうち、自社の製品／モデルが現実的に対応できる部分（地域、セグメント、チャネル）
- **SOM（Serviceable Obtainable Market）**: 競合とリソースを踏まえて、2〜3 年で現実的に獲得できる市場シェア

### Top-Down Calculation
業界レポートから始め、フィルターで絞り込む:
```
TAM = Total industry revenue for category
SAM = TAM x % in your geography x % in your segment
SOM = SAM x realistic capture rate (typically 1-5% for new entrants)
```

### Bottom-Up Calculation (More Reliable)
```
SOM = Target customers x Average deal size x Purchase frequency
```

両方の方法を使って比較する。大きく食い違う場合は、前提を調べる。

## Build vs Buy Assessment

### Evaluation Criteria

| 要因 | Build | Buy | スコア（Build -2〜+2） |
|--------|-------|-----|----------------------|
| コアの差別化要因か？ | 競争優位であれば Build | コモディティ化した機能は Buy | |
| 市場投入までの時間 | 数か月〜数年 | 数日〜数週間 | |
| 総コスト（3 年間） | 開発 + 保守 + 機会費用 | ライセンス + 統合 + ベンダーリスク | |
| 必要なカスタマイズ | 高度なカスタマイズは Build に有利 | 標準的なニーズは Buy に有利 | |
| データの機密性 | 完全に管理できる | ベンダーのデータ取り扱いポリシーに依存 | |
| チームの専門性 | 社内のスキルが必要 | 複雑さをベンダーに移せる | |
| 保守の負担 | 継続的な責任を負う | ベンダーが更新を担う | |

### Decision Rule
その機能がコアの差別化要因ではなく、成熟したベンダーのソリューションが存在する場合は、デフォルトで Buy とする。Build を選ぶのは、その機能が競争優位の中心である場合、またはベンダーのソリューションでは根本的に要件を満たせない場合に限る。

## Trend Analysis Methods

### Approaches
- **テクノロジーレーダー**: 新興技術を Adopt / Trial / Assess / Hold に分類する
- **ハイプサイクルのマッピング**: 関連する技術が Gartner のハイプサイクルのどこに位置するかを特定する
- **顧客シグナルの分析**: サポートチケット、機能要望、解約理由を掘り下げ、新たなパターンを見つける
- **隣接市場の監視**: 波及してくる可能性のある関連業界の動向を追跡する

### Validation
トレンドは仮説である。次の方法で検証する:
- トレンドが購買判断に影響していることを確認する顧客インタビュー
- 市場の動きを示す収益データ（メディアでの報道だけではないもの）
- 競合の投資シグナル（採用の傾向、買収、製品の発売）
