# TubeMixer API設計書

## 1. API概要

TubeMixerのAPIは2種類に分かれる:

| 種別 | 説明 |
|------|------|
| 内部API Routes | Next.js Route Handlers。フロントエンドから呼び出す |
| YouTube Data API v3 | 外部API。API Routes経由でサーバーサイドから呼び出す |

全ての内部API Routesは認証必須（未認証時は `401 Unauthorized`）。

---

## 2. 内部API Routes

### 2.1 認証API

#### `GET/POST /api/auth/[...nextauth]`

NextAuth.jsが自動的に処理するルート。直接設計する必要はない。

| パス | 説明 |
|------|------|
| `/api/auth/signin` | サインインページ |
| `/api/auth/callback/google` | Google OAuthコールバック |
| `/api/auth/signout` | サインアウト |
| `/api/auth/session` | セッション情報取得 |

---

### 2.2 チャンネル登録一覧取得

#### `GET /api/youtube/subscriptions`

ログインユーザーのチャンネル登録一覧を取得し、各チャンネルのアップロード再生リストIDも付与して返す。

**リクエスト**

| パラメータ | なし |
|-----------|------|

**レスポンス (200)**

```typescript
{
  channels: YouTubeChannel[];
}

interface YouTubeChannel {
  id: string;              // チャンネルID
  title: string;           // チャンネル名
  thumbnailUrl: string;    // チャンネルアイコンURL
  uploadsPlaylistId: string; // アップロード再生リストID
}
```

**処理フロー**

```
1. subscriptions.list で全チャンネル登録を取得（ページネーション対応、最大50件/ページ）
2. 取得したチャンネルIDで channels.list を呼び出し（最大50件一括）
   → contentDetails.relatedPlaylists.uploads を取得
3. チャンネル情報 + uploadsPlaylistId をマージして返却
```

**エラーレスポンス**

| ステータス | ボディ | 条件 |
|-----------|--------|------|
| 401 | `{ error: "Unauthorized" }` | 未認証 |
| 403 | `{ error: "YouTube API access denied" }` | スコープ不足・アカウント問題 |
| 429 | `{ error: "API quota exceeded" }` | YouTube APIクォータ上限 |
| 500 | `{ error: "Internal server error" }` | その他エラー |

---

### 2.3 チャンネル別動画取得

#### `GET /api/youtube/videos`

指定チャンネルの動画を日付範囲でフィルタして返す。

**リクエスト**

| パラメータ | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| `playlistId` | string | Yes | アップロード再生リストID |
| `after` | string | Yes | 開始日（ISO 8601、例: `2026-03-20T00:00:00Z`） |
| `before` | string | Yes | 終了日（ISO 8601、例: `2026-03-23T23:59:59Z`） |

**レスポンス (200)**

```typescript
{
  videos: YouTubeVideo[];
}

interface YouTubeVideo {
  id: string;              // 動画ID
  title: string;           // 動画タイトル
  channelId: string;       // チャンネルID
  channelTitle: string;    // チャンネル名
  thumbnailUrl: string;    // サムネイルURL
  publishedAt: string;     // 投稿日時（ISO 8601）
}
```

**処理フロー**

```
1. playlistItems.list でアップロード再生リストの動画を取得
   → maxResults=50、ページネーション対応
2. 各動画の publishedAt を確認し、日付範囲外ならスキップ
3. publishedAt が after より古い動画が見つかったらページネーション終了
   （動画は新しい順に並んでいるため、それ以降は全て範囲外）
4. フィルタ済みの動画リストを返却
```

**エラーレスポンス**

| ステータス | ボディ | 条件 |
|-----------|--------|------|
| 400 | `{ error: "Missing required parameters" }` | パラメータ不足 |
| 401 | `{ error: "Unauthorized" }` | 未認証 |
| 429 | `{ error: "API quota exceeded" }` | YouTube APIクォータ上限 |
| 500 | `{ error: "Internal server error" }` | その他エラー |

---

### 2.4 グループ一覧取得

#### `GET /api/groups`

ログインユーザーのグループ設定を取得。

**リクエスト**

| パラメータ | なし |
|-----------|------|

**レスポンス (200)**

```typescript
{
  groups: Group[];
}

interface Group {
  id: string;
  name: string;
  channelIds: string[];
  createdAt: string;
  updatedAt: string;
}
```

**処理フロー**

```
1. JWTからgoogleIdを取得
2. Vercel KV から user:{googleId}:groups を取得
3. 存在しない場合は空配列を返却
```

---

### 2.5 グループ一括保存

#### `PUT /api/groups`

グループ設定を一括で上書き保存。

**リクエスト**

```typescript
{
  groups: Group[];
}
```

**レスポンス (200)**

```typescript
{
  groups: Group[];
}
```

**処理フロー**

```
1. JWTからgoogleIdを取得
2. リクエストボディのバリデーション
   - groups が配列であること
   - 各グループに id, name, channelIds が存在すること
3. Vercel KV に user:{googleId}:groups を保存
4. 保存後のデータを返却
```

**エラーレスポンス**

| ステータス | ボディ | 条件 |
|-----------|--------|------|
| 400 | `{ error: "Invalid request body" }` | バリデーションエラー |
| 401 | `{ error: "Unauthorized" }` | 未認証 |
| 500 | `{ error: "Internal server error" }` | KV書き込みエラー |

