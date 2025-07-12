# サブドメイン運用戦略

*作成日: 2025-07-12*

## 🎯 現在の課題と機会

### 現状の設定
- `dev.example.com` に全てが集約
- `/develop` エイリアスで開発ディレクトリを公開
- 複数プロジェクトが同一ドメイン下で混在

### 改善の方向性
**各プロジェクト専用のサブドメインを作成し、独立したvhostで運用**

## 🌐 サブドメイン運用案

### 想定パターン
```
project1.example.com     → project1
myapp.example.com       → myappプロジェクト  
project2.example.com    → project2プロジェクト
tools.example.com       → 開発ツール集（汎用）
```

### DNS設定
- 各サブドメインを同一EIPに向ける
- ワイルドカード設定も可能: `*.example.com`

## 📋 vhost管理ツールへの反映

### 機能拡張
1. **サブドメイン自動生成**
   - プロジェクト名からサブドメイン提案
   - `{project}.example.com` パターン

2. **プロジェクト別設定**
   - DocumentRoot: `/var/www/html/{subdomain}`
   - 開発用Alias: `/develop/{project}` → `/home/{username}/develop/{project}`

3. **既存プロジェクト移行支援**
   - 現在の `/develop/*` アクセスから独立ドメインへ

## 🛠️ 実装案

### テンプレート修正
```apache
<VirtualHost *:443>
    ServerName {project}.example.com
    DocumentRoot /var/www/html/{project}.example.com
    
    # 開発用アクセス（移行期間用）
    Alias /develop /home/{username}/develop/{project}
    <Directory "/home/{username}/develop/{project}">
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    
    # 本番用設定
    <Directory "/var/www/html/{project}.example.com">
        Options -Indexes
        AllowOverride All
        Require all granted
        FallbackResource /index.html
    </Directory>
    
    # SSL設定...
</VirtualHost>
```

### コマンド例
```bash
# 新規プロジェクト用vhost作成
./create-vhost.sh ideanotes

# 生成される設定
# - ideanotes.example.com
# - DocumentRoot: /var/www/html/ideanotes.example.com
# - 開発Alias: /develop → /home/{username}/develop/ideanotes
```

## 🚀 移行戦略

### Phase 1: 新規プロジェクト
- 新しいプロジェクトから専用サブドメイン運用開始
- 既存の `/develop` アクセスは保持

### Phase 2: 既存プロジェクト移行  
- 段階的にサブドメインへ移行
- リダイレクト設定で互換性維持

### Phase 3: 最適化
- `/develop` エイリアスの整理
- 不要な設定の削除