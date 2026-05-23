---
form: cursor_pagination
topic: table
applies_to: [frontend, backend, sql]
data_scale: [medium, large, xlarge]
decision: data > 100k OR append-only time-sorted
status: stable
last_reviewed: 2026-05-19
---

# Form 2: Cursor / Keyset Pagination Table

For append-only large tables. Backend keyset on `(sort_col, id)`,
frontend "prev / next" nav only.

## When to use

- Data > 100k OR continuously appending
- No need to jump pages — sequential reading is fine
- Time-sorted log-like data: audit logs, event streams, login history

## When NOT to use

- Need arbitrary column sort — cursor locks one ORDER BY
  - Workaround: client-side sort on current page only (not "true" global sort)
- Need "total count" prominently — cursor has no built-in total
- Need "jump to last page" — impossible by design

## Pseudocode

```
function CursorListTable<T>(props):
  state pages: list of list of T = []   # cache of visited pages
  state pageIdx: int = 0
  state nextCursor: optional string = null
  state searchInput: string = ""
  state committedQ: string = ""           # only server-bound after Enter / button
  state loading: bool = false

  effect on (committedQ, deps change):
    loading = true
    result = await fetch({ q: committedQ, cursor: null, limit: pageSize })
    pages = [result.items]
    pageIdx = 0
    nextCursor = result.nextCursor
    loading = false

  function next():
    if pageIdx < length(pages) - 1:
      pageIdx++                          # cache hit, no fetch
      return
    if not nextCursor: return             # last page reached
    result = await fetch({ q: committedQ, cursor: nextCursor, limit: pageSize })
    pages.append(result.items)
    pageIdx++
    nextCursor = result.nextCursor

  function prev():
    if pageIdx > 0: pageIdx--

  function commitSearch():
    committedQ = searchInput.trim()

  render:
    toolbar:
      search input (Enter → commitSearch)
      "Search" button (click → commitSearch)
      "Clear" button if committedQ != ""
    table:
      thead: STATIC column headers (no sort UI — ORDER BY is fixed)
      tbody: pages[pageIdx].map(row => columns)
    footer: prev | "page N" | next
```

## Backend SQL pattern

```
SELECT *
FROM tbl
WHERE filters
  AND (sort_col, id) < (cursor_sort_value, cursor_id)   -- the key trick
ORDER BY sort_col DESC, id DESC
LIMIT N + 1                                              -- +1 to detect "has more"
```

The `(col_a, col_b) < (val_a, val_b)` row-constructor comparison is equivalent to:

```
col_a < val_a OR (col_a = val_a AND col_b < val_b)
```

This uses a B-tree index on `(sort_col DESC, id DESC)` for O(log N) seek to cursor position.

Cursor format: `<sort_value>,<id>` encoded as opaque string. Always use composite key
(at minimum `(timestamp, id)`) to handle ties at identical sort values.

## References

High-star OSS implementations (stars verified 2026-05-23 via GitHub API; ≥5,000★ bar):

- [TanStack/query](https://github.com/TanStack/query) — ~49k★: `useInfiniteQuery` with cursors, the data layer
- [TanStack/table](https://github.com/TanStack/table) — ~28k★: headless table, wire cursor state yourself
- [apollographql/apollo-client](https://github.com/apollographql/apollo-client) — ~20k★: cursor pagination via `fetchMore`
- [facebook/relay](https://github.com/facebook/relay) — ~19k★: GraphQL Connection / edges spec, cursor-first by design

No standalone "cursor table" widget exists — built atop a headless table + a cursor-aware data layer.

## Pitfalls / Anti-patterns

- ❌ Column-click sort UI — cursor depends on ORDER BY, switching invalidates it
  - **Fix**: keep ORDER BY fixed; offer filters instead of sort, or client-side sort on current page only with clear "current page only" indicator
- ❌ Cursor uses a single column (e.g. `created_at` only) — ties cause skipped/repeated rows
  - **Fix**: composite cursor `(created_at, id)` minimum
- ❌ Forgetting `LIMIT N + 1` trick — no way to know "is there more"
  - **Fix**: always fetch N+1, return N, set hasMore=true if N+1 returned
- ❌ Encoding cursor as raw JSON exposed to client → API consumers depend on internals
  - **Fix**: opaque base64-encoded cursor, treat as black box from client side
- ❌ Wanting "jump to last page" — impossible by design
  - **Fix**: tell users it's a log feed, not a paged dataset
