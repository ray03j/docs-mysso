# feature/database-schema/idp-auth

## 目的

`idp-auth` サービスの認可関連テーブル（`authorization_codes`, `access_tokens`, `refresh_tokens`, `consents`）をマイグレーションで定義し、テスト用の初期データを `seeds.rb` に投入する。

## ブランチ名

`feature/database-schema/idp-auth`

## 親ブランチ

`feature/database-schema/init`

## 作成・変更するファイル

### `idp-auth/db/migrate/001_create_authorization_codes.rb`

```ruby
class CreateAuthorizationCodes < ActiveRecord::Migration[7.1]
  def change
    create_table :authorization_codes do |t|
      t.string :code, null: false
      t.string :user_id, null: false
      t.string :client_id, null: false
      t.string :redirect_uri
      t.string :scope
      t.datetime :expires_at, null: false
      t.boolean :used, default: false, null: false
      t.timestamps
    end

    add_index :authorization_codes, :code, unique: true
    add_index :authorization_codes, [:client_id, :used]
  end
end
```

#### なぜこの設定か

- **`code` の `null: false` + UNIQUE**：認可コードは 1 回限りのランダム文字列であり、重複は許容できない。
- **`user_id` / `client_id` の `string`**：他サービスのテーブルと外部キー制約を持たない（マイクロサービス間で DB を分離しているため）。サービス間通信で整合性を担保する。
- **`expires_at` の `null: false`**：認可コードは短期間（例: 10 分）で失効する必要があるため、有効期限は必須。
- **`used` の `default: false`**：コード使用済みフラグ。OIDC では認可コードは 1 度しか使用できない（コード再利用攻撃の防止）。
- **複合インデックス `[client_id, used]`**：未使用コードの検索を高速化するため。なお、`:client_id` と `:used` は上記 `create_table` ブロック内で定義されたカラムです。

### `idp-auth/db/migrate/002_create_access_tokens.rb`

```ruby
class CreateAccessTokens < ActiveRecord::Migration[7.1]
  def change
    create_table :access_tokens do |t|
      t.string :token, null: false
      t.string :user_id, null: false
      t.string :client_id, null: false
      t.string :scope
      t.datetime :expires_at, null: false
      t.timestamps
    end

    add_index :access_tokens, :token, unique: true
    add_index :access_tokens, [:user_id, :client_id]
  end
end
```

#### なぜこの設定か

- **`token` の `null: false` + UNIQUE**：アクセストークンは JWT 文字列またはランダム文字列。重複は許容できない。
- **`expires_at` の `null: false`**：アクセストークンは短期間（例: 15 分）で失効するため必須。
- **複合インデックス `[user_id, client_id]`**：ユーザー・クライアント別のトークン検索を高速化。トークン失効（logout）時の一括削除にも利用。なお、`:user_id` と `:client_id` は上記 `create_table` ブロック内で定義されたカラムです。

### `idp-auth/db/migrate/003_create_refresh_tokens.rb`

```ruby
class CreateRefreshTokens < ActiveRecord::Migration[7.1]
  def change
    create_table :refresh_tokens do |t|
      t.string :token, null: false
      t.string :user_id, null: false
      t.string :client_id, null: false
      t.string :scope
      t.datetime :expires_at
      t.timestamps
    end

    add_index :refresh_tokens, :token, unique: true
    add_index :refresh_tokens, [:user_id, :client_id]
  end
end
```

#### なぜこの設定か

- **`expires_at` の `null: true`**：リフレッシュトークンは「回転（rotation）」戦略を採用する場合、失効期限を設けずに「使用済みフラグ」で管理する方式もある。現時点では柔軟性を持たせて NULL 許容とする。Phase 2 で方針を確定する。
- **それ以外は `access_tokens` と同様の理由**。リフレッシュトークンも同じくサービス間で DB が分離されているため `user_id` / `client_id` は string 型。
- **インデックス**：`access_tokens` と同様に、`:token` には単一 UNIQUE インデックス、`:user_id` / `:client_id` には複合インデックスを設定。これらは上記 `create_table` ブロック内で定義されたカラムです。

### `idp-auth/db/migrate/004_create_consents.rb`

```ruby
class CreateConsents < ActiveRecord::Migration[7.1]
  def change
    create_table :consents do |t|
      t.string :user_id, null: false
      t.string :client_id, null: false
      t.string :scope, null: false
      t.timestamps
    end

    add_index :consents, [:user_id, :client_id], unique: true
  end
end
```

#### なぜこの設定か

- **`user_id` + `client_id` の複合 UNIQUE**：同一ユーザー・同一クライアントの同意は 1 レコードで管理する。スコープが変更された場合は更新（UPSERT）を行う。なお、`:user_id` と `:client_id` は上記 `create_table` ブロック内で定義されたカラムです。
- **`scope` の `null: false`**：同意の対象となるスコープは必須。OIDC の同意画面では「どの情報を提供するか」をスコープ単位でユーザーに提示する。
- **同意履歴の簡易管理**：現時点では「最新の同意のみ」を保持するシンプルな設計。Phase 4 以降で監査ログ（`idp-audit`）を新設し、履歴を完全に追跡する予定。

### `idp-auth/db/seeds.rb`

```ruby
# 認可関連テーブルは運用データが主体のため、
# 本番では seeds.rb は使用しない。
# 開発・テスト環境用の最小データのみ記載。
#
# AuthorizationCode.create!(
#   code: 'test_code_12345',
#   user_id: '1',
#   client_id: 'demo_client',
#   redirect_uri: 'http://localhost:5174/callback',
#   scope: 'openid profile',
#   expires_at: 10.minutes.from_now
# )
```

#### なぜこの設定か

- **認可コード・トークンは動的生成**：seeds.rb で固定的な認可コードを投入するとセキュリティリスクになるため、コメントアウトで雛形のみ残す。
- **consents の初期データは不要**：ユーザーが初めてログインした際に動的に作成される。

## マージ基準（チェックリスト）

- [ ] `rails db:migrate` を `idp-auth/` で実行して以下のテーブルが作成される
  - [ ] `authorization_codes`
  - [ ] `access_tokens`
  - [ ] `refresh_tokens`
  - [ ] `consents`
- [ ] `strong_migrations` で警告が出ない
- [ ] 各テーブルの必須インデックス（UNIQUE / 複合）が設定されている
- [ ] `seeds.rb` は実行可能（コメントアウト済みのため空実行で通る）

## 備考

本ブランチではテーブル定義のみ。認可コード発行・検証、JWT 発行などのロジックは `feature/idp-auth-service` で実装する。
