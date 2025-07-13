# AVM家族メモ - テーブル定義書

*2025-07-14 - DynamoDB拡張設計*

## 📊 テーブル構造

### 1. Memosテーブル（拡張）

#### 既存構造
```typescript
interface MemoItem {
  userId: string;      // Alexa User ID or Google ID
  memoId: string;      // memo_YYYYMMDD_XXX形式
  text: string;        // メモ内容
  timestamp: string;   // ISO 8601形式の日時
  deleted: string;     // "true" or "false"
  createdAt: string;   // 作成日時
  updatedAt: string;   // 更新日時
  version: number;     // バージョン番号
}
```

#### 拡張後構造
```typescript
interface MemoItem {
  // 既存フィールド
  userId: string;      // 作成者ID（変更なし）
  memoId: string;      
  text: string;        
  timestamp: string;   
  deleted: string;     
  createdAt: string;   
  updatedAt: string;   
  version: number;     
  
  // 新規追加フィールド
  familyId: string;    // 所属ファミリー（筆頭者のuserId）
  createdByName?: string; // 作成者の表示名
  completedAt?: string;   // 完了日時
  completedBy?: string;   // 完了者ID
  completedByName?: string; // 完了者の表示名
}
```

#### インデックス構造
```typescript
// Primary Key
partitionKey: userId (STRING)
sortKey: memoId (STRING)

// GSI（既存）
timestamp-index:
  partitionKey: userId
  sortKey: timestamp

status-index:
  partitionKey: userId
  sortKey: deleted

// GSI（新規追加）
family-timestamp-index:
  partitionKey: familyId
  sortKey: timestamp
  projection: ALL
```

### 2. Usersテーブル（新規）

```typescript
interface UserItem {
  userId: string;      // Google ID or Amazon ID (PK)
  familyId: string;    // 所属ファミリー（筆頭者のuserId）
  name: string;        // 表示名
  email?: string;      // メールアドレス（Googleユーザーのみ）
  provider: 'google' | 'amazon';  // 認証プロバイダ
  createdAt: string;   // 作成日時
  lastLoginAt: string; // 最終ログイン日時
}
```

### 3. InviteCodesテーブル（新規）

```typescript
interface InviteCodeItem {
  code: string;        // 4-6桁の招待コード (PK)
  familyId: string;    // 招待先のfamilyId
  type: 'qr' | 'voice'; // QR用 or 音声用
  createdAt: string;   // 作成日時
  expiresAt: string;   // 有効期限（5分後）
  used: boolean;       // 使用済みフラグ
  ttl: number;         // DynamoDB TTL（自動削除用）
}
```

## 🔄 データ移行計画

### Phase 1: 既存メモの移行
```typescript
// 全ての既存メモに familyId を追加
UPDATE memos SET familyId = userId WHERE familyId IS NULL
```

### Phase 2: ユーザーテーブル作成
```typescript
// 既存のuserIdから初期ユーザーレコードを生成
INSERT INTO users (userId, familyId, provider) 
VALUES (userId, userId, 'amazon')
```

## 📐 ER図

```mermaid
erDiagram
    Users ||--o{ Memos : creates
    Users ||--o{ Users : "belongs to family of"
    InviteCodes ||--|| Users : "invites to family"

    Users {
        string userId PK
        string familyId "筆頭者のuserId"
        string name
        string email
        string provider
        string createdAt
        string lastLoginAt
    }

    Memos {
        string userId PK
        string memoId SK
        string familyId "筆頭者のuserId"
        string text
        string timestamp
        string deleted
        string createdByName
        string completedAt
        string completedBy
        string completedByName
        string createdAt
        string updatedAt
        number version
    }

    InviteCodes {
        string code PK
        string familyId "招待先の筆頭者userId"
        string type
        string createdAt
        string expiresAt
        boolean used
        number ttl
    }
```

## 🚀 CDK実装例

```typescript
// Memosテーブル拡張（GSI追加のみ）
memosTable.addGlobalSecondaryIndex({
  indexName: 'family-timestamp-index',
  partitionKey: { name: 'familyId', type: AttributeType.STRING },
  sortKey: { name: 'timestamp', type: AttributeType.STRING },
  projectionType: ProjectionType.ALL,
});

// Usersテーブル新規作成
const usersTable = new Table(this, 'UsersTable', {
  tableName: `${projectName}-${environment}-users`,
  partitionKey: { name: 'userId', type: AttributeType.STRING },
  billingMode: BillingMode.PAY_PER_REQUEST,
  encryption: TableEncryption.AWS_MANAGED,
  removalPolicy: RemovalPolicy.DESTROY,
});

// InviteCodesテーブル新規作成（TTL付き）
const inviteCodesTable = new Table(this, 'InviteCodesTable', {
  tableName: `${projectName}-${environment}-invite-codes`,
  partitionKey: { name: 'code', type: AttributeType.STRING },
  billingMode: BillingMode.PAY_PER_REQUEST,
  encryption: TableEncryption.AWS_MANAGED,
  removalPolicy: RemovalPolicy.DESTROY,
  timeToLiveAttribute: 'ttl',
});
```

## 📝 実装時の注意事項

1. **後方互換性**: 既存の個人メモは familyId = userId として扱う
2. **GSI追加タイミング**: 既存テーブルへのGSI追加は無停止で可能
3. **TTL設定**: InviteCodesは5分後に自動削除
4. **データ整合性**: ユーザー削除時は関連データのクリーンアップ必要

## 🎯 この設計のメリット

- **最小限の変更**: 既存メモテーブルはGSI追加のみ
- **シンプルな構造**: familyId = userIDで複雑性を排除
- **拡張性**: 将来的な機能追加も容易
- **パフォーマンス**: 適切なインデックスで高速クエリ

---

*17分実装を可能にする、判断ゼロの設計書*