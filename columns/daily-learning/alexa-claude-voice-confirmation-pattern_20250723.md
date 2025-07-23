# 確認プロンプトを音声対話に変換する賢いパターン

*2025-07-23*

## 📝 課題と解決策

「確認プロンプトはスキップするが、通常プロンプトで確認を取ってもらい、はい・いいえを選ぶ」- これは音声インターフェースならではの優れた設計。

## 🎯 実装パターン

### 基本的な流れ
```
1. claude --dangerously-skip-permissions --continue "削除して"
   ↓
2. Claude: 「10個のファイルを削除します。よろしいですか？」
   ↓
3. Alexa: ユーザーに音声で確認
   ↓
4. ユーザー: 「はい」
   ↓
5. claude --continue "はい、削除を実行してください"
```

### 2段階実行パターン
```python
# 第1段階：計画の提示
response1 = claude_execute(
    "--dangerously-skip-permissions --continue",
    user_command
)

if "確認" in response1 or "よろしいですか" in response1:
    # Alexaで音声確認
    user_confirmation = alexa_ask_confirmation(response1)
    
    # 第2段階：実行
    if user_confirmation == "はい":
        response2 = claude_execute(
            "--dangerously-skip-permissions --continue",
            "はい、実行してください"
        )
```

## 💡 音声確認のUXパターン

### 重要度別の確認方法

**低リスク操作**
```
Claude: 「キャッシュをクリアします」
→ 確認なしで実行
```

**中リスク操作**
```
Claude: 「3つのファイルを変更します。続けますか？」
Alexa: 「はい」か「いいえ」でお答えください
```

**高リスク操作**
```
Claude: 「本番環境にデプロイします。確認コード1234を言ってください」
Alexa: 確認コードをお願いします
ユーザー: 「1234」
```

### 確認内容の最適化
```
❌ 悪い例：
「/var/www/html/index.phpを削除し、
 /var/www/html/config.phpを変更し、
 /var/www/html/database.phpを...」

✅ 良い例：
「3つのPHPファイルを更新します。
 主な変更はデータベース接続設定です。
 続けますか？」
```

## 🚀 高度な実装

### コンテキスト保持
```
Claude: 「5つのテストが失敗しています。修正しますか？」
ユーザー: 「どんなエラー？」
Claude: 「主にTypeErrorです。型定義の不整合が原因です」
ユーザー: 「じゃあ修正して」
```

### 部分承認パターン
```
Claude: 「10個の提案があります」
ユーザー: 「最初の3つだけ実行」
Claude: 「了解。最初の3つを実行します」
```

### ロールバック確認
```
実行後...
Claude: 「変更が完了しました。問題があれば戻せます」
ユーザー: 「やっぱり戻して」
Claude: 「変更前の状態に戻します」
```

## 🤔 実装上の工夫

### 状態管理
```python
class ConfirmationState:
    NONE = "none"
    WAITING = "waiting_confirmation"
    CONFIRMED = "confirmed"
    REJECTED = "rejected"
```

### タイムアウト処理
- 30秒以内に返答がない場合は自動キャンセル
- 重要な操作は明示的な確認が必須
- 安全側に倒す設計

### ログ記録
```
2024-01-23 15:30:01 - 削除操作を提案
2024-01-23 15:30:05 - ユーザー確認: はい
2024-01-23 15:30:06 - 削除実行開始
```

## 🎯 メリット

1. **安全性**: システムプロンプトより柔軟な確認
2. **自然さ**: 音声での自然な対話フロー
3. **透明性**: 何が起きるか明確に説明
4. **制御性**: ユーザーが最終決定権を保持

「--dangerously-skip-permissions」を安全に使うための、音声インターフェースならではの解決策。