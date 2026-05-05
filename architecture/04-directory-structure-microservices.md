# ディレクトリ構成案：マイクロサービスアーキテクチャ編

本ドキュメントは、SSOシステムをマイクロサービス化する場合の**切り分け方のバリエーション**をまとめる。

- docs/directory-structure-microservices.md
  - 切り分け方①：ドメイン境界（DDDベース）
  - 切り分け方②：レイヤー責務（Gateway + BFF）
  - 切り分け方③：トランザクション特性（ステートフル/ステートレス分離）
  - 切り分け方④：イベント駆動型（PubSub中心）
  - 比較表と段階的進化の推奨パスを記載

---

## 前提

- **目的**: マイクロサービスアーキテクチャの設計・運用を学ぶ。
- **制約**: 個人開発・学習のため、Kubernetes等のオーケストレータは必須ではなく、Docker Composeで擬似的に再現する前提とする。
- **技術スタック**: 各サービスは原則Ruby on RailsまたはSinatra、フロントエンドはVue 3。

---

## 切り分け方①：ドメイン境界による分割（DDDベース）

ビジネスドメイン（認可・ユーザー・クライアント管理）をサービス境界とする、最も標準的な切り分け方。

```
my-sso/
├── infrastructure/                 # 共有インフラ
│   ├── nginx/
│   │   └── nginx.conf              # API Gateway
│   ├── redis/
│   │   └── redis.conf              # キャッシュ・PubSub
│   └── postgresql/
│       └── init/
│
├── services/
│   ├── idp-auth/                   # 認可・トークンサービス
│   │   ├── Dockerfile
│   │   ├── Gemfile
│   │   ├── app/
│   │   │   ├── controllers/
│   │   │   │   ├── authorize_controller.rb
│   │   │   │   ├── token_controller.rb
│   │   │   │   ├── revoke_controller.rb
│   │   │   │   └── well_known_controller.rb
│   │   │   ├── models/
│   │   │   │   ├── authorization_code.rb
│   │   │   │   ├── access_token.rb
│   │   │   │   ├── refresh_token.rb
│   │   │   │   └── id_token.rb
│   │   │   └── services/
│   │   │       ├── jwt_service.rb
│   │   │       └── pkce_service.rb
│   │   ├── config/
│   │   │   └── database.yml       # auth_db
│   │   └── db/
│   │
│   ├── idp-user/                   # ユーザー管理サービス
│   │   ├── Dockerfile
│   │   ├── app/
│   │   │   ├── controllers/
│   │   │   │   ├── users_controller.rb
│   │   │   │   └── session_controller.rb
│   │   │   ├── models/
│   │   │   │   └── user.rb
│   │   │   └── services/
│   │   │       └── password_service.rb
│   │   ├── config/
│   │   │   └── database.yml       # user_db
│   │   └── db/
│   │
│   ├── idp-client/                 # RP（クライアント）管理サービス
│   │   ├── Dockerfile
│   │   ├── app/
│   │   │   ├── controllers/
│   │   │   │   └── clients_controller.rb
│   │   │   └── models/
│   │   │       └── client.rb
│   │   ├── config/
│   │   │   └── database.yml       # client_db
│   │   └── db/
│   │
│   └── idp-admin/                  # 管理画面BFF（Backend for Frontend）
│       ├── Dockerfile
│       ├── app/
│       ├── config/
│       └── db/
│
├── web/
│   ├── idp-portal/                 # 一般ユーザー向け（ログイン・同意）
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── src/
│   │       ├── views/
│   │       │   ├── LoginView.vue
│   │       │   └── ConsentView.vue
│   │       └── api/
│   │           ├── auth.ts
│   │           └── user.ts
│   │
│   └── idp-admin-ui/               # 管理者向けポータル
│       ├── Dockerfile
│       └── src/
│           └── views/
│               └── DashboardView.vue
│
├── demo-rp/
│   └── ...
│
└── shared/
    └── ruby/
        └── sso-core/               # サービス間共有Gem
            ├── lib/
            │   ├── sso/
            │   │   ├── errors.rb
            │   │   ├── http_client.rb
            │   │   └── event.rb
            │   └── sso-core.rb
            └── sso-core.gemspec
```

### 特徴

