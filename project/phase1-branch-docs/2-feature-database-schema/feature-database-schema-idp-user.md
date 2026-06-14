# feature/database-schema/idp-user

## 目的

`idp-user` サービスの `users` テーブルをマイグレーションで定義し、テスト用の初期データを `seeds.rb` に投入する。

## ブランチ名

`feature/database-schema/idp-user`

## 親ブランチ

`feature/database-schema/init`

## 作成・変更するファイル

### `idp-user/db/migrate/001_create_users.rb`

```ruby
class CreateUsers < ActiveRecord::Migration[7.1]
  def change
    create_table :users do |t|
      t.string :email, null: false
      t.string :password_hash, null: false
      t.string :name
      t.timestamps
    end

    add_index :users, :email, unique: true
  end
end
```

#### なぜこの設定か

- **`email` の `null: false`**：OIDC の `sub` クレームや認証フローで email をユーザー識別子として使用するため、必須項目とする。
- **`password_hash` の `null: false`**：パスワード認証を前提とするサービスのため、空のパスワードは許容しない。
- **`name` の `null: true`**：必須ではあるが、初期登録時に省略可能にするため現時点では NULL を許容。Phase 2 でバリデーション強化時に `null: false` へ移行する。
- **インデックスは同一マイグレーション内で追加**：新規テーブル作成のため strong_migrations 警告は出ない。`unique: true` で email の重複登録を DB レベルで防ぐ。
- **マイグレーション番号 `001`**：サービスごとに独立したマイグレーション管理を行うため、連番ではなくサービス固有の prefix を付ける運用も検討可能だが、現時点では Rails 標準のタイムスタンプ形式を推奨。本ドキュメントでは簡略化のため `001` を使用。

### `idp-user/app/models/user.rb`

```ruby
class User < ApplicationRecord
  validates :email, presence: true, uniqueness: true
  validates :password_hash, presence: true
end
```

#### なぜこの設定か

- **モデル雛形**：`db:seed` で `User.create!` を実行するために最低限のモデルクラスが必要。本ブランチではテーブル定義のみを行うため、パスワード関連ロジックは後続ブランチで追加する。
- **DB 制約に対応する最小限のバリデーション**：`email` は `NOT NULL` + `UNIQUE`、`password_hash` は `NOT NULL` の制約があるため、モデル側でも `presence` / `uniqueness` を設定しておく。これにより、DB 例外（`ActiveRecord::NotNullViolation` 等）を発生させず、API としての扱いやすさを向上する。
- **`ApplicationRecord` を継承**：Rails 7.1 の標準的なモデル定義。

### `idp-user/db/seeds.rb`

```ruby
require 'bcrypt'

# テスト用ユーザー（development/test のみ）
if Rails.env.development? || Rails.env.test?
  user = User.find_or_initialize_by(email: 'test@example.com')
  cost = Rails.env.test? ? BCrypt::Engine::MIN_COST : BCrypt::Engine::DEFAULT_COST
  user.assign_attributes(
    password_hash: BCrypt::Password.create('password123', cost: cost),
    name: 'Test User'
  )
  user.save!
end
```

#### なぜこの設定か

- **テスト用ユーザー**：`feature/idp-user-service` の実装前に、手動でログイン API を試す際の確認用データ。`feature/frontend-login` の結合テストでも使用する予定。
- **`BCrypt::Password.create`**：`bcrypt` gem を使用した安全なハッシュ化。平文パスワードは保存しない。
- **環境ごとのコスト切り替え**：test 環境では `BCrypt::Engine::MIN_COST` を使用し、テスト実行速度を確保する。production/development では `DEFAULT_COST` を使用し、実環境に近い挙動を確認する。

## マージ基準（チェックリスト）

- [ ] `rails db:migrate` を `idp-user/` で実行して `users` テーブルが作成される
- [ ] `rails db:seed` を `idp-user/` で実行してテスト用ユーザーが投入できる
- [ ] `strong_migrations` で警告が出ない
- [ ] `users` テーブルの `email` に UNIQUE 制約が設定されている

## 備考

本ブランチではテーブル定義のみ。バリデーションやパスワード関連ロジックは `feature/idp-user-service` で実装する。
