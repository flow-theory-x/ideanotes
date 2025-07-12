# Alexa Voice Memo - 実装ガイド

## 🎯 実装概要

セットアップ完了後のLambda関数実装手順書です。S3永続化による音声メモ機能を実装します。

## 📋 実装前チェック

### 前提条件確認
- [ ] セットアップガイド完了
- [ ] Alexa-hostedスキル作成済み
- [ ] 対話モデル構築済み
- [ ] テスト環境動作確認済み

### 実装ファイル構成
```
lambda/
├── index.js              # メインハンドラー
├── package.json          # 依存関係定義
└── util.js              # ヘルパー関数（オプション）
```

## 💻 Step 1: package.json設定

### 1.1 依存関係更新

```json
{
  "name": "alexa-voice-memo",
  "version": "1.0.0",
  "description": "Voice memo skill for Alexa with S3 persistence",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "dependencies": {
    "ask-sdk-core": "^2.13.0",
    "ask-sdk-s3-persistence-adapter": "^2.13.0"
  }
}
```

### 1.2 インストール実行
- Alexa開発者コンソールで自動的にインストールされます
- パッケージ更新後は「Deploy」をクリック

## 🔧 Step 2: メインハンドラー実装

### 2.1 index.js基本構造

```javascript
const Alexa = require('ask-sdk-core');
const { S3PersistenceAdapter } = require('ask-sdk-s3-persistence-adapter');

// S3永続化アダプターの設定
const persistenceAdapter = new S3PersistenceAdapter({
    bucketName: process.env.S3_PERSISTENCE_BUCKET
});

// ========== ハンドラー定義 ==========

// スキル起動ハンドラー
const LaunchRequestHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'LaunchRequest';
    },
    handle(handlerInput) {
        const speakOutput = 'ボイスメモへようこそ。メモの追加、読み上げ、削除ができます。何をしますか？';
        const repromptText = 'メモに何かを追加、メモを読み上げ、または削除したい番号を教えてください。';
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt(repromptText)
            .getResponse();
    }
};

// メモ追加ハンドラー
const AddMemoIntentHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'AddMemoIntent';
    },
    async handle(handlerInput) {
        const attributesManager = handlerInput.attributesManager;
        const sessionAttributes = await attributesManager.getPersistentAttributes() || {};
        
        // スロットからメモテキストを取得
        const memoText = Alexa.getSlotValue(handlerInput.requestEnvelope, 'memoText');
        
        if (!memoText) {
            const speakOutput = '申し訳ありません。メモの内容を聞き取れませんでした。もう一度お試しください。';
            return handlerInput.responseBuilder
                .speak(speakOutput)
                .reprompt('メモに追加したい内容を教えてください。')
                .getResponse();
        }
        
        // メモリスト初期化
        if (!sessionAttributes.memos) {
            sessionAttributes.memos = [];
            sessionAttributes.nextId = 1;
        }
        
        // 新しいメモを追加
        const newMemo = {
            id: sessionAttributes.nextId,
            text: memoText,
            timestamp: new Date().toISOString(),
            deleted: false
        };
        
        sessionAttributes.memos.push(newMemo);
        sessionAttributes.nextId += 1;
        
        // S3に保存
        attributesManager.setPersistentAttributes(sessionAttributes);
        await attributesManager.savePersistentAttributes();
        
        const speakOutput = `${memoText}をメモに追加しました。`;
        const repromptText = '他にも何かメモしますか？';
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt(repromptText)
            .getResponse();
    }
};

// メモ読み上げハンドラー
const ReadMemosIntentHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'ReadMemosIntent';
    },
    async handle(handlerInput) {
        const attributesManager = handlerInput.attributesManager;
        const sessionAttributes = await attributesManager.getPersistentAttributes() || {};
        
        // アクティブなメモを取得（削除されていないもの）
        const activeMemos = (sessionAttributes.memos || []).filter(memo => !memo.deleted);
        
        let speakOutput;
        let repromptText = '他に何かお手伝いできることはありますか？';
        
        if (activeMemos.length === 0) {
            speakOutput = 'メモはありません。新しいメモを追加しますか？';
            repromptText = 'メモに追加したい内容を教えてください。';
        } else {
            const memoCount = activeMemos.length;
            const memoList = activeMemos
                .map((memo, index) => `${index + 1}番目、${memo.text}`)
                .join('。');
            
            speakOutput = `メモが${memoCount}件あります。${memoList}。`;
        }
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt(repromptText)
            .getResponse();
    }
};

// メモ削除ハンドラー
const DeleteMemoIntentHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'DeleteMemoIntent';
    },
    async handle(handlerInput) {
        const attributesManager = handlerInput.attributesManager;
        const sessionAttributes = await attributesManager.getPersistentAttributes() || {};
        
        // スロットから削除対象番号を取得
        const memoNumber = parseInt(Alexa.getSlotValue(handlerInput.requestEnvelope, 'memoNumber'));
        
        // アクティブなメモを取得
        const activeMemos = (sessionAttributes.memos || []).filter(memo => !memo.deleted);
        
        if (!memoNumber || memoNumber < 1 || memoNumber > activeMemos.length) {
            const speakOutput = `申し訳ありません。${memoNumber || '不明'}番目のメモは存在しません。1番目から${activeMemos.length}番目までの番号を指定してください。`;
            return handlerInput.responseBuilder
                .speak(speakOutput)
                .reprompt('削除したいメモの番号を教えてください。')
                .getResponse();
        }
        
        // 削除対象メモを特定（論理削除）
        const targetMemo = activeMemos[memoNumber - 1];
        const memoIndex = sessionAttributes.memos.findIndex(memo => memo.id === targetMemo.id);
        sessionAttributes.memos[memoIndex].deleted = true;
        
        // S3に保存
        attributesManager.setPersistentAttributes(sessionAttributes);
        await attributesManager.savePersistentAttributes();
        
        const speakOutput = `${memoNumber}番目のメモ「${targetMemo.text}」を削除しました。`;
        const repromptText = '他にも何かお手伝いできることはありますか？';
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt(repromptText)
            .getResponse();
    }
};

// ヘルプハンドラー
const HelpIntentHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && Alexa.getIntentName(handlerInput.requestEnvelope) === 'AMAZON.HelpIntent';
    },
    handle(handlerInput) {
        const speakOutput = 'ボイスメモでは、音声でメモの管理ができます。「メモに牛乳を買うを追加」でメモを追加、「メモを読んで」で一覧表示、「1番目のメモを削除」で削除できます。何をしますか？';
        const repromptText = 'メモの追加、読み上げ、削除のいずれかを選んでください。';
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt(repromptText)
            .getResponse();
    }
};

// 終了ハンドラー
const CancelAndStopIntentHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'IntentRequest'
            && (Alexa.getIntentName(handlerInput.requestEnvelope) === 'AMAZON.CancelIntent'
                || Alexa.getIntentName(handlerInput.requestEnvelope) === 'AMAZON.StopIntent');
    },
    handle(handlerInput) {
        const speakOutput = 'さようなら！メモが必要な時はまた呼んでください。';
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .getResponse();
    }
};

// セッション終了ハンドラー
const SessionEndedRequestHandler = {
    canHandle(handlerInput) {
        return Alexa.getRequestType(handlerInput.requestEnvelope) === 'SessionEndedRequest';
    },
    handle(handlerInput) {
        console.log(`~~~~ Session ended: ${JSON.stringify(handlerInput.requestEnvelope)}`);
        return handlerInput.responseBuilder.getResponse();
    }
};

// エラーハンドラー
const ErrorHandler = {
    canHandle() {
        return true;
    },
    handle(handlerInput, error) {
        const speakOutput = '申し訳ありません。問題が発生しました。もう一度お試しください。';
        console.log(`~~~~ Error handled: ${JSON.stringify(error)}`);
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt(speakOutput)
            .getResponse();
    }
};

// ========== スキルビルダー設定 ==========

exports.handler = Alexa.SkillBuilders.custom()
    .addRequestHandlers(
        LaunchRequestHandler,
        AddMemoIntentHandler,
        ReadMemosIntentHandler,
        DeleteMemoIntentHandler,
        HelpIntentHandler,
        CancelAndStopIntentHandler,
        SessionEndedRequestHandler
    )
    .addErrorHandlers(ErrorHandler)
    .withPersistenceAdapter(persistenceAdapter)
    .lambda();
```

