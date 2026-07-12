# feature/idp-user-service/init

## 目次

- [feature/idp-user-service/init](#featureidp-user-serviceinit)
  - [目次](#目次)
  - [目的](#目的)
  - [ブランチ名](#ブランチ名)
  - [親ブランチ](#親ブランチ)
  - [作成・変更するファイル](#作成変更するファイル)
    - [`idp-user/Gemfile`](#idp-usergemfile)
    - [`idp-user/config/routes.rb`](#idp-userconfigroutesrb)
    - [`idp-user/config/application.rb`](#idp-userconfigapplicationrb)
    - [`idp-user/app/models/user.rb`](#idp-userappmodelsuserrb)
  - [マージ基準（チェックリスト）](#マージ基準チェックリスト)
  - [備考](#備考)

---

## 目的

`idp-user` サービスにユーザー認証に必要な共通設定（`bcrypt` gem、ルーティング、モデル強化）を導入し、後続ブランチで API エンドポイントを実装できる基盤を整える。

## ブランチ名

`feature/idp-user-service/init`

## 親ブランチ

`feature/database-schema`（`main` へのマージ完了後）

## 作成・変更するファイル

### `idp-user/Gemfile`

```ruby
# 既存の gem に追加
gem 'bcrypt', '~> 3.1'
```

#### なぜこの設定か

- **`feature/project-setup` で既に追加済み**：`idp-user/Gemfile` は `feature/project-setup/idp-user` ブランチで作成されており、本ブランチでは存在確認のみを行う。マイクロサービス間で gem セットを統一することで、開発環境構築手順・Dockerfile・CI 設定を共通化できる。
- **`bcrypt '~> 3.1'`**：パスワードの安全なハッシュ化に必須。`bcrypt` はアダプティブハッシュ化アルゴリズムであり、コスト係数を調整することで計算機の性能向上に対応できる。Rails エコシステムで最も広く採用されており、セキュリティ監査でも標準的に認められている。

### `idp-user/config/routes.rb`

```ruby
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :users, only: [:create]
      namespace :auth do
        post 'verify', to: 'verify#create'
      end
    end
  end
end
```

#### なぜこの設定か

- **`namespace :api` + `namespace :v1`**：API バージョニングの標準的な構成。将来の v2 追加時に互換性を保ちながら移行できる。`idp-user` は内部マイクロサービスとして他サービスから呼ばれるため、明確なバージョン管理が重要。
- **`resources :users, only: [:create]`**：サインアップのみを提供し、ユーザー情報の取得・更新・削除は `idp-auth` や管理画面経由で行うため、原則として `create` のみを公開。最小権限の原則に従う。
- **`post 'verify'`**：ログイン検証は「状態を変更しないが機密情報を扱う」操作のため、GET ではなく POST を採用。パスワードをクエリパラメータに含めないようにし、HTTP ボディでの安全な送信を保証する。

### `idp-user/config/application.rb`

```ruby
require_relative 'boot'
require 'rails/all'

module IdpUser
  class Application < Rails::Application
    config.load_defaults 7.1
    config.api_only = true
    config.time_zone = 'Tokyo'
  end
end
```

#### なぜこの設定か

- **`feature/project-setup` で既に作成済み**：`idp-user/config/application.rb` は `feature/project-setup/idp-user` ブランチで作成されており、本ブランチでは変更なし。`idp-user` は内部マイクロサービスとして原則 API モードを維持し、セッション管理は `idp-auth` サービスで行う。`idp-user` 自体はステートレスな API 応答を提供する。
- **`require_relative 'boot'`**：現在のファイル（`application.rb`）からの相対パスで `boot.rb` を読み込む。`require` は `$LOAD_PATH` を基準に解決するのに対し、`require_relative` は実行中のファイル位置から解決するため、Rails アプリケーションの設定ファイル同士の依存関係を確実に解決できる。gem やライブラリの読み込み順序に依存せず、`config/boot.rb` が正しく読み込まれる。

### `idp-user/app/models/user.rb`

```ruby
class User < ApplicationRecord
  has_secure_password

  validates :email, presence: true, uniqueness: true
  validates :name, presence: true, length: { maximum: 100 }
end
```

#### なぜこの設定か

- **`class User < ApplicationRecord`（1行目）**：Rails の Active Record パターンに従い、`users` テーブルと1対1でマッピングする。これにより DB 操作をオブジェクト指向で記述でき、マイグレーションとの整合性を自動的に保つ。

- **`has_secure_password`（2行目）**：Rails 標準のパスワードハッシュ化機能。bcrypt によるハッシュ化、`password` / `password_confirmation` 仮想属性、存在バリデーション、`authenticate` メソッドを1行で導入する。Rails コア機能として長年の実績とセキュリティレビューを受けており、独自実装では生じうる「バリデーションとコールバックの競合」「意図しない二重ハッシュ化」などのリスクを回避できる。

- **`validates :email, presence: true, uniqueness: true`（4行目）**：メールアドレスはログイン ID として機能するため、必須かつ一意である必要がある。`presence` で空文字・nil を防ぎ、`uniqueness` で重複登録を防ぐ。DB レベルの UNIQUE インデックスとセットでモデル層でも保証し、レースコンディションによる重複を早期に検出する。

- **`validates :name, presence: true, length: { maximum: 100 }`（5行目）**：ユーザー表示名を必須とし、長さを100文字に制限。`feature/database-schema` では `name` は `null: false` と定義しており、モデル層の `presence: true` とDB層の `NOT NULL` 制約で二重化している。DB ストレージの無駄遣いと UI 表示崩れを防ぐため長さも制限する。

**`has_secure_password` とサービス層の両立**

パスワード処理をサービス層に切り出したい場合、`has_secure_password` と `PasswordService` は併用可能である。例えば `authenticate` メソッドをオーバーライドし、内部で `User::PasswordService.verify?` を呼び出すことで、標準機能の利便性とハッシュ化方式の一元管理を両立できる。

## マージ基準（チェックリスト）

- [ ] `idp-user/Gemfile` に `bcrypt` が含まれている
- [ ] `bundle install` が成功する
- [ ] `idp-user/config/routes.rb` に `/api/v1/users` と `/api/v1/auth/verify` が定義されている
- [ ] `rails routes` で上記ルートが確認できる
- [ ] `User` モデルに `has_secure_password` が設定されている
- [ ] `User` モデルのバリデーション（`email` の `presence` / `uniqueness`、`name` の `presence` / `length`）が正しく機能する

## 備考

本ブランチでは「認証基盤の前提整備」に留める。API エンドポイントとサービス層の実装は後続ブランチで行う。
