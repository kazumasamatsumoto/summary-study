# `angular.json` の `assets` 設定の詳細解説

`projects.architect.build.assets`は、**ビルド時に静的ファイルをどのように扱うか**を定義する設定です。

## 基本構造

```json
{
  "projects": {
    "my-app": {
      "architect": {
        "build": {
          "options": {
            "assets": [
              // ここに設定を書く
            ]
          }
        }
      }
    }
  }
}
```

## 設定方法（2つのパターン）

### パターン1: 文字列形式（シンプル）

```json
"assets": [
  "src/favicon.ico",
  "src/assets"
]
```

**動作**:
- `src/favicon.ico` → `dist/my-app/favicon.ico` にコピー
- `src/assets/**/*` → `dist/my-app/assets/**/*` にコピー（ディレクトリ全体）

### パターン2: オブジェクト形式（詳細制御）

```json
"assets": [
  {
    "glob": "**/*",
    "input": "src/assets",
    "output": "/assets"
  },
  {
    "glob": "favicon.ico",
    "input": "src",
    "output": "/"
  }
]
```

## 各プロパティの意味

### 1. `glob` (グロブパターン)

**どのファイルを対象にするか**を指定します。

```json
{
  "glob": "**/*",        // すべてのファイル
  "glob": "*.png",       // すべてのPNGファイル
  "glob": "*.{jpg,png}", // JPGとPNGファイル
  "glob": "**/*.svg",    // すべてのディレクトリ内のSVGファイル
}
```

**グロブパターンの記法**:
- `*`: 任意の文字列（ディレクトリ区切りを除く）
- `**`: 任意の深さのディレクトリ
- `?`: 任意の1文字
- `{a,b}`: aまたはb

### 2. `input` (入力元ディレクトリ)

**どこから**ファイルを取得するかを指定します。

```json
{
  "input": "src/assets",           // プロジェクト内
  "input": "node_modules/library", // node_modulesから
  "input": "../shared-assets"      // プロジェクト外の相対パス
}
```

### 3. `output` (出力先ディレクトリ)

**ビルド後のどこに**配置するかを指定します。

```json
{
  "output": "/assets",     // dist/my-app/assets/ に配置
  "output": "/",           // dist/my-app/ 直下に配置
  "output": "/images",     // dist/my-app/images/ に配置
}
```

### 4. `ignore` (除外パターン)

**除外したいファイル**を指定します（オプション）。

```json
{
  "glob": "**/*",
  "input": "src/assets",
  "output": "/assets",
  "ignore": [
    "**/*.md",      // Markdownファイルを除外
    "**/README.*",  // READMEファイルを除外
    "**/.gitkeep"   // .gitkeepを除外
  ]
}
```

### 5. `followSymlinks` (シンボリックリンク)

シンボリックリンクをたどるか（オプション、デフォルト: `false`）。

```json
{
  "glob": "**/*",
  "input": "src/assets",
  "output": "/assets",
  "followSymlinks": true  // シンボリックリンク先もコピー
}
```

## 実践的な設定例

### 例1: 標準的なAngularアプリ

```json
"assets": [
  "src/favicon.ico",
  "src/assets",
  {
    "glob": "**/*",
    "input": "src/assets",
    "output": "/assets"
  }
]
```

**ビルド結果**:
```
dist/my-app/
├── favicon.ico          ← src/favicon.ico
├── assets/
│   ├── images/
│   │   └── logo.png    ← src/assets/images/logo.png
│   ├── data/
│   │   └── config.json ← src/assets/data/config.json
│   └── styles/
│       └── theme.css   ← src/assets/styles/theme.css
```

### 例2: node_modulesからライブラリのアセットをコピー

```json
"assets": [
  "src/favicon.ico",
  "src/assets",
  {
    "glob": "**/*.{js,css}",
    "input": "node_modules/some-library/dist",
    "output": "/vendor/some-library"
  }
]
```

**ビルド結果**:
```
dist/my-app/
├── vendor/
│   └── some-library/
│       ├── library.js   ← node_modules/some-library/dist/library.js
│       └── library.css  ← node_modules/some-library/dist/library.css
```

### 例3: 環境別の設定ファイル

```json
"assets": [
  "src/favicon.ico",
  "src/assets",
  {
    "glob": "config.json",
    "input": "src/environments/production",
    "output": "/config"
  }
]
```

**ビルド結果**:
```
dist/my-app/
├── config/
│   └── config.json  ← src/environments/production/config.json
```

### 例4: 画像ファイルだけを別フォルダに

```json
"assets": [
  {
    "glob": "**/*.{png,jpg,jpeg,gif,svg}",
    "input": "src/assets/images",
    "output": "/images"
  },
  {
    "glob": "**/*",
    "input": "src/assets",
    "output": "/assets",
    "ignore": ["**/*.{png,jpg,jpeg,gif,svg}"]
  }
]
```

