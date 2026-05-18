# モジュール境界、物理的・論理的な分離、冪等性

> 出典: `hybrid-architecture-patterns.md` からの抽出・整理

---

## モジュール境界

### 概要
システム内の責務（認証、トークン発行、ユーザー管理、クライアント管理、同意管理など）を明確に切り分ける**ドメインの境界**のこと。

### SSO開発における具体例

**悪い例** — モジュール境界が曖昧な場合:
```go
// auth_handler.go （認証ハンドラの中で全てを処理している）
func LoginHandler(w http.ResponseWriter, r *http.Request) {
    // 1. パスワード検証
    user := db.Query("SELECT * FROM users WHERE email = ?", email)
    
    // 2. 認可コード生成
    code := generateAuthCode()
    db.Exec("INSERT INTO auth_codes ...", code)
    
    // 3. クライアント情報の検証（直接クエリ）
    client := db.Query("SELECT * FROM oauth_clients WHERE id = ?", clientID)
    
    // 4. 同意情報の確認（直接クエリ）
    consent := db.Query("SELECT * FROM consents WHERE user_id = ?", user.ID)
    
    // 5. セッション作成
    session := createSession(user.ID)
    
    // 6. 監査ログ記録
    db.Exec("INSERT INTO audit_logs ...", "login", user.ID)
}
```

このコードの問題点:
- 認証・トークン・クライアント・同意・セッション・監査の責務が全て混在
- 1つのファイルを変更するだけで、不関連な機能まで影響を受ける
- 後から「認証だけを切り出したい」と思っても、依存が絡み合って分離が困難

**良い例** — モジュール境界を明確に分離:
```go
// auth/handler.go （認証モジュールの責務のみ）
func LoginHandler(w http.ResponseWriter, r *http.Request) {
    // 認証モジュールの責務: パスワード検証のみ
    user, err := authModule.VerifyPassword(email, password)
    if err != nil { ... }
    
    // トークンモジュールに委譲
    authCode, err := tokenModule.GenerateAuthCode(user.ID, clientID)
    if err != nil { ... }
    
    // セッションモジュールに委譲
    session, err := sessionModule.Create(user.ID)
    if err != nil { ... }
    
    // 監査はイベントで非同期化
    eventBus.Publish("login.succeeded", user.ID)
}
```

**推奨されるSSOモジュール構成**:
```
認証モジュール   → パスワード/MFA検証
トークンモジュール → JWT/認可コード発行
ユーザーモジュール → プロフィール/属性管理
クライアントモジュール → OAuthクライアント登録
同意モジュール    → スコープ同意画面/記録
セッションモジュール → Cookie/セッション管理
```

---

## 物理的な分離

### 概要
データベースやプロセス、ネットワークなどを**実際に分離**すること。

### SSO開発における具体例

**悪い例** — 物理的な分離なし（同一DB）:
```sql
-- idp-auth サービスが、user テーブルに直接アクセス
SELECT password_hash FROM users WHERE id = 'user-123';

-- idp-auth サービスが、client テーブルに直接アクセス
SELECT client_secret FROM oauth_clients WHERE id = 'client-456';
```

この設計の問題点:
- 「結局JOINすればいい」となり、モジュール境界が崩壊
- `idp-auth` が `users` テーブルのスキーマ変更に影響を受ける
- 後から `idp-user` を独立させたい時に、依存が多すぎて分離不可能

**良い例** — 各モジュールに独立DB:
```
idp-auth DB:
  - auth_codes （認可コード）
  - sessions   （セッション）

idp-user DB:
  - users      （ユーザー情報）
  - passwords  （パスワードハッシュ）

idp-client DB:
  - clients    （クライアント登録情報）
```

```go
// idp-auth は idp-user のAPIを呼び出す（テーブルを直接参照しない）
resp, err := http.Post("http://idp-user:8080/auth/verify", ...)

// idp-auth は自DBにのみ書き込む
db.Exec("INSERT INTO auth_codes ...", code)
```

**物理的な分離の効果**:
| 分離前（同一DB） | 分離後（独立DB） |
|---|---|
| `idp-auth` が `users` テーブルを直接SELECT | `idp-auth` は `idp-user` のAPIを呼び出す |
| スキーマ変更が全体に影響 | 各サービスは自DBのスキーマのみ管理 |
| 「JOINで済ませよう」となり境界崩壊 | 物理的な壁で論理的な分離を強制 |

**注意点**: OIDCの認可コードフローでは、トランザクション的に整合性が必要な更新は**認可サービスのDB内だけで完結**する。`idp-user` への問い合わせは参照系なので分散トランザクションは不要。

---

## 論理的な分離

### 概要
コードや責務の境界を**概念的に分離**すること。同一プロセス・同一DB内でも適用可能。

### SSO開発における具体例

**悪い例** — 論理的な分離なし:
```go
// どのモジュールにも属さない「共通ユーティリティ」
package utils

func GetUserByID(db *sql.DB, userID string) (*User, error) {
    // 誰からでも呼べる → 依存関係が不明確
}

func GetClientByID(db *sql.DB, clientID string) (*Client, error) {
    // どのモジュールの責務か分からない
}
```