---

## 3. YouTube Data API v3 利用パターン

### 3.1 使用するAPIメソッド

| メソッド | コスト | 用途 |
|---------|--------|------|
| `subscriptions.list` | 1ユニット/リクエスト | チャンネル登録一覧取得 |
| `channels.list` | 1ユニット/リクエスト | チャンネル詳細取得（uploadsPlaylistId） |
| `playlistItems.list` | 1ユニット/リクエスト | チャンネル動画一覧取得 |

### 3.2 使用しないAPIメソッド

| メソッド | コスト | 不使用の理由 |
|---------|--------|-------------|
| `search.list` | 100ユニット/リクエスト | コスト高。playlistItems.listで代替 |
| `videos.list` | 1ユニット/リクエスト | 再生時間等の詳細情報は不要 |

### 3.3 呼び出しフロー

```
subscriptions.list (mine=true, part=snippet, maxResults=50)
  │  → チャンネルID一覧を取得
  │  → ページネーション: nextPageToken がある限り繰り返し
  │
  ▼
channels.list (id=<チャンネルIDをカンマ区切り>, part=contentDetails, maxResults=50)
  │  → uploadsPlaylistId を取得
  │  → 50件一括取得可能なので、1〜2リクエストで完了
  │
  ▼
playlistItems.list (playlistId=<uploadsPlaylistId>, part=snippet, maxResults=50)
     → 各チャンネルごとに呼び出し
     → publishedAt で日付フィルタ
     → 範囲外の古い動画が出たらページネーション終了
```

### 3.4 APIリクエストの実装例

```typescript
// subscriptions.list
const url = "https://www.googleapis.com/youtube/v3/subscriptions";
const params = {
  part: "snippet",
  mine: "true",
  maxResults: "50",
  pageToken: nextPageToken, // ページネーション用
};

// channels.list
const url = "https://www.googleapis.com/youtube/v3/channels";
const params = {
  part: "contentDetails",
  id: channelIds.join(","), // 最大50件
};

// playlistItems.list
const url = "https://www.googleapis.com/youtube/v3/playlistItems";
const params = {
  part: "snippet",
  playlistId: uploadsPlaylistId,
  maxResults: "50",
  pageToken: nextPageToken,
};
```

---

## 4. クォータ消費見積もり

### 4.1 1回のフィルタ操作（30チャンネルのグループ）

| 操作 | APIメソッド | リクエスト数 | ユニット |
|------|-----------|------------|---------|
| チャンネル登録一覧取得 | subscriptions.list | ~2 | 2 |
| チャンネル詳細取得 | channels.list | 1 | 1 |
| 各チャンネルの動画取得 | playlistItems.list | ~30 | 30 |
| **合計** | | **~33** | **33** |

### 4.2 1日あたりの想定使用量

| シナリオ | フィルタ回数 | ユニット消費 |
|---------|------------|------------|
| 軽い利用 | 5回 | ~165 |
| 通常利用 | 20回 | ~660 |
| ヘビー利用 | 50回 | ~1,650 |

- 1日の上限: 10,000ユニット
- ヘビー利用でも十分余裕あり

---

## 5. エラーハンドリング

### 5.1 YouTube API エラー

| HTTPステータス | 意味 | アプリの対応 |
|--------------|------|------------|
| 401 | アクセストークン無効 | トークンリフレッシュ → 失敗ならログイン画面へ |
| 403 | クォータ超過 or 権限不足 | クォータ超過メッセージ表示 |
| 404 | リソース未検出 | スキップして処理続行 |
| 429 | レートリミット | リトライ（exponential backoff） |
| 500/503 | YouTube側エラー | エラーメッセージ表示 |

### 5.2 フロントエンドでのエラー表示

```typescript
// エラーレスポンスの統一型
interface ApiError {
  error: string;
  details?: string;
}
```

| エラー種別 | 表示メッセージ |
|-----------|-------------|
| 認証エラー | 「セッションが切れました。再ログインしてください」 |
| クォータ超過 | 「本日のAPI上限に達しました。明日再度お試しください」 |
| ネットワークエラー | 「通信エラーが発生しました。接続を確認してください」 |
| その他 | 「エラーが発生しました。しばらくしてから再度お試しください」 |

---

## 6. 再生用URL生成（フロントエンド処理）

API Routesを使用せず、フロントエンドで直接生成する。

### 6.1 URL形式

```
https://www.youtube.com/watch_videos?video_ids={id1},{id2},{id3},...
```

### 6.2 YouTubeアプリ起動

```typescript
function openInYouTubeApp(videoIds: string[]): void {
  const ids = videoIds.join(",");
  const webUrl = `https://www.youtube.com/watch_videos?video_ids=${ids}`;
  const appUrl = `youtube://watch_videos?video_ids=${ids}`;

  // YouTubeアプリを試行
  window.location.href = appUrl;

  // フォールバック: 一定時間後にブラウザで開く
  setTimeout(() => {
    window.open(webUrl, "_blank");
  }, 2000);
}
```

### 6.3 制約

- 動画数が約50本を超えるとURLが長すぎて動作しない可能性あり
- 50本超の場合はUI上で警告メッセージを表示
- 非公式URLのため、将来仕様変更の可能性あり
