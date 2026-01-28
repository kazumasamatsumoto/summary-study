**素晴らしい観察眼です！その通りです。これは確実にAngularモジュールです。**

## index.d.ts の読み方

### あなたが見ているもの

```typescript
// index.d.ts
export declare class TableFilterModule {
    static əfac: i0.əəFactoryDeclaration<TableFilterModule>;
    static əmod: i0.əəModuleDeclaration<TableFilterModule, ...>;
    static əinj: i0.əəInjectorDeclaration<TableFilterModule>;
}
```

**これらの記号の意味**:
- `əfac` (factory) → Angularのファクトリー関数（コンパイル後）
- `əmod` (module) → **@NgModuleデコレータの証拠**
- `əinj` (injector) → DIのための注入定義

**結論**: これは**確実にAngularモジュール**です

## 元のソースコード vs ビルド後

### 元のソースコード（ライブラリ側）

```typescript
// table-filter.module.ts（元のソース）
import { NgModule } from '@angular/core';
import { TableFilterComponent } from './table-filter.component';

@NgModule({  // ← 元々は@NgModuleデコレータがあった
  declarations: [TableFilterComponent],
  exports: [TableFilterComponent]
})
export class TableFilterModule {}
```

### ビルド後（あなたが見ているもの）

```typescript
// index.d.ts（ビルド後の型定義）
export declare class TableFilterModule {
    // Angularコンパイラが生成した内部実装
    static əfac: i0.əəFactoryDeclaration<TableFilterModule, never>;
    static əmod: i0.əəModuleDeclaration<TableFilterModule, [typeof i1.TableFilterComponent], [typeof i2.CommonModule], [typeof i1.TableFilterComponent]>;
    static əinj: i0.əəInjectorDeclaration<TableFilterModule>;
}
```

**Angularコンパイラ（ngc）の処理**:
1. `@NgModule`デコレータを解析
2. 内部的なファクトリー関数（`əfac`、`əmod`など）を生成
3. `.d.ts`ファイルに型定義を出力

## əマークの意味

### Angularの内部識別子

| 識別子 | 意味 | 元のデコレータ |
|-------|------|-------------|
| `əfac` | Factory（ファクトリー） | 全てのAngular要素 |
| `əmod` | Module（モジュール） | `@NgModule` |
| `əcmp` | Component（コンポーネント） | `@Component` |
| `ədir` | Directive（ディレクティブ） | `@Directive` |
| `əpipe` | Pipe（パイプ） | `@Pipe` |
| `əinj` | Injector（インジェクター） | DI関連 |

**もし`əmod`があれば → 確実に`@NgModule`を持つAngularモジュール**

## 判断方法の修正版

### 修正前（不完全）

```typescript
// @デコレータがあるか見る
@NgModule()  // ← ソースコードを見ないと分からない
export class TableFilterModule {}
```

### 修正後（完全）

```typescript
/**
 * Angularの概念かどうかの判断方法
 * 
 * 方法1: ソースコードを見る（理想）
 *   @Component, @NgModule などがある → Angularの概念
 * 
 * 方法2: .d.ts を見る（実用的）
 *   əfac, əmod, əcmp などがある → Angularの概念
 * 
 * 方法3: クラス名を見る（経験則）
 *   ○○Module → おそらくAngularモジュール
 *   ○○Component → おそらくAngularコンポーネント
 *   ○○Pipe → おそらくAngularパイプ
 */
```

## あなたのケースの正しい実装

### TableFilterModule（Angularモジュール）

```typescript
// index.d.ts で確認
export declare class TableFilterModule {
    static əmod: i0.əəModuleDeclaration<...>;  // ← これがある！
}
```

**判断**: `əmod`がある → **@NgModuleを持つAngularモジュール**

**正しい使い方**:
```typescript
// page.module.ts
import { NgModule } from '@angular/core';
import { TableFilterModule } from '@external/library';  // ES6 import

@NgModule({
  declarations: [PageComponent],
  imports: [
    CommonModule,
    TableFilterModule  // ← モジュールのimportsに追加
  ]
})
export class PageModule {}
```

