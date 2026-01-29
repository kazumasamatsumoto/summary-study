# Apache Supersetクロスフィルタリングに関する調査結果まとめ

## 1. クロスフィルタリングの基本条件

### 1.1 対応チャートタイプ

**一次情報源**: [Preset公式ドキュメント - Cross-filtering](https://docs.preset.io/docs/cross-filtering)

> **引用**: "In order to use cross-filtering, you need to use an eligible chart:
> - All charts built with ECharts
> - Area Chart, Bar Chart, Line Chart, Pie Chart, etc.
> - Pivot Table Chart
> - Table Chart
> - World Maps"

**結論**: Table Chartはクロスフィルタリングに対応している

### 1.2 必須条件：ディメンション（Group By）の存在

**一次情報源**: [GitHub Discussion #34649](https://github.com/apache/superset/discussions/34649)

> **引用**: "You must have edit permissions on the dashboard, and each chart needs **at least one dimension (group-by field)**; charts with only metrics won't show the option."

> **引用**: "The most common reason cross filtering isn't visible in Table charts... is that the chart doesn't have at least one dimension (group-by field). Cross filtering is only enabled for charts with dimensions—**if your Table chart only shows metrics, the option won't appear**"

**結論**: クロスフィルタリングを有効にするには、最低1つのディメンション（Group by）が必須

---

## 2. 複数Group Byでのクロスフィルタリングの挙動

### 2.1 実装コードによる検証

**一次情報源**: [GitHub - TableChart.tsx ソースコード](https://github.com/apache/superset/blob/master/superset-frontend/plugins/plugin-chart-table/src/TableChart.tsx)

```typescript
crossFilter: cellPoint.isMetric
  ? undefined
  : getCrossFilterDataMask(cellPoint.key, cellPoint.value),
```

**解釈**: 
- `cellPoint.key` = クリックしたセルのカラム名（1つのみ）
- `cellPoint.value` = クリックしたセルの値（1つのみ）
- **複数のGroup By値を同時に渡す実装になっていない**

### 2.2 EChartsとの実装の違い

**一次情報源**: [GitHub Discussion #29489 - ECharts Timeseries実装](https://github.com/apache/superset/discussions/29489)

EChartsのTimeseriesチャートでは以下のような実装:

```javascript
const groupbyValues = values.map(value => labelMap[value]);
return {
  dataMask: {
    extraFormData: {
      filters: groupby.map((col, idx) => {
        const val = groupbyValues.map(v => v[idx]);
        return {
          col,
          op: 'IN' as const,
          val: val as (string | number | boolean)[],
        };
      }),
    },
  },
};
```

**解釈**: 
- EChartsでは`labelMap`を使って複数Group Byに対応
- しかしこれはECharts特有の実装
- **Table Chartには同様の実装が存在しない**

### 2.3 実際の動作検証結果

**ユーザーの検証結果**:
> "今私の環境で実行しているのですが。Modelの値をクリックしてもクロスフィルタリングの値はModelしか渡っていないのです。"

**結論**: **Table Chartで複数のGroup Byがある場合、クリックしたディメンションの値のみがフィルターとして渡される。同じ行の他のディメンション値は含まれない。**

---

## 3. 複合キーによる解決策

### 3.1 Virtual Columnの作成

**一次情報源**: [Superset公式ドキュメント - Creating Your First Dashboard](https://superset.apache.org/docs/using-superset/creating-your-first-dashboard/)

> **引用**: "Virtual calculated columns: you can write SQL queries that customize the appearance and behavior of a specific column (e.g. CAST(recovery_rate as float)). Aggregate functions aren't allowed in calculated columns."

**実装方法**:
```sql
-- データセットの Calculated Columns で追加
Column Name: model_region_key
SQL Expression: CONCAT(Model, '-', Region)
```

### 3.2 複合キーの有効性

**一次情報源**: [GitHub Issue #30345](https://github.com/apache/superset/issues/30345)

> **引用**: "In Apache Superset, all dimensions are clickable and will affect the tables where cross-filtering is enabled."

**解釈**: 
- 複合キーも1つのディメンションとして扱われる
- クリックすると、その複合キー全体（Model AND Regionの組み合わせ）でフィルタリングされる

---

## 4. Query Mode（AggregateとRaw records）の違い

### 4.1 クロスフィルタリング発行側の要件

**一次情報源**: [GitHub Discussion #34649](https://github.com/apache/superset/discussions/34649)

> **引用**: "each chart needs **at least one dimension (group-by field)**"

**解釈**:
- **Aggregate Mode**: Group byフィールドを持つ = ディメンションが存在 = クロスフィルタリング発行可能 ✅
- **Raw Records Mode**: Group byを使用しない = ディメンションが存在しない可能性 = クロスフィルタリング発行不可 ❌

### 4.2 クロスフィルタリング受信側の動作

**一次情報源**: データベースクエリの仕組みから推論

クロスフィルタリングは以下のようにWHERE句を追加する:

```sql
-- 概要テーブルで "ModelA-RegionX" をクリック
-- 詳細テーブルに適用されるクエリ:
SELECT Model, Region, 詳細情報
FROM table
WHERE model_region_key = 'ModelA-RegionX'  -- WHERE句が追加される
```

**解釈**:
- Raw Records Modeでも、WHERE句は適用可能
- データセットに`model_region_key`カラムが存在すれば、Query Modeに関わらずフィルタリングされる

---

## 5. 詳細テーブルでの複合キー表示の必要性

### 5.1 データセットレベルとチャート表示レベルの区別

**一次情報源**: Supersetのアーキテクチャから推論（直接的な文献は見つからず）

**検証可能な事実**:
1. データセットに存在するカラムは、チャート作成時に表示/非表示を選択できる
2. WHERE句はデータセットレベルで適用される（チャートの表示設定とは独立）

**実装検証**:

```
データセット（両方に必要）:
  ├─ Model
  ├─ Region
  └─ model_region_key (Virtual Column)  ← 必須

概要テーブル:
  - Query Mode: Aggregate
  - Group by: model_region_key  ← 表示必須
  
詳細テーブル:
  - Query Mode: Raw records
  - Columns: Model, Region
  - model_region_keyは非選択  ← 表示不要
```

**結論**: 
- **データセットには複合キーが必要**（両方）
- **チャート表示は任意**（詳細テーブルでは非表示でもOK）

---

## 6. 最終的な実装方法（根拠付き）

### 構成

#### 概要テーブル
```
設定:
  - Query Mode: Aggregate（理由: Group by必須）
  - Group by: model_region_key
  - Metrics: AVG(受入率), COUNT(*)
  - Columns: model_region_key（表示）

根拠: GitHub Discussion #34649
"charts with only metrics won't show the option"
→ Group byがないとクロスフィルタリング発行不可
```

#### 詳細テーブル
```
設定:
  - Query Mode: Raw records（または Aggregate）
  - Columns: Model, Region, その他詳細カラム
  - model_region_keyは非表示

根拠: WHERE句はデータセットレベルで適用される
→ 表示していなくてもフィルタリングは機能する
```

### データセット設定（両方に必要）

```sql
Calculated Column:
  Column Name: model_region_key
  SQL Expression: CONCAT(Model, '-', Region)

根拠: Superset公式ドキュメント
"Virtual calculated columns: you can write SQL queries..."
```

---

## 7. 制約事項と不可能なこと

### 7.1 標準機能で実現できないこと

**一次情報源**: [GitHub - TableChart.tsx 実装コード](https://github.com/apache/superset/blob/master/superset-frontend/plugins/plugin-chart-table/src/TableChart.tsx)

```typescript
getCrossFilterDataMask(cellPoint.key, cellPoint.value)
// ↑ 1つのカラムと値のみを渡している
```

**結論**: 
- ❌ 1つのディメンションをクリックして複数ディメンションの組み合わせでフィルタリング
- ❌ メトリクス（計算カラム）をクリックしてフィルタリング発動

### 7.2 回避策

**一次情報源**: 実装コードの制約から導かれる解決策

- ✅ 複合キー列を作成（実質的に複数ディメンションの組み合わせを1つのディメンションとして扱う）
- ✅ 詳細テーブルでは複合キーを非表示にできる

---

## 8. 動作確認方法

### 確認1: クロスフィルタリング設定の確認

**一次情報源**: [Preset公式ドキュメント](https://docs.preset.io/docs/cross-filtering)

> **引用**: "To use it: open your dashboard in edit mode, click the vertical dots on a supported chart's header, and select 'Cross-filtering scoping'"

### 確認2: フィルター適用の確認

**一次情報源**: [Preset公式ドキュメント](https://docs.preset.io/docs/cross-filtering)

> **引用**: "You can also clearly see the active filters through the filter indicators, where the **'blue colored indicator' means a cross-filter was applied**"

---

## まとめ

| 項目 | 結論 | 根拠 |
|------|------|------|
| 複数Group Byで1クリック→複数フィルタ | ❌ 不可能 | TableChart.tsx実装 |
| 複合キーによる代替 | ✅ 可能 | Virtual Column機能 |
| 詳細テーブルでの複合キー表示 | 非必須 | WHERE句の適用メカニズム |
| 概要テーブルのQuery Mode | Aggregate必須 | Discussion #34649 |
| 詳細テーブルのQuery Mode | Raw records可 | WHERE句の適用メカニズム |

**最終的な推奨実装**:
1. 両方のデータセットに同じ複合キー（Virtual Column）を作成
2. 概要テーブル: Aggregate + 複合キーを表示
3. 詳細テーブル: Raw records + 複合キーを非表示
4. クロスフィルタリングは正常に機能する
