# feature/idp-user-service/auth-verify

## 目次

- [目的](#目的)
- [ブランチ名](#ブランチ名)
- [親ブランチ](#親ブランチ)
- [作成・変更するファイル一覧](#作成変更するファイル一覧)
- [ユーザーモデル](#ユーザーモデル)
- [パスワードサービス](#パスワードサービス)
- [認証サービス](#認証サービス)
- [認証コントローラー](#認証コントローラー)
- [マージ基準（チェックリスト）](#マージ基準チェックリスト)
- [備考](#備考)

---

## 目的

`idp-user` サービスのログイン検証 API（`POST /api/v1/auth/verify`）と認証サービス層を実装する。`idp-auth` からの内部API呼び出しに応答し、ユーザー認証の可否を判定する。

## ブランチ名

`feature/idp-user-service/auth-verify`

## 親ブランチ

`feature/idp-user-service/init`

## 作成・変更するファイル一覧

| # | ファイルパス | 内容 |
|---|---|---|
| 1 | `idp-user/app/models/user.rb` | ユーザーモデル（`has_secure_password`） |
| 2 | `idp-user/app/services/user/password_service.rb` | パスワードハッシュ化・検証サービス |
| 3 | `idp-user/app/services/user/authentication_service.rb` | 認証サービス |
| 4 | `idp-user/app/controllers/api/v1/auth/verify_controller.rb` | ログイン検証 API コントローラー |
| 5 | `idp-user/spec/services/user/password_service_spec.rb` | パスワードサービスの単体テスト |
| 6 | `idp-user/spec/services/user/authentication_service_spec.rb` | 認証サービスの単体テスト |
| 7 | `idp-user/spec/requests/auth_verify_spec.rb` | ログイン検証 API のリクエストスペック |

---

## ユーザーモデル

### ファイルパス

`idp-user/app/models/user.rb`

### コード

```ruby
class User < ApplicationRecord
  has_secure_password

  validates :email, presence: true, uniqueness: true
  validates :name, presence: true, length: { maximum: 100 }
end
```

### なぜこの設定か

- **`class User < ApplicationRecord`（1行目）**：Rails の Active Record パターンに従い、`users` テーブルと1対1でマッピングする。これにより DB 操作をオブジェクト指向で記述でき、マイグレーションとの整合性を自動的に保つ。

- **`has_secure_password`（2行目）**：Rails 標準のパスワードハッシュ化機能。bcrypt によるハッシュ化、`password` / `password_confirmation` 仮想属性、存在バリデーション、`authenticate` メソッドを1行で導入する。Rails コア機能として長年の実績とセキュリティレビューを受けており、独自実装では生じうる「バリデーションとコールバックの競合」「意図しない二重ハッシュ化」などのリスクを回避できる。

- **`validates :email, presence: true, uniqueness: true`（4行目）**：メールアドレスはログイン ID として機能するため、必須かつ一意である必要がある。`presence` で空文字・nil を防ぎ、`uniqueness` で重複登録を防ぐ。DB レベルの UNIQUE インデックスとセットでモデル層でも保証し、レースコンディションによる重複を早期に検出する。

- **`validates :name, presence: true, length: { maximum: 100 }`（5行目）**：ユーザー表示名を必須とし、長さを100文字に制限。`feature/database-schema` では `name` は `null: false` と定義しており、モデル層の `presence: true` とDB層の `NOT NULL` 制約で二重化している。DB ストレージの無駄遣いと UI 表示崩れを防ぐため長さも制限する。

#### `has_secure_password` とサービス層の両立

パスワード処理をサービス層に切り出したい場合、`has_secure_password` と `PasswordService` は併用可能である。例えば `authenticate` メソッドをオーバーライドし、内部で `User::PasswordService.verify?` を呼び出すことで、標準機能の利便性とハッシュ化方式の一元管理を両立できる。

---

## パスワードサービス

### ファイルパス

`idp-user/app/services/user/password_service.rb`

### コード

```ruby
require 'bcrypt'

class User
  class PasswordService
    def self.hash(password)
      BCrypt::Password.create(password)
    end

    def self.verify?(password, password_digest)
      return false if password.blank? || password_digest.blank?

      BCrypt::Password.new(password_digest) == password
    end
  end
end
```

### なぜこの設定か

- **クラスメソッドとして定義**：`PasswordService` は状態を持たない純粋関数の集合。インスタンス化の必要がないため、クラスメソッドでシンプルに実装する。
- **`BCrypt::Password.create`**：コスト係数を自動調整した安全なハッシュ化。`User` モデルでは `has_secure_password` が標準のハッシュ化を担うが、マイクロサービス間で認証ロジックを共通化し、Rails 固有の機能に依存しないようにするため、サービス層にも `bcrypt` のラッパーを置く。`idp-user` だけでなく、将来の `idp-client` の `client_secret` 検証でも同様の `bcrypt` ロジックを使い回せる。
- **`verify?` のガード節**：`password` または `password_digest` が空の場合は即座に `false` を返す。`BCrypt::Password.new(nil)` を呼ぶと `BCrypt::Errors::InvalidHash` が発生するため、事前に防ぐ。Ruby の慣習に従い、真偽値を返すメソッド名は `?` で終わるようにする。

---

## 認証サービス

### ファイルパス

`idp-user/app/services/user/authentication_service.rb`

### コード

```ruby
class User
  class AuthenticationService
    def self.authenticate(email, password)
      user = User.find_by(email: email)
      return nil unless user

      return user if PasswordService.verify?(password, user.password_digest)

      nil
    end
  end
end
```

### なぜこの設定か

- **タイミング攻撃対策の注記**：現時点では「ユーザーが存在しない場合は即座に `nil` を返す」ため、メールアドレスの存在有無による処理時間の差が生じる。これはタイミング攻撃のリスクを残す。ただし、本ブランチでは「認証ロジックの基本構造を確立すること」を優先し、厳密なタイミング攻撃対策（「存在しないダミーユーザーで `BCrypt::Password.new` を実行して処理時間を均一化する」方式）は Phase 2 で実装する予定。現時点では「ユーザーの有無とパスワード検証の順序を固定化」することで、コードの可読性を保つ。
- **戻り値は `User` または `nil`**：認証成功時はユーザー情報を返し、失敗時は `nil` を返す。Boolean ではなく User オブジェクトを返すことで、`idp-auth` から呼び出した際に「認証成功 + ユーザー情報取得」を一気に行える。

---

## 認証コントローラー

### ファイルパス

`idp-user/app/controllers/api/v1/auth/verify_controller.rb`

### コード

```ruby
module Api
  module V1
    module Auth
      class VerifyController < ApplicationController
        def create
          user = User::AuthenticationService.authenticate(
            params[:email],
            params[:password]
          )

          if user
            render json: success_response(user), status: :ok
          else
            render json: { code: 'INVALID_CREDENTIALS' }, status: :unauthorized
          end
        end

        private

        def success_response(user)
          {
            user: {
              id: user.id,
              email: user.email,
              name: user.name
            }
          }
        end
      end
    end
  end
end
```

### なぜこの設定か

- **`params[:email]` / `params[:password]`**：同じ idp-user サービス内で、サインアップ API は user キーでネストしているのに対し、ログイン検証は「単一操作」であるため、ネストされたパラメータ（`user: { email: ... }`）ではなく、フラットな構造を採用する。これにより `idp-auth` からの内部API呼び出し時に JSON ボディをシンプルに保てる。`users` ブランチのサインアップ API（`POST /api/v1/users`）では `user` キーでネストしているが、ログイン検証は独立した操作であり、パラメータ構造を簡潔に保つ。
- **`success_response(user)`**：認証成功時の JSON レスポンスを生成する private ヘルパーメソッド。コントローラーアクションからレスポンス構造を切り出し、可読性と再利用性を高める。戻り値は `id` / `email` / `name` のみに絞り、機密情報は含めない。
- **`private` に置く理由**：`success_response` は外部から直接呼ばれるべきではないコントローラー内部のヘルパー。Rails の慣習に従い、クラス外からの呼び出しを防ぐため `private` 配下に定義する。これによりアクションと内部ロジックの責務を明確に分離できる。
- **`HTTP ステータス + 機械可読な code`**：認証成功時は `200 OK` + ユーザー情報、認証失敗時は `401 Unauthorized` + `code: INVALID_CREDENTIALS` を返す。`idp-auth` 側は HTTP ステータスだけで判定でき、詳細な理由が必要な場合は `code` を参照できる。
- **`status: :unauthorized`（401）**：認証失敗時は HTTP 401 を返す。RFC 7235 に準拠し、クライアントに「認証が必要である」を明確に伝える。ただし、セキュリティ上の理由から「メールアドレスは存在するがパスワードが違う」場合と「メールアドレスが存在しない」場合を区別しない（タイミング攻撃対策）。
- **レスポンスに `password_digest` を含めない**：ユーザー情報の一部として `password_digest` は絶対に含めない。`idp-auth` 側で誤ってログに出力するリスクも防ぐ。

---

## マージ基準（チェックリスト）

- [ ] `PasswordService` の単体テストが通る（bcrypt cost 10以上）
- [ ] `AuthenticationService` の単体テストが通る
- [ ] ログイン検証 API で正しいメール・パスワードに対して `200 OK` + ユーザー情報を返す
- [ ] ログイン検証 API で誤ったパスワードを拒否する（401 + `code: INVALID_CREDENTIALS`）
- [ ] ログイン検証 API で存在しないメールも拒否する（401 + `code: INVALID_CREDENTIALS`）
- [ ] レスポンスに `password_digest` が含まれない
- [ ] リクエストスペックで「正しい認証 → 誤った認証 → 存在しないメール」が通る
- [ ] `db/seeds.rb` が正常に動作し、テスト用ユーザー初期データが投入できる（冪等・development/test 限定）

## 備考

本ブランチではログイン検証 API とサービス層を実装する。サインアップ API は `feature/idp-user-service/users` で実装済み。

`idp-auth` からの内部API呼び出しは、後続の `feature/idp-auth-service` ブランチで実装される。本ブランチでは `idp-user` 単体での動作を保証する。
