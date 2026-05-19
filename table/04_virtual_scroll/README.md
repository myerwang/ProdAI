---
form: virtual_scroll
topic: table
applies_to: [frontend, mobile]
data_scale: [large, xlarge]
decision: > 1k rows visible OR very large in-memory dataset
status: stable
last_reviewed: 2026-05-19
---

# Form 4: Virtual Scroll Table

Render only rows within viewport. DOM stays at ~20 rows regardless of total count.

## When to use

- > 1k rows in memory and user can scroll through them (mandatory at scale)
- Row height fixed (easier) or measurable upfront
- Spreadsheet-like, huge log viewers, Notion-style databases

## When NOT to use

- < 500 rows — overhead exceeds benefit
- Highly varied row heights with rich media — measurement is fragile
- SEO-critical pages — virtual scroll content invisible to crawlers

## Pseudocode

```
function VirtualScrollList<T>(props):
  state items: list of T = []      # full cache (if total < 100k)
  state scrollTop: int = 0
  rowHeight = 40                    # fixed; or use measure function for variable

  derived:
    visibleStart = floor(scrollTop / rowHeight)
    visibleEnd = visibleStart + ceil(viewportHeight / rowHeight) + buffer
    visibleItems = items.slice(visibleStart, visibleEnd)
    totalHeight = length(items) * rowHeight
    offsetTop = visibleStart * rowHeight

  render:
    <scroll-container height=viewportHeight onScroll={e => scrollTop = e.scrollTop}>
      <div height=totalHeight>                    # spacer to claim full scroll range
        <div translateY=offsetTop>                # offset visible block
          visibleItems.map(item => render row at fixed height)
        </div>
      </div>
    </scroll-container>
```

**Core idea**: a huge placeholder div claims full scroll height; inside, only ~20 visible
rows render, positioned via `transform: translateY(...)`.

For variable row heights, lib stores measured heights in a cache and recalculates
offsets on resize.

## Representative libraries

- **react-window** (`bvaughn/react-window`, 16k+) — lightweight, fixed row height, simplest
- **react-virtualized** (`bvaughn/react-virtualized`, 15k+) — full-featured, variable height, heavier API
- **TanStack Virtual** (`TanStack/virtual`, 5k+) — headless, framework-agnostic (React/Vue/Solid/Svelte)
- **virtua** (`inokawa/virtua`, 3k+) — newer, best perf on React 18+, simpler API
- **vue-virtual-scroller** — Vue ecosystem standard

## Pitfalls / Anti-patterns

- ❌ Dynamic row heights without measurement → scroll jitter, overlapping rows
  - **Fix**: use libs with built-in `useDynamicSize` / `measure` hook, or pre-measure with ResizeObserver
- ❌ Buffer too small → blank rows on fast scroll
  - **Fix**: buffer 5-10 rows above and below viewport
- ❌ Using `<table>` semantics with virtual scroll → column alignment breaks (because spacer divs disrupt table layout)
  - **Fix**: use `display: grid` or `display: block` + manual column widths via `flex` / `grid-template-columns`
- ❌ Sticky headers don't sync with virtual scroll offset
  - **Fix**: render header outside the scroll container, or use `position: sticky` + careful CSS
- ❌ Selection / keyboard nav resets when row scrolls out of view
  - **Fix**: maintain selection state keyed by item id (not row index)
- ❌ Window resize doesn't recalculate visible range
  - **Fix**: re-measure on `ResizeObserver` or `window.resize`
