# Superset機能を使った実装解説

## 📋 概要

このドキュメントは、**Apache Supersetのどの機能を使って今回の要件を実装しているか**を技術的に解説します。

---

## 🎯 今回の要件

### 要件1: クロスフィルタでSelected/Total同時表示

チャート1でerror_code_idをクリック → チャート2で以下を同一行に表示：
- **Selected Error Count**: クリックされたerror_code_idのみ
- **Total Error Count**: 全エラー（クロスフィルタの影響を受けない）

### 要件2: 日付範囲でSelected/Total同時表示

- **Selected Pass Count**: 期間内（gather_date_from ~ gather_date_to）
- **Total Pass Count**: 累積（~ gather_date_to）

### 要件3: Embedded Dashboard連携

URL Parametersで10個のフィルタを受け取る

---

## 🔧 使用しているSupersetの機能

### 1. Virtual Dataset（仮想データセット）

#### 機能概要

- SQL Labで任意のSQLクエリを「データセット」として定義
- 物理テーブルと同様にチャート作成に使用可能
- **Jinja2テンプレート**をサポート

#### 使い方

1. **SQL Lab** → 新規クエリ作成
2. SQLを記述（Jinja2テンプレート使用可）
3. **Save** → **Save dataset**

#### 今回の利用方法

```sql
-- superset_dataset.sql
{% set selected_error_codes = filter_values('error_code_id', remove_filter=True)
    | reject('equalto', 'No filter') | list %}

WITH error_data AS (...)
SELECT
  company_name,
  {% if selected_error_codes %}
  CASE
    WHEN error_code_id::TEXT IN ('{{ selected_error_codes | join("','") }}')
    THEN error_incident_errors
    ELSE 0
  END AS selected_error_count,
  {% else %}
  0 AS selected_error_count,
  {% endif %}
  error_incident_errors AS total_error_count
FROM joined_data
```

**ポイント**: Jinja2で動的SQLを生成し、Selected/Totalを分離

---

### 2. Jinja2 Template Processing

#### 機能概要

Supersetは仮想データセット内で**Jinja2テンプレートエンジン**を使用できる

> **📖 Jinja2が初めての方へ**: [JINJA2_BASICS.md](JINJA2_BASICS.md)で基礎から解説しています

#### 有効化方法

```python
# superset_config.py
FEATURE_FLAGS = {
    "ENABLE_TEMPLATE_PROCESSING": True,
}
```

#### 利用可能な関数

| 関数 | 説明 | 戻り値 |
|-----|------|--------|
| `filter_values('column')` | フィルタUIから値を取得 | リスト |
| `url_param('param', default)` | URLパラメータから値を取得 | 文字列 |

#### Jinja2の機能

| 機能 | 構文 | 用途 |
|-----|------|------|
| 変数 | `{% set var = value %}` | 値を保持 |
| 条件分岐 | `{% if condition %}...{% endif %}` | 動的SQL生成 |
| フィルタ | `{{ list \| join(",") }}` | データ変換 |

#### 今回の利用方法

```jinja
-- 変数定義
{% set selected_error_codes = filter_values('error_code_id', remove_filter=True)
    | reject('equalto', 'No filter') | list %}

-- 条件分岐
{% if selected_error_codes %}
CASE WHEN error_code_id::TEXT IN ('{{ selected_error_codes | join("','") }}')
{% endif %}

-- URLパラメータ取得
{% set company_id_param = url_param('company_id', '') %}
{% set f_company_id = company_id_param.split(',') if company_id_param else [] %}
```

**ポイント**: クエリ実行時に動的にSQLを生成

---

### 3. Cross-filtering（クロスフィルタ）

#### 機能概要

チャート間で**インタラクティブなフィルタリング**を実現

- チャート1の要素をクリック → チャート2が自動的にフィルタされる
- ダッシュボード内のチャート間連携

#### 設定方法

**チャート1（送信側）**:
1. チャート編集 → **Chart Configuration**
2. **Cross-filtering** セクション
3. **Enable cross-filtering**: ✅ ON
4. **Emitting columns**: `error_code_id` を選択

**チャート2（受信側）**:
1. チャート編集 → **Chart Configuration**
2. **Cross-filtering** セクション
3. **Receiving filters**: ✅ ON

#### 内部動作

```
ユーザーがチャート1でerror_code_id=733をクリック
          ↓
Supersetが内部的にフィルタ状態を保持
          ↓
filter_values('error_code_id') → [733]
          ↓
チャート2のクエリ実行時にJinja2変数として利用可能
```

