# Phase 1 Feature ブランチ定義（サービスディレクトリ分離版）

`design.md` の **Phase 1: 基盤構築** を、各ドメインを**独立したRailsアプリディレクトリ**として分離した feature ブランチ粒度で再定義します。

> **設計変更の注記**: `design.md` では Phase 1 を「同一Railsアプリ内のモジュール分割」としていましたが、本定義では Phase 1 から `idp-auth` / `idp-user` / `idp-client` を独立したディレクトリ（＝独立したRailsアプリ雛形）として配置します。これにより Phase 2 の「`idp-auth` 切り出し」は不要となり、フェーズ定義の見直しが必要です。

---

## 前提：Phase 1 のサービス構成

各サービスは**独立したRailsアプリケーション**として、独自の `Gemfile`、`config/`、`app/`、`db/` を持ちます。

| ディレクトリ | サービス名 | 責務 | ポート例 |
|---|---|---|---|
| `idp-auth/` | `idp-auth` | 認可コード・トークン・セッション管理 | 3000 |
| `idp-user/` | `idp-user` | ユーザーアカウント・パスワード認証 | 3001 |
| `idp-client/` | `idp-client` | RP（連携アプリ）登録・管理・検証 | 3002 |
| `frontend/` | `idp-portal` | 一般ユーザー・管理者向けSPA | 5173 |
| `demo-rp/` | `demo-rp` | デモ用Relying Party | 5174 |

---

## 依存関係

```
feature/project-setup
    │
    ▼
feature/database-schema
    │
    ├──► feature/idp-user-service
    │         │
    │         ▼
    ├──► feature/idp-client-service
    │         │
    │         ▼
    ├──► feature/idp-auth-service
    │         │
    │         ▼
    └──► feature/frontend-login
```

> `idp-auth` は `idp-user` と `idp-client` の内部APIをHTTPで呼び出すため、実装上は後からでも可。ただしブランチマージ順は上記を推奨。

---

## 1. feature/project-setup

### 目的
開発環境とプロジェクト雛形を構築し、チーム全員が `docker compose up` だけで開発を始められる状態にする。

### 作成するディレクトリ・ファイル

```
my-sso/
├── compose.yml                     # PostgreSQL + 3 Rails + Vue + RP 一括起動
├── .env.example                    # 環境変数テンプレート
├── .gitignore
│
├── idp-auth/                       # 認可サービス（Rails API）
│   ├── Gemfile
│   ├── Gemfile.lock
│   ├── Rakefile
│   ├── config.ru
│   ├── Dockerfile
│   ├── app/
│   │   ├── controllers/
│   │   │   └── application_controller.rb
│   │   └── models/
│   │       └── application_record.rb
│   ├── config/
│   │   ├── application.rb
│   │   ├── boot.rb
│   │   ├── environment.rb
│   │   ├── routes.rb
│   │   ├── database.yml            # PostgreSQL接続設定（auth_db専用）
│   │   ├── environments/
│   │   │   ├── development.rb
│   │   │   ├── test.rb
│   │   │   └── production.rb
│   │   └── initializers/
│   ├── db/
│   │   └── seeds.rb
│   └── spec/
│       └── spec_helper.rb          # RSpec セットアップ
│
├── idp-user/                       # ユーザー管理サービス（Rails API）
│   ├── Gemfile
│   ├── Gemfile.lock
│   ├── Rakefile
│   ├── config.ru
│   ├── Dockerfile
│   ├── app/
│   │   ├── controllers/
│   │   │   └── application_controller.rb
│   │   └── models/
│   │       └── application_record.rb
│   ├── config/
│   │   ├── application.rb
│   │   ├── boot.rb
│   │   ├── environment.rb
│   │   ├── routes.rb
│   │   ├── database.yml            # PostgreSQL接続設定（user_db専用）
│   │   ├── environments/
│   │   │   ├── development.rb
│   │   │   ├── test.rb
│   │   │   └── production.rb
│   │   └── initializers/
│   ├── db/
│   │   └── seeds.rb
│   └── spec/
│       └── spec_helper.rb
│
├── idp-client/                     # クライアント管理サービス（Rails API）
│   ├── Gemfile
│   ├── Gemfile.lock
│   ├── Rakefile
│   ├── config.ru
│   ├── Dockerfile
│   ├── app/
│   │   ├── controllers/
│   │   │   └── application_controller.rb
│   │   └── models/
│   │       └── application_record.rb
│   ├── config/
│   │   ├── application.rb
│   │   ├── boot.rb
│   │   ├── environment.rb
│   │   ├── routes.rb
│   │   ├── database.yml            # PostgreSQL接続設定（client_db専用）
│   │   ├── environments/
│   │   │   ├── development.rb
│   │   │   ├── test.rb
│   │   │   └── production.rb
│   │   └── initializers/
│   ├── db/
│   │   └── seeds.rb
│   └── spec/
│       └── spec_helper.rb
│
├── frontend/                       # Vue 3 SPA（統合フロントエンド）
│   ├── package.json
│   ├── vite.config.ts
│   ├── index.html
│   ├── Dockerfile
│   └── src/
│       ├── main.ts
│       └── App.vue
│
├── demo-rp/                        # デモ用 Relying Party
│   ├── package.json
│   ├── vite.config.ts
│   ├── index.html
│   ├── .env.example
│   ├── Dockerfile
│   └── src/
│       ├── main.ts
│       └── App.vue
│
└── infrastructure/                 # 共有インフラ（Phase 2〜で拡張）
    ├── postgresql/
    │   └── init/
    │       └── 01_init.sql         # DB/スキーマ初期化（3DB作成）
    └── nginx/
        └── nginx.conf              # Phase 4 で API Gateway として導入（雛形）
```