- **Database per Service**: `auth_db`, `user_db`, `client_db` を分離。
- **API Gateway**: nginxがエンドポイント統合・ルーティングを担当。
- サービス間通信はREST（同期）を基本とし、必要に応じてRedis Pub/Subで非同期連携。

### 学習ポイント

- ドメイン境界の設計（どこまでを「認可」とし、どこまでを「ユーザー」とするか）。
- 分散トランザクションのない設計（各サービスが自己完結するデータを持つ）。

---

## 切り分け方②：レイヤー責務による分割（API Gateway + コア + BFF）

ビジネスドメインではなく、技術的責務（プロトコル変換、集約、画面表示）でサービスを分割する。

```
my-sso/
├── infrastructure/
│   ├── nginx/
│   ├── redis/
│   └── postgresql/
│
├── services/
│   ├── gateway/                    # API Gateway（エッジでの統合）
│   │   ├── Dockerfile
│   │   ├── Gemfile
│   │   ├── app/
│   │   │   └── controllers/
│   │   │       ├── authorize_proxy.rb    # ルーティング＆レートリミット
│   │   │       └── token_proxy.rb
│   │   └── config/
│   │
│   ├── idp-core/                   # ビジネスロジック集約サービス
│   │   ├── app/
│   │   │   ├── models/             # User, Client, Token全て
│   │   │   ├── services/
│   │   │   │   ├── auth_flow_service.rb
│   │   │   │   └── token_issuer_service.rb
│   │   │   └── controllers/
│   │   └── config/
│   │       └── database.yml       # core_db（単一DB）
│   │
│   ├── idp-bff-admin/              # 管理画面専用BFF
│   │   ├── app/
│   │   │   └── controllers/
│   │   │       └── admin_api_controller.rb   # idp-core集約呼び出し
│   │   └── config/
│   │
│   └── idp-bff-portal/             # エンドユーザー画面専用BFF
│       ├── app/
│       │   └── controllers/
│       │       └── portal_api_controller.rb  # ログイン・同意のAPI集約
│       └── config/
│
├── web/
│   ├── admin-ui/
│   └── portal-ui/
│
└── demo-rp/
    └── ...
```

### 特徴

- `idp-core` に全ビジネスロジックを集約し、BFFが画面に最適化したAPIを提供する。
- フロントエンドごとにBFFを設けることで、異なるデバイスやユースケースに最適化できる。
- Database per Serviceは採用せず、コアDBは1つ（学習初期の複雑性緩和）。

### 学習ポイント

- BFFパターン（Backend for Frontend）の実装。
- API Gatewayでの認証・レートリミット・ログ集約。

---

## 切り分け方③：トランザクション特性による分割（ステートフル/ステートレス分離）

データの一貫性要件やスケーリング特性を軸に分割する。

```
my-sso/
├── infrastructure/
│   ├── nginx/
│   ├── redis/                      # セッション・コード保存
│   ├── postgresql/
│   └── kafka/                      # イベントストリーム（任意）
│
├── services/
│   ├── idp-session/                # ステートフル：セッション・同意管理
│   │   ├── app/
│   │   │   ├── controllers/
│   │   │   │   └── session_controller.rb     # ログイン状態保持
│   │   │   └── models/
│   │   │       └── user_session.rb
│   │   └── config/
│   │
│   ├── idp-token/                  # ステートレス：JWT発行・検証
│   │   ├── app/
│   │   │   ├── controllers/
│   │   │   │   ├── token_controller.rb
│   │   │   │   └── jwks_controller.rb        # /jwks 公開鍵配布
│   │   │   └── services/
│   │   │       ├── jwt_issuer.rb
│   │   │       └── jwt_verifier.rb
│   │   └── config/
│   │       └── database.yml       # 最小（ログのみ）
│   │
│   ├── idp-metadata/               # ReadHeavy：設定・クライアント情報
│   │   ├── app/
│   │   │   ├── controllers/
│   │   │   │   ├── clients_controller.rb
│   │   │   │   └── discovery_controller.rb   # .well-known
│   │   │   └── models/
│   │   │       └── client.rb
│   │   └── config/
│   │       └── database.yml       # metadata_db
│   │
│   └── idp-event/                  # 非同期処理：監査ログ・失効伝播
│       ├── app/
│       │   └── consumers/
│       │       └── revocation_consumer.rb
│       └── config/
│
├── web/
│   └── portal/
│       └── src/
│           └── api/
│               ├── session.ts
│               └── token.ts
│
└── demo-rp/
    └── ...
```

