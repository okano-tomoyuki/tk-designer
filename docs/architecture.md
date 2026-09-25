# アーキテクチャ

関連する ADR: [0001](adr/0001-vscode-extension.md)（VS Code 拡張）、[0002](adr/0002-codegen-targets.md)（生成ターゲット）、
[0003](adr/0003-react-for-webview.md)（React）、[0005](adr/0005-monorepo-and-toolchain.md)（モノレポ・ツールチェーン）、
[0006](adr/0006-document-state-and-undo.md)（状態管理・Undo）、[0007](adr/0007-widget-catalog-from-tk.md)（ウィジェットカタログ）

## 1. リポジトリ構成

```
tk-designer/
├─ packages/
│  ├─ core/        DSL の型・JSON Schema・検証・ウィジェットカタログ・ドキュメント操作コマンド
│  ├─ codegen/     中間表現・C++/Python エミッタ・マーカー区間のマージ
│  ├─ cli/         コード生成 CLI（CMake 等から実行）
│  ├─ extension/   VS Code 拡張ホスト（Custom Editor、WorkspaceEdit、コード生成の実行）
│  └─ webview/     デザイナー UI（React）
├─ tools/
│  └─ catalog/     Tk からウィジェットカタログを抽出するスクリプト（開発時のみ使用）
├─ docs/
├─ pnpm-workspace.yaml
└─ package.json
```

## 2. パッケージの責務と依存関係

```
webview   ──────────────────────▶ core
codegen   ──────────────────────▶ core
cli       ──▶ codegen, core
extension ──▶ codegen, core
```

core はどのパッケージにも依存しない。矢印の逆向き（core → 他）や、webview ⇄ extension の直接 import は禁止する
（extension と webview の間はメッセージでのみ通信し、メッセージ型は core に置く）。

| パッケージ | 責務 | 使ってよいもの | 使ってはいけないもの |
|---|---|---|---|
| core | DSL の型定義、JSON Schema、検証、カタログの読み込み、ドキュメント操作コマンド（純粋関数）、決定的シリアライズ、拡張と Webview 間のメッセージ型 | Immer、スキーマ検証ライブラリ | VS Code API、DOM、Node API（fs 等） |
| codegen | DSL → 中間表現 → 各言語のコード片、既存ファイルとのマージ（文字列入力 → 文字列出力） | core | VS Code API、ファイル入出力 |
| cli | 引数解析、ファイル入出力、codegen の呼び出し | core、codegen、Node API | VS Code API |
| extension | Custom Editor の提供、Webview との通信、WorkspaceEdit の適用、コード生成・新しい画面の作成のコマンド | core、codegen、VS Code API | React |
| webview | デザイナー UI（キャンバス、ツリー、プロパティパネル、パレット、コード生成の画面） | core、codegen（名前の既定値など純粋関数のみ）、React、Zustand | VS Code API（`acquireVsCodeApi` 経由の通信を除く）、Node API |

**方針**: ロジックはできるだけ core / codegen に置き、extension と webview は薄く保つ。
core と codegen はファイル入出力を持たないので、入出力が文字列だけのテストで網羅できる。

## 3. 主要なデータの流れ

### 編集（ADR 0006）

```
[webview] 操作 → コマンド → postMessage
[extension] core でコマンド適用 → シリアライズ → WorkspaceEdit → TextDocument
[extension] onDidChangeTextDocument → パース・検証 → postMessage(document)
[webview] ドキュメントストア更新 → 再描画
```

### コード生成

```
[extension / cli] DSL を読み込み → core で検証
  → codegen: 中間表現 → エミッタ → 既存ファイルとマージ
  → extension: WorkspaceEdit で適用 ／ cli: ファイルに書き込み
```

## 4. 技術スタック

| 用途 | 採用 |
|---|---|
| 言語 | TypeScript（strict） |
| パッケージ管理 | pnpm workspaces |
| Webview UI | React + Zustand（+ Immer） |
| Webview ビルド | Vite |
| 拡張・CLI ビルド | esbuild |
| テスト | Vitest（コード生成はゴールデンファイルテスト） |
| 静的解析・整形 | ESLint（flat config + typescript-eslint）、Prettier |
| 拡張の梱包 | vsce（`--no-dependencies`、バンドル済み成果物のみ梱包） |

## 5. 未決の論点

| # | 論点 | メモ |
|---|---|---|
| A1 | ~~キャンバス上のレイアウト計算~~ | **決定**: Tk の pack / grid のアルゴリズムを TS で core に実装する（[ADR 0008](adr/0008-layout-engine-in-core.md)）。 |
| A2 | ~~DSL の型と JSON Schema の単一化~~ | **決定**: Zod 4 でスキーマを core に定義し、型と JSON Schema を導出する（[ADR 0009](adr/0009-dsl-schema-in-zod.md)）。 |
| A3 | Webview の UI 部品 | VS Code 公式の Webview UI Toolkit は開発終了済み。VS Code のテーマ変数（CSS 変数）を使った自前部品か、`@vscode-elements` 等のライブラリか。 |
| A4 | 実機プレビュー | 生成した Tcl / Python を wish / python で実行して本物の表示を確認する機能。A1 の案c とも関係する。 |
| A5 | ttk スタイルの扱い | ADR 0007 の影響欄を参照。 |
