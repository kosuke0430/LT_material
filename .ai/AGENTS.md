# AGENTS.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## リポジトリ概要

これは日本語の技術プレゼンテーション資料を管理するLT（Learning & Teaching）資料リポジトリです。Marpベースのプレゼンテーションを作成し、PDF形式で配布するための資料を管理しています。

## プロジェクト構造

```
LT_material/
├── FY25/           # 昨年度の資料
│   ├── *.md                # Marpプレゼンテーションソースファイル
│   ├── images/             # プレゼンテーション別に整理された画像
│   │   ├── architecture-test/
│   │   ├── design-training-fy2024/
│   │   ├── do-not-use-join-in-mysql/
│   │   └── how-to-use-marp/
│   ├── outputs/            # 生成されたPDFファイル
│   └── Base/               # ベーステンプレートや共有コンテンツ
├── FY26/           # 現年度の資料
│   ├── *.md                # Marpプレゼンテーションソースファイル
│   ├── images/             # プレゼンテーション別に整理された画像
│   │   ├── architecture-test/
│   │   ├── design-training-fy2024/
│   │   ├── do-not-use-join-in-mysql/
│   │   └── how-to-use-marp/
│   ├── outputs/            # 生成されたPDFファイル
│   └── Base/               # ベーステンプレートや共有コンテンツ
└── README.md
```

## コンテンツ種別

### プレゼンテーション資料
- **ArchitectureTest.md**: PHP/Laravel + Pestフレームワークを使用したアーキテクチャテスト
- **DesignTrainingFY2024.md**: ECサイトを例としたシステム設計トレーニング
- **DoNotUseJoinInMySQL.md**: MySQL JOINの最適化技術
- **HowToUseMarp.md**: Marp + Cursorを使った高速スライド作成ガイド

### 技術スタック
- **Marp**: Markdown からスライドへの変換ツール
- **PDF Export**: プレゼンテーションの最終出力形式

## Marpファイルの操作

### Marp設定
すべてのプレゼンテーションファイルは以下の共通設定を使用:
```yaml
---
marp: true
size: 4:3
theme: default
paginate: true
title: [プレゼンテーションタイトル]
author: [日付] [著者名]
---
```

### コンテンツ作成ワークフロー
1. 標準的なMarkdown形式でコンテンツを記述
2. CursorのAI機能を使ってMarp形式のスライドに変換
3. 対応する`images/`サブディレクトリから画像を追加
4. PDFにエクスポートして`outputs/`ディレクトリに保存

### 画像管理
- 画像はプレゼンテーション topic別に`images/`サブディレクトリで整理
- Markdownでは相対パスを使用: `./images/[topic]/[filename]`
- 対応形式: PNG、JPEG

## ファイル操作

### 新しいプレゼンテーションの作成
- `FY26/`ディレクトリに`.md`ファイルを作成
- 必要に応じて対応する画像サブディレクトリを作成
- 既存の命名規則に従う（ファイル名はPascalCase）

### PDF生成
- VS Code/CursorのMarp拡張機能を使用してPDFにエクスポート
- 生成されたPDFは`outputs/`ディレクトリに保存
- 同じベースファイル名を維持（例: `HowToUseMarp.md` → `HowToUseMarp.pdf`）

## 開発ノート

- コンテンツは主に日本語で記述
- プレゼンテーションはアーキテクチャ、データベース最適化、開発ツールなどの技術トピックを扱う
- 知識共有とトレーニング資料に焦点を当てている
- ビルドプロセスや依存関係なし - ドキュメント専用リポジトリ
- 15分程度の発表資料としてまとめること