#### 今回の利用方法

```jinja
{% set selected_error_codes = filter_values('error_code_id', remove_filter=True)
    | reject('equalto', 'No filter') | list %}
```

- `filter_values('error_code_id')`: クロスフィルタ値を取得
- `remove_filter=True`: **重要**（後述）
- `reject('equalto', 'No filter')`: 空フィルタを除外

**ポイント**: チャート1でクリックされた値をチャート2のSQLで利用

---

### 4. remove_filter パラメータ

#### 機能概要

`filter_values()`の第2引数で、**Supersetの自動フィルタ適用を制御**

```python
filter_values(column_name, remove_filter=False)
```

| パラメータ値 | 動作 |
|-------------|------|
| `False`（デフォルト） | フィルタ値を取得 + WHERE句に自動追加 |
| `True` | フィルタ値を取得のみ（WHERE句に追加しない） |

#### 動作の違い

**remove_filter=False の場合**:

```sql
-- Jinja2
{% set selected_error_codes = filter_values('error_code_id') | list %}

-- 生成されるSQL（Supersetが自動追加）
SELECT * FROM table
WHERE error_code_id IN (733)  -- ← 自動追加される

-- 結果: Total Error CountもSelected Error Countと同じ値になる（NG）
```

**remove_filter=True の場合**:

```sql
-- Jinja2
{% set selected_error_codes = filter_values('error_code_id', remove_filter=True) | list %}

-- 生成されるSQL
SELECT * FROM table
-- error_code_idのWHERE句は追加されない

-- CASE文で手動制御
CASE
  WHEN error_code_id IN (733) THEN error_count
  ELSE 0
END AS selected_error_count,
error_count AS total_error_count

-- 結果: Selected=10, Total=27 のように分離できる（OK）
```

#### 今回の利用方法

```jinja
{% set selected_error_codes = filter_values('error_code_id', remove_filter=True)
    | reject('equalto', 'No filter') | list %}
```

**ポイント**: WHERE句への自動追加を防ぎ、CASE文で手動制御することで、Selected/Totalの同時表示を実現

---

### 5. url_param() 関数

#### 機能概要

**URLクエリパラメータ**から値を取得

```jinja
{% set param_value = url_param('parameter_name', 'default_value') %}
```

| 引数 | 説明 |
|-----|------|
| 第1引数 | パラメータ名 |
| 第2引数 | デフォルト値（パラメータがない場合） |

#### URL例

```
https://superset.example.com/superset/dashboard/123/?company_id=1,2,3&gather_date_from=2024-01-01
```

#### 取得方法

```jinja
{% set company_id_param = url_param('company_id', '') %}
-- company_id_param = '1,2,3'

{% set gather_date_from = url_param('gather_date_from', '') %}
-- gather_date_from = '2024-01-01'
```

#### 複数値の処理

```jinja
{% set company_id_param = url_param('company_id', '') %}
{% set f_company_id = company_id_param.split(',') if company_id_param else [] %}
-- f_company_id = ['1', '2', '3']

-- SQLで使用
{% if f_company_id %}
AND company_id IN ('{{ f_company_id | join("','") }}')
{% endif %}
-- → AND company_id IN ('1','2','3')
```

#### 今回の利用方法

10個のURL Parametersを受け取る：

```jinja
-- 単一値
{% set gather_date_from = url_param('gather_date_from', '') %}
{% set gather_date_to = url_param('gather_date_to', '') %}
{% set f_min_note = url_param('min_note', '') %}

-- 複数値（カンマ区切り）
{% set company_id_param = url_param('company_id', '') %}
{% set f_company_id = company_id_param.split(',') if company_id_param else [] %}

{% set type_id_param = url_param('type_id', '') %}
{% set f_type_id = type_id_param.split(',') if type_id_param else [] %}
```

**ポイント**: Embedded Dashboard経由で外部アプリケーションからフィルタ値を受け取る

---

### 6. Embedded Dashboard

#### 機能概要

Supersetダッシュボードを外部アプリケーションに埋め込み

- **iframeで埋め込み**
- **URLパラメータで外部制御**

#### 使用例

```html
<iframe
  src="https://superset.example.com/superset/dashboard/123/?
       company_id=1&
       type_id=RCW100&
       gather_date_from=2024-01-01&
       gather_date_to=2024-12-31"
  width="100%"
  height="800px"
></iframe>
```

#### 今回の利用方法

外部アプリケーションから10個のパラメータを渡す：

