# feature/project-setup/infra

## 目的

共有インフラ（PostgreSQL 初期化スクリプト、nginx 雛形）を作成する。

## ブランチ名

`feature/project-setup/infra`

## 親ブランチ

`feature/project-setup/init`

## 作成するファイル

### `infrastructure/postgresql/init/01_init.sql`

```sql
CREATE DATABASE auth_db;
CREATE DATABASE user_db;
CREATE DATABASE client_db;
```

### `infrastructure/nginx/nginx.conf`

```nginx
# Phase 4 で API Gateway として本格導入
# 現時点では雛形のみ

upstream idp_auth {
    server idp-auth:3000;
}

upstream idp_user {
    server idp-user:3001;
}

upstream idp_client {
    server idp-client:3002;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://idp_auth;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## マージ基準（チェックリスト）

- [ ] `docker compose up postgres` で3つのDB（`auth_db`, `user_db`, `client_db`）が作成される
- [ ] `infrastructure/nginx/nginx.conf` が存在し、構文エラーがない

## 備考

nginx は現時点では `docker-compose.yml` に含めない。Phase 4 で正式に導入する。
