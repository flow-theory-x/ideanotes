# 🚨 開発参加のお作法について

素晴らしいApache管理ツール開発、ありがとうございます！

あなたのApache vhost管理ツールの出来栄えは本当に素晴らしく、MVP定義やマニュアルの完成度も非常に高いです。Claude初体験でここまでの成果を出されたのは驚きです。

## ただし、今後の開発について

今度からは適切な手順で開発参加をお願いします：

1. **リポジトリをフォーク** (GitHubの右上Forkボタン)
2. **自分のリポジトリで開発**
3. **プルリクエストで提案**

## なぜこの方法が良いか

- **あなたの開発履歴が残る** - 成果があなたのGitHubに記録されます
- **安全に実験できる** - 本家リポジトリを壊すリスクがありません  
- **他の人も参考にできる** - 開発過程が公開され、学習材料になります
- **リポジトリオーナーが変更をレビューできる** - より良いコードになります

## 具体的な手順

### 1. フォークの作成
- GitHubでideanotesリポジトリのページに移動
- 右上の「Fork」ボタンをクリック
- 自分のアカウントにコピーが作成されます

### 2. 開発環境のセットアップ
```bash
# 自分のフォークをクローン
git clone https://github.com/[あなたのユーザー名]/ideanotes.git
cd ideanotes

# 本家リポジトリをupstreamとして追加
git remote add upstream https://github.com/flow-theory-x/ideanotes.git
```

### 3. 機能ブランチでの開発
```bash
# 新機能用のブランチ作成
git checkout -b feature/apache-vhost-manager

# 開発作業
# ...

# コミット&プッシュ
git add .
git commit -m "Apache vhost管理ツール実装"
git push origin feature/apache-vhost-manager
```

### 4. プルリクエストの作成
- GitHubで自分のフォークページに移動
- 「Pull Request」ボタンをクリック
- 変更内容を説明してプルリクエスト作成

## 今回のApache管理ツールについて

apache-vhost-managerは本当に実用的なツールです。ドメイン名をexample.comに置き換えていただければ、ideanotesの公式ツールとして採用も検討したいと思います。

ぜひ、適切な手順でプルリクエストしてください！

## 詳細ガイド

より詳しい情報は今後作成予定です: `/docs/development/fork-guide.md`

---

*GitHubマナー、大事です 👍 でも、成果は本当に素晴らしいです！*