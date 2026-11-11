# Component Specification Template

`interaction-spec.md`（Stage 1.5 Refined Mockups）や、詳細な UI コンポーネント定義を必要とするあらゆるステージで、コンポーネントレベルの仕様にこのテンプレートを使用する。

---

## [Component Name]

| 項目 | 値 |
|---|---|
| Component | [名前] |
| Description | [目的を1行で] |
| Category | [input / display / layout / navigation / feedback] |

### States

| State | 説明 | トリガー |
|---|---|---|
| default | 初期描画の状態 | ページ読み込み |
| hover | 要素上にカーソルがある | mouseover |
| focus | キーボードフォーカス | Tab キー / クリック |
| disabled | 操作不可 | prop disabled=true |
| loading | 非同期処理の待機中 | 非同期処理の実行中 |
| error | バリデーションエラーまたはシステムエラー | バリデーション失敗 |
| empty | 表示するデータがない | データなし |

### Props / Inputs

| Prop | 型 | 必須 | デフォルト | 説明 |
|---|---|---|---|---|
| [prop 名] | [string \| boolean \| number \| object] | [yes/no] | [値 または —] | [制御する内容] |

### Responsive Behaviour

| ブレークポイント | 振る舞い |
|---|---|
| mobile (<768px) | [レイアウト/表示の変化] |
| tablet (768–1024px) | [レイアウト/表示の変化] |
| desktop (>1024px) | [デフォルトのレイアウト] |

### Accessibility

| 要件 | 実装 |
|---|---|
| ARIA ロール | [ロール — 例: button, listbox, dialog] |
| キーボード操作 | [Tab でフォーカス、Enter/Space で実行、Escape で閉じる] |
| ラベル / aria-label | [可視ラベル、aria-label、aria-labelledby のいずれの方式か] |
| コントラスト比 | WCAG AA（テキストは 4.5:1、UI コンポーネントは 3:1） |
| スクリーンリーダー | [何がいつ読み上げられるか] |
| フォーカス管理 | [開く/閉じる/実行時にフォーカスがどこへ移るか] |

### Usage Example

```
<ComponentName
  prop="value"
  onAction={handler}
/>
```

---
