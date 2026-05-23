---
form: infinite_scroll
topic: table
applies_to: [frontend, mobile, backend]
data_scale: [any]
decision: mobile UI OR feed-style consumption
status: stable
last_reviewed: 2026-05-19
---

# Form 3: Infinite Scroll Table

Mobile / feed standard. Auto-fetch more when scroll near bottom. Cursor under the hood.

## When to use

- Mobile UI (standard UX expectation)
- Feed-style consumption: notifications, timelines, message history
- Any data scale (cursor underneath handles size)

## When NOT to use

- Users need "page N" or "last page" navigation (use Form 1 or 2)
- Desktop admin tables where users expect deterministic pagination
- Need to track "scroll position" across navigation (hard with infinite scroll)

## Pseudocode

```
function InfiniteScrollList<T>(props):
  state items: list of T = []
  state nextCursor: optional string = null
  state loading: bool = false
  state hasMore: bool = true
  state error: optional Error = null

  effect on (q, deps change):
    items = []
    nextCursor = null
    hasMore = true
    error = null
    loadMore()

  function loadMore():
    if loading or not hasMore: return    # guard against duplicate fires
    loading = true
    try:
      result = await fetch({ q, cursor: nextCursor, limit: pageSize })
      items = items.concat(result.items)
      nextCursor = result.nextCursor
      hasMore = (result.nextCursor != null)
    catch err:
      error = err
    finally:
      loading = false

  # IntersectionObserver on sentinel element near bottom
  on sentinel enters viewport:
    loadMore()

  render:
    items.map(item => render row)
    if hasMore: <sentinel ref={observer} />
    if loading: spinner
    if error: retry button
```

## References

High-star OSS implementations (stars verified 2026-05-23 via GitHub API; ≥5,000★ bar):

- [flutter/flutter](https://github.com/flutter/flutter) — ~176k★: `ListView.builder` + `ScrollController` end-detection (mobile)
- [facebook/react-native](https://github.com/facebook/react-native) — ~126k★: `FlatList` `onEndReached` / `onEndReachedThreshold` (mobile)
- [TanStack/query](https://github.com/TanStack/query) — ~49k★: `useInfiniteQuery` abstracts cursor + pages
- [TanStack/virtual](https://github.com/TanStack/virtual) — ~6.9k★: headless, pairs windowing with infinite loading

## Pitfalls / Anti-patterns

- ❌ All loaded rows stay in DOM after 1000+ scrolls → frame drops, RAM bloat
  - **Fix**: combine with Form 4 (virtual scroll), keep DOM count constant
- ❌ Forgetting `hasMore` check → infinite fetch loop after last page
- ❌ Observer fires multiple times before fetch completes → duplicate loads
  - **Fix**: `loading` guard + optionally abort previous fetch
- ❌ No "scroll back to top" affordance → users get lost in long lists
  - **Fix**: floating back-to-top button after N pixels of scroll
- ❌ Losing scroll position on back-navigation → user has to rebuild context
  - **Fix**: persist `nextCursor` + scroll offset in history state or session storage
- ❌ Network failure mid-scroll → silent freeze
  - **Fix**: error state + visible retry button at bottom
