# Serializer 導入実装例

## 対象

`idp-user` サービスの `VerifyController` を例に、JSON レスポンスの生成を Serializer に移譲する方法。

## 現状（手組み）

`app/controllers/api/v1/auth/verify_controller.rb` でレスポンスハッシュをコントローラー内で直接構築している。

```ruby
def success_response(user)
  {
    authenticated: true,
    user: {
      id: user.id,
      email: user.email,
      name: user.name
    }
  }
end
```

## トレードオフ

| 観点 | 手組み（現状） | Serializer 導入 |
|------|--------------|----------------|
| **シンプルさ** | ファイル数が増えず、小規模なら即理解できる | Gem とファイルが増え、学習コストがかかる |
| **再利用性** | 同じモデルを別エンドポイントで返す場合、属性リストが重複・分散しやすい | `UserSerializer.new(user)` で一元管理・再利用できる |
| **機密フィールドの扱い** | 各コントローラーで除外忘れのリスクがある | ホワイトリスト方式なので、記載しない属性は自動的に出力されない |
| **出力形式の制御** | Hash を自由に組めるため柔軟 | `jsonapi-serializer` はデフォルトで JSON:API 形式（`data` → `attributes` ネスト）を返す。既存のプレーン JSON API と整合させるには追加の変換が必要 |

**判断基準**: 同一モデルを 2 箇所以上で JSON 返却する、または機密フィールドの管理が重要になる場合、Serializer の導入メリットが大きい。エンドポイントが 1 〜 2 個で構成が安定している場合は、手組みのままでも十分。

## 推奨ライブラリ

**jsonapi-serializer**（旧 fast_jsonapi）
- Rails 7.1 対応
- 軽量・高速
- 属性のホワイトリスト管理がシンプル

## 導入手順

### 1. Gemfile

```ruby
gem 'jsonapi-serializer'
```

```bash
bundle install
```

### 2. Serializer 作成

`app/serializers/user_serializer.rb`

```ruby
class UserSerializer
  include JSONAPI::Serializer

  attributes :email, :name
end
```

- ホワイトリスト方式：記載した属性のみ出力される
- `password_digest` は記載しないため自動的に除外される

### 3. コントローラー修正

`app/controllers/api/v1/auth/verify_controller.rb`

```ruby
def success_response(user)
  {
    authenticated: true,
    user: UserSerializer.new(user).serializable_hash[:data][:attributes]
                        .merge(id: user.id.to_s)
  }
end
```

**補足**: `jsonapi-serializer` はデフォルトで JSON:API 形式（`data` → `attributes` ネスト）を返す。既存 API がプレーンな JSON を期待する場合、上記のように `[:data][:attributes]` を取り出して整形する。

JSON:API 形式に統一する場合は以下のように書き換える。

```ruby
def success_response(user)
  {
    authenticated: true,
    user: UserSerializer.new(user).serializable_hash
  }
end
```

出力例（JSON:API 形式）:

```json
{
  "authenticated": true,
  "user": {
    "data": {
      "id": "1",
      "type": "user",
      "attributes": {
        "email": "test@example.com",
        "name": "Taro"
      }
    }
  }
}
```
