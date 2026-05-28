---
form: search_filter_form
topic: form
applies_to: [frontend, backend]
field_set: query criteria
decision: collect query/filter criteria to narrow a list, not submit a record
status: stable
last_reviewed: 2026-05-28
---

# Form 8: Search / Filter Form

Collects **query criteria** to narrow a list/result set — not to create or update a record.
Advanced search panels, list-page filter bars, faceted filters. The "submit" produces a
query, and the result usually feeds a `table/` form.

## When to use

- Users refine a result set by multiple criteria (text, ranges, enums, dates)
- Filter state should be shareable/bookmarkable (encode in the URL)
- An admin list, search results page, faceted catalog

## When NOT to use

- Submitting a record (create/edit) → Form 1/2
- A single global search box with no structured filters → just an input + debounce
- Deeply nested boolean logic builder (AND/OR trees) → a niche sub-pattern; see "Boundary"

## Conclusion

Filter state is the **URL query string** (single source of truth) → shareable, bookmarkable,
back-button friendly + debounced apply for text, immediate for discrete controls + reset
also resets pagination. The query drives the list/table fetch.

## Pseudocode

```
function SearchFilterForm(props):
  # URL query is the source of truth — not local state that drifts from the address bar
  filters = parseFromUrl()    # { q, status, dateFrom, dateTo, page }

  function apply(patch):
    next = merge(filters, patch)
    if patch changes anything except page: next.page = 1   # filter change resets paging
    writeToUrl(next)          # triggers the list fetch downstream

  debouncedApplyText = debounce(300ms, q => apply({ q }))

  render:
    text input(filters.q) on change → debouncedApplyText        # debounced
    select(filters.status) on change → apply({ status })         # immediate
    dateRange(filters.dateFrom, filters.dateTo) on commit → apply({...})
    active-filter chips: each removable → apply({ <key>: null })
    button "Reset all" → writeToUrl(defaults)

  # downstream list/table reads the same URL filters and fetches:
  effect on (filters change):
    result = await props.fetchList(filters)   # see table/ forms for rendering
```

## Apply timing

```
text fields:        debounce (~300ms) — avoid a request per keystroke
selects/checkboxes: apply immediately on change
range/date:         apply on commit (blur / explicit "apply"), not mid-drag
big/expensive query: explicit "Search" button instead of live apply
always: a filter change resets pagination to page 1
```

## Boundary: visual query-builder

Nested AND/OR condition trees ("status = open AND (priority = high OR assignee = me)") are a
distinct, heavier sub-pattern. The dedicated OSS for visual query-builders currently sits
**below the 5,000★ bar** (e.g. react-querybuilder ~1.7k★, react-awesome-query-builder
~2.3k★), so this form documents the mainstream advanced-search form and treats the
query-builder as an out-of-scope niche until a >5k★ implementation emerges.

## References

High-star OSS implementations (stars verified 2026-05-28 via GitHub API; ≥5,000★ bar):

- [ant-design/ant-design](https://github.com/ant-design/ant-design) — ~98k★: Form "advanced search" / collapsible filter form pattern over a Table
- [mui/material-ui](https://github.com/mui/material-ui) — ~98k★: filter toolbar / DataGrid filter panel
- [mantinedev/mantine](https://github.com/mantinedev/mantine) — ~31k★: filter inputs + URL-state-friendly form controls

## Pitfalls / Anti-patterns

- ❌ Filter state only in local component state → not shareable, lost on refresh, back-button breaks
  - **Fix**: encode filters in the URL query string as the source of truth
- ❌ Firing a request per keystroke on the text filter → request storm
  - **Fix**: debounce text (~300ms); apply discrete controls immediately
- ❌ Changing a filter but keeping the old page number → "no results" on page 5 of a 1-page result
  - **Fix**: reset to page 1 on any filter change
- ❌ Out-of-order responses: an older slow query overwrites a newer one
  - **Fix**: sequence-guard responses (drop superseded), or cancel in-flight requests
- ❌ No "active filters" visibility / no reset → users stuck with filters they forgot about
  - **Fix**: show removable filter chips + a clear "reset all"
- ❌ Sending empty/default filter keys to the backend → noisy queries, cache misses
  - **Fix**: omit empty/default keys from both URL and request
