# vibeSettings セキュリティガイドライン強化提案

*2025-07-12*

## 🎯 提案の背景

ideanotesプロジェクトでの**固有情報コミット事件**（実際のドメイン名、IPアドレス、ユーザー名等がGitHub公開リポジトリにコミットされた事件）を受けて、vibeSettingsのセキュリティガイドラインに重大な不備が発覚。

### 現在のvibeSettings セキュリティルール
```markdown
### 🔒 セキュリティルール
- 機密情報や個人情報を含む設定は作成しない
- 外部への不正アクセスを誘発する設定は禁止
```

**問題**: これだけでは**GitHub公開リポジトリでの固有情報露出**を防げない。

## 🚨 提案: 専用セキュリティ設定ファイル作成

### ファイル提案: `configs/github_security.md`

```markdown
# GitHub Security Guidelines

## STATUS: ON
<!-- 公開リポジトリ作業時は常に適用 -->

## 🚫 GitHub公開リポジトリに絶対コミットしてはいけない情報

### 機密情報カテゴリ
- **認証情報**: API keys, tokens, passwords, private keys
- **接続情報**: database URLs, connection strings, internal endpoints
- **固有情報**: real domain names, IP addresses, server names
- **個人情報**: usernames, email addresses, phone numbers
- **内部情報**: company names, project codenames, internal URLs

### 具体例
❌ **NG例**:
- `api_key: sk-1234567890abcdef`
- `database_url: postgresql://user:pass@127.0.0.1/db`
- `domain: my-company.com`
- `server_ip: 192.168.1.100`
- `username: john.doe`

✅ **OK例**:
- `api_key: your_api_key_here`
- `database_url: postgresql://user:pass@localhost/dbname`
- `domain: example.com`
- `server_ip: {IP_ADDRESS}`
- `username: {username}`

## 🛡️ 予防策

### コミット前チェックリスト
- [ ] 実際のドメイン名が含まれていないか
- [ ] IPアドレスが露出していないか
- [ ] ユーザー名やパス情報が含まれていないか
- [ ] API keyやtokenが含まれていないか
- [ ] 内部URL、サーバー名が含まれていないか

### .gitignore必須パターン
```gitignore
# 環境設定
.env
.env.local
.env.*.local

# 設定ファイル
config.local.*
settings.local.*
secrets.*

# ログファイル
*.log
logs/

# IDE設定
.vscode/settings.json
.idea/
```

### 設計パターン
1. **テンプレート分離**: `config.template.js` + `config.local.js`
2. **環境変数活用**: `process.env.API_KEY`
3. **プレースホルダー使用**: `{API_KEY}`, `example.com`

## 🔧 事故発生時の対処法

### Git履歴修正手順
1. **即座に固有情報を置換**
   ```bash
   find . -name "*.md" -type f -exec sed -i '' 's/actual-domain\.com/example.com/g' {} \;
   ```

2. **Git履歴整理**
   ```bash
   git reset --soft {clean_commit_hash}
   git commit -m "sensitive info sanitized"
   git push --force-with-lease
   ```

3. **affected filesの確認**
   ```bash
   git log --oneline --name-only
   ```

## ⚡ 緊急時対応

### 即座に実行すべきこと
1. **該当リポジトリをprivateに変更**（可能な場合）
2. **exposed credentialsの無効化**（API keys等）
3. **Git履歴の修正**
4. **team memberへの通知**

### 長期対策
- pre-commit hookの導入
- 定期的なリポジトリ監査
- team educationの実施

## 📋 チーム協力時の注意

### プルリクエスト受け取り時
- [ ] 固有情報チェックの実施
- [ ] .envファイル等の確認
- [ ] ハードコーディングされた値の確認

### 協力者への事前説明
「GitHub公開リポジトリなので、固有情報（ドメイン名、IP、ユーザー名等）は
example.com、{IP}、{username}等のプレースホルダーを使ってください」
```

### mode_profiles.mdへの追加提案

`current_settings`プロファイルに追加:
```markdown
### current_settings
- configs/english_learning.md
- configs/development.md
- configs/interactive_mode.md
- configs/system_analysis.md
- configs/talk_settings.md
- configs/strict_rules.md
+ configs/github_security.md  # 追加
```

## 🎯 期待効果

1. **予防効果**: コミット前の自動チェック習慣化
2. **教育効果**: 協力者への明確なガイドライン提示
3. **対処効果**: 事故発生時の迅速な対応手順明確化
4. **品質効果**: より安全なコード共有の実現

## 📝 実装タイミング

**提案**: この提案書をvibeSettingsディレクトリで別Claude セッションに提示し、
適切な実装方法を検討・実行してもらう。

---

*ideanotesプロジェクトでの実体験に基づく、実践的セキュリティガイドライン強化提案*