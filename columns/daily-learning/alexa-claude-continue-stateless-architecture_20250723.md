# 「--continue」で実現するAlexa×Claude Codeのステートレス対話アーキテクチャ

*2025-07-23*

## 📝 画期的な発見

「Alexaがメッセージ受け取ったら、--continueで実行し、1プロンプト毎にレスポンス返して終了」- これは天才的なアーキテクチャ！

## 🎯 実現方法の詳細

### 基本フロー
```
1. Alexa：「テストを実行して」
   ↓
2. Lambda：claude --continue "テストを実行して"
   ↓
3. Claude Code：前回の続きから実行
   ↓
4. 1レスポンス生成したら終了
   ↓
5. Alexa：結果を音声で返答
```

### ステートレスの利点
- Lambdaのタイムアウトを気にしない
- メモリ使用量を最小限に
- スケーラブルな設計
- コスト効率が高い

## 💡 実装イメージ

### Lambda関数の疑似コード
```python
def lambda_handler(event, context):
    # Alexaからのメッセージ取得
    user_message = event['request']['intent']['slots']['message']['value']
    
    # Claude Codeを--continueで実行
    result = subprocess.run([
        'claude', 
        '--continue',
        user_message
    ], capture_output=True, text=True)
    
    # 結果を返す
    return {
        'response': {
            'outputSpeech': {
                'type': 'PlainText',
                'text': result.stdout
            }
        }
    }
```

### 会話の継続性
```
初回：「新しいReactコンポーネント作って」
→ Claude: 作成開始、セッション保存

2回目：「TypeScriptで」（--continue）
→ Claude: 前回の続きでTS実装

3回目：「テストも追加」（--continue）
→ Claude: 同じコンポーネントにテスト追加
```

## 🚀 高度な活用

### 複数プロジェクトの管理
```bash
# プロジェクトごとに作業ディレクトリ分離
cd /projects/project-a && claude --continue "..."
cd /projects/project-b && claude --continue "..."
```

### コンテキスト切り替え
```
「プロジェクトAに切り替えて」
→ Lambda: cd /projects/project-a

「さっきの続き」
→ claude --continue で自動的にproject-aの文脈
```

### 長時間タスクの分割実行
```
「大規模リファクタリング開始」
→ 1ファイル処理して終了

「続き」
→ 次のファイル処理

（必要なだけ繰り返し）
```

## 🤔 実装上の考慮点

### セッション管理
- ユーザーごとの作業ディレクトリ
- プロジェクトごとの分離
- 適切な権限管理

### エラーハンドリング
- タイムアウト対策
- 部分的な成功の扱い
- ロールバック機能

### 応答の最適化
- 長い出力の要約
- 音声に適した簡潔な表現
- 進捗状況の分かりやすい伝達

## 🎯 革新的なポイント

1. **状態管理不要**: Claude Code自体が状態を保持
2. **シンプル**: 1リクエスト1レスポンス
3. **スケーラブル**: Lambdaの自動スケーリング活用
4. **低コスト**: 実行時間最小化

「もしかして」から始まったこのアイデアが、Alexa×Claude Codeの実装における最適解かもしれない！