### 含めるツール・設定

- **lefthook**（Gitフック管理）
- **RuboCop**（Ruby リンタ）
- **ESLint + Prettier**（フロントエンド リンタ）
- **dotenv**（環境変数管理）
- **strong_migrations**（安全なマイグレーション）
- **RSpec + FactoryBot**（テストフレームワーク）

### マージ基準（チェックリスト）

- [ ] `docker compose up` で PostgreSQL が起動する
- [ ] `docker compose up idp-auth` で Rails サーバ（ポート3000）が起動する
- [ ] `docker compose up idp-user` で Rails サーバ（ポート3001）が起動する
- [ ] `docker compose up idp-client` で Rails サーバ（ポート3002）が起動する
- [ ] `docker compose up frontend` で Vite 開発サーバ（ポート5173）が起動する
- [ ] `docker compose up demo-rp` で デモRP（ポート5174）が起動する
- [ ] RuboCop / ESLint / Prettier が各コンテナで実行できる
- [ ] 各 Rails サービスで `bundle exec rspec` が空テストで通る

---

## 2. feature/database-schema

### 目的
各サービスのDBをマルチDB設定で分離し、テーブルをマイグレーションで定義する。

### 作成するファイル

```
idp-user/
├── config/
│   └── database.yml                # user_db 専用設定
├── app/
│   └── models/
│       └── user.rb                 # バリデーション（email, password_hash）
└── db/
    ├── migrate/
    │   └── 001_create_users.rb
    ├── schema.rb                   # マイグレーション実行後に生成
    └── seeds.rb                    # 更新：テスト用ユーザー初期データ（冪等・環境限定）

idp-client/
├── config/
│   └── database.yml                # client_db 専用設定
├── app/
│   └── models/
│       └── client.rb               # バリデーション（client_id, client_secret_hash, name, ...）
└── db/
    ├── migrate/
    │   └── 001_create_clients.rb
    ├── schema.rb                   # マイグレーション実行後に生成
    └── seeds.rb                    # 更新：テスト用クライアント初期データ（冪等・環境限定）

idp-auth/
├── config/
│   └── database.yml                # auth_db 専用設定
├── app/
│   └── models/
│       ├── authorization_code.rb   # バリデーション（code, user_id, client_id, expires_at）
│       ├── access_token.rb         # バリデーション（token, user_id, client_id, expires_at）
│       ├── refresh_token.rb        # バリデーション（token, user_id, client_id）
│       └── consent.rb              # バリデーション（user_id, client_id, scope）
└── db/
    ├── migrate/
    │   ├── 001_create_authorization_codes.rb
    │   ├── 002_create_access_tokens.rb
    │   ├── 003_create_refresh_tokens.rb
    │   └── 004_create_consents.rb
    ├── schema.rb                   # マイグレーション実行後に生成
    └── seeds.rb                    # コメントアウト済み（空実行で通る）
```

