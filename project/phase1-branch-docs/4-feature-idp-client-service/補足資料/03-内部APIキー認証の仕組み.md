# 内部APIキー認証の仕組み

## 目次

- [背景](#背景)
- [対象コード](#対象コード)
- [各メソッドの役割](#各メソッドの役割)
- [before_action の継承と if: 条件の評価](#before_action-の継承と-if-条件の評価)
- [リクエスト処理の流れ](#リクエスト処理の流れ)
- [設計パターンとしての位置づけ](#設計パターンとしての位置づけ)

---

## 背景

`ApplicationController` に設置した次の1行について、「`VerifyController` の `internal_api?` は関係あるのか？」という疑問への回答として、仕組みを整理する。

```ruby
before_action :require_internal_api_key, if: :internal_api?
```

結論として、**`VerifyController` の `internal_api?` は直接関係する**。`if:` の条件は実行時に実際のコントローラインスタンス上で評価されるため、子クラスのオーバーライドがそのまま発動可否を決める。

## 対象コード

`ApplicationController`（`idp-client/app/controllers/application_controller.rb`）:

```ruby
class ApplicationController < ActionController::API
  before_action :require_internal_api_key, if: :internal_api?

  private

  def require_internal_api_key
    return if request.headers['X-Internal-API-Key'] == internal_api_key

    render json: { error: 'Unauthorized' }, status: :unauthorized
  end

  def internal_api?
    false
  end

  def internal_api_key
    ENV.fetch('INTERNAL_API_KEY')
  end
end
```

`VerifyController`（`idp-client/app/controllers/api/v1/verify_controller.rb`）:

```ruby
class VerifyController < ApplicationController
  private

  def internal_api?
    true # ApplicationController の false をオーバーライド
  end
end
```

## 各メソッドの役割

- **`require_internal_api_key`**：認証チェックの本体。リクエストヘッダ `X-Internal-API-Key` の値が `internal_api_key` と一致しない場合に 401 Unauthorized を返して処理を中断する。一致した場合は何もせずアクションへ進む。
- **`internal_api?`**：認証フィルタを適用するかどうかの判定メソッド。`before_action` の `if:` 条件として使われる。`ApplicationController` ではデフォルト `false`（認証不要の公開API）を返し、保護したいコントローラが `true` を返すようオーバーライドする（`ClientsController#show`、`VerifyController` など）。
- **`internal_api_key`**：期待されるキー値を環境変数 `INTERNAL_API_KEY` から取得する。`ENV['...']` ではなく `fetch` を使うことで、未設定時に `nil` ではなく `KeyError` を発生させ、空文字比較による認証バイパスを防ぐ。

## before_action の継承と if: 条件の評価

### 1. before_action は継承される

`ApplicationController` に定義したフィルタは、全ての子コントローラ（`VerifyController` を含む）に自動で引き継がれる。子クラス側に `before_action` を改めて書く必要はない。

### 2. if: の条件は子クラスのメソッドで評価される

Rails はリクエストを処理するたびに「このコントローラの `internal_api?`」を呼ぶ。Ruby のメソッド探索（オーバーライドが優先される）に従うため：

- `VerifyController` へのリクエスト → オーバーライドされた `internal_api?` が `true` を返す → **`require_internal_api_key` が発動**
- オーバーライドしていないコントローラ → デフォルトの `false` → フィルタはスキップ

つまり、継承した `before_action` が実際に動くかどうかは、**子クラスが `internal_api?` をどう定義しているかで決まる**。

### 役割分担のイメージ

```text
ApplicationController: before_action 登録 + internal_api? = false（雛形）
        ↓ 継承
VerifyController:      internal_api? = true に差し替え（有効化スイッチ）
```

- `before_action` 1行 = 「仕組みの設置」
- 各コントローラの `internal_api?` オーバーライド = 「スイッチON」

## リクエスト処理の流れ

1. `before_action` が `internal_api?` を評価する
2. `true` の場合のみ `require_internal_api_key` が実行される
3. ヘッダの値と `internal_api_key`（環境変数）を比較する
4. 一致すればアクションへ進み、不一致なら 401 を返す

## 設計パターンとしての位置づけ

この構成は **Template Method パターン** の適用例である。

- 親クラス（`ApplicationController`）が処理の骨格（認証フィルタ）を定義する
- 変動部分（認証をかけるかどうか）をフックメソッド（`internal_api?`）として切り出す
- 子クラスはフックメソッドのオーバーライドだけで振る舞いを変えられる

設計上の根拠は `phase1-branch-docs/4-feature-idp-client-service/01_feature-idp-client-service-init.md` の「なぜこの設定か」も参照。
