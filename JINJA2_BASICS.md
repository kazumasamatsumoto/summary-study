# Jinja2テンプレート 基礎解説

## 📋 このドキュメントについて

**Jinja2とは何か？どこで使われているのか？**を初心者向けに解説します。

---

## 🤔 Jinja2とは何か

### 一言で言うと

**Jinja2**（ジンジャツー）は、**テンプレートエンジン**です。

### テンプレートエンジンとは

**「穴あき文書」に動的に値を埋め込んで、最終的な文書を生成する仕組み**

#### 身近な例：メール送信

```
件名: {{name}}様へのご案内

{{name}}様

いつもご利用ありがとうございます。
本日の予約時刻は{{time}}です。
```

↓ Jinja2が処理

```
件名: 田中様へのご案内

田中様

いつもご利用ありがとうございます。
本日の予約時刻は14:00です。
```

**ポイント**: `{{name}}`や`{{time}}`の部分が動的に置き換わる

---

## 💡 なぜテンプレートエンジンが必要なのか

### 問題: 同じようなコードを何度も書く

**例: SQLで顧客を検索**

```sql
-- 顧客ID=1の場合
SELECT * FROM customers WHERE customer_id = 1

-- 顧客ID=2の場合
SELECT * FROM customers WHERE customer_id = 2

-- 顧客ID=3の場合
SELECT * FROM customers WHERE customer_id = 3
```

毎回SQLを書き直すのは面倒...

### 解決策: テンプレート化

```sql
-- テンプレート（Jinja2使用）
SELECT * FROM customers WHERE customer_id = {{ customer_id }}
```

↓ `customer_id = 1`を渡すと

```sql
SELECT * FROM customers WHERE customer_id = 1
```

↓ `customer_id = 2`を渡すと

```sql
SELECT * FROM customers WHERE customer_id = 2
```

**ポイント**: 1つのテンプレートで、様々な値に対応できる

---

## 🔧 Jinja2の構文

### 基本的な3つの記法

| 記法 | 用途 | 例 |
|-----|------|-----|
| `{{ variable }}` | 変数の値を出力 | `{{ name }}` |
| `{% statement %}` | 制御構文（if, for等） | `{% if condition %}...{% endif %}` |
| `{# comment #}` | コメント | `{# これはコメント #}` |

### 1. 変数の出力: `{{ }}`

**テンプレート**:
```jinja
こんにちは、{{ name }}さん！
```

**変数**:
```python
name = "田中"
```

**結果**:
```
こんにちは、田中さん！
```

---

### 2. 制御構文: `{% %}`

#### 条件分岐（if文）

**テンプレート**:
```jinja
{% if age >= 20 %}
成人です
{% else %}
未成年です
{% endif %}
```

**変数**:
```python
age = 25
```

**結果**:
```
成人です
```

---

#### 繰り返し（for文）

**テンプレート**:
```jinja
{% for item in items %}
- {{ item }}
{% endfor %}
```

**変数**:
```python
items = ["りんご", "バナナ", "みかん"]
```

**結果**:
```
- りんご
- バナナ
- みかん
```

---

### 3. フィルタ: `{{ variable | filter }}`

変数に対して**加工処理**を適用

**テンプレート**:
```jinja
{{ items | join(", ") }}
```

**変数**:
```python
items = ["りんご", "バナナ", "みかん"]
```

**結果**:
```
りんご, バナナ, みかん
```

#### よく使うフィルタ

| フィルタ | 説明 | 例 |
|---------|------|-----|
| `join(sep)` | リストを文字列で結合 | `{{ [1,2,3] \| join(",") }}` → `1,2,3` |
| `length` | 要素数を取得 | `{{ [1,2,3] \| length }}` → `3` |
| `upper` | 大文字に変換 | `{{ "hello" \| upper }}` → `HELLO` |
| `default(val)` | 空の場合のデフォルト値 | `{{ name \| default("匿名") }}` |

---

## 🎯 SupersetでのJinja2

### なぜSupersetでJinja2を使うのか

**問題**: 同じSQLを、条件ごとに書き換えたい

```sql
-- ユーザーがフィルタAを選択した場合
SELECT * FROM table WHERE country = 'JP'

-- ユーザーがフィルタBを選択した場合
SELECT * FROM table WHERE country = 'US'

-- フィルタなしの場合
SELECT * FROM table
```

**解決**: Jinja2で動的SQLを生成

```jinja
SELECT * FROM table
{% if selected_country %}
WHERE country = '{{ selected_country }}'
{% endif %}
```

### Supersetでの処理フロー

