# feature/project-setup/docker

## 目的

PostgreSQL + 3 Rails + Vue + RP を一括起動できる `docker-compose.yml` と各サービスの `Dockerfile` を作成する。

## ブランチ名

`feature/project-setup/docker`

## 親ブランチ

`feature/project-setup/init`

## 作成・変更するファイル

### `docker-compose.yml`

```yaml
version: '3.8'

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
    command: npm run dev
    volumes:
      - ./frontend:/app
    ports:
      - "5173:5173"
    networks:
      - sso_network

  demo-rp:
    build: ./demo-rp
    command: npm run dev
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
FROM ruby:3.3
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
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]
```

### `demo-rp/Dockerfile`

（`frontend/Dockerfile` と同様）

## マージ基準（チェックリスト）

- [ ] `docker compose up` で PostgreSQL が起動する
- [ ] `docker compose up idp-auth` で Rails サーバ（ポート3000）が起動する
- [ ] `docker compose up idp-user` で Rails サーバ（ポート3001）が起動する
- [ ] `docker compose up idp-client` で Rails サーバ（ポート3002）が起動する
- [ ] `docker compose up frontend` で Vite 開発サーバ（ポート5173）が起動する
- [ ] `docker compose up demo-rp` で デモRP（ポート5174）が起動する

## 備考

各 `Dockerfile` は最小構成とし、本番用のマルチステージビルドは Phase 4 以降で導入する。
