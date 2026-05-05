# ディレクトリ構成案：モノレポ（単一アプリケーション）編

本ドキュメントは、SSOシステムを単一リポジトリ・単一または複数デプロイ単位で構成する場合のディレクトリ構成案をまとめる。

- docs/directory-structure-monolithic.md
  - 案A：モノレポ統合型（推奨）
  - 案B：API分離型
  - 案C：Rails主体・最小Vue型
  - 各案の特徴・比較表・選定指針を記載

---

## 前提

- **言語**: Ruby
- **フレームワーク**: Ruby on Rails（API + SSR管理画面）
- **フロントエンド**: Vue 3（Composition API）
- **データベース**: PostgreSQL
- **永続化**: RDBMSのみ（キャッシュ等は要件外）
- **目的**: 学習・個人開発。セットアップの容易さとデバッグのしやすさを重視。

---

## 案A：モノレポ統合型（推奨）

RailsとVueを同一リポジトリで管理し、Docker Composeで一括起動する構成。
学習初期の認知負荷が最も低く、エンドツーエンドのフローをすぐに動作確認できる。

```
my-sso/
├── AGENTS.md
├── README.md
├── requirements.md
├── .env.example
├── docker-compose.yml              # PostgreSQL + Rails + Vue 一括起動
│
├── backend/                        # Ruby on Rails（IdP本体）
│   ├── Gemfile
│   ├── Gemfile.lock
│   ├── Rakefile
│   ├── config.ru
│   ├── app/
│   │   ├── controllers/
│   │   │   ├── api/
│   │   │   │   ├── v1/
│   │   │   │   │   ├── auth_controller.rb      # /authorize, /token
│   │   │   │   │   ├── userinfo_controller.rb  # /userinfo
│   │   │   │   │   └── revoke_controller.rb    # /revoke
│   │   │   │   └── well_known_controller.rb    # /.well-known/openid-configuration
│   │   │   └── admin/                          # SSR管理画面
│   │   │       ├── dashboard_controller.rb
│   │   │       ├── users_controller.rb
│   │   │       └── clients_controller.rb
│   │   ├── models/
│   │   │   ├── user.rb
│   │   │   ├── client.rb                       # RP登録情報
│   │   │   ├── authorization_code.rb
│   │   │   ├── access_token.rb
│   │   │   ├── refresh_token.rb
│   │   │   └── id_token.rb
│   │   ├── services/
│   │   │   ├── jwt_service.rb                  # JWT発行・検証
│   │   │   ├── pkce_service.rb
│   │   │   └── session_service.rb
│   │   ├── views/                              # 管理画面のERB
│   │   └── assets/
│   ├── config/
│   │   ├── routes.rb
│   │   ├── database.yml
│   │   └── initializers/
│   ├── db/
│   │   ├── migrate/
│   │   └── seeds.rb
│   ├── lib/
│   │   └── oauth2/                             # OIDCプロトコル実装
│   │       ├── grant_types.rb
│   │       └── scopes.rb
│   └── test/
│
├── frontend/                       # Vue 3（管理画面・同意画面のSPA化）
│   ├── package.json
│   ├── vite.config.ts
│   ├── index.html
│   ├── src/
│   │   ├── main.ts
│   │   ├── App.vue
│   │   ├── router/
│   │   ├── views/
│   │   │   ├── LoginView.vue
│   │   │   ├── ConsentView.vue
│   │   │   └── AdminView.vue
│   │   ├── components/
│   │   ├── stores/                 # Pinia
│   │   └── api/                    # Rails API クライアント
│   └── public/
│
└── demo-rp/                        # デモ用Relying Party
    ├── package.json
    ├── vite.config.ts
    ├── src/
    │   ├── main.ts
    │   ├── App.vue
    │   └── oauth2/                 # OAuth2/OIDCクライアント実装
    │       ├── authorize.ts
    │       ├── callback.ts
    │       └── userinfo.ts
    └── .env.example
```

