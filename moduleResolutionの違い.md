# TypeScript の `moduleResolution` の違い

TypeScriptの`moduleResolution`は、import文でモジュールを解決する際の戦略を決定する重要な設定です。`node`と`bundler`では、モジュール解決のアルゴリズムが根本的に異なります。

## `node` (Classic Node.js resolution)

Node.jsの従来のモジュール解決アルゴリズム（CommonJS時代の仕組み）を模倣します。

**特徴:**
- `node_modules`を再帰的に上位ディレクトリへ遡って探索
- `package.json`の`main`フィールドを参照
- 拡張子の自動補完（`.ts`, `.tsx`, `.d.ts`など）
- `index.ts`の暗黙的な解決

**解決順序の例:**
```typescript
import { foo } from 'my-module';
```

1. `node_modules/my-module/package.json`の`main`をチェック
2. `node_modules/my-module/index.ts`を探索
3. 親ディレクトリの`node_modules`へ遡る

## `bundler` (Modern bundler resolution)

Webpack、Vite、esbuildなどのモダンなバンドラーの解決戦略を前提とした設定です。TypeScript 4.7で導入されました。

**特徴:**
- `package.json`の`exports`フィールドを優先的に参照（Conditional Exports対応）
- ESM（ES Modules）を前提とした解決
- バンドラーが実行時にモジュール解決を行う前提なので、TypeScriptは型チェックに専念
- 拡張子なしimportを許容（バンドラーが解決するため）
- `types`や`typings`フィールドの柔軟な解決

**`exports`フィールドの例:**
```json
{
  "name": "my-library",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    },
    "./utils": {
      "import": "./dist/utils.mjs",
      "types": "./dist/utils.d.ts"
    }
  }
}
```

`bundler`では、このようなConditional Exportsを正しく解釈できます。

## 主な違いのまとめ

| 項目 | `node` | `bundler` |
|------|--------|-----------|
| 対象環境 | Node.js (CommonJS中心) | モダンバンドラー環境 |
| `package.json`参照 | `main`フィールド優先 | `exports`フィールド優先 |
| ESM対応 | 限定的 | フル対応 |
| 拡張子補完 | あり | バンドラー依存 |
| Conditional Exports | 非対応 | 対応 |

## Angularにおける推奨

Angularプロジェクトでは、Angular CLI（内部でWebpackやesbuildを使用）を使っているため、**`bundler`が推奨**されます。特にAngular 15以降ではesbuildベースのビルダーが標準となっているため、`bundler`との相性が良いです。

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler",
    "module": "ESNext"
  }
}
```

## 公式ドキュメント

- **TypeScript Handbook - Module Resolution**: https://www.typescriptlang.org/docs/handbook/module-resolution.html
- **TSConfig Reference - moduleResolution**: https://www.typescriptlang.org/tsconfig#moduleResolution
- **TypeScript 4.7 Release Notes (bundler導入)**: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-7.html#packagejson-exports-imports-and-self-referencing

これらのドキュメントでは、各解決戦略のアルゴリズム詳細やエッジケースについても記載されています。

コンピュータサイエンスの観点から見ると、モジュール解決は**グラフ探索アルゴリズム**（依存関係グラフのトラバース）と**ファイルシステムの木構造探索**の組み合わせです。`node`は深さ優先探索的に上位ディレクトリを遡り、`bundler`はより宣言的なマニフェスト（`exports`）を優先する戦略を取っています。




**結論から言うと、基本的にはソースコードの変更は不要です。**

`moduleResolution`はTypeScriptの**型チェック時のモジュール解決戦略**を変える設定であり、トランスパイル後のJavaScriptコードそのものには影響しません。実際のランタイム動作はバンドラー（Webpack、esbuildなど）が制御するためです。

## ソースコード変更が不要な理由

```typescript
// このコードは moduleResolution が node でも bundler でも変わらない
import { Component } from '@angular/core';
import { MyService } from './my-service';
```

上記のようなコードは、設定を変えても**そのまま動きます**。なぜなら：

1. **型チェックだけが変わる**: TypeScriptコンパイラがモジュールの型定義を探す方法が変わるだけ
2. **実際の解決はバンドラーが行う**: Angular CLIが使うWebpack/esbuildが実行時に解決
3. **出力されるJSは同じ**: トランスパイル結果のJavaScriptコードは変わらない

## ただし、例外的にコード変更が必要になるケース

### 1. 拡張子付きimportを使っている場合

```typescript
// node では許容されるが、bundler では警告が出る可能性
import { foo } from './module.ts';  // ← .ts 拡張子付き

// どちらでも問題ない書き方
import { foo } from './module';     // ← 拡張子なし
```

### 2. package.jsonの`exports`フィールドに依存している場合

```typescript
// bundler: exports フィールドから解決される
import { util } from 'my-lib/utils';

// node: main フィールドしか見ないため、エラーになる可能性
```

この場合でも、**ソースコード自体は変更不要**で、依存ライブラリの`package.json`の問題です。

## あなたの状況での推奨対応

### 1. tsconfig.jsonの統一（コード変更なし）

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler",  // または "node16"/"nodenext"
    "module": "ESNext"
  }
}
```

これを各マイクロフロントエンドで統一すれば、**既存コードに触れずに開発環境を揃えられます**。

### 2. 検証手順

```bash
# 1. 設定変更
# tsconfig.json を編集

# 2. 型チェック
ng build --configuration=production

# 3. テスト実行
ng test --code-coverage

# エラーが出なければ、コード変更不要
```

### 3. チームへの報告内容（例）

```
【開発環境の差分について】

現状、tsconfig.jsonのmoduleResolution設定が
各プロジェクトで統一されていないため、
型チェックとテスト実行時の挙動に差分があります。

■ 影響範囲
- TypeScriptの型チェックのみ
- 実際のランタイムコードには影響なし

■ 対応内容
- tsconfig.jsonの統一
- ソースコードの変更は不要

■ 確認方法
- ビルドとテストが通ることを確認済み
- カバレッジ100%達成
```

## マイクロフロントエンド環境での注意点

Module Federationを使っている場合：

```typescript
// Remote アプリのモジュール解決
const RemoteComponent = lazy(() => import('remote/Component'));
```

このような動的importは、`moduleResolution`の影響を受けません。実行時にWebpackが解決するためです。

## まとめ

- **ソースコード変更は基本的に不要**
- `moduleResolution`は型チェックの設定であり、ランタイムには影響しない
- テストカバレッジ達成には問題なし
- チームには「開発環境の統一」として説明すればOK

既存コードが正常に動いているなら、設定を統一するだけでビルドもテストも通るはずです。もしエラーが出る場合は、そのエラー内容を共有していただければ、具体的な対処法をお伝えできます。
