# RP（Relying Party）利用ガイド

本ドキュメントは、本SSO（IdP）を利用するクライアントアプリケーション（RP）側の実装方法をまとめます。

## 前提

- IdP は OAuth 2.0 / OpenID Connect (OIDC) に準拠しています。
- 認証フローは **Authorization Code Flow with PKCE** を採用しています。
- IdP のエンドポイントは `/.well-known/openid-configuration` から動的に取得できます。

## 1. IdP 側で事前に行うこと

RP は IdP 管理画面または API で以下を登録しておく必要があります。

| 項目 | 例 |
|---|---|
| `client_id` | `demo-rp-001` |
| `client_secret` | `xxxxxxxx`（機密クライアントの場合） |
| `redirect_uris` | `http://localhost:5173/callback` |
| `scope` | `openid profile email` |

## 2. 認証フロー全体

```
1. Discovery
   GET /.well-known/openid-configuration

2. Authorize（ブラウザリダイレクト）
   GET /authorize?client_id=...&redirect_uri=...&response_type=code&scope=openid%20profile%20email&code_challenge=...&code_challenge_method=S256&state=...

3. Callback（ブラウザから RP に戻る）
   GET /callback?code=xxx&state=xxx

4. Token（サーバー間通信推奨）
   POST /token

5. UserInfo
   GET /userinfo (Authorization: Bearer <access_token>)
```

## 3. コード実装例

### 3.1 Discovery & Authorize リダイレクト

```typescript
const IDP_BASE = import.meta.env.VITE_IDP_BASE_URL;

export async function redirectToAuthorize() {
  // 1. Discovery（初回のみキャッシュ可能）
  const discovery = await fetch(`${IDP_BASE}/.well-known/openid-configuration`).then(r => r.json());

  // 2. PKCE パラメータ生成
  const codeVerifier = generateCodeVerifier();
  const codeChallenge = await sha256Base64Url(codeVerifier);
  sessionStorage.setItem('pkce_verifier', codeVerifier);

  // 3. state 生成（CSRF 対策）
  const state = generateRandomString(32);
  sessionStorage.setItem('oauth_state', state);

  // 4. authorize リダイレクト
  const params = new URLSearchParams({
    client_id: import.meta.env.VITE_CLIENT_ID,
    redirect_uri: import.meta.env.VITE_REDIRECT_URI,
    response_type: 'code',
    scope: 'openid profile email',
    code_challenge: codeChallenge,
    code_challenge_method: 'S256',
    state: state,
  });

  window.location.href = `${discovery.authorization_endpoint}?${params.toString()}`;
}

function generateCodeVerifier() {
  const array = new Uint8Array(32);
  crypto.getRandomValues(array);
  return base64UrlEncode(array);
}

async function sha256Base64Url(plain: string) {
  const encoder = new TextEncoder();
  const data = encoder.encode(plain);
  const digest = await crypto.subtle.digest('SHA-256', data);
  return base64UrlEncode(new Uint8Array(digest));
}

function base64UrlEncode(buffer: Uint8Array) {
  return btoa(String.fromCharCode(...buffer))
    .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
}

function generateRandomString(len: number) {
  const array = new Uint8Array(len);
  crypto.getRandomValues(array);
  return base64UrlEncode(array);
}
```

### 3.2 Callback & Token 交換

```typescript
export async function handleCallback(url: URL) {
  const code = url.searchParams.get('code');
  const state = url.searchParams.get('state');
  const savedState = sessionStorage.getItem('oauth_state');

  if (state !== savedState) throw new Error('Invalid state');

  const codeVerifier = sessionStorage.getItem('pkce_verifier');
  if (!codeVerifier) throw new Error('Missing PKCE verifier');

  const discovery = await fetch(`${IDP_BASE}/.well-known/openid-configuration`).then(r => r.json());

  const tokenRes = await fetch(discovery.token_endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      code: code!,
      redirect_uri: import.meta.env.VITE_REDIRECT_URI,
      client_id: import.meta.env.VITE_CLIENT_ID,
      client_secret: import.meta.env.VITE_CLIENT_SECRET,
      code_verifier: codeVerifier,
    }),
  });

  if (!tokenRes.ok) throw new Error('Token request failed');
  const tokenSet = await tokenRes.json();

  sessionStorage.setItem('access_token', tokenSet.access_token);
  sessionStorage.setItem('id_token', tokenSet.id_token);
  if (tokenSet.refresh_token) sessionStorage.setItem('refresh_token', tokenSet.refresh_token);

  return tokenSet;
}
```

### 3.3 UserInfo 取得

```typescript
export async function fetchUserInfo() {
  const accessToken = sessionStorage.getItem('access_token');
  if (!accessToken) throw new Error('Not authenticated');

  const discovery = await fetch(`${IDP_BASE}/.well-known/openid-configuration`).then(r => r.json());

  const res = await fetch(discovery.userinfo_endpoint, {
    headers: { Authorization: `Bearer ${accessToken}` },
  });

  if (!res.ok) throw new Error('UserInfo request failed');
  return res.json(); // { sub, email, name, ... }
}
```

### 3.4 ID トークンの検証（推奨）

本 IdP の ID トークンは **RS256** で署名されています。RP 側で検証するには、`jwks_uri` から公開鍵を取得します。

```typescript
import * as jose from 'jose';

async function verifyIdToken(idToken: string) {
  const discovery = await fetch(`${IDP_BASE}/.well-known/openid-configuration`).then(r => r.json());
  const JWKS = jose.createRemoteJWKSet(new URL(discovery.jwks_uri));

  const { payload } = await jose.jwtVerify(idToken, JWKS, {
    issuer: IDP_BASE,
    audience: import.meta.env.VITE_CLIENT_ID,
  });

  return payload;
}
```

## 4. セキュリティ要件

| 項目 | 対応 |
|---|---|
| **PKCE** | `code_challenge_method=S256` で必須送信 |
| **HTTPS** | 開発環境では自己署名証明書可 |
| **state** | CSRF 対策として必須（推奨） |
| **トークン有効期限** | Access=15分 / Refresh=7日 |

## 5. 環境変数例（`.env`）

```
VITE_IDP_BASE_URL=http://localhost:3000
VITE_CLIENT_ID=demo-rp-001
VITE_CLIENT_SECRET=your-client-secret
VITE_REDIRECT_URI=http://localhost:5173/callback
```
