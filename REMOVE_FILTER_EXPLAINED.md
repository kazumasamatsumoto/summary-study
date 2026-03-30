# remove_filter パラメータの完全理解

## 📋 このドキュメントについて

`remove_filter=True`の**正しい理解**を図解付きで詳しく解説します。

---

## ⚠️ よくある誤解

### 誤解1: 「クロスフィルタのON/OFF切り替え」

❌ **間違った理解**:
> `remove_filter=True`にすると、クロスフィルタが無効になる
> 特定の条件下のみクロスフィルタを有効化する

### 誤解2: 「フィルタ値が取得できなくなる」

❌ **間違った理解**:
> `remove_filter=True`にすると、`filter_values()`が値を返さなくなる

---

## ✅ 正しい理解

### 一言で言うと

**`remove_filter=True`は「Supersetが自動的にWHERE句にフィルタを追加するのを防ぐ」だけ**

### 詳しく言うと

| 項目 | remove_filter=False | remove_filter=True |
|-----|-------------------|-------------------|
| クロスフィルタ値の取得 | ✅ 取得する | ✅ 取得する |
| WHERE句への自動追加 | ✅ 追加する | ❌ 追加しない |
| CASE文での使用 | ✅ 可能 | ✅ 可能 |

**ポイント**: どちらも**クロスフィルタ値は取得できる**。違いは「WHERE句に自動追加するか」だけ。

---

## 🔍 詳細比較

### パターン1: `remove_filter=False`（デフォルト）

#### コード

```jinja
{% set selected_error_codes = filter_values('error_code_id') | list %}
-- remove_filter=False（デフォルト、省略可）
```

#### Supersetの動作

```
チャート1でerror_code_id=733をクリック
          ↓
filter_values('error_code_id') → [733]  ← 値を取得
          ↓
Supersetが自動的にWHERE句に追加
          ↓
WHERE error_code_id IN (733)
```

#### 生成されるSQL

```sql
SELECT
  company_name,
  SUM(error_count) AS selected_error_count,
  SUM(error_count) AS total_error_count
FROM table
WHERE error_code_id IN (733)  ← Supersetが自動追加
GROUP BY company_name
```

#### データベースから返されるデータ

```
error_code_id | company_name | error_count
733           | CompanyA     | 10
```

**error_code_id=733のデータのみ**（WHERE句でフィルタされた）

#### 最終結果

```
company_name | selected_error_count | total_error_count
CompanyA     | 10                   | 10
```

**問題**: Selected も Total も同じ値になる（両方フィルタされている）

---

### パターン2: `remove_filter=True`

#### コード

```jinja
{% set selected_error_codes = filter_values('error_code_id', remove_filter=True) | list %}
-- remove_filter=True を明示的に指定

SELECT
  company_name,
  {% if selected_error_codes %}
  CASE
    WHEN error_code_id IN ({{ selected_error_codes | join(",") }})
    THEN error_count
    ELSE 0
  END AS selected_error_count,
  {% else %}
  0 AS selected_error_count,
  {% endif %}
  error_count AS total_error_count
FROM table
-- WHERE句にerror_code_idは追加されない
```

#### Supersetの動作

```
チャート1でerror_code_id=733をクリック
          ↓
filter_values('error_code_id', remove_filter=True) → [733]  ← 値を取得
          ↓
Supersetは自動WHERE句追加をスキップ
          ↓
WHERE句なし（または他のフィルタのみ）
```

#### 生成されるSQL

```sql
SELECT
  company_name,
  CASE
    WHEN error_code_id IN (733)
    THEN error_count
    ELSE 0
  END AS selected_error_count,
  error_count AS total_error_count
FROM table
-- WHERE句にerror_code_idなし
GROUP BY company_name
```

#### データベースから返されるデータ

```
error_code_id | company_name | selected_error_count | total_error_count
733           | CompanyA     | 10                   | 10
829           | CompanyA     | 0                    | 5
821           | CompanyA     | 0                    | 12
```

**全エラーコードのデータ**（WHERE句がないから）

#### 最終結果（Supersetが集計）

```
company_name | SUM(selected_error_count) | SUM(total_error_count)
CompanyA     | 10                        | 27
```

