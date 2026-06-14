# idp-auth と idp-user の前提知識

本ファイルは、`idp-auth`（認可サービス）と `idp-user`（ユーザー認証サービス）を理解するための前提知識をまとめたものです。  
OAuth 2.0 / OpenID Connect（OIDC）の標準用語との対応、なぜ分割するのか、それぞれの責務を説明します。

---

## 1. 標準仕様における役割

OAuth 2.0 / OIDC では、以下の用語が定義されています。

| 標準用語 | 日本語 | 役割 |
|---|---|---|
| **Authorization Server** | 認可サーバー | 認可コード、アクセストークン、リフレッシュトークンを発行・検証・失効する。RP（クライアント）から直接リクエストを受ける。 |
| **OpenID Provider (OP)** | OpenID プロバイダー | OIDC における Authorization Server。ID トークン（JWT）も発行する。 |
| **Resource Server** | リソースサーバー | アクセストークンを検証し、保護されたリソース（ユーザー情報など）を提供する。本プロジェクトでは `idp-auth` の `/userinfo` が該当。 |

**重要な点**：標準仕様では、**認証（パスワード検証など）も認可（トークン発行など）も、同じ Authorization Server / OP の中で行われる**ものとして定義されています。

```
【標準的な構成（1つのサービス）】
┌──────────────────────────────────────────┐
│  OpenID Provider (OP)                    │
│  = Authorization Server                  │
│                                          │
│  - ユーザーが入力したパスワードを検証      │
│  - 認可コードを発行                       │
│  - アクセストークン / ID トークンを発行   │
│  - トークンを検証・失効                   │
└──────────────────────────────────────────┘
```

---

## 2. なぜ分割するのか

標準では1つだが、マイクロサービス化するときに `idp-auth` と `idp-user` に分割する理由は以下の通りです。

| 理由 | 説明 |
|---|---|
| **責務の分離** | 「誰が本物か（認証）」と「何が許可されるか（認可）」は別の問題。混ぜるとコードが複雑になる。 |
| **セキュリティの隔離** | パスワード情報（高センシティブ）を `idp-user` に閉じ込め、他サービスからの直接アクセスを防ぐ。 |
| **独立したスケーリング** | ログイン時（`idp-user` 負荷高）と API アクセス時（`idp-auth` 負荷高）のピークが異なる場合、別々にスケールできる。 |
| **チーム分業** | 認証基盤チームと認可基盤チームが別れている大規模組織では、自然な分割線になる。 |

```
【本プロジェクトの構成（分割）】
┌─────────────────────┐     ┌─────────────────────────┐
│  idp-user           │     │  idp-auth               │
│  （認証担当）        │◄────│  （認可サーバー相当）    │
│  - パスワード検証    │ API │  - /authorize           │
│  - ユーザー情報管理  │     │  - /token               │
│                     │     │  - トークン発行・検証    │
└─────────────────────┘     └─────────────────────────┘
        ↑                            ↑
   標準では「OPの内部」           標準では「OPそのもの」
```

**本プロジェクトの文脈では**：個人開発・学習レベルでも、ドメイン境界の理解とサービス間通信の実装練習のため、この分割を採用しています。

---

## 3. idp-auth（認可サービス）

### 3.1 標準用語との対応

| 本プロジェクト | 標準用語 |
|---|---|
| `idp-auth` | **Authorization Server** / **OpenID Provider (OP)** |

### 3.2 主な責務

| 責務 | 具体例 |
|---|---|
| **認可コードの発行・検証** | `/authorize` エンドポイント。RP からのリクエストを受け、認可コードを生成。 |
| **トークンの発行・検証・失効** | `/token` でアクセストークン・ID トークン・リフレッシュトークンを発行。`/revoke` で失効。 |
| **同意（Consent）の管理** | ユーザーが RP にどのスコープを許可したかを記録。同意画面の制御。 |
| **OIDC Discovery** | `/.well-known/openid-configuration` でエンドポイント一覧などを公開。 |
| **セッション管理** | ブラウザ Cookie でログイン状態を保持（本プロジェクトの場合）。 |

### 3.3 持つデータ（例）

