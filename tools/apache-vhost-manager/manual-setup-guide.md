# vhost作成 手順書

*作成日: 2025-07-12*

## 🎯 前提条件

- DNS設定済み（サブドメインがEIPを向いている）
- Apache httpd稼働中
- sudo権限あり
- certbot導入済み

## 📋 手順一覧

### Step 1: ディレクトリ準備
```bash
# DocumentRoot作成
sudo mkdir -p /var/www/html/{DOMAIN_NAME}

# 権限設定
sudo chown apache:apache /var/www/html/{DOMAIN_NAME}
sudo chmod 755 /var/www/html/{DOMAIN_NAME}

# テスト用index.html作成
sudo tee /var/www/html/{DOMAIN_NAME}/index.html > /dev/null << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>{DOMAIN_NAME}</title>
</head>
<body>
    <h1>{DOMAIN_NAME} - Test Page</h1>
    <p>vhost設定が正常に動作しています</p>
</body>
</html>
EOF
```

### Step 2: HTTP設定（Port 80）
```bash
# 設定ファイル作成
sudo tee /etc/httpd/conf.d/{DOMAIN_NAME}.conf > /dev/null << 'EOF'
<VirtualHost *:80>
    ServerName {DOMAIN_NAME}
    DocumentRoot /var/www/html/{DOMAIN_NAME}
    
    # SSL証明書取得用
    Alias /.well-known/acme-challenge /var/www/html/.well-known/acme-challenge
    <Directory "/var/www/html/.well-known/acme-challenge">
        Options None
        AllowOverride None
        Require all granted
    </Directory>
    
    # HTTPS redirect
    RewriteEngine on
    RewriteCond %{SERVER_NAME} ={DOMAIN_NAME}
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>
EOF
```

### Step 3: Apache設定テスト・リロード
```bash
# 設定ファイルの構文チェック
sudo apache2ctl configtest

# エラーがなければリロード
sudo systemctl reload httpd

# HTTP接続テスト
curl -I http://{DOMAIN_NAME}/
# 期待: 301 Redirect to HTTPS
```

### Step 4: SSL証明書取得
```bash
# Let's Encrypt証明書取得
sudo certbot certonly --webroot -w /var/www/html -d {DOMAIN_NAME}

# 証明書確認
sudo ls -la /etc/letsencrypt/live/{DOMAIN_NAME}/
```

### Step 5: HTTPS設定（Port 443）
```bash
# HTTPS設定ファイル作成
sudo tee /etc/httpd/conf.d/{DOMAIN_NAME}-le-ssl.conf > /dev/null << 'EOF'
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName {DOMAIN_NAME}
    DocumentRoot /var/www/html/{DOMAIN_NAME}
    
    # SSL設定
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/{DOMAIN_NAME}/cert.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/{DOMAIN_NAME}/privkey.pem
    SSLCertificateChainFile /etc/letsencrypt/live/{DOMAIN_NAME}/chain.pem
    
    # セキュリティヘッダー
    Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    Header always set X-Frame-Options DENY
    Header always set X-Content-Type-Options nosniff
    
    # SPA対応（JS webアプリ用）
    <Directory "/var/www/html/{DOMAIN_NAME}">
        Options -Indexes
        AllowOverride All
        Require all granted
        
        # History API fallback
        RewriteEngine On
        RewriteBase /
        RewriteRule ^index\.html$ - [L]
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteCond %{REQUEST_FILENAME} !-d
        RewriteRule . /index.html [L]
    </Directory>
    
    # 静的ファイルキャッシュ
    <FilesMatch "\.(css|js|png|jpg|jpeg|gif|ico|svg)$">
        ExpiresActive On
        ExpiresDefault "access plus 1 month"
    </FilesMatch>
    
    # SSL証明書更新用
    Alias /.well-known/acme-challenge /var/www/html/.well-known/acme-challenge
    <Directory "/var/www/html/.well-known/acme-challenge">
        Options None
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
</IfModule>
EOF
```

### Step 6: 最終確認
```bash
# 設定テスト
sudo apache2ctl configtest

# Apache再読み込み
sudo systemctl reload httpd

# HTTPS接続テスト
curl -I https://{DOMAIN_NAME}/
# 期待: 200 OK

# SSL証明書テスト
openssl s_client -connect {DOMAIN_NAME}:443 -servername {DOMAIN_NAME} < /dev/null
```

## 🛠️ 実行例

### 実際のコマンド（ドメイン: myapp.example.com）
```bash
# 1. ディレクトリ準備
sudo mkdir -p /var/www/html/myapp.example.com
sudo chown apache:apache /var/www/html/myapp.example.com

# 2. HTTP設定
sudo tee /etc/httpd/conf.d/myapp.example.com.conf > /dev/null << 'EOF'
<VirtualHost *:80>
    ServerName myapp.example.com
    DocumentRoot /var/www/html/myapp.example.com
    
    Alias /.well-known/acme-challenge /var/www/html/.well-known/acme-challenge
    <Directory "/var/www/html/.well-known/acme-challenge">
        Options None
        AllowOverride None
        Require all granted
    </Directory>
    
    RewriteEngine on
    RewriteCond %{SERVER_NAME} =myapp.example.com
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>
EOF

# 3. Apache reload
sudo systemctl reload httpd

# 4. SSL証明書取得
sudo certbot certonly --webroot -w /var/www/html -d myapp.example.com

# 5. HTTPS設定（上記テンプレートで作成）

# 6. 最終reload
sudo systemctl reload httpd
```

### myapp配置

#### Option A: 開発/本番分離 + シンボリックリンク（推奨）
```bash
# 本番用ディレクトリ作成
sudo mkdir -p /home/ec2-user/production

# 開発版から本番用にクローン（またはコピー）
cd /home/ec2-user/production
git clone /home/ec2-user/develop/myapp myapp
# または: sudo cp -r /home/ec2-user/develop/myapp /home/ec2-user/production/

# DocumentRootをシンボリックリンクに
sudo rm -rf /var/www/html/myapp.example.com
sudo ln -s /home/ec2-user/production/myapp /var/www/html/myapp.example.com

# 権限調整
sudo chown -R ec2-user:apache /home/ec2-user/production/myapp
sudo chmod -R 755 /home/ec2-user/production/myapp
```

#### 運用フロー
```
開発: dev.example.com/develop/myapp/     # 開発環境
       ↓ (安定化確認後)
本番: myapp.example.com/                  # 本番環境
```

#### Option B: コピー方式
```bash
# 開発版から本番へコピー
sudo cp -r /home/ec2-user/develop/myapp/* /var/www/html/myapp.example.com/

# 権限調整
sudo chown -R apache:apache /var/www/html/myapp.example.com/
```

## ⚠️ トラブルシューティング

### よくある問題
1. **DNS浸透待ち**: 設定後すぐは反映されない場合あり
2. **証明書取得失敗**: DNS設定を確認
3. **権限エラー**: chown/chmod を確認
4. **Apache起動失敗**: configtest で構文確認

### ログ確認
```bash
# Apache エラーログ
sudo tail -f /var/log/httpd/error_log

# SSL証明書ログ
sudo journalctl -u certbot
```

## 📝 次回からの簡略化

手順確認後、この手順をbashスクリプト化してワンコマンド実行可能にする予定。