# jsontools プロジェクト概要

*取り込み日: 2025-07-12*

## 📋 基本情報

- **プロジェクト名**: JSON Tools
- **パス**: `/home/ec2-user/develop/jsontools`
- **説明**: JSONデータを効率的に操作・閲覧するためのWebベースツール集
- **技術**: HTML + JavaScript (Vanilla)
- **現状**: GitHub Pages公開済み

## 🛠️ 既存機能

### 1. JSON Formatter (`json-formatter.html`)
- ラベルソート: JSONキーのアルファベット順ソート
- 空白値除去: null、空文字列等の自動削除
- シンタックスハイライト: 見やすい色分け表示

### 2. JSON Hierarchy Viewer (`json-hierarchy-viewer.html`)  
- 階層構造の可視化

### 3. XPath Researcher (`xpath-researcher.html`)
- XPath抽出テスト機能

### 4. メインページ (`index.html`)
- ツール一覧とナビゲーション

## 🌐 現在の公開状況

- **GitHub Pages**: https://goodsun.github.io/jsontools/
- **開発用アクセス**: dev2.bon-soleil.com/develop/jsontools/

## 💡 改善・拡張の可能性

### UI/UX改善
- レスポンシブデザイン対応
- ダークモード対応
- ドラッグ&ドロップでファイル読み込み

### 新機能候補
- JSON差分比較ツール
- JSON→CSV変換
- JSONスキーマ検証
- APIレスポンス用フォーマッター

### 運用改善
- 専用サブドメイン: `jsontools.bon-soleil.com`
- 使用統計の取得
- ユーザーフィードバック収集

## 🎯 ideanotes連携の価値

1. **継続開発**: 思いついた改善をすぐ記録・議論
2. **機能拡張**: スモールスタート原則で段階的追加
3. **運用最適化**: vhost管理ツールで簡単デプロイ
4. **ユーザー視点**: 実際の使用から生まれるアイデア

## 📝 次のアクション候補

1. 専用vhostでの独立公開
2. 使い勝手の改善アイデア収集
3. 新機能の優先順位付け