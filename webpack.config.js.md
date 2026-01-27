# Module Federation の `exposes` の仕組み

マイクロフロントエンドの核心部分ですね。`exposes`は**他のアプリケーションに公開するモジュール**を定義する設定です。

## 基本的な概念

Module Federationでは、アプリケーションを2つの役割に分けます：

- **Host（ホスト）**: 他のアプリケーションのモジュールを使う側
- **Remote（リモート）**: 自分のモジュールを他に提供する側

`exposes`は**Remote側**で使う設定です。

## 具体例で理解する

### Remote側（モジュールを提供する）

```javascript
// apps/product-catalog/webpack.config.js
const { withModuleFederationPlugin } = require('@angular-architects/module-federation/webpack');

module.exports = withModuleFederationPlugin({
  name: 'productCatalog',  // このRemoteの名前
  
  // ★ exposes: 外部に公開するモジュール
  exposes: {
    // キー: 公開名（他のアプリから参照する名前）
    // 値: 実際のファイルパス
    './ProductModule': './src/app/product/product.module.ts',
    './ProductList': './src/app/product/product-list.component.ts',
  },
  
  shared: {
    '@angular/core': { singleton: true },
    '@angular/common': { singleton: true },
  }
});
```

### Host側（モジュールを使う）

```javascript
// apps/main-shell/webpack.config.js
const { withModuleFederationPlugin } = require('@angular-architects/module-federation/webpack');

module.exports = withModuleFederationPlugin({
  // ★ remotes: どのRemoteを使うか宣言
  remotes: {
    // キー: このアプリ内での参照名
    // 値: Remote名@URL
    productCatalog: 'http://localhost:4201/remoteEntry.js',
  },
  
  shared: {
    '@angular/core': { singleton: true },
    '@angular/common': { singleton: true },
  }
});
```

### Hostから使う

```typescript
// apps/main-shell/src/app/app.routes.ts
import { Routes } from '@angular/router';
import { loadRemoteModule } from '@angular-architects/module-federation';

export const routes: Routes = [
  {
    path: 'products',
    // ★ Remote側の exposes で定義したモジュールを読み込む
    loadChildren: () =>
      loadRemoteModule({
        type: 'module',
        remoteEntry: 'http://localhost:4201/remoteEntry.js',
        exposedModule: './ProductModule'  // ← exposes のキーと一致
      }).then(m => m.ProductModule)
  }
];
```

または動的importで：

```typescript
// Component内で動的に読み込む
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `<button (click)="loadProduct()">商品を表示</button>`
})
export class AppComponent {
  async loadProduct() {
    // ★ productCatalog/ProductList を読み込む
    const { ProductListComponent } = await import('productCatalog/ProductList');
    // コンポーネントを動的に生成して使う
  }
}
```

## `exposes` の詳細な仕組み

### 1. ビルド時の動作

```javascript
exposes: {
  './ProductModule': './src/app/product/product.module.ts',
}
```

このように設定すると、Webpackは：

1. **エントリーポイント作成**: `ProductModule`用の独立したJavaScriptチャンクを生成
2. **remoteEntry.js生成**: すべてのexposesを管理するマニフェストファイル
3. **非同期ロード可能化**: Host側から動的に読み込めるようにする

### 2. 生成されるファイル構造

```
dist/product-catalog/
├── remoteEntry.js           # ← これが重要！すべてのexposesの入口
├── main.js
├── src_app_product_product_module_ts.js  # ← ProductModule用チャンク
└── ...
```

### 3. remoteEntry.jsの中身（簡略版）

```javascript
// remoteEntry.js（概念的なコード）
const moduleMap = {
  './ProductModule': () => {
    return import('./src_app_product_product_module_ts.js')
      .then(module => module.ProductModule);
  },
  './ProductList': () => {
    return import('./src_app_product_product_list_component_ts.js')
      .then(module => module.ProductListComponent);
  }
};

// Host側からリクエストされたら、該当モジュールを返す
window.productCatalog = {
  get: (module) => moduleMap[module](),
  init: (shared) => { /* 共有モジュールの初期化 */ }
};
```

## 実践的な設定パターン

### パターン1: モジュール全体を公開