### 変更するファイル

- `idp-user/Gemfile` — `pg` 追加
- `idp-client/Gemfile` — `pg` 追加
- `idp-auth/Gemfile` — `pg` 追加
- `infrastructure/postgresql/init/01_init.sql` — 3DB作成（`auth_db`, `user_db`, `client_db`）

### マージ基準（チェックリスト）

- [ ] `docker compose up postgres` で3つのDBが作成される
- [ ] `rails db:migrate` を `idp-user/` で実行して `users` テーブルが作成される
- [ ] `rails db:migrate` を `idp-client/` で実行して `clients` テーブルが作成される
- [ ] `rails db:migrate` を `idp-auth/` で実行して `authorization_codes`, `access_tokens`, `refresh_tokens`, `consents` テーブルが作成される
- [ ] strong_migrations で警告が出ない
- [ ] `idp-auth` の `db/seeds.rb` はコメントアウト済みで、空実行（`rails db:seed`）がエラーなく通る（セキュリティリスクの有無の違いのため）
- [ ] `idp-user` の `db/seeds.rb` を実行してテスト用ユーザーが投入できる（冪等・development/test 限定）
- [ ] `idp-client` の `db/seeds.rb` を実行してテスト用クライアントが投入できる（冪等・development/test 限定）

---

## 3. feature/idp-user-service

### 目的
`idp-user` サービス（ユーザーアカウント・パスワード認証）を実装する。独立したRailsアプリとして `idp-user/` に配置。

### 作成するファイル

```
idp-user/
├── app/
│   ├── controllers/
│   │   └── api/
│   │       └── v1/
│   │           ├── users_controller.rb      # POST /api/v1/users（サインアップ）
│   │           └── auth/
│   │               └── verify_controller.rb # POST /api/v1/auth/verify（ログイン検証）
│   ├── models/
│   │   └── user.rb                          # バリデーション・パスワード関連
│   └── services/
│       └── user/
│           ├── password_service.rb          # bcryptハッシュ化・検証
│           └── authentication_service.rb    # 認証フロードライバ
└── spec/
    ├── models/
    │   └── user_spec.rb
    ├── services/
    │   └── user/
    │       ├── password_service_spec.rb
    │       └── authentication_service_spec.rb
    ├── requests/
    │   ├── users_spec.rb
    │   └── auth_verify_spec.rb
    └── support/
        └── factories/
            └── users.rb                     # FactoryBot ファクトリ
```

### 変更するファイル

- `idp-user/config/routes.rb` — `/api/v1/users`, `/api/v1/auth/verify` 追加
- `idp-user/Gemfile` — `bcrypt` 追加
- `idp-user/config/application.rb` — セッション設定（APIモードで不要なら最小限）

### マージ基準（チェックリスト）

- [ ] サインアップ API でユーザーが作成され、password_hash が bcrypt で保存される
- [ ] `PasswordService` の単体テストが通る（bcrypt cost 10以上）
- [ ] `AuthenticationService` の単体テストが通る
- [ ] ログイン検証 API で正しいメール・パスワードに対して `authenticated: true` を返す
- [ ] ログイン検証 API で誤ったパスワードを拒否する
- [ ] リクエストスペックで「サインアップ → ログイン検証」が通る

---

## 4. feature/idp-client-service

### 目的
`idp-client` サービス（RP登録・管理・検証）を実装する。独立したRailsアプリとして `idp-client/` に配置。

### 作成するファイル

```
idp-client/
├── app/
│   ├── controllers/
│   │   └── api/
│   │       └── v1/
│   │           ├── clients_controller.rb         # RP登録・一覧API
│   │           └── verify_controller.rb          # client_id/secret 検証API（内部用）
│   ├── models/
│   │   └── client.rb                             # バリデーション
│   └── services/
│       └── client/
│           └── client_verification_service.rb    # client_id/secret検証ロジック
└── spec/
    ├── models/
    │   └── client_spec.rb
    ├── services/
    │   └── client/
    │       └── client_verification_service_spec.rb
    ├── requests/
    │   ├── clients_spec.rb
    │   └── verify_spec.rb
    └── support/
        └── factories/
            └── clients.rb                        # FactoryBot ファクトリ
```

### 変更するファイル

