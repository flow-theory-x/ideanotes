# Alexa Voice Memo Service

## 📋 プロジェクト概要

音声でメモを追加・読み上げ・削除できるAlexaスキル。Alexa-hostedで完全無料実装。

**コンセプト**: 手軽な音声メモ管理の最小実装

## 🎯 機能仕様

### 基本機能（3つ）
1. **メモ追加**: 「アレクサ、メモに〜を追加」
2. **メモ読み上げ**: 「アレクサ、メモを読んで」（番号付きで読み上げ）
3. **メモ削除**: 「アレクサ、{番号}番目のメモを削除」

### 対話フロー例
```
ユーザー: 「アレクサ、ボイスメモを開いて」
Alexa: 「ボイスメモです。メモの追加、読み上げ、削除ができます。」

ユーザー: 「メモに牛乳を買うを追加」
Alexa: 「牛乳を買うをメモに追加しました」

ユーザー: 「メモを読んで」
Alexa: 「メモが2件あります。1番目、牛乳を買う。2番目、会議資料準備。」

ユーザー: 「1番目のメモを削除」
Alexa: 「1番目のメモ『牛乳を買う』を削除しました」
```

## 🏗️ 技術仕様

### アーキテクチャ
```
Alexa Device → Skills Kit → Lambda (Alexa-hosted) → S3 (データ永続化)
```

### 使用技術
- **プラットフォーム**: Alexa-hosted（Node.js 16.x）
- **データ永続化**: Amazon S3
- **開発環境**: Alexa開発者コンソール
- **コスト**: 完全無料（AWS無料枠内）

### データ設計

#### S3保存形式
```json
{
  "userId": "amzn1.ask.account.xxx",
  "memos": [
    {
      "id": 1,
      "text": "牛乳を買う",
      "timestamp": "2025-07-12T10:30:00Z",
      "deleted": false
    },
    {
      "id": 2, 
      "text": "会議資料準備",
      "timestamp": "2025-07-12T11:00:00Z",
      "deleted": false
    }
  ],
  "nextId": 3
}
```

#### ファイル構造
- **ファイル名**: `memo-{userId}.json`
- **保存場所**: S3バケット（自動作成）
- **文字エンコーディング**: UTF-8

## 🗣️ 対話モデル設計

### インテント定義

#### 1. AddMemoIntent
```json
{
  "name": "AddMemoIntent",
  "slots": [
    {
      "name": "memoText",
      "type": "AMAZON.AlphaNumeric"
    }
  ],
  "samples": [
    "メモに {memoText} を追加",
    "{memoText} をメモして",
    "{memoText} を追加",
    "メモに {memoText}",
    "{memoText} をメモに入れて"
  ]
}
```

#### 2. ReadMemosIntent
```json
{
  "name": "ReadMemosIntent",
  "slots": [],
  "samples": [
    "メモを読んで",
    "メモ一覧",
    "メモを教えて",
    "メモの内容は",
    "メモを聞かせて"
  ]
}
```

#### 3. DeleteMemoIntent
```json
{
  "name": "DeleteMemoIntent",
  "slots": [
    {
      "name": "memoNumber",
      "type": "AMAZON.NUMBER"
    }
  ],
  "samples": [
    "{memoNumber} 番目のメモを削除",
    "{memoNumber} 番を削除",
    "{memoNumber} 番目を消して",
    "メモの {memoNumber} 番を削除"
  ]
}
```

### 標準インテント
- **LaunchRequest**: スキル起動時の挨拶
- **AMAZON.HelpIntent**: ヘルプ情報提供
- **AMAZON.StopIntent / AMAZON.CancelIntent**: スキル終了

## 🔧 実装詳細

### Lambda関数構成

#### 必須パッケージ
```json
{
  "dependencies": {
    "ask-sdk-core": "^2.13.0",
    "ask-sdk-s3-persistence-adapter": "^2.13.0"
  }
}
```

#### ハンドラー設計
```javascript
// 主要ハンドラー
- LaunchRequestHandler: スキル起動
- AddMemoIntentHandler: メモ追加処理
- ReadMemosIntentHandler: メモ読み上げ処理  
- DeleteMemoIntentHandler: メモ削除処理
- HelpIntentHandler: ヘルプ情報
- CancelAndStopIntentHandler: 終了処理
- ErrorHandler: エラー処理
```

### S3永続化実装

#### 初期化
```javascript
const { S3PersistenceAdapter } = require('ask-sdk-s3-persistence-adapter');

const persistenceAdapter = new S3PersistenceAdapter({
    bucketName: process.env.S3_PERSISTENCE_BUCKET
});
```

#### データ操作パターン
```javascript
// 読み込み
const attributesManager = handlerInput.attributesManager;
const sessionAttributes = await attributesManager.getPersistentAttributes();

// 保存
sessionAttributes.memos = updatedMemos;
attributesManager.setPersistentAttributes(sessionAttributes);
await attributesManager.savePersistentAttributes();
```

## 🧪 テスト仕様

### テストケース

#### 基本機能テスト
1. **メモ追加**
   - 空リストへの初回追加
   - 既存リストへの追加
   - 特殊文字含むテキスト追加

2. **メモ読み上げ**
   - 空リスト時の対応
   - 1件の場合
   - 複数件の場合

3. **メモ削除**
   - 有効な番号指定
   - 無効な番号指定（0、負数、存在しない番号）
   - 削除後の番号振り直し

#### エラーハンドリングテスト
- 音声認識失敗時の対応
- S3アクセスエラー時の対応
- 不正なスロット値の処理

### テスト手順
1. Alexa開発者コンソールでのシミュレーターテスト
2. 実機（Echo）でのテスト
3. 各エラーケースの確認

## 📚 ファイル構造
```
alexa-voice-memo/
├── README.md                    # このファイル
├── setup-guide.md              # セットアップ手順書
├── implementation-guide.md     # 実装ガイド
├── testing-checklist.md        # テストチェックリスト
└── development-logs/           # 開発ログ
    └── progress-log.md
```

## 🚀 開発ステップ

### Phase 1: セットアップ（完了目標：1日）
1. Amazon開発者アカウント作成
2. Alexa-hostedスキル作成
3. 基本対話モデル設定

### Phase 2: 実装（完了目標：2-3日）
1. Lambda関数実装
2. S3永続化実装
3. エラーハンドリング実装

### Phase 3: テスト・改善（完了目標：1日）
1. 機能テスト
2. エラーケーステスト
3. 音声認識精度調整

### Phase 4: 公開準備（オプション）
1. スキル情報整備
2. 認定申請
3. 公開

---

**プロジェクト開始日**: 2025-07-12  
**想定完了日**: 2025-07-17  
**実装者**: Claude & User  
**プロジェクト方針**: スモールスタート原則による段階的開発