```
?asset_id=011867
&company_id=1,2,3
&country_code=JP
&gather_date_from=2024-01-01
&gather_date_to=2024-12-31
&location_id=10
&min_note=5
&region_id=1
&state=Tokyo
&type_id=RCW100
```

これらの値を`url_param()`で取得し、SQLクエリに反映

**ポイント**: 外部システムとの連携インターフェース

---

## 🔗 機能の組み合わせ

### パターン1: クロスフィルタでSelected/Total同時表示

#### 使用する機能

1. **Cross-filtering** - チャート間連携
2. **filter_values()** - クロスフィルタ値取得
3. **remove_filter=True** - 自動WHERE句回避
4. **Jinja2 CASE文** - 条件分岐

#### 実装

```jinja
-- 1. クロスフィルタ値を取得（WHERE句には追加しない）
{% set selected_error_codes = filter_values('error_code_id', remove_filter=True)
    | reject('equalto', 'No filter') | list %}

-- 2. CASE文で条件分岐
{% if selected_error_codes %}
CASE
  WHEN error_code_id::TEXT IN ('{{ selected_error_codes | join("','") }}')
  THEN error_incident_errors
  ELSE 0
END AS selected_error_count,
{% else %}
0 AS selected_error_count,
{% endif %}

-- 3. Totalは無条件
error_incident_errors AS total_error_count
```

#### データフロー

```
チャート1でerror_code_id=733をクリック
          ↓
Cross-filtering機能でチャート2に伝達
          ↓
filter_values('error_code_id', remove_filter=True) → [733]
          ↓
Jinja2がSQLを生成:
  CASE WHEN error_code_id IN ('733') THEN error_count ELSE 0 END AS selected,
  error_count AS total
          ↓
行レベルデータ:
  error_code_id=733: selected=10, total=10
  error_code_id=829: selected=0, total=5
          ↓
Supersetが集計（GROUP BY company, model）:
  SUM(selected)=10, SUM(total)=15
```

---

### パターン2: 日付範囲でSelected/Total同時表示

#### 使用する機能

1. **url_param()** - URLパラメータ取得
2. **Jinja2 CASE文** - 期間制御
3. **WHERE句を使わない** - 全期間データ取得

#### 実装

```jinja
-- 1. URLパラメータ取得
{% set gather_date_from = url_param('gather_date_from', '') %}
{% set gather_date_to = url_param('gather_date_to', '') %}

-- 2. CASE文で期間制御（WHERE句ではなく）
{% if gather_date_from and gather_date_to %}
CASE
  WHEN gather_date >= '{{ gather_date_from }}'::date
   AND gather_date <= '{{ gather_date_to }}'::date
  THEN pass_count
  ELSE 0
END AS selected_pass_count,
{% else %}
pass_count AS selected_pass_count,
{% endif %}

-- 3. Total（累積）
{% if gather_date_to %}
CASE
  WHEN gather_date <= '{{ gather_date_to }}'::date
  THEN pass_count
  ELSE 0
END AS total_pass_count,
{% else %}
pass_count AS total_pass_count,
{% endif %}

-- 4. WHERE句には日付フィルタを含めない
WHERE 1=1
  {% if f_company_id %}
  AND company_id IN ('{{ f_company_id | join("','") }}')
  {% endif %}
  -- gather_dateは含めない
```

#### データフロー

```
URL: ?gather_date_from=2024-01-01&gather_date_to=2024-01-31
          ↓
url_param()で取得
          ↓
Jinja2がSQLを生成（全期間データを取得）
          ↓
行レベルデータ:
  2023-12-01: selected=0, total=100
  2024-01-15: selected=200, total=200
  2024-02-01: selected=0, total=150
          ↓
Supersetが集計:
  SUM(selected)=200（期間内）, SUM(total)=450（累積）
```

---

### パターン3: URL Parameterフィルタリング

#### 使用する機能

1. **url_param()** - URLパラメータ取得
2. **Jinja2 split()** - 複数値を分割
3. **WHERE句** - 通常フィルタリング

#### 実装

```jinja
-- 1. URLパラメータ取得
{% set company_id_param = url_param('company_id', '') %}
{% set f_company_id = company_id_param.split(',') if company_id_param else [] %}

{% set type_id_param = url_param('type_id', '') %}
{% set f_type_id = type_id_param.split(',') if type_id_param else [] %}

-- 2. WHERE句でフィルタリング
WHERE 1=1
  {% if f_company_id %}
  AND company_id IN ('{{ f_company_id | join("','") }}')
  {% endif %}

  {% if f_type_id %}
  AND type_id IN ('{{ f_type_id | join("','") }}')
  {% endif %}
```

