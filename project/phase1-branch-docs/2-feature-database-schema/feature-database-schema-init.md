# feature/database-schema/init

## 目的

PostgreSQL 初期化スクリプトを更新し、3 サービス用の DB を作成する。また、各 Rails サービスの `Gemfile` に `pg` gem を追加し（`feature/project-setup` で既に含まれているが、本ブランチで明示的に確定）、マルチDB 分離の基盤を整える。

## ブランチ名

`feature/database-schema/init`

## 親ブランチ

`feature/project-setup`（`main` へのマージ完了後）

## 作成・変更するファイル

### `infrastructure/postgresql/init/01_init.sql`

```sql
CREATE DATABASE auth_db;
CREATE DATABASE user_db;
CREATE DATABASE client_db;
```

#### なぜこの設定か

- **3 DB 分離**：`idp-auth` / `idp-user` / `idp-client` を独立した DB で運用し、サービス間の影響を排除する。スキーマ分離ではなく DB 分離を採用する理由は、バックアップ・リストア・権限管理をサービス単位で行いやすくするため。
- **初期化スクリプト**：PostgreSQL の Docker イメージは `/docker-entrypoint-initdb.d` に配置されたファイルを辞書順に実行するため、`01_init.sql` で最初に実行される。`feature/project-setup` で既に雛形が作成されているが、本ブランチで内容を確定させる。

### `idp-user/Gemfile`

`gem 'pg', '~> 1.5'` を追加（`feature/project-setup` で既に含まれている場合は変更なし）。

#### なぜこの設定か

- **`pg '~> 1.5'`**：PostgreSQL 16 と公式アダプタの組み合わせで検証されているバージョン。Rails 7.1 のデフォルト推奨。`feature/project-setup` の雛形作成時に既に追加済みのため、本ブランチでは存在確認を行う。

### `idp-client/Gemfile`

`gem 'pg', '~> 1.5'` を追加（`feature/project-setup` で既に含まれている場合は変更なし）。

### `idp-auth/Gemfile`

`gem 'pg', '~> 1.5'` を追加（`feature/project-setup` で既に含まれている場合は変更なし）。

### `idp-user/config/database.yml`

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  host: <%= ENV.fetch("DATABASE_HOST") { "postgres" } %>
  username: <%= ENV.fetch("POSTGRES_USER") %>
  password: <%= ENV.fetch("POSTGRES_PASSWORD") %>

development:
  <<: *default
  database: user_db

test:
  <<: *default
  database: user_db_test

production:
  <<: *default
  database: user_db
```

#### なぜこの設定か

- `feature/project-setup` で作成済みの `database.yml` を本ブランチで確認・確定する。`user_db` / `user_db_test` を明示し、他サービスとの分離を保証する。

### `idp-client/config/database.yml`

`database: client_db` / `client_db_test` を確認・確定。

### `idp-auth/config/database.yml`

`database: auth_db` / `auth_db_test` を確認・確定。

## マージ基準（チェックリスト）

- [ ] `docker compose up postgres` で 3 つの DB（`auth_db`, `user_db`, `client_db`）が作成される
- [ ] 各 Rails サービスの `Gemfile` に `pg` が含まれている
- [ ] 各 Rails サービスの `database.yml` で正しい DB 名が設定されている

## 備考

本ブランチでは「DB 接続の前提整備」に留める。テーブル作成は後続ブランチで行う。
