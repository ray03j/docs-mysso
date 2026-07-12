# feature/idp-user-service/users

## 目次

- [目的](#目的)
- [ブランチ名](#ブランチ名)
- [親ブランチ](#親ブランチ)
- [作成・変更するファイル](#作成変更するファイル)
  - [`idp-user/app/controllers/api/v1/users_controller.rb`](#idp-userappcontrollersapiv1users_controllerrb)
  - [`idp-user/app/models/user.rb`](#idp-userappmodelsuserrb)
  - [`idp-user/spec/factories/users.rb`](#idp-userspecfactoriesusersrb)
- [マージ基準（チェックリスト）](#マージ基準チェックリスト)
- [備考](#備考)
- [解説](#解説)
  - [`ApplicationController` とは](#applicationcontroller-とは)
  - [`UsersController` の責務](#userscontroller-の責務)
  - [`render` とは](#render-とは)
  - [Controller と Model の関係](#controller-と-model-の関係)

---

## 目的

`idp-user` サービスのサインアップ API（`POST /api/v1/users`）を実装し、ユーザーアカウントの作成機能を提供する。

## ブランチ名

`feature/idp-user-service/users`

## 親ブランチ

`feature/idp-user-service/init`

## 作成・変更するファイル

### `idp-user/app/controllers/api/v1/users_controller.rb`

```ruby
module Api
  module V1
    class UsersController < ApplicationController
      def create
        user = User.new(user_params)
        if user.save
          render json: { id: user.id, email: user.email, name: user.name }, status: :created
        else
          render json: { errors: user.errors.full_messages }, status: :unprocessable_content
        end
      end

      private

      def user_params
        params.require(:user).permit(:email, :password, :name)
      end
    end
  end
end
```

#### なぜこの設定か

- **`params.require(:user)`**：Strong Parameters で `user` キーの存在を必須とし、予期しないパラメータの注入を防ぐ。`permit` でホワイトリスト化し、メール・パスワード・名前のみを受け付ける。
- **`status: :created`（201）**：リソース作成成功時は HTTP 201 を返す。RESTful API の慣習に従い、クライアントが「作成完了」を正確に認識できるようにする。
- **`status: :unprocessable_content`（422）**：バリデーションエラー時は HTTP 422 を返す。これによりフロントエンドが「リクエストは受理されたが内容が不正」を判断できる。エラー内容は `full_messages` で配列形式で返し、フロントエンドでの表示を容易にする。
- **レスポンスに `password_digest` を含めない**：セキュリティ上の理由から、ハッシュ化されたパスワードは API レスポンスに絶対に含めない。作成されたユーザーの `id`, `email`, `name` のみを返す。

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

### `idp-user/spec/factories/users.rb`

```ruby
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    password { 'password123' }
    name { 'Test User' }
  end
end
```

#### なぜこの設定か

- **`sequence(:email)`**：各ファクトリで一意なメールアドレスを自動生成。`uniqueness` バリデーションがあるため、固定値だと複数の `create(:user)` で衝突する。
- **`password` 属性**：`has_secure_password` が提供する `password` 仮想属性に対応。ファクトリ作成時に平文パスワードを渡すと、保存時に自動的に `password_digest` へハッシュ化される。

## マージ基準（チェックリスト）

- [ ] サインアップ API でユーザーが作成される
- [ ] `password_digest` が bcrypt で保存される（`$2a$` prefix）
- [ ] レスポンスに `password_digest` が含まれない
- [ ] 重複メールアドレスで 422 エラーが返る
- [ ] 無効なパラメータで 422 エラーが返る
- [ ] `users_spec.rb` が全て通る
- [ ] `user_spec.rb` が全て通る
- [ ] `db/seeds.rb` が正常に動作し、テスト用ユーザー初期データが投入できる（冪等・development/test 限定）

## 備考

本ブランチではサインアップ API とモデルテストのみを実装する。ログイン検証は `feature/idp-user-service/auth-verify` で行う。

## 解説

### `ApplicationController` とは

`ApplicationController` は、この Rails アプリケーションの全コントローラの基底クラスです。`idp-user` サービスでは以下のように定義されています。

```ruby
class ApplicationController < ActionController::API
end
```

`ActionController::API` を継承することで、JSON API 向けのコントローラとして動作します。

### `UsersController` の責務

`Api::V1::UsersController` の責務は **「ユーザー登録 API リクエストを受け付け、ユーザーを作成して結果を返す」**ことです。

| 責務 | 内容 |
|------|------|
| **リクエストの受け口** | `POST /api/v1/users` を処理する |
| **パラメータの洗い出し** | `user_params` で許可された項目だけ取り出す |
| **ユーザー作成の実行** | `User.new` → `save` で保存を試みる |
| **結果の返却** | 成功・失敗に応じて JSON + HTTP ステータスを返す |

認証・認可・メール送信・トークン発行などの責務は持ちません。

### `render` とは

`render` は、Rails コントローラがクライアントに返すレスポンスを指定するメソッドです。

```ruby
render json: { id: user.id, email: user.email, name: user.name }, status: :created
```

- `json:` に指定した Hash を JSON 形式に変換して返す
- `status: :created` で HTTP ステータスコードを `201` に設定

### Controller と Model の関係

DB 操作は Model クラスが行い、Controller はその手前の受け渡しを担当します。

```ruby
def create
  user = User.new(user_params)   # Controller → Model にデータを受け渡す
  if user.save                   # Controller → Model に保存を依頼
    render json: { id: user.id, email: user.email, name: user.name }, status: :created
  else
    render json: { errors: user.errors.full_messages }, status: :unprocessable_content
  end
end
```

| 行 | 何が起きているか |
|---|---|
| `User.new(user_params)` | Controller → Model **データの受け渡し** |
| `user.save` | Controller → Model **保存処理の依頼** |

つまり、Controller は HTTP リクエストとドメインロジックの接続役であり、Model はビジネスルールと DB 操作を担当します。
