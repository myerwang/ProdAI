---
form: server_side_row_model
topic: table
applies_to: [frontend, backend, sql]
data_scale: [xlarge]
decision: data > 1M with full sort/filter/group capability needed
status: stable
last_reviewed: 2026-05-19
---

# Form 8: Server-Side Row Model

Million-row tables. Server holds all state. Client is a thin viewer that requests row ranges.

## When to use

- Data > 1M rows, with any sort/filter/group needed
- Financial trading systems, massive logs, huge ETL output
- Frontend cannot reasonably cache; every interaction queries server

## When NOT to use

- Anything smaller — overkill, Form 4 + 1/2 suffices
- Without backend keyset / index optimization — defeats the purpose
- Without dedicated DBA / query performance ownership

## Pseudocode

```
function ServerSideTable<T>(props):
  # No client cache; server is authoritative
  state visibleRows: list of T = []
  state scrollTop: int = 0
  state sortModel: list of { field, dir } = []
  state filterModel: list of { field, op, value } = []
  state totalCount: optional int = null      # lazy, server may return on last page

  effect on (scrollTop, sortModel, filterModel):
    startRow = floor(scrollTop / rowHeight)
    endRow = startRow + visibleCount + buffer

    result = await fetch({
      startRow, endRow,
      sort: sortModel,
      filter: filterModel,
    })
    visibleRows = result.rows
    if result.lastRow exists:
      totalCount = result.lastRow

  render:
    virtual scroll container with scrollbar height = totalCount * rowHeight or "infinite"
    visibleRows.map(row => columns)
    if totalCount: "showing rows N-M of TOTAL"
    else: "showing rows N-M, total computing..."
```

## Backend design

Two challenges: **fast row range fetch** and **lazy total count**.

### Row range fetch (use keyset, not OFFSET)

```
-- DON'T:
SELECT * FROM tbl ORDER BY x LIMIT 100 OFFSET 999900   -- O(N), slow

-- DO: keyset based on previous range's last row
SELECT * FROM tbl
WHERE (x, id) < (cursor_x, cursor_id)
ORDER BY x DESC, id DESC LIMIT 100                      -- O(log N)
```

Map from row index → cursor: client tells server "start row 999900", server uses
a sparse index (e.g., every 10000th row's cursor cached) to jump close, then sequential.

### Lazy total count

```
-- DON'T:
SELECT COUNT(*) FROM tbl WHERE filter        -- blocks main query

-- DO either:
-- A. estimate from EXPLAIN
EXPLAIN (FORMAT JSON) SELECT * FROM tbl WHERE filter

-- B. defer until client reaches last page
-- (return null for totalCount until last page detected)
```

## References

High-star OSS implementations (stars verified 2026-05-23 via GitHub API; ≥5,000★ bar):

- [TanStack/table](https://github.com/TanStack/table) — ~28k★: headless; pair with virtualization + a backend keyset endpoint to build the row model yourself
- [bvaughn/react-window](https://github.com/bvaughn/react-window) — ~17k★: viewport virtualization for the thin-client viewer
- [ag-grid/ag-grid](https://github.com/ag-grid/ag-grid) — ~15k★: Server-Side Row Model (the canonical implementation; Enterprise)

No mature pure-OSS equivalent for the full SSRM; AG Grid Enterprise dominates this niche.

## Pitfalls / Anti-patterns

- ❌ Server uses `OFFSET 999000 LIMIT 50` → painfully slow query, defeats the whole point
  - **Fix**: keyset / cursor-based queries even when seeking to arbitrary row
- ❌ `SELECT COUNT(*)` on every request → blocks main query
  - **Fix**: lazy total (returned only when last page reached), or estimated count via EXPLAIN
- ❌ Filter columns without indexes → full table scan on every filter change
  - **Fix**: cover frequently-filtered columns with indexes; warn user about unindexed filters
- ❌ Client caching old row ranges → stale data when other users edit
  - **Fix**: short TTL on cached ranges, or invalidate on write events (websocket / SSE)
- ❌ Buffer too small → user sees blank rows when scrolling fast
  - **Fix**: prefetch adjacent ranges in background
- ❌ Group-by with millions of groups → group payload too big for client
  - **Fix**: paginate groups themselves; expand-on-demand for group children
- ❌ No abort logic when filter changes mid-fetch → race condition, wrong rows shown
  - **Fix**: cancel inflight request when filter / sort changes