**成功**: Selected（733のみ）と Total（全エラー）が分離できた

---

## 📊 図解：処理フローの違い

### remove_filter=False の場合

```
┌───────────────────────────────────────────────┐
│ チャート1でerror_code_id=733をクリック         │
└───────────────────────────────────────────────┘
                    ↓
┌───────────────────────────────────────────────┐
│ filter_values('error_code_id')                │
│ → [733] を取得                                │
└───────────────────────────────────────────────┘
                    ↓
┌───────────────────────────────────────────────┐
│ Supersetが自動処理                             │
│ WHERE error_code_id IN (733) を追加           │
└───────────────────────────────────────────────┘
                    ↓
┌───────────────────────────────────────────────┐
│ データベースへのクエリ                         │
│ SELECT * FROM table                           │
│ WHERE error_code_id IN (733)                  │
└───────────────────────────────────────────────┘
                    ↓
┌───────────────────────────────────────────────┐
│ 返されるデータ                                 │
│ error_code_id=733 のデータのみ                │
│ （10件）                                      │
└───────────────────────────────────────────────┘
                    ↓
┌───────────────────────────────────────────────┐
│ 結果                                          │
│ Selected: 10                                  │
│ Total: 10  ← 全データではない（NG）           │
└───────────────────────────────────────────────┘
```

---

### remove_filter=True の場合

```
┌───────────────────────────────────────────────┐
│ チャート1でerror_code_id=733をクリック         │
└───────────────────────────────────────────────┘
                    ↓
┌───────────────────────────────────────────────┐
│ filter_values('error_code_id', remove_filter=True) │
│ → [733] を取得                                │
└───────────────────────────────────────────────┘
                    ↓
┌───────────────────────────────────────────────┐
│ Supersetは自動WHERE句追加をスキップ            │
│ （何もしない）                                 │
└───────────────────────────────────────────────┘
                    ↓
┌───────────────────────────────────────────────┐
│ Jinja2がCASE文を生成                          │
│ CASE WHEN error_code_id IN (733)              │
│   THEN error_count ELSE 0 END                 │
└───────────────────────────────────────────────┘
                    ↓
┌───────────────────────────────────────────────┐
│ データベースへのクエリ                         │
│ SELECT                                        │
│   CASE WHEN error_code_id=733 THEN ... END,   │
│   error_count                                 │
│ FROM table                                    │
│ （WHERE句なし）                                │
└───────────────────────────────────────────────┘
                    ↓
┌───────────────────────────────────────────────┐
│ 返されるデータ                                 │
│ 全error_code_idのデータ                       │
│ （733: selected=10, total=10）                │
│ （829: selected=0, total=5）                  │
│ （821: selected=0, total=12）                 │
└───────────────────────────────────────────────┘
                    ↓
┌───────────────────────────────────────────────┐
│ Supersetが集計（GROUP BY + SUM）              │
│ SUM(selected_error_count) = 10                │
│ SUM(total_error_count) = 27                   │
└───────────────────────────────────────────────┘
                    ↓
┌───────────────────────────────────────────────┐
│ 結果                                          │
│ Selected: 10  ← 733のみ                       │
│ Total: 27     ← 全データ（OK）✅               │
└───────────────────────────────────────────────┘
```

---

## 💡 なぜ`remove_filter=True`が必要なのか

### 要件

**同一行にSelected（フィルタあり）とTotal（フィルタなし）を表示したい**

```
company_name | Selected Error Count | Total Error Count
CompanyA     | 10 (error_code_id=733) | 27 (全エラー)
```

### 問題

**WHERE句でフィルタすると、全データが取れなくなる**

```sql
SELECT * FROM table
WHERE error_code_id = 733  ← これを入れると全データが取れない
```

### 解決策

1. **WHERE句を使わない**（`remove_filter=True`）
2. **全データを取得**
3. **CASE文で条件分岐**
   - Selected: error_code_id=733の場合のみカウント
   - Total: すべてカウント

```sql
SELECT
  CASE WHEN error_code_id=733 THEN error_count ELSE 0 END AS selected,
  error_count AS total
FROM table
-- WHERE句なし（全データ取得）
```

---