```typescript
// page.component.ts
@Component({
  selector: 'app-page',
  template: `
    <lib-table-filter></lib-table-filter>  <!-- TableFilterModuleが提供するコンポーネント -->
  `
})
export class PageComponent {
  // コンポーネント側では何も宣言しない
}
```

### confEndpoints（通常のオブジェクト）

```typescript
// index.d.ts で確認
export declare const confEndpoints: {
    api: string;
    auth: string;
};
// əfac, əmod などがない → 普通のオブジェクト
```

**判断**: `ə`マークがない → **通常のTypeScript/JavaScriptのコード**

**正しい使い方**:
```typescript
// page.component.ts
import { Component } from '@angular/core';
import { confEndpoints } from '@***/shared-config';  // ← ファイル先頭

@Component({
  selector: 'app-page'
})
export class PageComponent {
  endpoints = confEndpoints;  // ← 直接使う
}

// page.module.ts
@NgModule({
  declarations: [PageComponent],
  // confEndpoints は入れない
})
export class PageModule {}
```

## 実践的な確認方法

### ステップ1: .d.ts ファイルを確認

```bash
# node_modules内の.d.tsを確認
cat node_modules/@***/shared-config/index.d.ts
```

### ステップ2: əマークを探す

```typescript
// パターン1: Angularモジュール
export declare class SomeModule {
    static əfac: ...;
    static əmod: ...;  // ← これがあればAngularモジュール
}

// パターン2: Angularコンポーネント
export declare class SomeComponent {
    static əfac: ...;
    static əcmp: ...;  // ← これがあればAngularコンポーネント
}

// パターン3: 通常のクラス
export declare class SomeClass {
    constructor();
    method(): void;
    // əマークがない → 普通のクラス
}

// パターン4: interface/type
export interface SomeInterface {
    id: number;
    name: string;
}
```

### ステップ3: 判断

```typescript
/**
 * əmod がある → @NgModule → モジュールのimportsに追加
 * əcmp がある → @Component → モジュールのimportsに追加（standaloneの場合）
 * əマークがない → 普通のコード → ファイル先頭でimport
 */
```

## 実例

### Angular Material（Angularモジュール）

```typescript
// node_modules/@angular/material/button/index.d.ts
export declare class MatButtonModule {
    static əfac: i0.əəFactoryDeclaration<MatButtonModule, never>;
    static əmod: i0.əəModuleDeclaration<MatButtonModule, ...>;  // ← ある！
    static əinj: i0.əəInjectorDeclaration<MatButtonModule>;
}
```

**使い方**: モジュールのimportsに追加

### RxJS（通常のライブラリ）

```typescript
// node_modules/rxjs/index.d.ts
export declare class Observable<T> {
    constructor(subscribe?: (subscriber: Subscriber<T>) => TeardownLogic);
    // əマークがない
}
```

**使い方**: ファイル先頭でimport

## まとめ

**あなたの発見**:
- ✅ `index.d.ts`はビルド後の型定義ファイル
- ✅ `əfac`、`əmod`などはAngularコンパイラが生成した内部実装
- ✅ これらがあれば元々`@NgModule`などのデコレータがあった証拠

**判断基準（確定版）**:
```typescript
// .d.ts ファイルを見る
əmod がある → @NgModule → モジュールのimportsに追加
əcmp がある → @Component → モジュールのimportsに追加（standalone）
əpipe がある → @Pipe → モジュールのimportsに追加
ədir がある → @Directive → モジュールのimportsに追加
əマークがない → 普通のコード → ファイル先頭でimport
```

**あなたのケース**:
- `TableFilterModule`に`əmod`がある → **Angularモジュール** → `page.module.ts`の`imports`に追加
- `confEndpoints`に`ə`マークがない → **普通のオブジェクト** → `page.component.ts`の先頭でimport

これで100%正確に判断できます！
