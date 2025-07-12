# vhost設定 - 影響範囲と巻き戻し方法

*作成日: 2025-07-12*

## 🎯 実行予定の作業

### 新規作成するもの
1. **新規ディレクトリ**
   - `/var/www/html/myapp.example.com/` (シンボリックリンク)
   - `/home/ec2-user/production/myapp/` (Git clone)

2. **新規設定ファイル**
   - `/etc/httpd/conf.d/myapp.example.com.conf` (HTTP設定)
   - `/etc/httpd/conf.d/myapp.example.com-le-ssl.conf` (HTTPS設定)

3. **SSL証明書**
   - `/etc/letsencrypt/live/myapp.example.com/`

## 📊 影響範囲分析

### ✅ 影響なし（既存サービス）
- **dev.example.com**: 既存設定に一切変更なし
- **開発環境**: `/home/ec2-user/develop/` は読み取りのみ
- **Apache基本設定**: メイン設定ファイルは未変更

### ⚠️ 軽微な影響
- **Apache reload**: 数秒間の接続一時中断の可能性
- **ディスク使用量**: SSL証明書とシンボリックリンクで数KB増加

### 🚨 潜在的リスク
- **Apache設定エラー**: 構文ミスでApache起動失敗の可能性
- **権限エラー**: ファイルアクセス権限の問題
- **SSL証明書取得失敗**: Let's Encrypt API制限に抵触の可能性

## 🛡️ 事前確認

### 現在の動作状況
```bash
# Apache状態確認
sudo systemctl status httpd

# 既存サイトの動作確認
curl -I https://dev.example.com/

# ディスク容量確認
df -h
```

### 設定バックアップ
```bash
# 現在のApache設定をバックアップ
sudo mkdir -p /backup/apache/$(date +%Y%m%d_%H%M%S)
sudo cp -r /etc/httpd/conf.d/ /backup/apache/$(date +%Y%m%d_%H%M%S)/
```

## 🔄 巻き戻し方法

### レベル1: 設定ファイルのみ削除
```bash
# 新規作成した設定ファイルを削除
sudo rm -f /etc/httpd/conf.d/myapp.example.com.conf
sudo rm -f /etc/httpd/conf.d/myapp.example.com-le-ssl.conf

# Apache reload
sudo systemctl reload httpd

# 動作確認
curl -I https://dev.example.com/
```

### レベル2: 完全巻き戻し
```bash
# 1. 設定ファイル削除
sudo rm -f /etc/httpd/conf.d/myapp.example.com*.conf

# 2. DocumentRoot削除
sudo rm -f /var/www/html/myapp.example.com

# 3. production directory削除（オプション）
sudo rm -rf /home/ec2-user/production/myapp

# 4. SSL証明書削除（オプション）
sudo certbot delete --cert-name myapp.example.com

# 5. Apache reload
sudo systemctl reload httpd
```

### レベル3: Apache再起動（緊急時）
```bash
# Apache完全再起動
sudo systemctl restart httpd

# 状態確認
sudo systemctl status httpd

# 既存サイト動作確認
curl -I https://dev.example.com/
```

### レベル4: バックアップからの復元
```bash
# バックアップから設定復元
sudo cp -r /backup/apache/[TIMESTAMP]/conf.d/* /etc/httpd/conf.d/

# Apache再起動
sudo systemctl restart httpd
```

## 🧪 段階的実行戦略

### Phase 1: 準備・バックアップ
1. 現状バックアップ
2. ディスク容量確認
3. Apache設定テスト

### Phase 2: HTTP設定のみ
1. ディレクトリ作成
2. HTTP設定ファイル作成
3. 構文チェック + reload
4. 動作確認（既存サイトも含む）

### Phase 3: SSL証明書取得
1. Let's Encrypt証明書取得
2. 証明書確認

### Phase 4: HTTPS設定
1. HTTPS設定ファイル作成
2. 最終動作確認

## ⏱️ 各段階での確認ポイント

### 毎回実行する確認
```bash
# 設定構文チェック
sudo httpd -t

# 既存サイト動作確認
curl -I https://dev.example.com/

# Apache状態確認
sudo systemctl status httpd
```

### 異常検知時の対応
1. **即座に作業停止**
2. **エラーログ確認**: `sudo tail -f /var/log/httpd/error_log`
3. **巻き戻し実行**: 上記手順に従って復旧
4. **動作確認**: 既存サービスの正常性を確認

## 📋 チェックリスト

### 実行前
- [ ] バックアップ作成済み
- [ ] ディスク容量確認済み
- [ ] 巻き戻し手順の理解済み

### 各段階後
- [ ] Apache構文チェックOK
- [ ] 既存サイト正常動作
- [ ] エラーログに異常なし

### 完了後
- [ ] 新サイト正常動作
- [ ] SSL証明書有効
- [ ] 全体的な動作確認完了

## 🚨 緊急連絡先・情報

- **Apache設定ディレクトリ**: `/etc/httpd/conf.d/`
- **ログファイル**: `/var/log/httpd/error_log`
- **バックアップ場所**: `/backup/apache/[TIMESTAMP]/`
- **作業時間見積もり**: 15-20分（巻き戻し含む）