## 📝 実際のコード例

### 今回の実装

```jinja
-- ①クロスフィルタ値を取得（WHERE句には追加しない）
{% set selected_error_codes = filter_values('error_code_id', remove_filter=True)
    | reject('equalto', 'No filter') | list %}

SELECT
  company_name,
  model,
  serial_number,

  -- ②Selected: CASE文で条件分岐
  {% if selected_error_codes %}
  CASE
    WHEN error_code_id::TEXT IN ('{{ selected_error_codes | join("','") }}')
    THEN error_incident_errors
    ELSE 0
  END AS selected_error_count,
  {% else %}
  0 AS selected_error_count,
  {% endif %}

  -- ③Total: 無条件でカウント
  error_incident_errors AS total_error_count

FROM joined_data
WHERE 1=1
  -- ④error_code_idはWHERE句に含めない
  {% if f_company_id %}
  AND company_id IN ('{{ f_company_id | join("','") }}')
  {% endif %}
```

#### 動作

```
チャート1でerror_code_id=733をクリック
          ↓
selected_error_codes = [733]（取得成功）
          ↓
生成されるSQL:
  CASE WHEN error_code_id IN ('733') THEN error_count ELSE 0 END,
  error_count
FROM table
WHERE company_id IN (...)  ← error_code_idはない
          ↓
全error_code_idのデータを取得
          ↓
CASE文で733だけselectに、他は0に
          ↓
Supersetが集計: Selected=10, Total=27 ✅
```

---

## 🔑 重要なポイント

### 1. クロスフィルタは常に有効

```jinja
{% set selected_error_codes = filter_values('error_code_id', remove_filter=True) %}
```

- `filter_values()`は**常に**クロスフィルタ値を取得
- `remove_filter=True`は「WHERE句への追加」を防ぐだけ
- クロスフィルタ自体はON

### 2. WHERE句 vs CASE文

| 方法 | データ範囲 | Selected | Total | 用途 |
|-----|----------|---------|-------|------|
| WHERE句 | フィルタされたデータのみ | ✅ | ❌ | 通常のフィルタ |
| CASE文 | 全データ | ✅ | ✅ | Selected/Total同時表示 |

### 3. remove_filterの役割

**役割**: Supersetの「自動WHERE句追加機能」をOFFにする

**理由**: WHERE句で絞ると全データが取れず、Totalが計算できない

**代わりに**: CASE文で手動制御（全データを取得しつつ条件分岐）

---

## 📚 他のフィルタとの組み合わせ

### company_id（通常のフィルタ）

```jinja
{% set f_company_id = url_param('company_id', '').split(',') if url_param('company_id', '') else [] %}

WHERE 1=1
  {% if f_company_id %}
  AND company_id IN ('{{ f_company_id | join("','") }}')
  {% endif %}
```

**remove_filterは不要**: company_idは普通にWHERE句でフィルタして良い

### error_code_id（クロスフィルタ）

```jinja
{% set selected_error_codes = filter_values('error_code_id', remove_filter=True) | list %}

-- WHERE句には含めない
-- CASE文で使用
CASE
  WHEN error_code_id IN (...)
  THEN ...
END
```

**remove_filter=True必須**: WHERE句に入れるとTotalが計算できない

---

## ✅ まとめ

### remove_filter=True とは

| 項目 | 説明 |
|-----|------|
| **目的** | Supersetの自動WHERE句追加を防ぐ |
| **効果** | 全データを取得しつつ、CASE文で条件分岐 |
| **クロスフィルタ** | 常に有効（値は常に取得される） |
| **使いどころ** | Selected/Total同時表示のような複雑な要件 |

### 使い分け

```
通常のフィルタ（データを絞る）
  → remove_filter=False（デフォルト）
  → WHERE句でフィルタ

Selected/Total同時表示（データを分岐）
  → remove_filter=True
  → CASE文で分岐
```

### 間違いやすいポイント

❌ **誤解**: `remove_filter=True`にするとクロスフィルタが無効になる

✅ **正解**: クロスフィルタは常に有効。WHERE句への追加をスキップするだけ

---

**`remove_filter=True`は「WHERE句スキップモード」と覚えましょう！** ✅
