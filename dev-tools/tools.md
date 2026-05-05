# 開発環境整備ツール一覧（OSS 複数候補版）

本プロジェクト（Ruby on Rails + Vue 3 + PostgreSQL による自作 SSO）の品質・セキュリティ・開発効率を担保するため、**各カテゴリごとに OSS を 3〜4 候補**ずつ列挙します。  
**★40K 以上**と**★40K 未満**を混在させ、実績と新興のバランスを考慮しています。

> スター数は 2025/05 時点の GitHub 公開値です。

---

## 1. Pre-commit Hooks（コミット前品質担保）

| ツール | Stars | 区分 | 用途 |
|---|---|---|---|
| [husky](https://github.com/typicode/husky) | ★ 35,016 | < 40K | Node.js エコシステムで最も普及している Git フックマネージャ |
| [pre-commit](https://github.com/pre-commit/pre-commit) | ★ 15,225 | < 40K | Python 製。マルチ言語対応で設定ファイル一本管理 |
| [lefthook](https://github.com/evilmartians/lefthook) | ★ 8,128 | < 40K | Go 製で高速。Rails + Vue の並列実行に最適 |

**推奨**: `lefthook`（高速・並列実行・YAML 設定が簡潔）

---

## 2. Linting / Formatting（Ruby）

| ツール | Stars | 区分 | 用途 |
|---|---|---|---|
| [RuboCop](https://github.com/rubocop/rubocop) | ★ 12,854 | < 40K | Rails 業界標準。豊富な Cop と自動修正機能 |
| [StandardRB](https://github.com/standardrb/standard) | ★ 2,894 | < 40K | RuboCop の設定不要版。「議論より実行」 |
| [erb_lint](https://github.com/Shopify/erb-lint) | ★ 1,200 程度 | < 40K | Shopify 製。Rails ERB テンプレート専用 Linter |

**推奨**: `RuboCop`（エコシステム最大。StandardRB は移行後の簡略化に）

---

## 3. Linting / Formatting（JS / Vue）

| ツール | Stars | 区分 | 用途 |
|---|---|---|---|
| [Prettier](https://github.com/prettier/prettier) | ★ 51,837 | **≥ 40K** | フロントエンド_formatter の事実上の標準 |
| [ESLint](https://github.com/eslint/eslint) | ★ 27,220 | < 40K | JS/TS/Vue の静的解析。プラグイン充実 |
| [Biome](https://github.com/biomejs/biome) | ★ 24,513 | < 40K | Rust 製で高速。Lint + Format を一元化 |

**推奨**: `ESLint` + `Prettier`（分離運用が Vue エコシステムで安定）

---

## 4. Security Scanning（SSO に必須）

| ツール | Stars | 区分 | 用途 |
|---|---|---|---|
| [Trivy](https://github.com/aquasecurity/trivy) | ★ 34,809 | < 40K | コンテナ・依存関係・IaC の総合脆弱性スキャナ |
| [OWASP ZAP](https://github.com/zaproxy/zaproxy) | ★ 15,075 | < 40K | Web アプリ動的セキュリティスキャンの OSS 標準 |
| [Brakeman](https://github.com/presidentbeef/brakeman) | ★ 7,228 | < 40K | Rails 専用セキュリティ静的解析 |
| [bundler-audit](https://github.com/rubysec/bundler-audit) | ★ 2,748 | < 40K | Ruby Gem の既知脆弱性チェック |

**推奨**: `Brakeman` + `bundler-audit` + `Trivy`（Rails + コンテナ両面カバー）

---

## 5. CI / CD（継続的インテグレーション）

| ツール | Stars | 区分 | 用途 |
|---|---|---|---|
| [Gitea](https://github.com/go-gitea/gitea) | ★ 55,289 | **≥ 40K** | 軽量 Git ホスティング + Actions 機能（自前運用向け） |
| [Jenkins](https://github.com/jenkinsci/jenkins) | ★ 25,241 | < 40K | プラグイン数No.1。オンプレ/クラウド両対応 |
| [Woodpecker CI](https://github.com/woodpecker-ci/woodpecker) | ★ 6,935 | < 40K | Drone フォーク。コンテナネイティブで軽量 |
| GitHub Actions | — | — | スター数なし。GitHub 提供。最も手軽 |

**推奨**: **GitHub Actions**（本プロジェクトは GitHub 上で管理する前提）

---

## 6. Testing（Ruby / Rails）

| ツール | Stars | 区分 | 用途 |
|---|---|---|---|
| [Capybara](https://github.com/teamcapybara/capybara) | ★ 10,157 | < 40K | Rails E2E テスト。ブラウザ操作をコード化 |
| [RSpec](https://github.com/rspec/rspec-metagem) | ★ 2,832 | < 40K | BDD スタイルのテストフレームワーク。Ruby 界隈で圧倒的シェア |
| [FactoryBot](https://github.com/thoughtbot/factory_bot) | ★ 7,800 程度 | < 40K | テストデータ生成のデファクトスタンダード |

**推奨**: `RSpec` + `FactoryBot` + `Capybara`（Rails 開発の鉄板セット）

---

## 7. Testing（JS / Vue フロントエンド）

| ツール | Stars | 区分 | 用途 |
|---|---|---|---|
| [Playwright](https://github.com/microsoft/playwright) | ★ 87,810 | **≥ 40K** | Microsoft 製 E2E。マルチブラウザ・高速・高機能 |
| [Cypress](https://github.com/cypress-io/cypress) | ★ 49,624 | **≥ 40K** | フロントエンド E2E で圧倒的シェア。開発体験が優秀 |
| [Jest](https://github.com/jestjs/jest) | ★ 45,336 | **≥ 40K** | Meta 製。JS 単体テストの老舗。スナップショット強力 |
| [Vitest](https://github.com/vitest-dev/vitest) | ★ 16,452 | < 40K | Vite ネイティブ。高速・TypeScript 最適化 |

**推奨**: **Vitest**（Vite 連携）+ **Playwright**（E2E）

---

## 8. Database（スキーマ・マイグレーション管理）

| ツール | Stars | 区分 | 用途 |
|---|---|---|---|
| [strong_migrations](https://github.com/ankane/strong_migrations) | ★ 4,391 | < 40K | 危険なマイグレーション（カラム削除等）を実行前に阻止 |
| [rails-erd](https://github.com/voormedia/rails-erd) | ★ 4,075 | < 40K | `bundle exec erd` で ER 図を自動生成 |
| [Scenic](https://github.com/scenic-views/scenic) | ★ 3,200 程度 | < 40K | PostgreSQL の View をマイグレーション管理 |

**推奨**: `strong_migrations`（必須）+ `rails-erd`（設計レビュー用）

---

## 9. Documentation（仕様可視化）

| ツール | Stars | 区分 | 用途 |
|---|---|---|---|
| [Docusaurus](https://github.com/facebook/docusaurus) | ★ 64,791 | **≥ 40K** | Meta 製。Markdown ベースのドキュメントサイト構築 |
| [Swagger UI](https://github.com/swagger-api/swagger-ui) | ★ 28,767 | < 40K | OpenAPI 仕様をインタラクティブに可視化 |
| [VitePress](https://github.com/vuejs/vitepress) | ★ 17,636 | < 40K | Vue チーム製。Vite ベースの高速静的サイト生成 |
| [YARD](https://github.com/lsegal/yard) | ★ 1,100 程度 | < 40K | Ruby コードのドキュメント生成 |

**推奨**: **Swagger UI**（OIDC エンドポイント仕様）+ **VitePress**（プロジェクト Wiki）

---

## 10. Containerization（環境統一）

| ツール | Stars | 区分 | 用途 |
|---|---|---|---|
| [Docker / Moby](https://github.com/moby/moby) | ★ 71,526 | **≥ 40K** | コンテナランタイムの事実上の標準 |
| [Podman](https://github.com/containers/podman) | ★ 31,564 | < 40K | Daemon 不要・rootless。RedHat 主導の Docker 代替 |
| [Docker Compose](https://github.com/docker/compose) | ★ 37,330 | < 40K | 複数コンテナ（Rails + PostgreSQL + Vue）を一括管理 |
| [Dev Containers](https://github.com/devcontainers/cli) | ★ 3,000 程度 | < 40K | VS Code 連携で完全に再現可能な開発コンテナ |

**推奨**: **Docker Compose**（チーム開発の環境統一に最も手軽）

---

## 11. Secret / Config Management（設定・機密情報）

| ツール | Stars | 区分 | 用途 |
|---|---|---|---|
| [HashiCorp Vault](https://github.com/hashicorp/vault) | ★ 35,554 | < 40K | 企業級の暗号化・動的シークレット管理 |
| [Mozilla SOPS](https://github.com/getsops/sops) | ★ 21,664 | < 40K | Git 管理可能な暗号化設定ファイル。YAML/JSON 対応 |
| [dotenv](https://github.com/bkeepers/dotenv) | ★ 6,742 | < 40K | 開発環境の環境変数を `.env` で管理 |
| [Rails Credentials](https://edgeguides.rubyonrails.org/security.html#custom-credentials) | — | — | Rails 標準。マスターキー方式の暗号化設定 |

**推奨**: **dotenv**（開発）+ **Rails Credentials**（本番・ステージング）

---

## 12. SSO / JWT 特有の検証・デバッグツール

| ツール | Stars | 区分 | 用途 |
|---|---|---|---|
| [Insomnia](https://github.com/Kong/insomnia) | ★ 38,368 | < 40K | REST/GraphQL クライアント。OIDC エンドポイント検証に便利 |
| [jwt-cli](https://github.com/mike-engel/jwt-cli) | ★ 1,467 | < 40K | ターミナルから JWT のデコード・検証 |
| [OpenID Connect Debugger](https://oidcdebugger.com/) | — | — | Web 上で Authorization Code Flow を手動デバッグ |
| Postman | — | — | スター数なし。最も普及した API クライアント |

**推奨**: **Insomnia**（コレクション保存・チーム共有が容易）

---

## 推奨導入セット（フェーズ別）

### Phase 1: プロジェクト初期（今すぐ）
| カテゴリ | ツール |
|---|---|
| Pre-commit | lefthook |
| Lint Ruby | RuboCop |
| Lint JS/Vue | ESLint + Prettier |
| Test Ruby | RSpec + FactoryBot |
| DB 安全 | strong_migrations |
| 設定管理 | dotenv |
| 環境統一 | Docker Compose |

### Phase 2: 開発本格化時
| カテゴリ | ツール |
|---|---|
| CI/CD | GitHub Actions |
| Security | Brakeman + bundler-audit |
| Test E2E | Playwright |
| API 仕様 | Swagger UI |

### Phase 3: セキュリティ強化・リリース前
| カテゴリ | ツール |
|---|---|
| 動的スキャン | OWASP ZAP |
| 脆弱性監視 | Dependabot（GitHub 標準） |
| 設定暗号化 | Rails Credentials |
| ドキュメント | VitePress |

---

## 補足: 本プロジェクト固有の注意点

- **JWT は自作する**ため、トークンの署名・検証ロジックに対するユニットテストを徹底する（RSpec で網羅的に）
- **PKCE 対応**があるため、認証フローの E2E テスト（Playwright/Capybara）で PKCE パラメータの検証も含める
- **HTTPS（自己署名証明書）**環境での開発を前提とするため、CI でも証明書関連の設定を考慮する
- ★40K 以上のツールは**コミュニティ大・情報豊富**だが、本プロジェクトの技術スタック（Rails/Vue）と必ずしも最適解ではない。スター数より**開発体験・連携性**を優先して選定しています。


40K 以上の代表的ツール
- Prettier (51K), Jest (45K), Playwright (87K), Cypress (49K)
- Docker / Moby (71K), Docusaurus (64K), Gitea (55K)
40K 未満だが本プロジェクトに最適なツール
- lefthook (8K), RuboCop (12K), Brakeman (7K), RSpec (2.8K)