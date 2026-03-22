# TubeMixer データベース設計書

## 1. ストレージ概要

| ストレージ | 用途 | 技術 |
|-----------|------|------|
| Vercel KV | グループ設定の永続化 | Redis互換キーバリューストア |
| JWT Cookie | セッション・トークン管理 | NextAuth.js JWTセッション |

- RDBは使用しない（個人専用アプリで、永続化が必要なデータはグループ設定のみ）
- 動画データ・チャンネルデータはキャッシュせず、毎回YouTube APIから取得

---

## 2. Vercel KV キー設計

### 2.1 キー一覧

| キー | 型 | 説明 |
|------|-----|------|
| `user:{googleId}:groups` | `string` (JSON) | ユーザーの全グループ設定 |

### 2.2 設計方針

- **1キーに全グループを集約**: グループごとにキーを分けるとKVリクエスト数が増加し、無料枠（3,000リクエスト/日）を圧迫するため
- **チャンネルはIDのみ保存**: チャンネル名・アイコンはYouTube APIから都度取得（データ鮮度を担保）
- **ログアウト時もデータ保持**: 再ログインで同じGoogleアカウントなら復元可能

### 2.3 データ構造

```typescript
// キー: user:{googleId}:groups
// 値: GroupData[] のJSON文字列

interface GroupData {
  id: string;           // UUID v4
  name: string;         // グループ名
  channelIds: string[]; // YouTubeチャンネルIDの配列
  createdAt: string;    // ISO 8601 作成日時
  updatedAt: string;    // ISO 8601 更新日時
}
```

### 2.4 データ例

```json
// キー: user:1234567890:groups
[
  {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "name": "ゲーム実況",
    "channelIds": [
      "UC1234567890abcdef",
      "UC0987654321fedcba"
    ],
    "createdAt": "2026-03-20T10:00:00.000Z",
    "updatedAt": "2026-03-22T15:30:00.000Z"
  },
  {
    "id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
    "name": "音楽",
    "channelIds": [
      "UC1111111111aaaaa",
      "UC2222222222bbbbb",
      "UC3333333333ccccc"
    ],
    "createdAt": "2026-03-20T10:05:00.000Z",
    "updatedAt": "2026-03-21T09:00:00.000Z"
  }
]
```

---

## 3. KV操作パターン

### 3.1 グループ一覧取得

```typescript
import { kv } from "@vercel/kv";

async function getGroups(googleId: string): Promise<GroupData[]> {
  const groups = await kv.get<GroupData[]>(`user:${googleId}:groups`);
  return groups ?? [];
}
```

### 3.2 グループ一括保存

```typescript
async function saveGroups(googleId: string, groups: GroupData[]): Promise<void> {
  await kv.set(`user:${googleId}:groups`, groups);
}
```

### 3.3 KVリクエスト見積もり

| 操作 | リクエスト数 | 頻度 |
|------|------------|------|
| グループ一覧取得 | 1 | ページ表示ごと |
| グループ保存 | 1 | グループ変更ごと |

- 1日あたりの想定: 数十リクエスト（個人利用）
- 無料枠 3,000リクエスト/日に対して十分余裕あり

---

## 4. JWTトークン構造

### 4.1 NextAuth.js JWT ペイロード

```typescript
interface JWTPayload {
  // NextAuth.js 標準フィールド
  sub: string;              // GoogleアカウントID
  name: string;             // ユーザー名
  email: string;            // メールアドレス
  picture: string;          // プロフィール画像URL

  // カスタムフィールド
  accessToken: string;      // Google OAuth アクセストークン
  refreshToken: string;     // Google OAuth リフレッシュトークン
  accessTokenExpires: number; // アクセストークン有効期限（UNIXタイムスタンプ）
  googleId: string;         // GoogleアカウントID（KVキーに使用）
  error?: string;           // トークンリフレッシュエラー
}
```

### 4.2 トークンライフサイクル

```
初回ログイン
  │
  ├─ Google OAuthから accessToken + refreshToken を取得
  ├─ JWTに格納してCookieに保存
  │
  ▼
API リクエスト時
  │
  ├─ JWTからaccessTokenを取得
  ├─ 有効期限チェック
  │   ├─ 有効 → そのまま使用
  │   └─ 期限切れ → refreshTokenで更新
  │       ├─ 成功 → 新しいaccessTokenをJWTに保存
  │       └─ 失敗 → error: "RefreshAccessTokenError"
  │                 → フロントでログイン画面にリダイレクト
  │
  ▼
7日後（テストモード制約）
  │
  └─ refreshToken失効 → 再ログインが必要
```

### 4.3 トークンリフレッシュ処理

```typescript
async function refreshAccessToken(token: JWTPayload): Promise<JWTPayload> {
  const response = await fetch("https://oauth2.googleapis.com/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      client_id: process.env.GOOGLE_CLIENT_ID!,
      client_secret: process.env.GOOGLE_CLIENT_SECRET!,
      grant_type: "refresh_token",
      refresh_token: token.refreshToken,
    }),
  });

  const refreshed = await response.json();

  if (!response.ok) {
    return { ...token, error: "RefreshAccessTokenError" };
  }

  return {
    ...token,
    accessToken: refreshed.access_token,
    accessTokenExpires: Date.now() + refreshed.expires_in * 1000,
  };
}
```

---

## 5. 容量見積もり

### 5.1 1ユーザーあたりのデータサイズ

| 項目 | サイズ見積もり |
|------|-------------|
| グループ1件 | 約200バイト（ID + 名前 + 日時） |
| チャンネルID 1件 | 約30バイト |
| グループ10件 × 各30チャンネル | 約11KB |

### 5.2 Vercel KV 制限との比較

| 制限 | 値 | 使用見込み |
|------|-----|----------|
| ストレージ容量 | 256MB | ~11KB（十分余裕） |
| 日次リクエスト | 3,000 | 数十回（十分余裕） |
| 最大値サイズ | 1MB | ~11KB（十分余裕） |

---

## 6. 型定義ファイル

### 6.1 `src/types/group.ts`

```typescript
export interface Group {
  id: string;
  name: string;
  channelIds: string[];
  createdAt: string;
  updatedAt: string;
}
```

### 6.2 `src/types/youtube.ts`

```typescript
export interface YouTubeChannel {
  id: string;
  title: string;
  thumbnailUrl: string;
  uploadsPlaylistId: string;
}

export interface YouTubeVideo {
  id: string;
  title: string;
  channelId: string;
  channelTitle: string;
  thumbnailUrl: string;
  publishedAt: string;
}
```
