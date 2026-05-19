---
form: tree_table
topic: table
applies_to: [frontend, backend]
data_scale: [small, medium, large]
decision: hierarchical / parent-child data with expand-collapse UX
status: stable
last_reviewed: 2026-05-19
---

# Form 6: Tree Table (Nested / Expandable)

Hierarchical data with expand / collapse rows.

## When to use

- File system, org chart, comment threads, config trees, category trees
- Data naturally has parent-child relations
- User needs to drill down progressively

## When NOT to use

- Flat data — adds complexity for nothing
- Very deep nesting (> 10 levels) — UX breaks down regardless of implementation
- Mobile (limited horizontal space for indentation)

## Pseudocode

```
function TreeTable<T>(props):
  # Each node: { id, parentId, name, children, childrenLoaded }
  state nodes: list of T
  state expanded: set of id = {}
  state loading: set of id = {}         # children being fetched

  derived:
    # DFS flatten only expanded subtrees
    visibleRows = []
    walk(node, depth):
      visibleRows.push({ node, depth })
      if node.id in expanded:
        for child in node.children:
          walk(child, depth + 1)
    for root in nodes: walk(root, 0)

  function onExpand(node):
    if node.id in expanded:
      expanded.remove(node.id)          # collapse
      return
    expanded.add(node.id)
    if not node.childrenLoaded:         # lazy fetch on first expand
      loading.add(node.id)
      try:
        children = await fetchChildren(node.id)
        node.children = children
        node.childrenLoaded = true
      finally:
        loading.remove(node.id)

  render:
    table:
      visibleRows.map(({ node, depth }) =>
        <row>
          <cell paddingLeft={depth * 20}>
            <ExpandIcon
              expanded={node.id in expanded}
              loading={node.id in loading}
              onClick={() => onExpand(node)}
            />
            {node.name}
          </cell>
          other columns
        </row>
      )
```

## Backend design

For lazy loading, two API patterns:

**Pattern A: per-node children fetch** (recommended)

```
GET /api/tree/:nodeId/children → [Node]
```

**Pattern B: materialized path** (faster for deep trees, but rebuilds on insert)

```
SELECT * FROM tbl WHERE path LIKE 'a/b/c/%' AND depth = 4
```

## Representative libraries

- **AG Grid** — `treeData` + grouping, strongest implementation, paid Enterprise for huge trees
- **Ant Design Table** — `expandable` + `childrenColumnName`, batteries-included
- **TanStack Table** — `getRowCanExpand` + `subRows`, headless, you control rendering
- **React Aria** (`adobe/react-spectrum`) — accessibility-first, low-level primitives

## Pitfalls / Anti-patterns

- ❌ Loading the full tree at once → tens of thousands of nodes freeze the page
  - **Fix**: lazy-load children on expand; root level may also paginate
- ❌ Using `expandedKeys: string[]` for expansion tracking → key collisions across deep trees
  - **Fix**: `Set<id>` with guaranteed-unique IDs (UUID, not just labels)
- ❌ Deep nesting overflows horizontally
  - **Fix**: max indent depth (e.g., 6); beyond that collapse rest under "..." or icon-only
- ❌ Search "find all matching nodes" — naive impl expands every parent of every match → janky UI
  - **Fix**: build expansion set once, then auto-scroll to first match
- ❌ Drag-drop reordering breaks parent-child invariant
  - **Fix**: validate drop target (can't drop parent into own descendant); recompute path
- ❌ Children fetched after expand but parent's count badge stale
  - **Fix**: include child count in parent payload, or refresh badge after fetch