### 特徴

- **idp-session**: Redisを活用したログインセッション管理。水平スケール時の整合性確保が焦点。
- **idp-token**: 完全ステートレス。DBアクセスなしでJWTを発行・検証。/jwks エンドポイントも担当。
- **idp-metadata**: クライアント設定やDiscovery情報を提供。変更頻度は低く、キャッシュ効果が高い。
- **idp-event**: トークン失効イベントの非同期伝播や監査ログ書き込みを担当。

### 学習ポイント

- ステートフルとステートレスサービスの混在設計。
- ReadHeavyサービスにおけるキャッシュ戦略（Redis等）。
- イベント駆動による最終一貫性の実現。

---

## 切り分け方④：イベント駆動型（非同期・PubSub中心）

サービス間の連携をRESTではなく、イベントバス（Redis Pub/SubまたはKafka）で行う構成。

```
my-sso/
├── infrastructure/
│   ├── nginx/
│   ├── redis/                      # Pub/Sub + ストリーム
│   ├── postgresql/
│   │   ├── user_db/
│   │   ├── auth_db/
│   │   └── audit_db/
│   └── kafka/                      # イベント永続化（オプション）
│
├── services/
│   ├── idp-auth/
│   │   ├── app/
│   │   │   ├── controllers/
│   │   │   └── publishers/         # イベント発行
│   │   │       └── token_issued_publisher.rb
│   │   └── config/
│   │
│   ├── idp-user/
│   │   ├── app/
│   │   │   ├── controllers/
│   │   │   └── subscribers/        # イベント購読
│   │   │       └── user_logged_in_subscriber.rb
│   │   └── config/
│   │
│   ├── idp-audit/                  # 監査ログサービス
│   │   ├── app/
│   │   │   └── subscribers/
│   │   │       └── all_events_subscriber.rb
│   │   └── config/
│   │       └── database.yml       # audit_db
│   │
│   └── idp-sync/                   # データ同期・投影サービス（CQRS）
│       ├── app/
│       │   └── consumers/
│       │       └── materialized_view_builder.rb
│       └── config/
│
├── web/
│   └── ...
│
└── demo-rp/
    └── ...
```

### 特徴

- サービス間は非同期イベントで疎結合。
- `idp-auth` はトークン発行イベントを発行し、`idp-audit` が非同期で記録する。
- `idp-sync` はイベントを元に集計用のマテリアライズドビューを構築（CQRSの簡易版）。

### 学習ポイント

- イベント駆動アーキテクチャ（EDA）の基礎。
- 冪等性（idemponent）なイベント処理の実装。
- 分散システムにおける「最終一貫性」と補償トランザクションの考え方。

---

## 比較と選択指針

| 切り分け方 | 学習価値 | 複雑度 | 推奨フェーズ |
|---|---|---|---|
| **①ドメイン境界** | 高（標準的なMSA設計） | 中 | Phase 3〜4 |
| **②レイヤー責務** | 中（BFF/Gatewayの理解） | 中 | Phase 3〜4 |
| **③トランザクション特性** | 高（スケーリング設計） | 高 | Phase 4〜5 |
| **④イベント駆動** | 最高（分散システムの本質） | 最高 | Phase 5以降 |

---

## 段階的進化の推奨パス

マイクロサービスは最初から全てを分離すると運用・デバッグが困難になる。
以下の順序で段階的にサービスを切り出すことを推奨する。

1. **Step 0**: 案A（モノレpo統合型）でエンドツーエンド動作を確認。
2. **Step 1**: `idp-auth` と `idp-user` を分離（切り分け方①の最小版）。
3. **Step 2**: `idp-client` を分離し、DBを分ける。
4. **Step 3**: `nginx` でAPI Gatewayを導入し、BFFパターンを試す（切り分け方②）。
5. **Step 4**: トークン検証を完全ステートレス化し、Redisでセッションを分離（切り分け方③）。
6. **Step 5**: Redis Pub/Subでイベント連携を導入し、監査ログサービスを追加（切り分け方④）。
