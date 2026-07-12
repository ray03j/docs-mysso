# RSpec 基本構文とマッチャー

## 目次

1. [RSpec とは](#rspec-とは)
2. [FactoryBot とは](#factorybot-とは)
3. [Rails のバリデーションとは](#rails-のバリデーションとは)
4. [ファイル全体の構造](#ファイル全体の構造)
5. [構文解説](#構文解説)
   1. [`require 'rails_helper'`](#require-rails_helper)
   2. [`RSpec.describe User, type: :model do`](#rspecdescribe-user-type--model-do)
   3. [`describe 'バリデーション' do`](#describe-バリデーション-do)
   4. [`it '...' do`](#it--do)
6. [テスト例](#テスト例)
   1. [email、password、name が必須であるテスト](#emailpasswordname-が必須であるテスト)
   2. [email が一意であるテスト](#email-が一意であるテスト)
   3. [password がハッシュ化されるテスト](#password-がハッシュ化されるテスト)
7. [よく使う RSpec の書き方](#よく使う-rspec-の書き方)

---

## RSpec とは

RSpec（アールスペック）は、Ruby プログラムの動作をテストするためのツールです。
自然言語に近い形でテストを書くことができ、
「〜であるべき」「〜でないべき」という期待をコードで表現します。

## FactoryBot とは

FactoryBot（ファクトリーボット）は、テストで使うダミーのデータを簡単に作るためのツールです。

| メソッド | 意味 |
| --- | --- |
| `create(:user)` | ダミーのユーザーをデータベースに保存する |
| `build(:user)` | ダミーのユーザーを作るが、データベースには保存しない |

`create` は実際にデータベースにレコードを作ります。
`build` はメモリ上にだけ作り、まだ保存されていない状態です。

## Rails のバリデーションとは

バリデーションは「入力値がルールに合っているかチェックする仕組み」です。
例えば「名前は必須」「メールアドレスは他と重複してはいけない」といったルールを、
モデルに設定できます。

`user.valid?` でチェックでき、`be_valid` は「有効であること」を表します。
逆に `not_to be_valid` は「無効であること」を表します。

## ファイル全体の構造

```ruby
require 'rails_helper'

RSpec.describe User, type: :model do
  describe 'バリデーション' do
    it 'email, password, name が必須' do
      # テスト内容
    end

    it 'email は一意である' do
      # テスト内容
    end

    it 'password が present の場合、保存時にハッシュ化される' do
      # テスト内容
    end
  end
end
```

- `describe` は「テストのグループ」を作ります。
- `it` は「個別のテストケース」を作ります。
- 日本語で書かれた `'...'` の部分は、テストのタイトル（人間が読む説明）です。

## 構文解説

### `require 'rails_helper'`

RSpec を実行するときに必要な Rails の設定を読み込みます。
これにより、データベースやモデルなどが使えるようになります。

### `RSpec.describe User, type: :model do`

「`User` モデルについてテストを書くよ」という宣言です。
`type: :model` は「これはモデルのテストです」と明示しています。

### `describe 'バリデーション' do`

「バリデーションに関するテストをまとめるよ」というグループです。
複数のテストを意味のある単位で分けて整理するために使います。

### `it '...' do`

「これが 1 つのテストです」という意味です。
`it` の中に、実際に確認したい内容を書いていきます。

## テスト例

### email、password、name が必須であるテスト

```ruby
it 'email, password, name が必須' do
  user = User.new
  expect(user).not_to be_valid
  expect(user.errors[:email]).to be_present
  expect(user.errors[:password]).to be_present
  expect(user.errors[:name]).to be_present
end
```

1. `user = User.new`
   - `User` モデルの新しいインスタンスを作ります。
   - ここでは `email` も `password` も `name` も何も入れていません。

2. `expect(user).not_to be_valid`
   - `user` が「有効（valid）」でないことを期待します。
   - つまり、何も入っていないので保存できてはいけない、という確認です。

3. `expect(user.errors[:email]).to be_present`
   - `user.errors[:email]` にエラーメッセージが入っていることを確認します。
   - `be_present` は「空でないこと」を意味します。
   - `password`、`name` についても同様です。

### email が一意であるテスト

```ruby
it 'email は一意である' do
  create(:user, email: 'test@example.com')
  user = build(:user, email: 'test@example.com')
  expect(user).not_to be_valid
  expect(user.errors[:email]).to be_present
end
```

1. `create(:user, email: 'test@example.com')`
   - `test@example.com` というメールアドレスのユーザーをデータベースに作ります。

2. `user = build(:user, email: 'test@example.com')`
   - 同じメールアドレスのユーザーを新しく作ろうとします。
   - `build` なので、まだデータベースには保存されていません。

3. `expect(user).not_to be_valid`
   - 同じ `email` のユーザーは作れないので、無効であることを期待します。

4. `expect(user.errors[:email]).to be_present`
   - `email` に関するエラーが発生していることを確認します。

### password がハッシュ化されるテスト

```ruby
it 'password が present の場合、保存時にハッシュ化される' do
  user = User.new(email: 'test@example.com', password: 'password123', name: 'Test')
  user.save!
  expect(user.password_digest).to start_with('$2a$')
  expect(user.password_digest).not_to eq('password123')
end
```

1. `user = User.new(...)`
   - `email`、`password`、`name` を指定して新しいユーザーを作ります。

2. `user.save!`
   - ユーザーをデータベースに保存します。
   - `save!` の `!` は「保存に失敗したらエラーを出す」という意味です。

3. `expect(user.password_digest).to start_with('$2a$')`
   - `password_digest`（パスワードのハッシュ値）が `$2a$` で始まることを確認します。
   - `$2a$` は bcrypt というハッシュ化アルゴリズムの印です。

4. `expect(user.password_digest).not_to eq('password123')`
   - ハッシュ値が平文の `password123` と同じでないことを確認します。
   - これにより、パスワードがそのまま保存されていないことが保証されます。

## よく使う RSpec の書き方

| 書き方 | 意味 |
| --- | --- |
| `expect(x).to be_valid` | `x` が有効であること |
| `expect(x).not_to be_valid` | `x` が無効であること |
| `expect(x).to be_present` | `x` が空でないこと |
| `expect(x).not_to eq(y)` | `x` が `y` と等しくないこと |
| `expect(x).to start_with(y)` | `x` が `y` で始まること |

`expect(...).to` は「〜であることを期待する」、
`expect(...).not_to` は「〜でないことを期待する」という意味になります。
