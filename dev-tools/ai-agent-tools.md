# AIエージェントツール一覧

本ファイルは、requirements.md に基づく自作 SSO プロジェクトにおいて、AI エージェントが活用すべき **Skills**、**AGENTS.md**、**Subagent** を一覧化したものです。  
単体テストを徹底する方針のため、テスト支援ツールを特に重視しています。

---

## 1. Skills

AI エージェントが `skill` ツールでロードする専門スキルです。

| スキル名 | 用途 | 必要性の根拠 |
|---|---|---|
| `rails-development` | Ruby on Rails アプリケーションの開発ワークフロー、規約、ベストプラクティス | 技術要件で Rails を採用（requirements.md 5.1） |
| `ruby-programming` | Ruby 言語のイディオム、リファクタリングパターン、パフォーマンスTips | バックエンド言語が Ruby（requirements.md 5.1） |
| `rspec-testing` | RSpec を使った単体テスト・結合テストの作成パターン、モック/スタブの使い分け | **単体テストを書く予定**であり、RSpec は推奨ツール（tools.md 6） |
| `oauth2-oidc` | OAuth 2.0 / OpenID Connect プロトコルの実装、エンドポイント仕様、JWT 取り扱い | コア機能が Authorization Code Flow + OIDC（requirements.md 3.1） |
| `security-practices` | セキュリティ要件の実装（PKCE、bcrypt、JWT署名、SameSite Cookie、CSP等） | 非機能要件にセキュリティが詳細に定義（requirements.md 4.1） |
| `vue-frontend` | Vue 3 / Composition API の開発パターン、状態管理、コンポーネント設計 | フロントエンドが Vue 3（requirements.md 5.1）、Phase 4 で使用 |
| `docker-compose` | Docker Compose を使ったマルチコンテナ開発環境の構築・運用 | Phase 1 推奨（tools.md 10）。Rails + PostgreSQL の環境統一に必須 |
| `postgresql` | PostgreSQL のスキーマ設計、インデックス戦略、クエリ最適化 | 性能要件（ステートレス化、DBアクセス最小化）の実現に必須（requirements.md 4.2） |
| `github-actions` | GitHub Actions による CI/CD ワークフロー構築 | Phase 2 推奨（tools.md 5）。自動テスト・セキュリティスキャン用 |
| `ruby-security-tools` | Brakeman、bundler-audit 等、Ruby エコシステムのセキュリティツール運用 | Phase 2 推奨（tools.md 4）。ただし `security-practices` と役割が一部重複 |

---

## 2. AGENTS.md

プロジェクト内の各ディレクトリに配置し、AI エージェントの行動指針とコンテキストを提供するファイルです。

| 配置パス | 役割 | 記載すべき内容の例 |
|---|---|---|
| `AGENTS.md`（プロジェクトルート） | 全体共通のガイドライン | 技術スタック、ブランチ戦略、コーディング規約、単体テスト方針、Phase 優先順位 |
| `app/controllers/api/AGENTS.md` | API エンドポイント実装のガイドライン | OIDC エンドポイント（/authorize, /token, /userinfo, /revoke）の入出力仕様、エラーレスポンス形式、JWT の取り扱い |
| `app/models/AGENTS.md` | モデル層のガイドライン | ユーザー・クライアント（RP）モデルの責務、パスワードハッシュ化（bcrypt）の扱い、バリデーション規約 |
| `lib/oidc/AGENTS.md` | OIDC プロトコル実装のガイドライン | JWT 署名（RS256）・検証ロジック、Authorization Code の生成・検証、PKCE パラメータの検証フロー |
| `app/views/AGENTS.md` | 画面実装のガイドライン | ログイン画面・同意画面の UI/UX 要件、CSRF トークンの埋め込み、CSP 対応 |
| `spec/AGENTS.md` | テスト作成のガイドライン | **単体テスト徹底のための方針**、FactoryBot の使い方、JWT 署名検証のモック戦略、PKCE フローのテストパターン |
| `config/AGENTS.md` | 設定管理のガイドライン | 環境変数の扱い、Rails Credentials の使い分け、OIDC Discovery メタデータの設定 |
| `db/migrate/AGENTS.md` | マイグレーション運用のガイドライン | strong_migrations 準拠、危険な DDL（カラム削除等）の回避策、ロールバック手順 |

---

## 3. Subagent

`task` ツールでデリゲートする専門エージェントです。複雑な作業を並列・専門化して効率化します。

| エージェント名 | 役割 | デリゲートするタイミング |
|---|---|---|
| `code-reviewer` | 実装コードのレビュー（可読性、Rails 規約、セキュリティ観点） | 主要な機能実装後、PR 作成前 |
| `test-writer` | **単体テスト・結合テストの作成**（RSpec + FactoryBot） | モデル・サービスクラス実装後、またはテスト不足を検知したとき |
| `security-auditor` | セキュリティ監査（SQLi、XSS、CSRF、JWT の弱署名、PKCE 抜け等） | Phase 5（セキュリティ強化）時、または要件に関連するコード変更後 |
| `oidc-protocol-validator` | OIDC プロトコル準拠の検証（Discovery エンドポイント、Token レスポンス形式、スコープ処理等） | /authorize, /token, /userinfo 実装後、または仕様変更時 |
| `frontend-reviewer` | Vue 3 フロントエンドコードのレビュー（Composition API、型安全性、アクセシビリティ） | Phase 4（デモ RP 実装）時 |
| `database-design-reviewer` | スキーマ設計・マイグレーションのレビュー（正規化、インデックス、strong_migrations 準拠） | Phase 1（DB 設計）時、またはマイグレーション追加時 |
| `api-integration-tester` | API エンドポイント間の連携テスト（OIDC フロー全体の検証） | 単体テストでは不十分な /authorize → /token → /userinfo の統合検証時 |
| `migration-reviewer` | マイグレーションコードの安全性レビュー（strong_migrations ルール適用） | 本番環境向け DDL 変更を行う前 |

---

## 4. 補足：単体テストとの関連

requirements.md に「単体テストは書く予定です」とあるため、以下を特に重視します：

- **Skill `rspec-testing`** は必ずロードし、テストファースト or テスト併記の開発を支援
- **Subagent `test-writer`** はモデル・サービスクラス・JWT 署名検証ロジックの実装後に積極的に起動
- **AGENTS.md `spec/AGENTS.md`** には、JWT の自作部分に対する網羅的なテスト方針を明記（tools.md 補足参照）