**良い例** — 論理的な分離あり（同一DBでも適用可能）:
```go
// user/repository.go （ユーザーモジュールの内部）
package user

type Repository interface {
    FindByID(ctx context.Context, id string) (*User, error)
    VerifyPassword(ctx context.Context, email, password string) (*User, error)
}

type repository struct {
    db *sql.DB  // ユーザーモジュールのDB（論理的には独立）
}

// auth/service.go （認証モジュール）
package auth

type Service struct {
    userRepo user.Repository  // インターフェース経由のみアクセス
    // authモジュールはuserのテーブル構造を知らない
}

func (s *Service) Authenticate(ctx context.Context, email, password string) (*AuthResult, error) {
    user, err := s.userRepo.VerifyPassword(ctx, email, password)
    // userのパスワードハッシュ形式（bcrypt? argon2?）を知らない
    // userテーブルのカラム名を知らない
}
```

**論理的な分離の実践**:
1. **モジュール間は公開API経由で通信** — 同一プロセス内でも直接DBテーブルを跨いでSQLを書かない
2. **インターフェースを定義してDIする** — `idp-auth` が「どのDBを使うか」を知らないようにする
3. **インメモリイベントバス** — 認証成功イベント → 監査ログ記録、同意更新イベント → キャッシュ無効化など、同一プロセス内の疎結合を実現

**物理的 vs 論理的な分離の比較**:

| | 物理的な分離 | 論理的な分離 |
|---|---|---|
| **対象** | DB、プロセス、ネットワーク | コード、責務、インターフェース |
| **同一DBで可能？** | × | ○ |
| **強制力** | 強い（実際に分かれている） | 弱い（自律が必要） |
| **学習コスト** | 高い（分散システムの知識が必要） | 低い（設計パターンで実現） |

---

## 冪等性（Idempotency）

### 概要
**同じ操作を何度実行しても、結果が変わらない性質**。分散システムやイベント駆動アーキテクチャで特に重要。

### SSO開発における具体例

#### 例1: 認証APIの冪等性

**非冪等（問題がある）**:
```go
func (s *UserService) RecordAuthHistory(ctx context.Context, userID string) error {
    // 同じリクエストが2回来ると、履歴が2重登録される
    _, err := s.db.Exec("INSERT INTO auth_history (user_id, created_at) VALUES (?, NOW())", userID)
    return err
}
```

**冪等（正しい）**:
```go
func (s *UserService) RecordAuthHistory(ctx context.Context, userID string, requestID string) error {
    // request_id で重複を防止
    _, err := s.db.Exec(`
        INSERT INTO auth_history (user_id, request_id, created_at) 
        VALUES (?, ?, NOW())
        ON CONFLICT (request_id) DO NOTHING
    `, userID, requestID)
    return err
}
```

**シナリオ**:
1. クライアントが `POST /auth/verify` を送信（`request_id: abc-123`）
2. サーバーが処理中にタイムアウト → クライアントが再試行
3. 同じ `request_id: abc-123` で2回目のリクエスト
4. **冪等設計により、認証履歴は1件のみ記録される**

#### 例2: イベント購読の冪等性

**非冪等**:
```go
func HandleLoginEvent(event LoginEvent) error {
    // イベントが重複配送されると、メールが2通届く
    email.Send(event.UserID, "ログイン通知")
    return nil
}
```

**冪等**:
```go
func HandleLoginEvent(event LoginEvent) error {
    // event_id で重複排除
    if alreadyProcessed(event.EventID) {
        return nil // 処理済みなので無視
    }
    
    email.Send(event.UserID, "ログイン通知")
    markAsProcessed(event.EventID)
    return nil
}
```

#### 例3: CQRS投影の冪等性

**非冪等**:
```go
func ProjectUserCreated(event UserCreatedEvent) error {
    // 同じイベントを2回適用すると、ユーザーのポイントが2倍加算される
    db.Exec("UPDATE user_stats SET total_users = total_users + 1")
    return nil
}
```

**冪等**:
```go
func ProjectUserCreated(event UserCreatedEvent) error {
    // UPSERT で冪等に
    db.Exec(`
        INSERT INTO user_stats (user_id, total_users, created_at)
        VALUES (?, 1, ?)
        ON CONFLICT (user_id) DO NOTHING
    `, event.UserID, event.Timestamp)
    return nil
}
```

**SSOにおける冪等性の確認ポイント**:

| # | 確認項目 | 非冪等のリスク | 冪等の実装 |
|---|---|---|---|
| 1 | 認証履歴の記録 | ネットワーク重試行で履歴が2重登録 | `request_id` でUPSERT |
| 2 | 認可コードの発行 | 同じフローで2つのコードが発行 | フローIDで既存コードを返却 |
| 3 | イベント購読 | メール通知が複数回届く | `event_id` で重複排除 |
| 4 | スナップショット取得 | 再適用時に状態がずれる | イベントシーケンス番号で管理 |
| 5 | セッション作成 | 同時ログインでセッションが複数 | ユーザーIDで既存セッションを置換 |

---

## まとめ：4つの概念の関係性

```
モジュール境界（責務の定義）
    ↓
論理的な分離（コードレベルでの分離）
    ↓
物理的な分離（DB/プロセスレベルでの分離）
    ↓
冪等性（分散後の安全な運用）
```

| 概念 | SSO開発での目的 | 具体例 |
|---|---|---|
| **モジュール境界** | 認証・トークン・同意などの責務を明確に分ける | `auth` と `token` を別パッケージに分離 |
| **論理的な分離** | 同一プロセス内でも依存を制御する | `auth` が `user.Repository` インターフェース経由で呼び出し |
| **物理的な分離** | 実際のDB/プロセスを分けて強制する | `idp-auth` と `idp-user` で独立DB |
| **冪等性** | 分散後の重複・重試行を安全に処理する | `request_id` で認証履歴の2重登録を防止 |
