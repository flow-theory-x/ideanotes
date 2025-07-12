# 既存プロジェクトのideanotes連携

*作成日: 2025-07-12*

## 🎯 コンセプト

既に存在するプロジェクトをideanotesの管理下に取り込み、継続的な改善・拡張を行う仕組み。

## 🔗 連携方法

### 1. プロジェクト情報の取り込み
```bash
# 既存プロジェクトの基本情報を収集
./import-project.sh /home/ec2-user/develop/jsontools
```

**自動収集する情報：**
- README.md の内容
- package.json の依存関係
- ディレクトリ構造
- Git履歴の分析

### 2. ideanotesディレクトリ構造への配置
```
ideanotes/
├── services/
│   └── imported-jsontools/          # 既存プロジェクト用
│       ├── project-overview.md     # 自動生成
│       ├── improvement-ideas.md    # 改善アイデア
│       └── deployment-plan.md      # 公開計画
```

### 3. 改善・拡張の管理
- 既存機能の分析と文書化
- 新機能アイデアの追加
- 公開・運用計画の策定

## 🛠️ 実装案

### プロジェクト分析スクリプト
```bash
#!/bin/bash
# analyze-existing-project.sh

PROJECT_PATH=$1
PROJECT_NAME=$(basename $PROJECT_PATH)

# 基本情報収集
echo "## 📋 プロジェクト分析: $PROJECT_NAME"
echo "- パス: $PROJECT_PATH"
echo "- 最終更新: $(stat -c %y $PROJECT_PATH)"

# ファイル構造分析
echo "### ディレクトリ構造"
tree $PROJECT_PATH -L 2

# 技術スタック推定
if [ -f "$PROJECT_PATH/package.json" ]; then
    echo "### 技術スタック: Node.js/JavaScript"
    echo "#### 依存関係"
    cat $PROJECT_PATH/package.json | jq '.dependencies'
fi

# Git情報
if [ -d "$PROJECT_PATH/.git" ]; then
    echo "### Git履歴"
    cd $PROJECT_PATH && git log --oneline -5
fi
```

### ideanotes連携フロー
1. **分析**: 既存プロジェクトの自動分析
2. **文書化**: ideanotes形式での整理
3. **改善計画**: スモールスタート原則での拡張方針
4. **公開準備**: vhost管理ツールとの連携

## 💡 具体例: jsontools プロジェクト

### 想定される管理内容
- **現状分析**: 既存機能の棚卸し
- **改善アイデア**: UI/UX向上、新機能追加
- **公開計画**: `jsontools.bon-soleil.com` での運用
- **継続開発**: 段階的機能拡張

### ideanotesでの議論例
```
「jsontoolsのJSONバリデーター機能、もう少し視覚的にできそう」
→ services/imported-jsontools/improvement-ideas.md に記録
→ 議論を重ねて仕様化
→ 50%決まったら実装開始
```

## 🚀 メリット

1. **既存資産の活用**: 作りかけプロジェクトも無駄にならない
2. **段階的改善**: ideanotesの方法論で継続開発
3. **統一管理**: 全プロジェクトを一箇所で把握
4. **公開促進**: vhost管理と連携して簡単デプロイ

## 📝 次のアクション

1. プロジェクト分析スクリプトの作成
2. jsontools の実際の分析・取り込み
3. 改善アイデアのブレインストーミング