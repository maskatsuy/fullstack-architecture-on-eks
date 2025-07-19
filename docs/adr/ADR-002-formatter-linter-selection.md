# ADR-002: フォーマッター・リンターツールの選定

- Status: Accepted
- Date: 2025-07-19
- Deciders: [@maskatsuy]

## コンテキスト
本プロジェクトでは、TypeScript/JavaScript、GraphQL、CSSなど複数の言語を使用するモノレポ構成において、統一的なコードスタイルとコード品質の維持が必要である。開発効率とコード品質の両立を図るため、適切なフォーマッター・リンターツールの選定が求められる。

### 要件
1. TypeScript/JavaScript、GraphQL、CSSのフォーマット対応
2. 高速な実行速度（モノレポでの大量ファイル処理）
3. ESLintルールとの互換性または統合
4. 主要エディタ（VSCode、IntelliJ）のサポート
5. CI/CDパイプラインへの容易な統合
6. 開発者体験（DX）の向上

## 検討した選択肢

### 1. Prettier + ESLint（従来の標準構成）
- **利点**: 
  - デファクトスタンダード、豊富なドキュメント
  - 幅広いプラグインエコシステム
  - あらゆる言語・フレームワークに対応
- **欠点**: 
  - 実行速度が遅い（特に大規模プロジェクト）
  - ESLintとの設定競合の可能性
  - 2つのツールの設定管理が必要

### 2. Biome（Rust製の統合ツール）
- **利点**: 
  - Prettierの25倍高速（Rust製、マルチスレッド）[^1]
    - 公式ベンチマーク: 96,000ファイル（~650MB）をPrettier 10.3秒 vs Biome 0.4秒
    - 環境: Linux 5.19, AMD Ryzen 5950X, 64GB RAM
  - フォーマッターとリンターを統合
  - Prettier互換モードで移行が容易
  - 97%のPrettier互換性
  - VSCode/IntelliJ公式拡張機能
- **欠点**: 
  - YAMLやMDXは未対応
  - 一部のPrettierプラグイン（tailwindcss等）が使用不可
  - 比較的新しいツール（ただし活発に開発中）

### 3. dprint（Rust製、プラグインベース）
- **利点**: 
  - 高速実行
  - プラグインによる拡張性
- **欠点**: 
  - コミュニティが小さい
  - ドキュメントが少ない
  - エディタサポートが限定的

### 4. ESLint単体（--fixオプション使用）
- **利点**: 
  - 単一ツールでの管理
  - 既存のESLintルールをそのまま使用
- **欠点**: 
  - フォーマット機能が限定的
  - Prettierより遅い
  - スタイル統一が困難

## 決定
**Biome** を主要なフォーマッター・リンターとして採用し、未対応の言語には補助ツールを使用する。

## 理由
1. **パフォーマンス**: Prettierの25倍の速度により、開発時の待ち時間を大幅削減
2. **統合性**: フォーマッターとリンターが統合されており、設定の競合がない
3. **移行容易性**: Prettier互換モードにより、既存のコードベースから段階的に移行可能
4. **エディタサポート**: VSCode/IntelliJ公式拡張機能により、保存時の自動フォーマットが高速
5. **将来性**: 元Rome（現Biome）チームによる活発な開発、2025年にはプラグインシステムも計画

## 実装方針
1. **Biomeの適用範囲**:
   - TypeScript/JavaScript/JSX/TSX
   - JSON
   - GraphQL
   - CSS

2. **補助ツール**:
   - YAML: yamlfmt または prettier（YAML専用）
     - コマンド: `pnpm format:yaml`
     - VSCode: `defaultFormatter: "esbenp.prettier-vscode"`
   - Markdown/MDX: markdownlint-cli2
     - コマンド: `pnpm lint:md`
     - VSCode: `defaultFormatter: "esbenp.prettier-vscode"`
   - Rust: rustfmt（標準ツール）
     - コマンド: `cargo fmt`
     - VSCode: rust-analyzer拡張機能が自動処理

3. **段階的移行**:
   - 初期設定はPrettier互換モードを使用
   - チームが慣れた後、Biome推奨ルールへ段階的に移行

4. **CI/CD統合**:
   - `biome check`でフォーマットとリントを一括実行
   - プルリクエスト時の自動チェック
   - CI実行順: 1) `biome check` 2) `yamlfmt` 3) `markdownlint` 4) `cargo fmt --check`

## 影響
- **ポジティブ**: 
  - 開発時のフィードバックループ高速化
  - CI/CD時間の短縮（特にlint/format検証）
  - 設定ファイルの簡素化
  - エディタでの快適な開発体験

- **ネガティブ**: 
  - チームメンバーの学習コスト（ただし、基本的な使用方法はPrettierと同様）
  - 一部のPrettierプラグインが使用不可
  - YAML/MDXには別ツールが必要

## 参考資料
- [Biome公式ドキュメント](https://biomejs.dev/)
- [Biome vs Prettier比較](https://biomejs.dev/formatter/differences-with-prettier/)
- [Migrating from Prettier and ESLint](https://biomejs.dev/guides/migrate-eslint-prettier/)

[^1]: [Biome公式ブログ - パフォーマンスベンチマーク](https://biomejs.dev/blog/biome-wins-prettier-challenge/)