```
①ユーザーがダッシュボードでフィルタを選択
          ↓
②Supersetが値を取得（例: country = 'JP'）
          ↓
③Jinja2テンプレートに値を渡す
          ↓
④Jinja2がSQLテキストを生成
          ↓
⑤生成されたSQLをデータベースに送信
          ↓
⑥結果をチャートに表示
```

---

## 📝 今回のコードでJinja2を使っている箇所

### コード全体

```sql
-- これがJinja2テンプレート！
{% set selected_error_codes = filter_values('error_code_id', remove_filter=True)
    | reject('equalto', 'No filter') | list %}

{% set gather_date_from = url_param('gather_date_from', '') %}
{% set gather_date_to = url_param('gather_date_to', '') %}

SELECT
  company_name,

  {% if selected_error_codes %}
  CASE
    WHEN error_code_id::TEXT IN ('{{ selected_error_codes | join("','") }}')
    THEN error_count
    ELSE 0
  END AS selected_error_count,
  {% else %}
  0 AS selected_error_count,
  {% endif %}

  error_count AS total_error_count

FROM table
WHERE 1=1
  {% if gather_date_from %}
  AND gather_date >= '{{ gather_date_from }}'
  {% endif %}
```

### どこがJinja2なのか？

#### Jinja2の部分（色付き）

```jinja
{% set selected_error_codes = ... %}  ← Jinja2
{% set gather_date_from = ... %}      ← Jinja2

SELECT
  company_name,

  {% if selected_error_codes %}        ← Jinja2
  CASE
    WHEN error_code_id::TEXT IN ('{{ selected_error_codes | join("','") }}')  ← Jinja2
    THEN error_count
    ELSE 0
  END AS selected_error_count,
  {% else %}                           ← Jinja2
  0 AS selected_error_count,
  {% endif %}                          ← Jinja2
```

#### SQLの部分

```sql
SELECT
  company_name,
  CASE
    WHEN error_code_id::TEXT IN (...)
    THEN error_count
    ELSE 0
  END AS selected_error_count,
  error_count AS total_error_count
FROM table
WHERE 1=1
```

---

## 🔍 具体例で理解する

### 例1: クロスフィルタ（error_code_id）

#### テンプレート（Jinja2）

```jinja
{% set selected_error_codes = filter_values('error_code_id', remove_filter=True)
    | reject('equalto', 'No filter') | list %}

{% if selected_error_codes %}
CASE
  WHEN error_code_id::TEXT IN ('{{ selected_error_codes | join("','") }}')
  THEN error_count
  ELSE 0
END AS selected_error_count,
{% else %}
0 AS selected_error_count,
{% endif %}
```

#### 状況1: ユーザーがerror_code_id=733をクリック

**Jinja2が取得する値**:
```python
selected_error_codes = [733]
```

**Jinja2が生成するSQL**:
```sql
CASE
  WHEN error_code_id::TEXT IN ('733')
  THEN error_count
  ELSE 0
END AS selected_error_count,
```

#### 状況2: クリックなし（フィルタなし）

**Jinja2が取得する値**:
```python
selected_error_codes = []
```

**Jinja2が生成するSQL**:
```sql
0 AS selected_error_count,
```

---

### 例2: URL Parameter（gather_date_from）

#### テンプレート（Jinja2）

```jinja
{% set gather_date_from = url_param('gather_date_from', '') %}

WHERE 1=1
  {% if gather_date_from %}
  AND gather_date >= '{{ gather_date_from }}'
  {% endif %}
```

#### 状況1: URL = `?gather_date_from=2024-01-01`

**Jinja2が取得する値**:
```python
gather_date_from = '2024-01-01'
```

**Jinja2が生成するSQL**:
```sql
WHERE 1=1
  AND gather_date >= '2024-01-01'
```

#### 状況2: URLパラメータなし

**Jinja2が取得する値**:
```python
gather_date_from = ''
```

**Jinja2が生成するSQL**:
```sql
WHERE 1=1
```

---

### 例3: 複数値のJOIN処理

#### テンプレート（Jinja2）

```jinja
{% set selected_error_codes = [733, 829, 821] %}

WHERE error_code_id IN ('{{ selected_error_codes | join("','") }}')
```

#### Jinja2の処理

**ステップ1**: `join("','")`フィルタを適用
```python
[733, 829, 821] → "733','829','821"
```

**ステップ2**: `{{ }}`で出力
```sql
WHERE error_code_id IN ('733','829','821')
```

---

## 🆚 Jinja2あり vs Jinja2なし

### Jinja2なしの場合（不可能）

**問題**: Selected/Total同時表示ができない

```sql
-- error_code_id=733でフィルタした場合
SELECT
  company_name,
  SUM(error_count) AS selected_error_count,
  SUM(error_count) AS total_error_count  -- ← これもフィルタされる
FROM table
WHERE error_code_id = 733  -- ← フィルタ
GROUP BY company_name

-- 結果: Selected=10, Total=10（同じ値）
```

