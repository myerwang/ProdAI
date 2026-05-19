---
form: offset_table
topic: table
applies_to: [frontend, backend, sql]
data_scale: [small, medium]
decision: data < 100k + mutating CRUD + arbitrary sort/jump pages
status: stable
last_reviewed: 2026-05-19
---

# Form 1: Standard List Table (Offset Pagination)

The 90% case. Backend `OFFSET / LIMIT`, frontend column-click sort, page-number nav.

## When to use

- Data < 100k rows
- Mutating CRUD (status changes frequently)
- Users want arbitrary column sort, jump to any page
- Need visible "total count" prominently

## When NOT to use

- Data > 100k → `OFFSET 99000` becomes O(N), slow
- Append-only time-series → Form 2 (cursor) is better
- Mobile UX → Form 3 (infinite scroll)

## Pseudocode

```
function StandardListTable<T>(props):
  state items: list of T = []
  state page: int = 1
  state pageSize: int = 20
  state sortKey: optional string
  state sortDir: "asc" | "desc" = "desc"
  state q: string = ""
  state total: int = 0
  state loading: bool = false

  effect on (page, pageSize, sortKey, sortDir, q):
    loading = true
    result = await fetch({
      offset: (page - 1) * pageSize,
      limit: pageSize,
      sort: sortKey ? format(sortKey, sortDir) : null,
      q,
    })
    items = result.items
    total = result.total
    loading = false

  render:
    toolbar(search input, filters)
    table:
      thead: sortable column headers
        on click: setSortKey + toggle sortDir
      tbody: items.map(row => render columns)
    footer:
      "total N items" display
      page selector (1, 2, 3, ..., last)
      pageSize selector
```

## Backend SQL pattern

```
SELECT *
FROM tbl
WHERE filters...
ORDER BY sort_col DIR
LIMIT pageSize
OFFSET (page - 1) * pageSize
```

For total count:

```
SELECT COUNT(*) FROM tbl WHERE filters...
```

## Representative libraries

- **shadcn/ui Data Table** (`shadcn-ui/ui`, 80k+ stars) — paste-and-own, TanStack-based
- **Ant Design Table** (`ant-design/ant-design`, 90k+ overall) — full-featured, China-favored
- **MUI X DataGrid** (`mui/mui-x`) — Material design, partial paid

## Pitfalls / Anti-patterns

- ❌ Using this for > 100k rows → deep pagination is O(N), SQL gets slow
  - **Fix**: switch to Form 2 (cursor) or Form 4 (virtual scroll)
- ❌ Computing offset client-side and ignoring it server-side → drops pages
  - **Fix**: always pass offset to backend
- ❌ New rows inserted while user paginates → row duplication / loss between pages
  - **Fix**: stable ordering with tiebreaker (e.g., `ORDER BY created_at DESC, id DESC`), or freeze a snapshot
- ❌ `SELECT COUNT(*)` on every request → unnecessary load
  - **Fix**: cache total per filter combo; refresh on filter change only
- ❌ Page selector showing "1 2 3 ... 9999" → ugly + meaningless
  - **Fix**: show first/last + neighbors only
