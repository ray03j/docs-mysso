# Phase1 Rails コマンドで生成されるファイル一覧

## 目次

- [1. はじめに](#1-はじめに)
- [2. `rails new`](#2-rails-new)
- [3. `rails generate model`](#3-rails-generate-model)
- [4. `rails generate controller`](#4-rails-generate-controller)
- [5. `rails generate migration`](#5-rails-generate-migration)
- [6. `rails db:migrate` / `rails db:seed`](#6-rails-dbmigrate--rails-dbseed)
- [7. RSpec / FactoryBot 関連の生成コマンド](#7-rspec--factorybot-関連の生成コマンド)
- [8. Phase1 各ブランチで作成される主なファイル](#8-phase1-各ブランチで作成される主なファイル)
- [9. 注意点：Rails コマンドで生成されないファイル](#9-注意点rails-コマンドで生成されないファイル)

---

## 1. はじめに

本ドキュメントは、Phase1 の各 feature ブランチで作成されるファイルのうち、Rails 標準コマンド（`rails new` / `rails generate` / `rails db:*` など）で生成可能なファイルをまとめたものです。

Phase1 では以下の 3 つの Rails API サービスを対象としています。

| サービス | ディレクトリ | ポート |
|---|---|---|
| `idp-auth` | `idp-auth/` | 3000 |
| `idp-user` | `idp-user/` | 3001 |
| `idp-client` | `idp-client/` | 3002 |

各サービスは `feature/project-setup` ブランチで `rails new` により雛形を作成し、後続ブランチでモデル・コントローラー・マイグレーション等を追加しています。

---

## 2. `rails new`

### 使用ブランチ

- `feature/project-setup/idp-auth`
- `feature/project-setup/idp-user`
- `feature/project-setup/idp-client`

### 実行コマンド

```bash
rails new . --api --database=postgresql --skip-test
```

### 生成される主なファイル・ディレクトリ

各サービス配下に以下のファイル群が生成されます。

| ファイル / ディレクトリ | 用途 |
|---|---|
| `Gemfile` | gem 依存定義 |
| `Gemfile.lock` | `bundle install` 後に生成されるロックファイル |
| `Rakefile` | Rake タスクのエントリポイント |
| `config.ru` | Rack サーバー起動用 |
| `config/application.rb` | Rails アプリケーション設定 |
| `config/boot.rb` | Bundler 初期化 |
| `config/environment.rb` | 環境読み込み |
| `config/environments/development.rb` | development 環境設定 |
| `config/environments/test.rb` | test 環境設定 |
| `config/environments/production.rb` | production 環境設定 |
| `config/initializers/` | 初期化ファイル置き場（空ディレクトリ） |
| `config/routes.rb` | ルーティング定義 |
| `config/database.yml` | DB 接続設定 |
| `app/controllers/application_controller.rb` | 全コントローラーの継承元 |
| `app/models/application_record.rb` | 全モデルの継承元 |
| `db/seeds.rb` | 初期データ投入スクリプト |
| `spec/spec_helper.rb` | RSpec 用グローバル設定 |

### 備考

- `--api` によりビュー関連のミドルウェア・ファイルは生成されません。
- `--skip-test` により Minitest のファイル（`test/` ディレクトリ等）は生成されません。
- 各サービスの `application.rb` では `config.api_only = true`（`idp-auth` は `init` ブランチで `false` に変更）と `config.time_zone = 'Tokyo'` を設定します。

---

## 3. `rails generate model`

### 3.1 `idp-user` サービス

#### 使用ブランチ

- `feature/database-schema/idp-user`

#### 対応コマンド例

```bash
rails generate model User email:string:uniq password_hash:string name:string
```

#### 生成される主なファイル

| ファイル | 用途 |
|---|---|
| `app/models/user.rb` | `User` モデル |
| `db/migrate/xxx_create_users.rb` | `users` テーブル作成マイグレーション |
| `spec/factories/users.rb` | FactoryBot ファクトリ（`factory_bot_rails` 有効時） |
| `spec/models/user_spec.rb` | モデルスペック |

#### 備考

- ドキュメントではマイグレーション番号を `001_create_users.rb` としていますが、`rails generate` 実行時はタイムスタンプ形式のファイル名になります。
- `email` に `unique: true` のインデックスが追加されます。

---

### 3.2 `idp-client` サービス

#### 使用ブランチ

- `feature/database-schema/idp-client`

#### 対応コマンド例

```bash
rails generate model Client \
  client_id:string:uniq \
  client_secret_hash:string \
  name:string \
  redirect_uris:text \
  allowed_scopes:text
```

#### 生成される主なファイル

| ファイル | 用途 |
|---|---|
| `app/models/client.rb` | `Client` モデル |
| `db/migrate/xxx_create_clients.rb` | `clients` テーブル作成マイグレーション |
| `spec/factories/clients.rb` | FactoryBot ファクトリ |
| `spec/models/client_spec.rb` | モデルスペック |

---

### 3.3 `idp-auth` サービス

#### 使用ブランチ

- `feature/database-schema/idp-auth`

#### 対応コマンド例

```bash
rails generate model AuthorizationCode \
  code:string:uniq \
  user_id:uuid \
  client_id:uuid \
  redirect_uri:string \
  scope:string \
  expires_at:datetime \
  used:boolean

rails generate model AccessToken \
  token:string:uniq \
  user_id:uuid \
  client_id:uuid \
  scope:string \
  expires_at:datetime

rails generate model RefreshToken \
  token:string:uniq \
  user_id:uuid \
  client_id:uuid \
  scope:string \
  expires_at:datetime

rails generate model Consent \
  user_id:uuid \
  client_id:uuid \
  scope:string
```

#### 生成される主なファイル

| ファイル | 用途 |
|---|---|
| `app/models/authorization_code.rb` | 認可コードモデル |
| `app/models/access_token.rb` | アクセストークンモデル |
| `app/models/refresh_token.rb` | リフレッシュトークンモデル |
| `app/models/consent.rb` | 同意管理モデル |
| `db/migrate/xxx_create_authorization_codes.rb` | 認可コードテーブル作成 |
| `db/migrate/xxx_create_access_tokens.rb` | アクセストークンテーブル作成 |
| `db/migrate/xxx_create_refresh_tokens.rb` | リフレッシュトークンテーブル作成 |
| `db/migrate/xxx_create_consents.rb` | 同意テーブル作成 |
| `spec/factories/authorization_codes.rb` | 認可コードファクトリ |
| `spec/factories/access_tokens.rb` | アクセストークンファクトリ |
| `spec/factories/refresh_tokens.rb` | リフレッシュトークンファクトリ |
| `spec/factories/consents.rb` | 同意ファクトリ |

#### 備考

- `consents` テーブルは `user_id` と `client_id` の複合 UNIQUE インデックスが必要です。別途 `rails generate migration` で追加します。
- ドキュメントでは追加の複合インデックス（`authorization_codes` の `[client_id, used]` 等）もマイグレーション内で定義しています。

---

## 4. `rails generate controller`

### 4.1 `idp-user` サービス

#### 使用ブランチ

- `feature/idp-user-service/users`
- `feature/idp-user-service/auth-verify`

#### 対応コマンド例

```bash
# feature/idp-user-service/users
rails generate controller api/v1/users create

# feature/idp-user-service/auth-verify
rails generate controller api/v1/auth/verify create
```

#### 生成される主なファイル

| ファイル | 用途 |
|---|---|
| `app/controllers/api/v1/users_controller.rb` | サインアップ API コントローラー |
| `app/controllers/api/v1/auth/verify_controller.rb` | ログイン検証 API コントローラー |
| `spec/requests/users_spec.rb` | サインアップリクエストスペック |
| `spec/requests/auth_verify_spec.rb` | ログイン検証リクエストスペック |

---

### 4.2 `idp-client` サービス

#### 使用ブランチ

- `feature/idp-client-service/clients`
- `feature/idp-client-service/verify`

#### 対応コマンド例

```bash
# feature/idp-client-service/clients
rails generate controller api/v1/clients index create show

# feature/idp-client-service/verify
rails generate controller api/v1/verify create
```

#### 生成される主なファイル

| ファイル | 用途 |
|---|---|
| `app/controllers/api/v1/clients_controller.rb` | RP 登録・一覧・詳細 API コントローラー |
| `app/controllers/api/v1/verify_controller.rb` | 内部検証 API コントローラー |
| `spec/requests/clients_spec.rb` | クライアント API リクエストスペック |
| `spec/requests/verify_spec.rb` | 内部検証 API リクエストスペック |

---

### 4.3 `idp-auth` サービス

#### 使用ブランチ

- `feature/idp-auth-service/session`

#### 対応コマンド例

```bash
rails generate controller api/v1/auth/session create destroy
```

#### 生成される主なファイル

| ファイル | 用途 |
|---|---|
| `app/controllers/api/v1/auth/session_controller.rb` | ログインセッション API コントローラー |
| `spec/requests/session_spec.rb` | セッション API リクエストスペック |

---

## 5. `rails generate migration`

### 使用例

- `feature/database-schema/idp-auth` で `consents` テーブルの複合 UNIQUE インデックスを追加する場合

### 対応コマンド例

```bash
rails generate migration AddUniqueIndexToConsents user_id:client_id:uniq
```

### 生成される主なファイル

| ファイル | 用途 |
|---|---|
| `db/migrate/xxx_add_unique_index_to_consents.rb` | 複合 UNIQUE インデックス追加マイグレーション |

### 備考

- ドキュメント内では `create_consents` マイグレーションに `add_index` を含めて定義していますが、分離したい場合は上記コマンドで追加できます。

---

## 6. `rails db:migrate` / `rails db:seed`

### 使用ブランチ

- `feature/database-schema/idp-user`
- `feature/database-schema/idp-client`
- `feature/database-schema/idp-auth`
- `feature/idp-client-service/init`

### 実行コマンド

```bash
rails db:migrate
rails db:seed
```

### 生成・更新されるファイル

| ファイル | 用途 |
|---|---|
| `db/schema.rb` | マイグレーション適用後の DB スキーマ定義（`rails db:migrate` で更新） |
| `db/seeds.rb` | 初期データ投入スクリプト（`rails db:seed` で実行） |

### 備考

- `db/schema.rb` は `rails db:migrate` 実行時に自動生成・更新されます。
- `db/seeds.rb` は手動で作成したスクリプトであり、`rails db:seed` により実行されます。

---

## 7. RSpec / FactoryBot 関連の生成コマンド

### 7.1 `rails generate rspec:install`

#### 使用タイミング

- `feature/project-setup/idp-auth`
- `feature/project-setup/idp-user`
- `feature/project-setup/idp-client`

`rspec-rails` gem を追加後、以下を実行します。

```bash
bundle exec rails generate rspec:install
```

#### 生成される主なファイル

| ファイル | 用途 |
|---|---|
| `.rspec` | RSpec 実行オプション |
| `spec/spec_helper.rb` | RSpec グローバル設定 |
| `spec/rails_helper.rb` | Rails 統合用 RSpec 設定 |

---

### 7.2 `rails generate factory_bot:model`

#### 使用タイミング

- `feature/idp-user-service/users`
- `feature/idp-client-service/clients`

`factory_bot_rails` gem が有効な状態で以下を実行できます。

```bash
rails generate factory_bot:model user
rails generate factory_bot:model client
```

#### 生成される主なファイル

| ファイル | 用途 |
|---|---|
| `spec/factories/users.rb` | `User` ファクトリ |
| `spec/factories/clients.rb` | `Client` ファクトリ |

---

## 8. Phase1 各ブランチで作成される主なファイル

### 8.1 `feature/project-setup/*`

| ブランチ | 主な作成ファイル |
|---|---|
| `feature/project-setup/idp-auth` | `idp-auth/` 以下の Rails 雛形一式 |
| `feature/project-setup/idp-user` | `idp-user/` 以下の Rails 雛形一式 |
| `feature/project-setup/idp-client` | `idp-client/` 以下の Rails 雛形一式 |

### 8.2 `feature/database-schema/*`

| ブランチ | 主な作成ファイル |
|---|---|
| `feature/database-schema/idp-user` | `app/models/user.rb`, `db/migrate/xxx_create_users.rb`, `db/seeds.rb` |
| `feature/database-schema/idp-client` | `app/models/client.rb`, `db/migrate/xxx_create_clients.rb`, `db/seeds.rb` |
| `feature/database-schema/idp-auth` | `app/models/authorization_code.rb`, `app/models/access_token.rb`, `app/models/refresh_token.rb`, `app/models/consent.rb`, 各マイグレーション, `db/seeds.rb` |

### 8.3 `feature/idp-user-service/*`

| ブランチ | 主な作成ファイル |
|---|---|
| `feature/idp-user-service/init` | `config/routes.rb`, `app/models/user.rb` |
| `feature/idp-user-service/users` | `app/controllers/api/v1/users_controller.rb`, `spec/requests/users_spec.rb`, `spec/models/user_spec.rb`, `spec/factories/users.rb` |
| `feature/idp-user-service/auth-verify` | `app/controllers/api/v1/auth/verify_controller.rb`, `app/services/user/password_service.rb`, `app/services/user/authentication_service.rb`, 各種 spec |

### 8.4 `feature/idp-client-service/*`

| ブランチ | 主な作成ファイル |
|---|---|
| `feature/idp-client-service/init` | `config/routes.rb`, `app/controllers/application_controller.rb`, `app/models/client.rb`, `db/seeds.rb` |
| `feature/idp-client-service/clients` | `app/controllers/api/v1/clients_controller.rb`, `spec/requests/clients_spec.rb`, `spec/models/client_spec.rb`, `spec/factories/clients.rb` |
| `feature/idp-client-service/verify` | `app/controllers/api/v1/verify_controller.rb`, `app/services/client_verification_service.rb`, `spec/requests/verify_spec.rb`, `spec/services/client_verification_service_spec.rb` |

### 8.5 `feature/idp-auth-service/*`

| ブランチ | 主な作成ファイル |
|---|---|
| `feature/idp-auth-service/init` | `config/routes.rb`, `config/application.rb`, `app/controllers/application_controller.rb`, `app/models/authorization_code.rb`, `app/models/access_token.rb`, `app/models/refresh_token.rb`, `app/models/consent.rb` |
| `feature/idp-auth-service/session` | `app/controllers/api/v1/auth/session_controller.rb`, `app/services/auth/session_service.rb`, `spec/requests/session_spec.rb`, `spec/services/auth/session_service_spec.rb` |
| `feature/idp-auth-service/token` | `app/services/auth/authorization_code_service.rb`, `app/services/auth/token_service.rb`, 各種 spec |
| `feature/idp-auth-service/oidc` | `lib/oidc/scopes.rb`, `lib/oidc/grant_types.rb`, `lib/oidc/discovery.rb`, `lib/oidc/jwt_signer.rb`, 各種 spec |

---

## 9. 注意点：Rails コマンドで生成されないファイル

以下のファイルは Rails 標準コマンドでは生成されず、手動で作成する必要があります。

| ファイル | 作成ブランチ |
|---|---|
| `app/services/user/password_service.rb` | `feature/idp-user-service/auth-verify` |
| `app/services/user/authentication_service.rb` | `feature/idp-user-service/auth-verify` |
| `app/services/client_verification_service.rb` | `feature/idp-client-service/verify` |
| `app/services/auth/session_service.rb` | `feature/idp-auth-service/session` |
| `app/services/auth/authorization_code_service.rb` | `feature/idp-auth-service/token` |
| `app/services/auth/token_service.rb` | `feature/idp-auth-service/token` |
| `lib/oidc/scopes.rb` | `feature/idp-auth-service/oidc` |
| `lib/oidc/grant_types.rb` | `feature/idp-auth-service/oidc` |
| `lib/oidc/discovery.rb` | `feature/idp-auth-service/oidc` |
| `lib/oidc/jwt_signer.rb` | `feature/idp-auth-service/oidc` |
| `config/environments/development.rb` の追記（セキュリティヘッダー等） | `feature/idp-auth-service/session` |
| `lefthook.yml`, `.rubocop.yml` 等 | `feature/project-setup/tools` |

---

## まとめ

Phase1 では主に以下の Rails コマンドを使用してファイルを生成します。

- `rails new . --api --database=postgresql --skip-test`：Rails API 雛形一式
- `rails generate model`：モデル・マイグレーション・ファクトリ
- `rails generate controller`：コントローラー・リクエストスペック
- `rails generate migration`：追加マイグレーション
- `rails db:migrate` / `rails db:seed`：スキーマ適用・初期データ投入
- `rails generate rspec:install` / `rails generate factory_bot:model`：テスト関連ファイル

サービス層や `lib/oidc/` 配下のファイルは Rails コマンドでは生成されないため、手動作成が必要です。
