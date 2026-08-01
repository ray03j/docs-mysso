# idp-client-service のレビューしやすいコミット順

## 目次

- [推奨するコミット順](#推奨するコミット順)
- [この順が良い理由](#この順が良い理由)
- [コミットの流儀について](#コミットの流儀について)
- [コミットメッセージ案](#コミットメッセージ案)

---

## 推奨するコミット順

依存関係の下から積み上げる順。前回の idp-user-service (#20) と同じ型にすると、レビュアーが既知のパターンとして読める。

1. ✅テスト基盤
    - 対象: `idp-client/.rspec`, `spec/spec_helper.rb`, `spec/factories/clients.rb`
    - 根拠: 後続コミットのテストが全てこれに依存。factory は遅延評価なのでこの時点でも green
2. ✅Model 層
    - 対象: `app/models/client.rb`, `spec/models/client_spec.rb`
    - 根拠: ドメイン知識（ハッシュ化・配列変換・バリデーション）の土台。ここだけで `rspec spec/models` が通る
3. ✅Service 層
    - 対象: `app/services/client_verification_service.rb`, `spec/services/client_verification_service_spec.rb`
    - 根拠: Model の `client_secret_valid?` 等を利用する側。利用箇所を見て Model 設計の妥当性を確認できる
4. ✅認証基盤 + ルーティング + コントローラー
    - 対象: `app/controllers/application_controller.rb`, `config/routes.rb`, `app/controllers/api/v1/`
    - 根拠: Service を呼ぶ層。内部APIキー認証はコントローラーとセットでないと意味をなさない
5. ✅リクエストスペック
    - 対象: `spec/requests/clients_spec.rb`, `spec/requests/verify_spec.rb`
    - 根拠: 4 の API 契約の検証。実装の直後に置くと「仕様の確認」として読める
6. 付帯変更
    - 対象: `idp-client/db/seeds.rb`, `.env.example`, `demo-rp/.env.example`
    - 根拠: 本質と独立。最後にまとめるとノイズにならない

## この順が良い理由

- 各コミットが前のコミットの上に成り立つ
    - Model → Service → Controller の依存方向と一致し、「このメソッドは誰が使うのか」を後続で追える
- 実装とテストを同コミットにすると、コミット単位で `rspec` が green になる
    - レビュアーが各段階で動作確認できる
- 2 → 3 → 4 → 5 が「テスト → 実装 → その上のテスト」の繰り返しになる
    - 前回 PR (#20) と同じリズムで読める

## コミットの流儀について

- 前回 (#20) は `test: → feat:` のテスト先行コミットだった
- どちらかに統一するとよい
    - 前回の流儀に揃えるなら: 各レイヤーで test コミットを先に分ける
    - コミット単位の green を優先するなら: 実装とテストをペアにする

## コミットメッセージ案

既存スタイル（Conventional Commits + 日本語の具体的な説明）に合わせる。

### 実装+テストをペアにする場合（6 コミット）

```text
1. chore: RSpec と FactoryBot でテストを実行できる基盤を整備

2. feat: クライアントシークレットのハッシュ化と redirect_uris / allowed_scopes の配列変換・バリデーションをモデルに定義

3. feat: client_id と client_secret でクライアントを検証して結果を返すサービスクラスを定義

4. feat: 内部APIキー認証を備えたクライアントの登録・参照・検証 API エンドポイントを定義

5. test: クライアント管理・認証検証 API のリクエストスペックを追加

6. chore: デモ用クライアントのシードと環境変数の設定例を更新
```

### 前回 (#20) のテスト先行スタイルに揃える場合

2 と 3 を test / feat に分割する（5 は前回も feat の後ろで独立していたので同じ）。

```text
2a. test: クライアントのバリデーションとシークレットハッシュ化に関するテストを追加
2b. feat: クライアントシークレットのハッシュ化と redirect_uris / allowed_scopes の配列変換・バリデーションをモデルに定義

3a. test: クライアント認証検証の単体テストを追加
3b. feat: client_id と client_secret でクライアントを検証して結果を返すサービスクラスを定義
```

### 補足

- 4 をさらに細かく分けるなら、以下の 2 つに割れる
    - `feat: 内部API向けにAPIキーでリクエストを認証する仕組みを定義`（application_controller）
    - `feat: クライアントの登録・参照・検証 API エンドポイントを定義`（routes + api/v1）
- 5 は前回表記（`test: 認証情報検証 API のリクエストスペックを追加`）に揃えるなら、以下の 2 つに分けるのもあり
    - `test: クライアント登録 API のリクエストスペックを追加`
    - `test: クライアント認証検証 API のリクエストスペックを追加`