- `idp-client/config/routes.rb` — `/api/v1/clients`, `/api/v1/verify` 追加
- `idp-client/config/application.rb` — 内部API保護用 `X-Internal-API-Key` 認証の before_action 雛形追加

### マージ基準（チェックリスト）

- [ ] RPクライアント登録 API で `client_id`, `client_secret`, `redirect_uris`, `allowed_scopes` が保存できる
- [ ] `redirect_uris` と `allowed_scopes` のバリデーションが正しく機能する
- [ ] RPクライアント一覧 API で登録済みクライアントが取得できる
- [ ] 内部検証 API で `client_id` / `client_secret` の組み合わせを正しく検証できる
- [ ] `ClientVerificationService` の単体テストが通る
- [ ] 内部API用 `X-Internal-API-Key` ヘッダー認証の before_action が設置されている

---

## 5. feature/idp-auth-service

### 目的
`idp-auth` サービス（セッション管理・認可コード雛形）を実装する。独立したRailsアプリとして `idp-auth/` に配置。`idp-user` と `idp-client` の内部APIをHTTPで呼び出す。

### 作成するファイル

```
idp-auth/
├── app/
│   ├── controllers/
│   │   └── api/
│   │       └── v1/
│   │           └── auth/
│   │               └── session_controller.rb    # POST /api/v1/auth/session（ログイン・セッション発行）
│   ├── models/
│   │   ├── authorization_code.rb                # 認可コード（短期間有効）
│   │   ├── access_token.rb                      # アクセストークン（JWT、15分）
│   │   ├── refresh_token.rb                     # リフレッシュトークン（ランダム文字列、7日）
│   │   └── consent.rb                           # 同意履歴
│   ├── services/
│   │   └── auth/
│   │       ├── authorization_code_service.rb    # 認可コード発行・検証（雛形）
│   │       ├── session_service.rb               # セッション管理
│   │       └── token_service.rb                 # JWT発行・検証（雛形）
│   └── lib/
│       └── oidc/
│           ├── discovery.rb                     # OIDC Discoveryメタデータ（雛形）
│           ├── grant_types.rb                   # サポートするgrant_type一覧
│           ├── scopes.rb                        # openid, profile, email
│           └── jwt_signer.rb                    # RS256署名・検証ロジック（雛形）
└── spec/
    ├── models/
    │   ├── authorization_code_spec.rb
    │   ├── access_token_spec.rb
    │   ├── refresh_token_spec.rb
    │   └── consent_spec.rb
    ├── services/
    │   └── auth/
    │       ├── authorization_code_service_spec.rb
    │       ├── session_service_spec.rb
    │       └── token_service_spec.rb
    └── requests/
        └── session_spec.rb
```

### 変更するファイル

- `idp-auth/config/routes.rb` — `/api/v1/auth/session` 追加
- `idp-auth/config/application.rb` — セッションCookie設定
- `idp-auth/app/controllers/application_controller.rb` — セッション管理・CSRF保護設定
- `idp-auth/Gemfile` — `httparty` or `faraday`（内部API呼び出し用）追加

### マージ基準（チェックリスト）

- [ ] `SessionController` が `idp-user` の `/api/v1/auth/verify` をHTTPで呼び出して認証できる
- [ ] 認証成功時にセッションCookie（`HttpOnly`, `Secure`, `SameSite=Lax`）を発行する
- [ ] 認証失敗時にエラーを返す
- [ ] `SessionService` の単体テストが通る
- [ ] `AuthorizationCodeService` の雛形が単体テストで呼び出せる
- [ ] `idp-auth` → `idp-user` の内部API呼び出しが `X-Internal-API-Key` ヘッダー付きで行われる

---

## 6. feature/frontend-login

### 目的
フロントエンド（Vue）のログイン画面を実装し、`idp-auth` サービスと接続する。

### 作成するファイル

```
frontend/
├── src/
│   ├── router/
│   │   └── index.ts                # /login, /consent, /admin/* ルーティング
│   ├── views/
│   │   └── LoginView.vue           # エンドユーザーログイン画面
│   ├── stores/
│   │   └── auth.ts                 # 認証状態管理（Pinia）
│   ├── api/
│   │   ├── auth.ts                 # idp-auth API クライアント（/api/v1/auth/session）
│   │   └── user.ts                 # idp-user API クライアント（雛形）
│   └── composables/
│       └── useOAuth.ts             # PKCE生成・Discovery取得（雛形）
```

