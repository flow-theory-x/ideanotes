# Apache vhost管理ツール - 初期アイデア

*作成日: 2025-07-12*

## 🎯 背景・ニーズ

EC2環境でApacheが稼働中、vhostで複数のサービスを公開予定。
現在のvhost管理の課題と、ツール化による効率化を検討。

## 💡 初期アイデア

### 想定する機能
- vhost設定の一覧表示
- 新規vhost設定の追加（テンプレート利用）
- 既存vhost設定の編集・削除
- 設定の自動バックアップ
- SSL証明書の管理（Let's Encrypt連携？）
- サービスの稼働状況モニタリング

### 技術スタック候補
- バックエンド: Python/Node.js（Apache設定ファイル操作）
- フロントエンド: シンプルなWeb UI
- 設定管理: YAML/JSONで管理？

## 🤔 検討事項

- 権限管理（Apache設定の変更にはsudo必要）
- 設定変更時の自動reload/restart
- 設定ミスの検証機能
- 複数環境への展開方法

## 🔍 現状確認 (2025-07-12)

### Apache環境
- ✅ Apache httpd稼働中 (ポート80,443)
- ✅ 既存vhost: `dev.example.com` 
- ✅ SSL証明書導入済み (Let's Encrypt)
- 📁 設定ファイル: `/etc/httpd/conf.d/`

### 現在の設定パターン
```
/etc/httpd/conf.d/
├── dev.example.com.conf (HTTP→HTTPS redirect)
└── dev.example.com-le-ssl.conf (HTTPS設定)
```

### 現在の運用状況
- **メインアプリ**: `/var/www/html/dev.example.com` (SPA)
- **開発ディレクトリ**: `/develop` エイリアス → `/home/{username}/develop`
- **API プロキシ**: AWS API Gateway連携
- **プロジェクト例**: project1, project2, project3等

### サブドメイン運用の可能性
- 同一EIPに複数サブドメイン向け可能
- 各プロジェクト専用vhost作成で分離
- 開発環境の整理・本格化が可能

## 🎯 対象サービス

### JS製webアプリケーション
- 静的ファイル配信がメイン（HTML, CSS, JS）
- SPAパターンが多い（React, Vue等）
- API連携は別サーバー想定

## 📝 次のステップ

1. ✅ 現在のApache設定を確認 
2. ✅ 管理したいサービスのリストアップ → JS製webアプリ
3. 最小機能セット（MVP）の定義
4. JS webアプリ向け設定テンプレートの作成