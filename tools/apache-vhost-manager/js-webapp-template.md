# JS製Webアプリ向けvhost設定テンプレート

*作成日: 2025-07-12*

## 🎯 想定パターン

### 1. SPA (Single Page Application)
- React, Vue, Angular等のフロントエンドフレームワーク
- ビルド済みの静的ファイル配信
- History API対応（URLルーティング）

### 2. 静的サイト
- Vanilla JS, jQuery等
- 従来型のマルチページ

## 📋 設定テンプレート案

### HTTP設定 (port 80)
```apache
<VirtualHost *:80>
    ServerName {domain_name}
    DocumentRoot /var/www/html/{domain_name}
    
    # SSL証明書取得用
    Alias /.well-known/acme-challenge /var/www/html/.well-known/acme-challenge
    <Directory "/var/www/html/.well-known/acme-challenge">
        Options None
        AllowOverride None
        Require all granted
    </Directory>
    
    # HTTPS redirect
    RewriteEngine on
    RewriteCond %{SERVER_NAME} ={domain_name}
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>
```

### HTTPS設定 (port 443)
```apache
<VirtualHost *:443>
    ServerName {domain_name}
    DocumentRoot /var/www/html/{domain_name}
    
    # SSL設定
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/{domain_name}/cert.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/{domain_name}/privkey.pem
    SSLCertificateChainFile /etc/letsencrypt/live/{domain_name}/chain.pem
    
    # セキュリティヘッダー
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    Header always set X-Frame-Options DENY
    Header always set X-Content-Type-Options nosniff
    
    # SPA対応 (History API)
    <Directory "/var/www/html/{domain_name}">
        Options -Indexes
        AllowOverride All
        Require all granted
        
        # SPA用fallback
        RewriteEngine On
        RewriteBase /
        RewriteRule ^index\.html$ - [L]
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteCond %{REQUEST_FILENAME} !-d
        RewriteRule . /index.html [L]
    </Directory>
    
    # 静的ファイルのキャッシュ設定
    <FilesMatch "\.(css|js|png|jpg|jpeg|gif|ico|svg)$">
        ExpiresActive On
        ExpiresDefault "access plus 1 month"
    </FilesMatch>
</VirtualHost>
```

## 🛠️ 必要な作業

### 1. ディレクトリ作成
```bash
sudo mkdir -p /var/www/html/{domain_name}
sudo chown apache:apache /var/www/html/{domain_name}
```

### 2. SSL証明書取得
```bash
sudo certbot certonly --webroot -w /var/www/html -d {domain_name}
```

### 3. Apache設定適用
```bash
sudo systemctl reload httpd
```

## 🎛️ 設定のカスタマイズポイント

- **キャッシュ期間**: 開発中は短く、本番は長く
- **セキュリティヘッダー**: 要件に応じて調整
- **ディレクトリ権限**: 開発環境では緩く設定も
- **ログ設定**: 必要に応じて個別ログファイル