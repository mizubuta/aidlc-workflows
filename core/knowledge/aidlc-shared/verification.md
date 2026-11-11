# Automatic Verification — Element-Level Traceability

以下の `<record>` はアクティブなワークフローの記録
`aidlc/spaces/<active-space>/intents/<active-intent>` を指し、
`aidlc/active-space`（デフォルトは `default`）と
`aidlc/spaces/<active-space>/intents/active-intent` によって選択される。

## Per-Stage Coverage

要件、ストーリー、設計、またはコードを変換するステージは、宣言された
`traceability.json` 成果物を生成する。`traceability` センサーは各ファイルが
書き込まれた時点でそれを検証し、ステージの成果物コントラクトによって、その欠落は
通常のディレクティブ、レビュー、完了エビデンスの経路で可視化される。

すべてのファイルは安定した ID を使用する:

| Prefix | Meaning | Example |
|--------|---------|---------|
| `FR{n}` / `FR{n}.{m}` | 機能要件 | `FR1`, `FR1.2` |
| `NFR{n}` | Inception の非機能要件 | `NFR2` |
| `US{n}.{m}` | ユーザーストーリー | `US1.3` |
| `AC{n}.{m}.{seq}` | 受け入れ基準 | `AC1.3.2` |
| `U{n}` / `u{n}-{description}` | ユニット ID / ユニット名（その construction ディレクトリ） | `U1`, `u1-auth` |
| `BR{group}.{seq}` | ビジネスルール | `BR1.1` |
| `NFRx.y` | 詳細な NFR 要件 | `NFR2.1` |

JSON の形式は次のとおり:

```json
{
  "stage": "functional-design",
  "unit": "u1-auth",
  "upstream_ids": ["AC1.1.1"],
  "coverage": [
    { "id": "AC1.1.1", "status": "OK", "target": "BR1.1" }
  ],
  "reverse": [
    { "id": "BR1.3", "status": "N/A", "target": "technical validation rule" }
  ]
}
```

有効なステータスは `OK`、`GAP`、`ORPHAN`、`Deferred`、`N/A` である。`OK`、
`Deferred`、`N/A` には、空でないターゲットまたは根拠が必要である。上流の
すべての ID は `upstream_ids` と `coverage` に入れ、該当しないものはそこで `N/A` とし、
該当しない理由をターゲットとして記す。`reverse` には、ステージが生成したもののうち、
上流の何にも辿れないものだけを列挙する。サブ要件を持つ要件は、その
`FR{n}.{m}` ID を通じてトレースされ、それ自体の行は不要である。センサーは
ソース成果物から上流 ID を照合し、可能な場合は決定的なターゲットを検証し、
functional-design のビジネスルールの孤立項目（orphan）を導出する。

## When Verification Runs

| Trigger | What's Checked |
|---------|---------------|
| **Ideation → Inception** | インテント → スコープ → インテントバックログの一貫性。すべてのスコープ項目に実現可能性の裏付けがあること |
| **Inception → Construction** | 要件 → ストーリー → アーキテクチャの整合。すべてのストーリーが要件にトレースされること。アーキテクチャがすべてのストーリーをカバーすること |
| **Construction → Operation** | アーキテクチャ → コード → テストの整合。すべてのコードが設計にトレースされること。受け入れ基準に対するテストカバレッジ |
| **On demand** | 人間はいつでも検証を要求できる |
| **Stage output write** | ステージの要素レベルのカバレッジとターゲットを検証する |

## Phase Check Output

各フェーズ境界のチェックは `<record>/verification/phase-check-<phase>.md` を生成する:
- カバレッジの割合（ストーリーを持つ要件、コンポーネントを持つストーリーなど）
- 警告（不完全なマッピング）
- 一貫性チェック（フェーズ間に矛盾がないこと）
- 人間による承認のチェックボックス

## Verification Process

1. 完了したフェーズから、宣言されたすべてのトレーサビリティ成果物を読み込む
2. 安定した ID からトレーサビリティチェーンを再構築する
3. ギャップ、無効なターゲット、導出された孤立項目、矛盾を特定する
4. 検証レポートを生成する
5. 結果をレビューのために提示する
6. アクティブな記録の `audit/<host>-<clone-id>.md` シャードへの `PHASE_VERIFIED` の記録は
   エンジンに任せる。手作業で追記しない
