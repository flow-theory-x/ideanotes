# 実装ガイド

> Lambda関数の実装とデプロイの完全ガイド

## 🎯 実装概要

セットアップ完了後のLambda関数実装手順です。S3永続化による音声メモ機能を実装します。

## 📋 実装前チェック

- [ ] [セットアップガイド](setup-guide.md)完了
- [ ] Alexa-hostedスキル作成済み
- [ ] 対話モデル構築済み
- [ ] テスト環境動作確認済み

## 💻 Step 1: 依存関係設定

### package.json更新

```json
{
  "name": "alexa-voice-memo",
  "version": "1.0.0",
  "description": "Voice memo skill for Alexa",
  "main": "index.js",
  "dependencies": {
    "ask-sdk-core": "^2.13.0",
    "ask-sdk-s3-persistence-adapter": "^2.13.0"
  }
}
```

## 🔧 Step 2: メインハンドラー実装

### index.js完全実装

```javascript
const Alexa = require('ask-sdk-core');
const { S3PersistenceAdapter } = require('ask-sdk-s3-persistence-adapter');

// S3永続化アダプターの設定
const persistenceAdapter = new S3PersistenceAdapter({
    bucketName: process.env.S3_PERSISTENCE_BUCKET
});

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
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt('他にも何かメモしますか？')
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
        
        const memoNumber = parseInt(Alexa.getSlotValue(handlerInput.requestEnvelope, 'memoNumber'));
        const activeMemos = (sessionAttributes.memos || []).filter(memo => !memo.deleted);
        
        if (!memoNumber || memoNumber < 1 || memoNumber > activeMemos.length) {
            const speakOutput = `申し訳ありません。${memoNumber || '不明'}番目のメモは存在しません。1番目から${activeMemos.length}番目までの番号を指定してください。`;
            return handlerInput.responseBuilder
                .speak(speakOutput)
                .reprompt('削除したいメモの番号を教えてください。')
                .getResponse();
        }
        
        // 論理削除
        const targetMemo = activeMemos[memoNumber - 1];
        const memoIndex = sessionAttributes.memos.findIndex(memo => memo.id === targetMemo.id);
        sessionAttributes.memos[memoIndex].deleted = true;
        
        // S3に保存
        attributesManager.setPersistentAttributes(sessionAttributes);
        await attributesManager.savePersistentAttributes();
        
        const speakOutput = `${memoNumber}番目のメモ「${targetMemo.text}」を削除しました。`;
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt('他に何かお手伝いできることはありますか？')
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
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt('メモの追加、読み上げ、削除のいずれかを選んでください。')
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
        console.log(`Session ended: ${JSON.stringify(handlerInput.requestEnvelope)}`);
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
        console.log(`Error handled: ${JSON.stringify(error)}`);
        
        return handlerInput.responseBuilder
            .speak(speakOutput)
            .reprompt(speakOutput)
            .getResponse();
    }
};

// スキルビルダー設定
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

## 🚀 Step 3: デプロイ・テスト

### デプロイ実行

1. **コード保存**: 「Save」ボタンをクリック
2. **デプロイ実行**: 「Deploy」ボタンをクリック
3. **完了確認**: デプロイ成功メッセージを確認

### 基本テスト

#### テストケース1: メモ追加
```
入力: 「ボイスメモを開いて」
期待: スキル起動メッセージ

入力: 「メモに牛乳を買うを追加」
期待: 「牛乳を買うをメモに追加しました」
```

#### テストケース2: メモ読み上げ
```
入力: 「メモを読んで」
期待: 「メモが1件あります。1番目、牛乳を買う。」
```

#### テストケース3: メモ削除
```
入力: 「1番目のメモを削除」
期待: 「1番目のメモ『牛乳を買う』を削除しました」
```

## 🔍 Step 4: デバッグ・最適化

### ログ確認
- Alexa開発者コンソール > コード > ログ
- CloudWatchでエラーログを確認

### 音声認識改善
追加のサンプル発話を設定：
```
「〜をメモに記録して」
「〜を覚えておいて」  
「〜番を消して」
```

## ✅ 実装完了チェック

### 機能確認
- [ ] スキル起動確認
- [ ] メモ追加機能動作確認
- [ ] メモ読み上げ機能動作確認
- [ ] メモ削除機能動作確認
- [ ] エラーハンドリング確認

### 品質確認
- [ ] レスポンス速度（3秒以内）
- [ ] S3データ永続化確認
- [ ] 音声認識精度確認

## 📖 次のステップ

実装が完了したら、[テストガイド](testing-guide.md)で包括的なテストを実行してください。

---

**トラブルシューティング**
問題が発生した場合は、コンソールのログを確認し、エラーメッセージを参考に修正してください。