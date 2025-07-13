# Alexa Voice Memo 「家族メモ」完全仕様書 v2.0

*2025-07-14 - オッカムの剃刀が導いた究極のシンプル設計*

## 📋 概要

「包丁持ってても、買い物忘れない」をコンセプトに、音声で家族のメモを共有するサービス。
徹底的にシンプルな設計により、技術的堅牢性と使いやすさを両立。

## 🎯 設計思想

### オッカムの剃刀
- 必要以上に要素を増やさない
- 既存のもので解決できるなら新しいものを作らない
- シンプルな解が、たいてい正解

### 基本原則
- **1ユーザー = 1ファミリー**: グループ切り替えなし
- **familyID = userID**: 無駄なID生成を排除
- **家族の信頼前提**: 複雑な承認フロー不要

## 🏗️ システム構成

### 認証方式
- **Web UI**: Google SSO（ワンクリックログイン）
- **Alexa**: Amazon ID（自動認証済み）
- **統一的な扱い**: どちらもuserIDとして平等に扱う

### データ構造

```typescript
interface User {
  userId: string;        // Google ID or Amazon ID
  familyId: string;      // 所属ファミリー（= 筆頭者のuserId）
  name: string;          // 表示名
  createdAt: Date;
}

// Familiesテーブルは不要！
// - メンバー一覧: UsersをfamilyIdでクエリ
// - 筆頭者: familyId = userIdの人
// - 買い物メモなので大量UPDATEの心配なし

interface Memo {
  memoId: string;        // UUID
  familyId: string;      // 所属ファミリー
  content: string;       // メモ内容
  createdBy: string;     // 作成者userId
  createdAt: Date;
  completedAt?: Date;    // 完了日時
  completedBy?: string;  // 完了者userId
}

interface InviteCode {
  code: string;          // 4-6桁の数字
  familyId: string;      // 招待先のfamilyId
  type: 'qr' | 'voice';  // QR用 or 音声用
  expiresAt: Date;       // 5分後失効
  used: boolean;
}
```

## 🤝 家族管理フロー

### 初回利用
```
【Web UI】
1. Googleでログイン → 即利用開始
2. familyId = 自分のGoogle ID

【Alexa】
1. スキル有効化 → 即利用開始
2. familyId = 自分のAmazon ID
```

### 家族への招待

#### QRコード招待（人間用）
```
1. 招待する側: 「家族を追加」→ QRコード表示
2. 招待される側: スマホでQR読み取り
3. 自動的にfamilyId更新 & メモマージ
4. 完了！
```

#### 音声招待（Alexa用）
```
1. 招待する側: 「Alexaを追加」→ 4桁コード表示（例: 3792）
2. 「アレクサ、ボイスメモを開いて」
3. 「招待コード3792」
4. Alexa「太郎さんの家族に参加しました」
```

### ライフイベント対応

#### 結婚（家族への参加）
```sql
-- 花子さんが太郎さんの家族に参加
UPDATE users SET familyId = 'taro@gmail.com' 
WHERE userId = 'hanako@gmail.com';

UPDATE memos SET familyId = 'taro@gmail.com' 
WHERE familyId = 'hanako@gmail.com';
```

#### 離婚（家族からの退出）
```sql
-- 花子さんが独立
UPDATE users SET familyId = 'hanako@gmail.com' 
WHERE userId = 'hanako@gmail.com';
-- 過去のメモは太郎さんの家族に残る
```

#### 筆頭者の離婚（全員のfamilyId更新）
```sql
-- 太郎さんが花子さんに筆頭者を移譲
UPDATE users SET familyId = 'hanako@gmail.com' 
WHERE familyId = 'taro@gmail.com';

UPDATE memos SET familyId = 'hanako@gmail.com' 
WHERE familyId = 'taro@gmail.com';

-- 太郎さんが退出して独立
UPDATE users SET familyId = 'taro@gmail.com' 
WHERE userId = 'taro@gmail.com';

-- 買い物メモなので量は少ない！問題なし
```

## 🎯 機能仕様

### メモ作成
- **Alexa**: 「アレクサ、ボイスメモを開いて」→ 音声入力
- **Web UI**: マイクボタン or テキスト入力
- **統一体験**: どちらも「声で言えば家族メモに入る」

### メモ表示
- 家族全員の未完了メモを新しい順に表示
- 作成者名を表示（誰が頼んだか分かる）
- 完了済みは別セクション or 非表示

### メモ完了
- チェックボックスで完了マーク
- 完了者と完了時刻を記録

## 🚀 実装のポイント

### 最小限の実装
- 複雑なグループ管理 → 不要
- UUID生成ロジック → 不要
- アカウント連携 → 不要
- 承認フロー → 不要

### セキュリティ
- OAuth 2.0による安全な認証
- ワンタイムコードは5分で失効
- familyIdによる確実なアクセス制御

### スケーラビリティ
- DynamoDBによる自動スケール
- Lambdaによるサーバーレス構成
- 家族単位での独立したデータ管理

## 📱 UI/UX設計

### 初回体験
1. 「Googleでログイン」ボタンのみ
2. ログイン後、即メモ画面
3. 説明不要の直感的UI

### 家族追加
- 大きな「家族を追加」ボタン
- QRコード or 音声コード選択
- 5分のカウントダウン表示

### 日常利用
- メモ一覧がメイン画面
- 大きなマイクボタン
- 完了チェックは直感的に

## 🎬 まとめ

**技術的シンプルさ = 人間的な自然さ**

- familyID = userIDという天才的発想
- QR/音声による直感的な招待
- 2行のUPDATEで人生の転機を表現

これは単なる買い物メモアプリではなく、
**家族のコミュニケーションを音声でつなぐ**、
新しい形の家族アプリケーション。

「なんだかんだでいいものができた」を超えて、
**設計思想として完璧なもの**が完成した。

---

*オッカムの剃刀が導いた、究極の「ちょうどいい」設計。*