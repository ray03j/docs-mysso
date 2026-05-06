# feature/project-setup/init

## 目的

プロジェクトのルート構造を作成し、各サービス用の空ディレクトリを配置する。チーム全員が一貫したディレクトリ構成で開発を開始できる状態にする。

## ブランチ名

`feature/project-setup/init`

## 親ブランチ

なし（`main` から直接分岐）

## 作成するディレクトリ構造

```
my-sso/
├── .gitignore
├── .env.example
├── docker-compose.yml          # ← 空ファイル or 後続ブランチで上書き
│
├── idp-auth/
│   ├── app/
│   │   ├── controllers/
│   │   └── models/
│   ├── config/
│   │   └── environments/
│   ├── db/
│   └── spec/
│
├── idp-user/
│   ├── app/
│   │   ├── controllers/
│   │   └── models/
│   ├── config/
│   │   └── environments/
│   ├── db/
│   └── spec/
│
├── idp-client/
│   ├── app/
│   │   ├── controllers/
│   │   └── models/
│   ├── config/
│   │   └── environments/
│   ├── db/
│   └── spec/
│
├── frontend/
│   └── src/
│
├── demo-rp/
│   └── src/
│
└── infrastructure/
    ├── postgresql/
    │   └── init/
    └── nginx/
```

## 作成するファイル

### `.gitignore`

```gitignore
# Docker
.env

# Rails
*/tmp/
*/log/
*/vendor/bundle/
*/.bundle/
*/config/master.key

# Node
node_modules/
dist/
*/.vite/

# IDE
.idea/
.vscode/
*.swp

# OS
.DS_Store
```

### `.env.example`

```env
# PostgreSQL
POSTGRES_USER=sso_user
POSTGRES_PASSWORD=sso_password
POSTGRES_DB=postgres

# Rails
RAILS_ENV=development
SECRET_KEY_BASE=replace_in_production

# Frontend
VITE_API_BASE_URL=http://localhost:3000
```

### `docker-compose.yml`

空ファイル（`feature/project-setup/docker` で上書き）

## マージ基準（チェックリスト）

- [ ] `my-sso/` 以下に全サービス用ディレクトリが作成されている
- [ ] `.gitignore` が適切に設定されている
- [ ] `.env.example` に必要な環境変数が定義されている
- [ ] `docker-compose.yml` が存在する（空でも可）

## 備考

本ブランチでは**空ディレクトリと最小限の設定ファイルのみ**を作成する。各サービスの実装（Gemfile, package.json など）は後続ブランチで行う。