#### データフロー

```
URL: ?company_id=1,2,3&type_id=RCW100
          ↓
url_param()で取得: '1,2,3', 'RCW100'
          ↓
split(',')で分割: ['1','2','3'], ['RCW100']
          ↓
Jinja2がSQLを生成:
  WHERE company_id IN ('1','2','3')
    AND type_id IN ('RCW100')
          ↓
フィルタされたデータのみ返却
```

---

## 📊 機能マッピング表

| 要件 | 使用するSuperset機能 | 技術的アプローチ |
|-----|---------------------|----------------|
| クロスフィルタでSelected/Total同時表示 | Cross-filtering + filter_values() + remove_filter=True | WHERE句を避けてCASE文で分岐 |
| 日付範囲でSelected/Total同時表示 | url_param() + Jinja2 CASE文 | WHERE句を避けてCASE文で期間制御 |
| URL Parameterフィルタリング | url_param() + Embedded Dashboard | WHERE句で通常フィルタ |
| 動的SQL生成 | Jinja2 Template Processing | 条件分岐・変数・フィルタ関数 |
| 外部連携 | Embedded Dashboard | URLパラメータ経由 |

---

## 🔑 重要な技術的ポイント

### 1. remove_filter=True の重要性

**なぜ必要か**:
- Supersetはデフォルトで`filter_values()`で取得した値をWHERE句に自動追加する
- Selected/Total同時表示には、WHERE句でデータを絞らずにCASE文で分岐する必要がある

**使い分け**:
- Selected/Total同時表示: `remove_filter=True` + CASE文
- 通常のフィルタ: `remove_filter=False`（デフォルト）+ WHERE句

### 2. WHERE句 vs CASE文

| 目的 | 使用箇所 | 理由 |
|-----|---------|------|
| データを絞る | WHERE句 | 不要なデータを除外（パフォーマンス向上） |
| データを分岐 | CASE文 | 全データを取得しつつ、条件で値を変更 |

**例**:
- company_idでフィルタ → WHERE句（会社Aのデータのみ取得）
- error_code_idでSelected/Total → CASE文（全エラーデータを取得し、条件で0/実数を切り替え）

### 3. 仮想データセットの設計思想

**仮想データセットの役割**:
- **行レベルデータ**を返す（集計しない）
- Supersetが外側でGROUP BYを実行

**チャートの役割**:
- Dimensionsを指定してGROUP BY
- Metricsで`SUM()`, `AVG()`等の集計関数を実行

**なぜこの設計か**:
```sql
-- ❌ 仮想データセット内で集計（二重集計になる）
SELECT company, SUM(error_count) FROM ...

-- ✅ 行レベルデータを返す
SELECT company, error_count FROM ...
-- Supersetが SUM(error_count) を実行
```

### 4. Jinja2とSQLの実行タイミング

```
クエリ実行リクエスト
          ↓
1. Jinja2テンプレート処理
   - filter_values(), url_param()で値取得
   - {% if %}で条件分岐
   - SQLテキスト生成
          ↓
2. 生成されたSQLを実行
   - データベースにクエリ送信
   - 結果を取得
          ↓
3. Supersetが集計
   - GROUP BY
   - SUM(), AVG()等
          ↓
4. チャートに表示
```

---

## ✅ 前提条件

### Superset設定

```python
# superset_config.py
FEATURE_FLAGS = {
    "ENABLE_TEMPLATE_PROCESSING": True,  # Jinja2を有効化
}
```

### バージョン要件

- Apache Superset 1.2.0以降
- PostgreSQL（array_agg, BOOL_OR等の関数使用）

---

## 📚 参考リソース

### Superset公式ドキュメント

- [SQL Lab](https://superset.apache.org/docs/creating-charts-dashboards/creating-your-first-dashboard/#creating-charts-in-explore-view)
- [Jinja Templating](https://superset.apache.org/docs/installation/sql-templating/)
- [Cross-filtering](https://superset.apache.org/docs/using-superset/exploring-data/#cross-filtering)

### 今回の実装に関連するドキュメント

| ファイル | 内容 |
|---------|------|
| JINJA2_BASICS.md | Jinja2テンプレート基礎解説 |
| superset_dataset.sql | 実装SQL |
| IMPLEMENTATION_DETAILS.md | 詳細実装解説 |
| README.md | 概要・クイックリファレンス |

---

**Supersetの機能を組み合わせることで、複雑な要件を実現できます！** ✅