```
authorization_codes  # 認可コード（一時的、短期間有効）
access_tokens        # アクセストークン（JWT、15分有効）
refresh_tokens       # リフレッシュトークン（ランダム文字列、7日有効）
consents             # 同意履歴（どのRPにどのスコープを許可したか）
```

### 3.4 知らないこと（他サービスに委譲）

- パスワードのハッシュ値 → `idp-user` に問い合わせる
- ユーザーのプロファイル詳細 → `idp-user` から取得
- RP（クライアント）の登録情報 → `idp-client` から取得

---

## 4. idp-user（ユーザー認証サービス）

### 4.1 標準用語との対応

| 本プロジェクト | 標準用語 |
|---|---|
| `idp-user` | **標準には独立した用語はない**。「OP の内部に含まれる認証エンジン」と捉える。 |

### 4.2 主な責務

| 責務 | 具体例 |
|---|---|
| **ユーザーアカウントの管理** | サインアップ、プロファイル更新、アカウント停止・削除。 |
| **パスワードの検証** | `idp-auth` からメールアドレスと平文パスワードを受け取り、合否を返す。 |
| **パスワードの安全な保存** | bcrypt などでハッシュ化してデータベースに保存。 |
| **ユーザー情報の提供** | `idp-auth` や他サービスに、ユーザー基本情報（ID、メール、名前）を提供。 |

### 4.3 持つデータ（例）

```
users  # ユーザーアカウント（id, email, password_hash, name, created_at）
```

### 4.4 知らないこと（他サービスに委譲）

- OAuth の認可コード、トークン → `idp-auth` の管轄
- `client_id` や `client_secret` の検証 → `idp-client` の管轄
- スコープの意味や同意の有無 → `idp-auth` の管轄

---

## 5. 両者の連携イメージ

### 5.1 シンプルな呼び出し関係

```
[RP] → [idp-auth] /authorize
              │
              ├──► [idp-user] POST /api/v1/auth/verify（認証問い合わせ）
              │         ↓
              │◄── 認証結果（Yes/No + ユーザー情報）
              │
              └── 認可コード発行 → [RP] へリダイレクト
```

### 5.2 実際の HTTP 通信例

**① `idp-auth` から `idp-user` へのリクエスト**

```http
POST /api/v1/auth/verify HTTP/1.1
Host: idp-user.internal:3000
Content-Type: application/json
X-Internal-API-Key: xxxxxxxx

{
  "email": "user@example.com",
  "password": "my-password"
}
```

**② `idp-user` からのレスポンス**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "authenticated": true,
  "user": {
    "id": "u-123456",
    "email": "user@example.com",
    "name": "Taro Yamada"
  }
}
```

**③ 認証失敗時**

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "authenticated": false,
  "reason": "invalid_credentials"
}
```

> `idp-auth` は **「パスワードが合っていたか」という結果だけ** を受け取る。パスワードハッシュの中身は知らない。

---

## 6. まとめ

| 比較項目 | `idp-auth`（認可） | `idp-user`（認証） |
|---|---|---|
| **標準用語** | Authorization Server / OP | （標準にはない、独自の分割） |
| **核心の問い** | 「このユーザーに代わって RP が API を呼んでいいか？」 | 「このユーザーは本物か？」 |
| **対象** | RP（クライアントアプリ） | エンドユーザー（人） |
| **持つ秘密** | client_secret、JWT 署名鍵 | パスワードハッシュ（bcrypt） |
| **RP が直接使う？** | **はい**（`/authorize`, `/token`） | **いいえ**（内部サービスのみ） |
| **公開エンドポイント** | はい（インターネット向け） | 基本的にいいえ（内部向け） |
| **本プロジェクトでの優先度** | Phase 2 で実装 | Phase 1 で実装 |

---

## 7. 関連ドキュメント

- [02-domain-boundary-design.md](../architecture/02-domain-boundary-design.md) — ドメイン境界の設計判断（なぜこの分割を選んだか）
- [04-directory-structure-microservices.md](../architecture/04-directory-structure-microservices.md) — マイクロサービス全体構成
- [requirements.md](../project/requirements.md) — プロジェクトの機能要件・非機能要件
