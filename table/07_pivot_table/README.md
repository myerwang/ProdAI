---
form: pivot_table
topic: table
applies_to: [frontend, backend, sql, bi]
data_scale: [server_aggregated]
decision: multi-dimensional analysis (rows × cols × values × filters)
status: stable
last_reviewed: 2026-05-19
---

# Form 7: Pivot Table

Multi-dimensional analysis: 4 dimensions of Rows × Cols × Values × Filters.

## When to use

- BI dashboards, reports, OLAP-style analysis
- Users want "sales by month × by region" rollups
- Cross-tab data exploration

## When NOT to use

- Simple list display — overkill
- Front-end reducing huge raw data — must reduce server-side first
- One-dimensional grouping — use plain GROUP BY result with Form 1

## Pseudocode

```
function PivotTable<T>(props):
  data: list of T                                    # raw rows (or pre-aggregated from server)
  rowFields: list of fieldName                       # group by these → row keys
  colFields: list of fieldName                       # group by these → col keys
  valueFields: list of { field, aggregator }         # sum | avg | count | max | min

  derived:
    rowKeys = unique(data.map(r => rowFields.map(f => r[f]).join("|")))
    colKeys = unique(data.map(r => colFields.map(f => r[f]).join("|")))

    matrix = {}                                      # map of "rowKey||colKey" to aggregated value
    for each r in data:
      rk = rowFields.map(f => r[f]).join("|")
      ck = colFields.map(f => r[f]).join("|")
      key = rk + "||" + ck
      matrix[key] = aggregate(matrix[key], r, valueFields)

  render:
    table:
      thead:
        # First N header rows for each colField (hierarchical)
        for each colField at depth d:
          <tr>
            <th colspan={rowFields.length}></th>
            colKeys.map(ck => <th>{extract part d of ck}</th>)
          </tr>
      tbody:
        rowKeys.map(rk =>
          <tr>
            for each rowField at depth d:
              <th>{extract part d of rk}</th>
            colKeys.map(ck => <td>{matrix[rk + "||" + ck] or "—"}</td>)
          </tr>
        )
      tfoot:
        # Optional totals row per col
        <tr>
          colKeys.map(ck => <td>{aggregate(matrix, all rows, ck)}</td>)
        </tr>
```

## Backend approach

For non-trivial data sizes, do aggregation in SQL:

```
SELECT row_field_1, row_field_2, col_field_1, SUM(value_field)
FROM tbl
WHERE filters
GROUP BY row_field_1, row_field_2, col_field_1
WITH ROLLUP  -- or CUBE for full cross-products
```

Then front-end only pivots the result (small matrix).

## Representative libraries

- **AG Grid Enterprise** — pivoting + row grouping + aggregation, industry standard, paid
- **PivotTable.js** (`nicolaskruchten/pivottable`, 4k+) — open source, jQuery-era but maintained
- **Apache Superset** — full BI tool with pivot
- **Metabase** — open-source BI, includes pivot
- **DevExtreme PivotGrid** — commercial alternative

## Pitfalls / Anti-patterns

- ❌ Aggregating raw rows in the frontend → thousands of rows crash the browser
  - **Fix**: backend `GROUP BY ... WITH ROLLUP / CUBE`, send pre-aggregated rows
- ❌ High-cardinality columns as pivot col (e.g., `user_id`) → column explosion (10000 columns)
  - **Fix**: limit pivot col to low-cardinality fields; offer top-N + "Other" bucketing
- ❌ Wrong aggregator (sum vs avg vs count) silently produces wrong numbers
  - **Fix**: explicit aggregator UI; consider double-display ("Total: $X / Avg: $Y")
- ❌ Mixing currencies / units across rows → meaningless sum
  - **Fix**: normalize units server-side, or split by currency dimension
- ❌ Empty cells displayed as 0 → misleading (no data ≠ value of 0)
  - **Fix**: explicit "—" or "n/a" for missing combinations
- ❌ Export to Excel mangles formatting → users can't reuse
  - **Fix**: use library's native Excel export (preserves row/col headers, merged cells)