### 特徴

- `backend/` がAPIサーバーおよびSSR管理画面を兼ねる。
- `frontend/` はVue SPAとしてビルドし、Railsの `public/` 配下または別ポートで配信してもよい。
- `demo-rp/` はデバッグ・動作確認用の独立したアプリケーション。
- データベースは1つのPostgreSQLインスタンスで複数DBまたはスキーマを使い分けてもよい。

---

## 案B：API分離型（フロントエンド完全分離）

Railsを純粋なAPIモードに特化し、フロントエンドは完全に分離する。
本番環境でのNginxやCORSの学習にも適する。

```
my-sso/
├── docker-compose.yml
│
├── idp-api/                        # Rails APIモードのみ
│   ├── app/
│   │   ├── controllers/api/v1/
│   │   ├── models/
│   │   ├── services/oauth/
│   │   └── serializers/            # JSONレスポンス整形
│   ├── config/
│   └── db/
│
├── idp-web/                        # Vue 3 + Vite（完全SPA）
│   ├── src/
│   │   ├── views/
│   │   │   ├── LoginPage.vue
│   │   │   ├── ConsentPage.vue
│   │   │   └── AdminDashboard.vue
│   │   └── composables/useAuth.ts
│   └── nginx.conf                  # 本番用リバースプロキシ設定
│
└── demo-rp/
    └── ...
```

### 特徴

- RailsはAPIモード（`rails new --api`）で軽量化。
- `idp-web` はNginxで配信し、APIへのリバースプロキシも兼ねる構成が考えられる。
- 開発時はCORS設定が必要となるが、それ自体が学習価値となる。

---

## 案C：Rails主体・最小Vue型（シンプル重視）

管理画面はRailsのSSR（ERB）で賄い、Vueは同意画面やログイン画面など、ユーザーの認証フローのみに使用する。

```
my-sso/
├── Gemfile
├── app/
│   ├── controllers/
│   │   ├── oauth/                  # OIDCエンドポイント
│   │   │   ├── authorize_controller.rb
│   │   │   └── token_controller.rb
│   │   ├── admin/                  # Rails SSR管理画面
│   │   └── application_controller.rb
│   ├── models/
│   ├── views/
│   │   ├── layouts/application.html.erb
│   │   └── admin/                  # ERBで管理画面
│   └── frontend/                   # Vue 3（Vite Ruby or Webpacker）
│       ├── components/
│       │   ├── LoginForm.vue
│       │   └── ConsentScreen.vue
│       └── entrypoints/
│           ├── login.ts
│           └── consent.ts
├── config/
├── db/
└── demo-rp/                        # 別ディレクトリ or 別リポジトリ
```

### 特徴

- Railsのセッション管理・CSRF保護をそのまま活用できる。
- Vueは部分的に導入（ページ単位またはコンポーネント単位）し、段階的にSPA化できる。
- フロントエンドの自由度はやや低下するが、セキュリティ的には最も容易。

---

## 比較

| 観点 | 案A：モノレポ統合型 | 案B：API分離型 | 案C：Rails主体型 |
|---|---|---|---|
| **学習コスト** | 低い | 中（CORS等） | 最低 |
| **拡張性** | 高い | 最高 | 中 |
| **Vueの活用度** | 管理画面・同意画面 | 全面 | 一部のみ |
| **デプロイ複雑度** | 低 | 中 | 最低 |
| **セッション管理** | Cookie（SameSite） | Token or Cookie | Cookie（Rails標準） |
| **推奨フェーズ** | Phase 1〜4 | Phase 3以降 | Phase 1〜2 |

---

## 選定指針

- **すぐに動かしたい、エンドツーエンドで理解したい** → **案A**
- **フロントエンドの最新ツールチェーンを深く学びたい** → **案B**
- **Railsのセキュリティ機構を最大限使い、後からVueを増やしたい** → **案C**
