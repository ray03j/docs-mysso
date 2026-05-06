# feature/project-setup/demo-rp

## 目的

デモ用 Relying Party `demo-rp` の Vue 3 SPA 雛形を作成する。ポート5174で動作。

## ブランチ名

`feature/project-setup/demo-rp`

## 親ブランチ

`feature/project-setup/init`

## 作成するファイル

### `demo-rp/package.json`

```json
{
  "name": "demo-rp",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite --host --port 5174",
    "build": "vue-tsc && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext .vue,.ts,.tsx --fix",
    "format": "prettier --write ."
  },
  "dependencies": {
    "vue": "^3.4"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.0",
    "typescript": "^5.3",
    "vite": "^5.0",
    "vue-tsc": "^1.8",
    "eslint": "^8.57",
    "eslint-plugin-vue": "^9.24",
    "@typescript-eslint/eslint-plugin": "^7.5",
    "@typescript-eslint/parser": "^7.5",
    "prettier": "^3.2"
  }
}
```

### `demo-rp/vite.config.ts`

```typescript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    port: 5174,
    host: true,
  },
})
```

### `demo-rp/index.html`

```html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Demo RP</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

### `demo-rp/.env.example`

```env
VITE_IDP_BASE_URL=http://localhost:3000
VITE_CLIENT_ID=demo_client
```

### `demo-rp/src/main.ts`

```typescript
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)
app.mount('#app')
```

### `demo-rp/src/App.vue`

```vue
<template>
  <div id="app">
    <h1>Demo Relying Party</h1>
  </div>
</template>

<script setup lang="ts">
</script>
```

## マージ基準（チェックリスト）

- [ ] `docker compose up demo-rp` で デモRP（ポート5174）が起動する
- [ ] `npm run lint` が実行できる
- [ ] `npm run format` が実行できる

## 備考

認可フロー実装は Phase 3 で行う。本ブランチでは最小構成の Vue アプリのみ作成する。
