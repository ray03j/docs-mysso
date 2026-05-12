# feature/project-setup/docker

## 目的

PostgreSQL + 3 Rails + Vue + RP を一括起動できる `compose.yml` と各サービスの `Dockerfile` を作成する。

## ブランチ名

`feature/project-setup/docker`

## 親ブランチ

`feature/project-setup/init`

## 作成・変更するファイル

### `compose.yml`

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./infrastructure/postgresql/init:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    networks:
      - sso_network

  idp-auth:
    build: ./idp-auth
    command: bundle exec rails server -b 0.0.0.0 -p 3000
    volumes:
      - ./idp-auth:/app
    ports:
      - "3000:3000"
    environment:
      RAILS_ENV: development
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/auth_db
    depends_on:
      - postgres
    networks:
      - sso_network

  idp-user:
    build: ./idp-user
    command: bundle exec rails server -b 0.0.0.0 -p 3001
    volumes:
      - ./idp-user:/app
    ports:
      - "3001:3001"
    environment:
      RAILS_ENV: development
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/user_db
    depends_on:
      - postgres
    networks:
      - sso_network

  idp-client:
    build: ./idp-client
    command: bundle exec rails server -b 0.0.0.0 -p 3002
    volumes:
      - ./idp-client:/app
    ports:
      - "3002:3002"
    environment:
      RAILS_ENV: development
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/client_db
    depends_on:
      - postgres
    networks:
      - sso_network

  frontend:
    build: ./frontend
    command: pnpm dev
    volumes:
      - ./frontend:/app
    ports:
      - "5173:5173"
    networks:
      - sso_network

  demo-rp:
    build: ./demo-rp
    command: pnpm dev
    volumes:
      - ./demo-rp:/app
    ports:
      - "5174:5174"
    networks:
      - sso_network

volumes:
  pg_data:

networks:
  sso_network:
    driver: bridge
```

### `idp-auth/Dockerfile`

```dockerfile
FROM ruby:3.3.0
WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle install
COPY . .
CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]
```

### `idp-user/Dockerfile`

（`idp-auth/Dockerfile` と同様、ポートのみ調整）

### `idp-client/Dockerfile`

（`idp-auth/Dockerfile` と同様、ポートのみ調整）

### `frontend/Dockerfile`

```dockerfile
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN corepack enable && pnpm install
COPY . .
CMD ["pnpm", "dev"]
```

### `demo-rp/Dockerfile`

（`frontend/Dockerfile` と同様）

#### なぜこの設定か


- **ポート割り当て**：
  - `3000`：`idp-auth`（認可サービス）は OIDC のエントリーポイントとなり、フロントエンド・RP 双方から最も頻繁に呼ばれるため、覚えやすいデフォルトポートを割り当てた。
  - `3001`/`3002`：`idp-user`・`idp-client` は内部マイクロサービスとして呼ばれることが多いが、開発時の直接アクセス（デバッグ）を考慮して連番にした。
  - `5173`/`5174`：Vite 開発サーバのデフォルトポートが `5173` なので、`frontend` にそのまま使用。`demo-rp` は `5174` にずらし、同時起動時の競合を回避。
- **`depends_on` のみ指定し、`links` は使わない**：`links` はレガシー機能であり、代わりに共有ネットワーク `sso_network` + サービス名 DNS で解決する。これにより、後からサービス追加・削除が容易になる。
- **`volumes: ./<service>:/app`**：ホストのソースコードをコンテナにマウントし、ファイル変更時に即座にホットリロードが効くようにする。開発生産性を優先した設定（本番ではマルチステージビルドで排除する）。

## マージ基準（チェックリスト）

- [ ] `docker compose up` で PostgreSQL が起動する
- [ ] `docker compose up idp-auth` で Rails サーバ（ポート3000）が起動する
- [ ] `docker compose up idp-user` で Rails サーバ（ポート3001）が起動する
- [ ] `docker compose up idp-client` で Rails サーバ（ポート3002）が起動する
- [ ] `docker compose up frontend` で Vite 開発サーバ（ポート5173）が起動する
- [ ] `docker compose up demo-rp` で デモRP（ポート5174）が起動する

#### なぜこの設定か

- **`ruby:3.3.0`**：Gemfile で `ruby '3.3.0'` を固定しているため、ビルド時の Ruby バージョンを厳密に一致させる。パッチバージョンまで固定することで、開発者間・CI 間の Ruby バージョン不一致を防ぎ、`bundle install` 時の「Your Ruby version is ... but your Gemfile specified ...」エラーを回避する。
- **`bundle install` を `COPY . .` より先に実行**：`Gemfile`/`Gemfile.lock` が変更されない限り、Docker レイヤキャッシュが効き、ビルド時間を短縮できる。ソースコードの細かい変更のたびに `bundle install` が走ると開発効率が著しく低下する。
- **`node:20`**：package.json のエコシステム（Vite, ESLint など）が Node 20 LTS で検証されているため。LTS 版を使うことで、開発中に Node の破壊的変更によるトラブルを回避する。

## 備考

各 `Dockerfile` は最小構成とし、本番用のマルチステージビルドは Phase 4 以降で導入する。