**ビルド結果**:
```
dist/my-app/
├── images/           ← 画像ファイルのみ
│   ├── logo.png
│   └── banner.jpg
├── assets/           ← 画像以外
│   ├── data/
│   └── styles/
```

### 例5: マイクロフロントエンドでの共有アセット

```json
"assets": [
  "src/favicon.ico",
  "src/assets",
  {
    "glob": "**/*",
    "input": "../shared-library/assets",
    "output": "/shared-assets"
  }
]
```

**ビルド結果**:
```
dist/my-app/
├── shared-assets/    ← 他のプロジェクトと共有
│   ├── fonts/
│   └── icons/
```

## よくある使用ケース

### ケース1: PWA（Progressive Web App）

```json
"assets": [
  "src/favicon.ico",
  "src/assets",
  "src/manifest.webmanifest"  // PWAマニフェスト
]
```

### ケース2: i18n（多言語対応）

```json
"assets": [
  "src/favicon.ico",
  "src/assets",
  {
    "glob": "*.json",
    "input": "src/i18n",
    "output": "/i18n"
  }
]
```

**ビルド結果**:
```
dist/my-app/
├── i18n/
│   ├── en.json
│   ├── ja.json
│   └── zh.json
```

### ケース3: サードパーティライブラリのフォント

```json
"assets": [
  "src/favicon.ico",
  "src/assets",
  {
    "glob": "**/*.{woff,woff2,ttf,eot}",
    "input": "node_modules/@fortawesome/fontawesome-free/webfonts",
    "output": "/webfonts"
  }
]
```

## 注意点とベストプラクティス

### 1. パフォーマンスへの影響

```json
// ❌ 避けるべき：node_modules全体をコピー
"assets": [
  {
    "glob": "**/*",
    "input": "node_modules",
    "output": "/vendor"
  }
]

// ✅ 推奨：必要なファイルだけを指定
"assets": [
  {
    "glob": "*.{js,css}",
    "input": "node_modules/specific-library/dist",
    "output": "/vendor/specific-library"
  }
]
```

### 2. セキュリティ

```json
// ✅ 機密情報を含むファイルは除外
"assets": [
  {
    "glob": "**/*",
    "input": "src/assets",
    "output": "/assets",
    "ignore": [
      "**/*.env",           // 環境変数ファイル
      "**/*.key",           // 秘密鍵
      "**/secrets/**"       // 機密情報フォルダ
    ]
  }
]
```

### 3. ビルド時間の最適化

```json
// 開発環境では最小限
"configurations": {
  "development": {
    "assets": [
      "src/favicon.ico",
      "src/assets"
    ]
  },
  // 本番環境で追加アセット
  "production": {
    "assets": [
      "src/favicon.ico",
      "src/assets",
      {
        "glob": "**/*",
        "input": "node_modules/library/dist",
        "output": "/vendor"
      }
    ]
  }
}
```

## コンピュータサイエンス的な仕組み

この設定は、内部的に以下のように動作します：

1. **ファイルシステムの走査**: `glob`パターンで指定されたファイルを探索
2. **ファイルツリーの構築**: `input`から`output`へのマッピングを作成
3. **コピー処理**: ビルド時に指定されたファイルを`dist`ディレクトリにコピー
4. **ハッシュ計算**: （オプション）ファイル内容のハッシュ値でキャッシュバスティング

```
入力ディレクトリ（input）
    ↓ glob パターンでフィルタ
対象ファイルリスト
    ↓ ignore パターンで除外
コピー対象リスト
    ↓ output パスに配置
dist ディレクトリ
```

## 確認方法

ビルド後にどのファイルがコピーされたか確認：

```bash
# 本番ビルド
ng build --configuration=production

# コピーされたファイルを確認
ls -R dist/my-app/assets

# 特定のファイルが含まれているか検索
find dist/my-app -name "*.json"
```

## 公式ドキュメント

- **Angular Workspace Configuration**: https://angular.dev/reference/configs/workspace-config
- **Angular CLI - Assets**: https://angular.dev/tools/cli/build#assets

## まとめ

```json
{
  "glob": "**/*",           // どのファイルを
  "input": "src/assets",    // どこから取得し
  "output": "/assets",      // どこに配置するか
  "ignore": ["**/*.md"]     // 何を除外するか
}
```

- **`assets`**: ビルド時に静的ファイルをコピーする設定
- **用途**: 画像、フォント、設定ファイル、サードパーティライブラリなど
- **ポイント**: 必要なファイルだけを指定してビルド時間とバンドルサイズを最適化

あなたのプロジェクトの`angular.json`の`assets`設定を見せていただければ、具体的にどのような処理が行われているか詳しく解説できます！