## 🧪 Step 3: テスト実装

### 3.1 基本機能テスト

#### テストケース1: メモ追加
```
入力: 「ボイスメモを開いて」
期待: スキル起動メッセージ

入力: 「メモに牛乳を買うを追加」
期待: 「牛乳を買うをメモに追加しました」

入力: 「メモにパンを買うを追加」  
期待: 「パンを買うをメモに追加しました」
```

#### テストケース2: メモ読み上げ
```
入力: 「メモを読んで」
期待: 「メモが2件あります。1番目、牛乳を買う。2番目、パンを買う。」
```

#### テストケース3: メモ削除
```
入力: 「1番目のメモを削除」
期待: 「1番目のメモ『牛乳を買う』を削除しました」

入力: 「メモを読んで」
期待: 「メモが1件あります。1番目、パンを買う。」
```

### 3.2 エラーケーステスト

#### 無効な削除番号
```
入力: 「5番目のメモを削除」
期待: 「申し訳ありません。5番目のメモは存在しません...」
```

#### 空のメモリスト
```
（全メモ削除後）
入力: 「メモを読んで」
期待: 「メモはありません。新しいメモを追加しますか？」
```

## 🚀 Step 4: デプロイ・テスト

### 4.1 デプロイ実行

1. **コード保存**
   - 「Save」ボタンをクリック
   - 自動保存が有効な場合は自動的に保存