**できないこと**:
- Selected（フィルタあり）とTotal（フィルタなし）を同時に取得

---

### Jinja2ありの場合（可能）

```jinja
{% set selected_error_codes = [733] %}

SELECT
  company_name,

  -- Selected: CASE文でフィルタ
  CASE
    WHEN error_code_id IN ({{ selected_error_codes | join(",") }})
    THEN error_count
    ELSE 0
  END AS selected_error_count,

  -- Total: フィルタなし
  error_count AS total_error_count

FROM table
-- WHERE句にerror_code_idを含めない
GROUP BY company_name
```

**データ**:
```
error_code_id | company_name | selected_error_count | total_error_count
733           | CompanyA     | 10                   | 10
829           | CompanyA     | 0                    | 5
821           | CompanyA     | 0                    | 12
```

**集計後**:
```
company_name | SUM(selected) | SUM(total)
CompanyA     | 10            | 27
```

**実現できること**:
- Selected（フィルタあり）= 10
- Total（フィルタなし）= 27
- 同一行に表示 ✅

---

## 📊 Jinja2の処理タイミング

```
[クエリ実行リクエスト]
          ↓
┌─────────────────────────────────┐
│ フェーズ1: Jinja2処理           │
│ (Supersetサーバー内)             │
├─────────────────────────────────┤
│ 1. filter_values()で値取得       │
│    → selected_error_codes = [733]│
│                                 │
│ 2. url_param()で値取得          │
│    → gather_date_from = '2024-01-01' │
│                                 │
│ 3. {% if %}で条件分岐           │
│                                 │
│ 4. {{ }}で変数展開              │
│                                 │
│ 5. 純粋なSQLテキストを生成      │
└─────────────────────────────────┘
          ↓
[生成されたSQL]
SELECT company_name,
CASE WHEN error_code_id IN ('733')
  THEN error_count ELSE 0 END,
error_count
FROM table
WHERE gather_date >= '2024-01-01'
          ↓
┌─────────────────────────────────┐
│ フェーズ2: SQL実行              │
│ (データベース)                   │
├─────────────────────────────────┤
│ 1. SQLを解析                    │
│ 2. クエリを実行                 │
│ 3. 結果を返却                   │
└─────────────────────────────────┘
          ↓
┌─────────────────────────────────┐
│ フェーズ3: Superset集計         │
│ (Supersetサーバー内)             │
├─────────────────────────────────┤
│ 1. GROUP BYを実行               │
│ 2. SUM()等の集計関数を実行      │
│ 3. チャートに表示               │
└─────────────────────────────────┘
```

**重要**: Jinja2は**SQLを実行する前**に処理される

---

## 🔑 重要なポイント

### 1. Jinja2はSQLの前に実行される

```
Jinja2テンプレート → (処理) → 純粋なSQL → データベース
```

データベースには**純粋なSQL**のみが送られる

### 2. `{% %}` vs `{{ }}`の違い

| 記法 | 用途 | 出力 |
|-----|------|------|
| `{% if ... %}` | 制御（条件・ループ） | 出力なし（制御のみ） |
| `{{ variable }}` | 変数の値 | 値を出力 |

**例**:
```jinja
{% if True %}  ← これは出力されない
Hello          ← これが出力される
{% endif %}    ← これは出力されない
```

### 3. フィルタの連鎖

```jinja
{{ values | reject('equalto', 'No filter') | list | join(",") }}
```

左から右へ順に処理：
```
values → reject() → list() → join() → 結果
```

---

## 📚 Jinja2の公式ドキュメント

- [Jinja2公式サイト](https://jinja.palletsprojects.com/)
- [Superset SQL Templating](https://superset.apache.org/docs/installation/sql-templating/)

---

## ✅ まとめ

### Jinja2とは

- **テンプレートエンジン**（穴あき文書に値を埋め込む）
- **動的にSQLを生成**するために使用
- `{% %}`と`{{ }}`の2つの記法

### SupersetでのJinja2

- **仮想データセット（Virtual Dataset）**内で使用
- **ユーザーの操作**（クロスフィルタ、URLパラメータ）に応じてSQLを変更
- **Selected/Total同時表示**などの高度な機能を実現

### 今回の実装での役割

| 機能 | Jinja2の役割 |
|-----|------------|
| クロスフィルタ | `filter_values()`で値取得 → CASE文生成 |
| 日付範囲フィルタ | `url_param()`で値取得 → CASE文生成 |
| URL Parameterフィルタ | `url_param()`で値取得 → WHERE句生成 |

**Jinja2があるからこそ、柔軟で動的なダッシュボードが作れる！** ✅