### 変更するファイル

- `frontend/src/App.vue` — ルータービュー配置
- `frontend/vite.config.ts` — 開発サーバのプロキシ設定（`idp-auth` への転送）
- `frontend/package.json` — `pinia`, `vue-router` 追加
- `idp-auth/config/environments/development.rb` — CORS 設定（開発環境用）
- `idp-auth/config/application.rb` — Cookie属性設定

### マージ基準（チェックリスト）

- [ ] ブラウザで `/login` にアクセスしてログイン画面が表示される
- [ ] メール・パスワード入力後、`idp-auth` の `/api/v1/auth/session` が呼ばれる
- [ ] `idp-auth` が `idp-user` を内部APIで呼び出して認証する
- [ ] 正しい認証情報でセッションCookieが発行される
- [ ] Pinia `auth` ストアでログイン状態が反映される
- [ ] 誤ったパスワードでエラーメッセージが表示される
- [ ] 開発環境で CORS エラーが出ない

---

## 補足：Phase 1 完了後のサービス分離状態

上記6ブランチを全てマージすると、以下のサービス構造が完成します。

```
my-sso/
├── idp-auth/                       # 認可サービス（ポート3000）
│   ├── app/
│   │   ├── controllers/
│   │   │   └── api/
│   │   │       └── v1/
│   │   │           └── auth/
│   │   │               └── session_controller.rb
│   │   ├── models/
│   │   │   ├── authorization_code.rb
│   │   │   ├── access_token.rb
│   │   │   ├── refresh_token.rb
│   │   │   └── consent.rb
│   │   ├── services/
│   │   │   └── auth/
│   │   │       ├── authorization_code_service.rb
│   │   │       ├── session_service.rb
│   │   │       └── token_service.rb
│   │   └── lib/
│   │       └── oidc/
│   │           ├── discovery.rb
│   │           ├── grant_types.rb
│   │           ├── scopes.rb
│   │           └── jwt_signer.rb
│   ├── config/
│   ├── db/
│   └── spec/
│
├── idp-user/                       # ユーザー管理サービス（ポート3001）
│   ├── app/
│   │   ├── controllers/
│   │   │   └── api/
│   │   │       └── v1/
│   │   │           ├── users_controller.rb
│   │   │           └── auth/
│   │   │               └── verify_controller.rb
│   │   ├── models/
│   │   │   └── user.rb
│   │   └── services/
│   │       └── user/
│   │           ├── password_service.rb
│   │           └── authentication_service.rb
│   ├── config/
│   ├── db/
│   └── spec/
│
├── idp-client/                     # クライアント管理サービス（ポート3002）
│   ├── app/
│   │   ├── controllers/
│   │   │   └── api/
│   │   │       └── v1/
│   │   │           ├── clients_controller.rb
│   │   │           └── verify_controller.rb
│   │   ├── models/
│   │   │   └── client.rb
│   │   └── services/
│   │       └── client/
│   │           └── client_verification_service.rb
│   ├── config/
│   ├── db/
│   └── spec/
│
├── frontend/                       # Vue SPA（ポート5173）
│   └── src/
│       ├── router/
│       ├── views/
│       ├── stores/
│       ├── api/
│       └── composables/
│
├── demo-rp/                        # デモRP（ポート5174）
├── compose.yml
├── .env.example
└── infrastructure/
    ├── postgresql/
    └── nginx/
```

### Phase 2 以降の再定義（案）

`design.md` では Phase 2 で「`idp-auth` 切り出し」を定義していましたが、Phase 1 から既にディレクトリ分離しているため、以下のように再定義する必要があります。

| フェーズ | 内容（修正案） |
|---|---|
| **Phase 1** | 各サービスの雛形構築・DB分離・認証フロー基盤（本定義） |
| **Phase 2** | OAuth2/OIDC エンドポイント実装（`/authorize`, `/token`, `/revoke`）+ 内部HTTP API連携の本格化 |
| **Phase 3** | JWT発行・検証・UserInfo + `/jwks.json` + デモRP完成 |
| **Phase 4** | フロントエンド分離（`idp-portal` / `idp-admin-ui`）+ nginx API Gateway 導入 |
| **Phase 5** | Redis Pub/Sub + `idp-audit` 新設 + 監査ログ + セキュリティスキャン |