2. **デプロイ実行**
   - 「Deploy」ボタンをクリック
   - デプロイ完了まで30秒程度待機

3. **デプロイ確認**
   - 成功メッセージの確認
   - エラーログの確認

### 4.2 テスト実行

#### Alexaシミュレーターでのテスト
1. **テストタブに移動**
2. **テスト有効化**
   - 「開発中」を選択
3. **音声/テキスト入力でテスト**

#### 実機テスト（Echoデバイス）
1. **デバイス連携確認**
   - 同じAmazonアカウントでログイン
2. **音声テスト**
   - 「アレクサ、ボイスメモを開いて」

## 🔍 Step 5: デバッグ・最適化

### 5.1 ログ確認

#### CloudWatch Logsでのデバッグ
- Alexa開発者コンソール > コード > ログ
- エラーメッセージの確認
- console.log出力の確認

#### 一般的なエラーパターン
```javascript
// デバッグ用ログ追加例
console.log('Received slot value:', memoText);
console.log('Current memos:', JSON.stringify(sessionAttributes.memos));
console.log('Active memos count:', activeMemos.length);
```

### 5.2 音声認識精度改善

#### サンプル発話の追加
```
// より自然な発話パターンを追加
「メモに〜をメモして」
「〜をメモに記録して」
「〜を覚えておいて」
```

#### スロットタイプの調整
```
// より広範囲のテキスト認識のため
AMAZON.SearchQuery （より柔軟な認識）
```

## ✅ 実装完了チェックリスト

### コード実装
- [ ] package.json設定完了
- [ ] index.js実装完了
- [ ] S3永続化設定完了
- [ ] 全ハンドラー実装完了

### 機能テスト
- [ ] スキル起動確認
- [ ] メモ追加機能動作確認
- [ ] メモ読み上げ機能動作確認
- [ ] メモ削除機能動作確認
- [ ] ヘルプ機能動作確認

### エラーハンドリング
- [ ] 無効な入力時の適切な応答
- [ ] 空リスト時の適切な応答
- [ ] エラー時の適切なログ出力

### パフォーマンス
- [ ] S3読み書き動作確認
- [ ] レスポンス速度確認（3秒以内）
- [ ] メモリ使用量確認

### 次のステップ準備
- [ ] テストチェックリスト確認
- [ ] 公開準備（オプション）
- [ ] 運用・保守計画確認

## 🆘 トラブルシューティング

### よくある実装エラー

#### 1. S3権限エラー
**症状**: 「Access Denied」エラー
**原因**: Alexa-hostedのS3アクセス権限設定
**解決**: 環境変数の確認、デプロイ再実行

#### 2. スロット値が取得できない
**症状**: memoTextがundefined
**原因**: インテント設計とコードの不整合
**解決**: サンプル発話とスロット名の確認

#### 3. 永続化データが保存されない
**症状**: スキル再起動後にメモが消える
**原因**: savePersistentAttributes()の未実行
**解決**: awaitキーワードの確認

#### 4. 音声認識精度が低い
**症状**: 意図しないテキストが認識される
**原因**: サンプル発話の不足
**解決**: より多様な発話パターンの追加

### デバッグ方法
```javascript
// エラー詳細ログの追加
console.log('Error details:', JSON.stringify(error, null, 2));
console.log('Request envelope:', JSON.stringify(handlerInput.requestEnvelope, null, 2));
```

---

**実装完了後**: [testing-checklist.md](testing-checklist.md) に進んでください。