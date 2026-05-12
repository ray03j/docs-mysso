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

#### なぜこの設定か

- **命名 `01_init.sql`**：PostgreSQL の Docker イメージは `/docker-entrypoint-initdb.d` に配置されたファイルを辞書順に実行するため、先頭に `01_` を付けて「最初に実行される」ことを保証。後から `02_seed.sql` などを追加しやすい命名規則。
- **1 ファイルで 3 DB を作成**：`compose.yml` の `postgres` サービスは単一コンテナで動作するため、Rails 各サービスが分離された DB を使えるようにする必要がある。1 ファイルにまとめることで、初期化スクリプトの散在を防ぐ。

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

#### なぜこの設定か

- **雛形としての配置**：現時点では `compose.yml` に含めないが、Phase 4 で API Gateway として導入する際に設定の起点となる。 upstream を事前に定義しておくことで、後続ブランチで `location /api/auth` などに振り分ける際の変更差分が最小化される。
- **`proxy_set_header`**：Rails がリモート IP やホスト名を正しく認識できるようにするための必須ヘッダ。特に OIDC の `redirect_uri` 検証時に `Host` ヘッダが正しくないとエラーになるため、初期段階から含めている。

## マージ基準（チェックリスト）

- [ ] `docker compose up postgres` で3つのDB（`auth_db`, `user_db`, `client_db`）が作成される
- [ ] `infrastructure/nginx/nginx.conf` が存在し、構文エラーがない

## 備考

nginx は現時点では `compose.yml` に含めない。Phase 4 で正式に導入する。