```javascript
exposes: {
  './ProductModule': './src/app/product/product.module.ts',
}
```

**用途**: 遅延ロードルーティング

```typescript
// Host側
{
  path: 'products',
  loadChildren: () => import('productCatalog/ProductModule')
    .then(m => m.ProductModule)
}
```

### パターン2: 個別コンポーネントを公開

```javascript
exposes: {
  './ProductCard': './src/app/components/product-card.component.ts',
  './ProductDetail': './src/app/components/product-detail.component.ts',
}
```

**用途**: 特定のコンポーネントだけを使いたい場合

```typescript
// Host側
const { ProductCardComponent } = await import('productCatalog/ProductCard');
```

### パターン3: サービスやユーティリティを公開

```javascript
exposes: {
  './ProductService': './src/app/services/product.service.ts',
  './utils': './src/app/shared/utils.ts',
}
```

**用途**: ビジネスロジックの共有

```typescript
// Host側
const { ProductService } = await import('productCatalog/ProductService');
```

## 実際の開発での注意点

### 1. パス解決の設定（TypeScript）

```json
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      // Remote のモジュールを TypeScript に認識させる
      "productCatalog/*": ["projects/product-catalog/src/decl.d.ts"]
    }
  }
}
```

```typescript
// decl.d.ts（型定義）
declare module 'productCatalog/ProductModule' {
  export class ProductModule {}
}

declare module 'productCatalog/ProductList' {
  export class ProductListComponent {}
}
```

### 2. 共有依存関係（shared）

```javascript
// Remote と Host の両方で同じ設定が必要
shared: {
  '@angular/core': {
    singleton: true,      // 1つのインスタンスのみ
    strictVersion: true,  // バージョン厳密チェック
    requiredVersion: '^17.0.0'
  },
  '@angular/common': {
    singleton: true,
    strictVersion: true,
    requiredVersion: '^17.0.0'
  }
}
```

**重要**: Remote と Host で Angular のバージョンが異なると、実行時エラーになります。

### 3. 実行時のエラーハンドリング

```typescript
// Host側での安全な読み込み
async loadRemoteModule() {
  try {
    const module = await import('productCatalog/ProductModule');
    return module.ProductModule;
  } catch (error) {
    console.error('Remote module loading failed:', error);
    // フォールバック処理
    return null;
  }
}
```

## コンピュータサイエンス的な視点

Module Federation は、以下の概念を組み合わせた仕組みです：

1. **動的モジュールローダー**: require.jsやSystemJSと同様の仕組み
2. **依存関係グラフの分散管理**: 各Remoteが独立した依存関係グラフを持つ
3. **コード分割（Code Splitting）**: チャンク単位での遅延ロード
4. **サービスレジストリパターン**: `remoteEntry.js`がモジュールのレジストリとして機能

```
┌─────────────┐
│   Host      │
│  (Shell)    │
└──────┬──────┘
       │
       │ HTTP Request: remoteEntry.js
       ↓
┌─────────────────────────────┐
│  Remote (productCatalog)    │
│                             │
│  remoteEntry.js             │
│  ├─ ./ProductModule  ──→ Chunk A
│  ├─ ./ProductList    ──→ Chunk B
│  └─ ./ProductService ──→ Chunk C
└─────────────────────────────┘
```

## 公式リソース

- **Webpack Module Federation**: https://webpack.js.org/concepts/module-federation/
- **Angular Architects (Module Federation for Angular)**: https://www.angulararchitects.io/en/blog/multi-framework-and-version-micro-frontends-with-module-federation-your-4-step-guide/
- **Module Federation Examples**: https://github.com/module-federation/module-federation-examples

## まとめ

```javascript
exposes: {
  './PublicName': './src/actual/path/to/module.ts'
}
```

- **`./PublicName`**: 他のアプリから`import('remoteName/PublicName')`で参照する名前
- **`./src/actual/path/to/module.ts`**: 実際のファイルパス
- **目的**: マイクロフロントエンド間でコードを共有する仕組み

あなたのプロジェクトで具体的にどのモジュールを公開しているか、`webpack.config.js`の内容を見せていただければ、より詳しく解説できます！
