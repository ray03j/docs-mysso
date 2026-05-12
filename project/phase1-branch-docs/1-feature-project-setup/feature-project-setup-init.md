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
├── compose.yml                 # ← 空ファイル or 後続ブランチで上書き
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

### なぜこの構成か

- **monorepo 構成**：複数サービスを単一リポジトリで管理し、Docker Compose で一括起動できるようにするため。サービス間の連携（API 呼び出し、認可フロー）が頻繁に発生するため、リポジトリを分割すると開発サイクルが遅くなる。
- **`idp-*` 配下の Rails 標準ディレクトリ**：`rails new --api` で生成される構成をそのまま採用し、チームメンバーが既知のレイアウトで開発できるようにするため。`app/controllers`・`app/models`・`config`・`db`・`spec` は Rails API モードの最小構成。
- **`frontend`/`demo-rp` は `src/` のみ**：Vue 3 + Vite の標準的なソース配置。ビルド成果物は `dist/` に出力するため、ソースコードは `src/` に集約する。
- **`infrastructure/` を分離**：DB 初期化スクリプトや nginx 設定など、複数サービスで共有するインフラリソースを一元管理するため。各サービスのディレクトリに散らばると、インフラ変更時に探し回るコストが発生する。

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

#### なぜこの設定か

- **`.env`**：`.env.example` はテンプレートとしてコミットするが、実際の `.env` は開発者ごとに異なる値（ローカルでのポート変更、一時的なシークレットなど）が入るため Git 管理外にする。Docker Compose も `.env` を自動読み込みするため、意図しない上書きを防ぐ。
- **`*/` プレフィックス（Rails・Node）**：monorepo 構成で複数サービスが同じリポジトリルートに存在するため、各サービス配下の同名ディレクトリを一括で無視する。例えば `idp-auth/tmp/`、`idp-user/tmp/` などを個別に書かずに済む。
- **`*/config/master.key`**：Rails の暗号化キー。漏洩すると `credentials.yml.enc` が復号できてしまうため、絶対にコミットしない。
- **IDE・OS 項目**：`.idea/`（IntelliJ）、`.vscode/`（VS Code）、`.DS_Store`（macOS）などは、開発者のローカル環境に依存するファイルで、プロジェクトの本質ではない。これらがコミットされると、不要な差分が PR に混入し、レビュー効率が落ちる。チームに Windows/macOS/Linux が混在していても安全な共通最小設定とするため、IDE・OS 双方を含めている。

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

#### なぜこの設定か

- **`.env.example` を配置する理由**：`.env` は Git 管理外だが、新規参入者や CI が必要な環境変数を把握するためにテンプレートが必要。これがないと「どの変数を設定すればいいか」が暗黙知になってしまう。
- **`POSTGRES_*`**：`compose.yml` の `postgres` サービスと Rails の `database.yml` の双方で参照する共通変数。開発環境では固定値で十分だが、`.env` 経由にすることで、ローカルで別のポートやパスワードに変更したい場合に柔軟に対応できる。
- **`POSTGRES_DB=postgres`**：PostgreSQL イメージのデフォルトスーパーユーザ DB 名。初期化スクリプトで `auth_db` などを作成する前の接続先として必要。
- **`SECRET_KEY_BASE=replace_in_production`**：Rails のセッション署名に使用。開発環境では自動生成されるが、明示的に置き換えを促すことで、本番デプロイ時の設定漏れを防ぐ。
- **`VITE_API_BASE_URL=http://localhost:3000`**：フロントエンドが `idp-auth`（ポート3000）に向けて API リクエストするための基点 URL。`localhost` を使う理由は、開発者がブラウザで直接 `frontend`（5173）にアクセスした際に CORS なしで `idp-auth`（3000）に到達できるようにするため。コンテナ間通信では `http://idp-auth:3000` を使うが、ブラウザからはホスト名解決できないため分けている。

### `compose.yml`

空ファイル（`feature/project-setup/docker` で上書き）

#### なぜ空ファイルか

- **存在確認のため**：`init` ブランチ時点でファイルがないと、後続ブランチで「新規作成」扱いになり、`git merge` 時に競合検出が曖昧になる可能性がある。空ファイルを先に置くことで、後続ブランチは「内容追加」として扱われ、差分が明確になる。
- **チェックリストの明確化**：`init` のマージ基準に「`compose.yml` が存在する（空でも可）」と記載することで、Phase 1 の進捗を可視化するため。

## マージ基準（チェックリスト）

- [ ] `my-sso/` 以下に全サービス用ディレクトリが作成されている
- [ ] `.gitignore` が適切に設定されている
- [ ] `.env.example` に必要な環境変数が定義されている
- [ ] `compose.yml` が存在する（空でも可）

## 備考

本ブランチでは**空ディレクトリと最小限の設定ファイルのみ**を作成する。各サービスの実装（Gemfile, package.json など）は後続ブランチで行う。
