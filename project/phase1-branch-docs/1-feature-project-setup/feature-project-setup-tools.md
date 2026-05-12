# feature/project-setup/tools

## 目的

プロジェクト全体の品質担保ツール（Lefthook, RuboCop, ESLint, oxfmt）を設定する。

## ブランチ名

`feature/project-setup/tools`

## 親ブランチ

`feature/project-setup/init`

## 作成・変更するファイル

### `lefthook.yml`

```yaml
pre-commit:
  commands:
    rubocop:
      glob: "*.{rb,gemfile}"
      run: bundle exec rubocop --force-exclusion {staged_files}
    eslint-frontend:
      root: frontend/
      glob: "*.{ts,vue}"
      run: pnpm eslint --fix {staged_files}
    eslint-demo-rp:
      root: demo-rp/
      glob: "*.{ts,vue}"
      run: pnpm eslint --fix {staged_files}

pre-push:
  commands:
    rspec-auth:
      root: idp-auth/
      run: bundle exec rspec
    rspec-user:
      root: idp-user/
      run: bundle exec rspec
    rspec-client:
      root: idp-client/
      run: bundle exec rspec
```

#### なぜこの設定か

- **Lefthook を選定した理由**：Git フックマネージャの中で Go 製で高速であり、monorepo 構成でも `root` 指定でサブディレクトリごとにコマンドを実行できる。Husky は Node 依存が強く、Ruby プロジェクトとの親和性が低いため採用しない。
- **`pre-commit` で lint のみ**：コミット前は変更ファイルのみを対象にして高速化。 staged_files を使うことで、未ステージのファイルに影響されず、コミット単位で品質を担保する。
- **`pre-push` で RSpec**：プッシュ前に全テストを実行し、壊れたコードをリモートに送らないようにする。lint はコミット時に済ませているため、プッシュ時はテストに集中させ、待ち時間を最小化するバランス。
- **`--force-exclusion`**：RuboCop が `.rubocop.yml` の `Exclude` を無視することがあるため、明示的に除外を強制。monorepo では `vendor/bundle` などが誤検出しやすい。

### `.rubocop.yml`

```yaml
AllCops:
  TargetRubyVersion: 3.3
  Exclude:
    - '*/db/schema.rb'
    - '*/vendor/**/*'
    - '*/tmp/**/*'
    - '*/bin/**/*'

Style/Documentation:
  Enabled: false

Metrics/BlockLength:
  Exclude:
    - '*/spec/**/*'
    - '*/config/routes.rb'
```

#### なぜこの設定か

- **`TargetRubyVersion: 3.3`**：Gemfile で固定した Ruby バージョンと一致させ、新しい構文（パターンマッチングなど）を誤って古いスタイルに書き換えられないようにする。
- **`*/` プレフィックス**：monorepo 配下の各 Rails サービスに共通適用するため。個別にサービス名を書くとサービス追加時に設定漏れが発生する。
- **`Style/Documentation: false`**：API モードの Rails ではコントローラー・モデルのクラス説明を必須にすると、短いクラスでも冗長なコメントを強制されて生産性が落ちる。必要な箇所のみ任意で記述するスタイルを採用。
- **`Metrics/BlockLength` の除外**：RSpec は DSL でネストが深くなるため、ブロック長の警告が大量に出る。`routes.rb` も同様に Rails DSL なので除外。

### `frontend/.eslintrc.cjs`

```javascript
module.exports = {
  root: true,
  env: { browser: true, es2021: true },
  extends: [
    'eslint:recommended',
    'plugin:vue/vue3-recommended',
    'plugin:@typescript-eslint/recommended',
  ],
  parser: 'vue-eslint-parser',
  parserOptions: {
    parser: '@typescript-eslint/parser',
    sourceType: 'module',
  },
  plugins: ['@typescript-eslint'],
  rules: {
    'vue/multi-word-component-names': 'off',
  },
}
```

#### なぜこの設定か

- **`root: true`**：monorepo 内で ESLint が親ディレクトリの設定を誤って継承しないようにし、各フロントエンドプロジェクトの設定を独立させる。
- **`plugin:vue/vue3-recommended`**：Vue 3 の Composition API と `<script setup>` を正しく解析するための推奨ルールセット。`vue3-essential` では不十分で、`vue3-strongly-recommended` まで入れると初期段階で厳しすぎるため、推奨（recommended）を採用。
- **`vue/multi-word-component-names: off`**：トップレベルの `App.vue` やデモ用の短いコンポーネントで「複数単語必須」の警告が出るのを防ぐ。本格的な UI 構築時には個別に有効化してもよいが、雛形段階では無効にしておく。

### `frontend/.oxfmt.json`

```json
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "es5"
}
```

#### なぜこの設定か

- **`semi: false`**：Vue / TypeScript コミュニティでは ASI（自動セミコロン挿入）を信頼して省略するスタイルが広く使われており、見た目がすっきりする。チーム内で統一されていれば可読性に問題はない。
- **`singleQuote: true`**：Ruby（シングルクォートが主流）と JavaScript / Vue の見た目を近づけ、フルスタック開発時の文脈切り替えコストを下げる。
- **`trailingComma: es5`**：配列・オブジェクトの末尾カンマを許可し、Git diff を行単位で綺麗に保つ。`all` にすると関数引数まで付与され、他のツールと衝突しやすいため `es5`（オブジェクト・配列のみ）を採用。

### `demo-rp/.eslintrc.cjs`

（`frontend/.eslintrc.cjs` と同様）

### `demo-rp/.oxfmt.json`

（`frontend/.oxfmt.json` と同様）

## マージ基準（チェックリスト）

- [ ] RuboCop が各 Rails サービスで実行できる
- [ ] ESLint / oxfmt が `frontend/` で実行できる
- [ ] ESLint / oxfmt が `demo-rp/` で実行できる
- [ ] `lefthook install` 後、pre-commit フックが動作する

## 備考

Lefthook のインストールは各開発者がローカルで `lefthook install` を実行する必要がある。CI では `lefthook run pre-commit` を実行する想定。
