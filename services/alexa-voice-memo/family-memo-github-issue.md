# GitHub Issue テンプレート: AVM家族メモ機能実装

## 🎯 概要

Alexa Voice Memoを「家族専用共有メモ」に進化させる。
**「包丁持ってても、買い物忘れない」** がコンセプト。

## 💡 背景とコンセプト

### なぜ作るのか
- 買い物頼まれた → 忘れた → 「なにやっとんじゃ〜！」を防ぐ
- 共有メモアプリはたくさんあるけど、わざわざ開かない
- 包丁持ってる時でも「アレクサ、ボイスメモ」で済む手軽さ

### 核となる思想
- **1ユーザー = 1ファミリー**: グループ切り替えなし
- **familyID = userID**: 無駄なID生成を排除（オッカムの剃刀）
- **Alexaも家族の一員**: 音声で招待、QRで招待

### 簡単離婚システム
- 退出 → 自分のfamilyIdに戻るだけ
- 筆頭者移譲 → 2行のUPDATEで完了
- 過去のメモは残る（新しい人生をゼロから）

## 📚 設計書

以下の設計書を読んで実装してください：

1. **[コンセプトドキュメント](https://github.com/flow-theory-x/ideanotes/blob/main/services/alexa-voice-memo/family-memo-concept.md)**
   - ユーザー視点での価値
   - 使用シーン
   - キャッチコピー

2. **[完全仕様書v2.0](https://github.com/flow-theory-x/ideanotes/blob/main/services/alexa-voice-memo/family-memo-specification-v2.md)**
   - システム設計
   - データフロー
   - 実装詳細

3. **[テーブル定義書](https://github.com/flow-theory-x/ideanotes/blob/main/services/alexa-voice-memo/family-memo-table-definition.md)**
   - DynamoDB構造
   - ER図
   - CDKコード例

4. **[実装タスク一覧](https://github.com/flow-theory-x/ideanotes/blob/main/services/alexa-voice-memo/family-memo-implementation-tasks.md)**
   - 5フェーズ分解
   - 優先順位付き
   - 時間見積もり

## 🚀 実装対象

### Phase 3: 家族機能コア（17分実装チャレンジ）

```typescript
POST   /invite-codes      // 招待コード生成
POST   /join-family       // 家族に参加
POST   /leave-family      // 家族から退出
POST   /transfer-owner    // 筆頭者移譲
GET    /family-members    // メンバー一覧
```

**なぜ17分で可能か**：
- 仕様が完全に固定
- SQL文まで設計済み
- 判断箇所ゼロ
- 既存APIの拡張で済む

## ✅ 受け入れ条件

- [ ] 家族管理APIが動作する
- [ ] familyIdでメモが共有される
- [ ] 招待コードが5分で失効する
- [ ] 筆頭者移譲が2行UPDATEで完了する
- [ ] テストが通る

## 🎬 補足

このプロジェクトは「528%効率化」を実証したAVM開発の続編です。
設計の完成度により、実装は純粋な作業となることを目指しています。

**「なんだかんだでいいものができた」を超えて、「完璧な設計」を。**

---

*オッカムの剃刀が導いた、究極の「ちょうどいい」実装へ。*