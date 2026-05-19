---
form: editable_data_grid
topic: table
applies_to: [frontend]
data_scale: [small, medium]
decision: cell-level edit-in-place needed, Excel-like UX expected
status: stable
last_reviewed: 2026-05-19
---

# Form 5: Editable Data Grid (Excel-like)

Cell-level edit-in-place. Keyboard navigation, copy-paste, undo, optionally formulas.

## When to use

- Financial sheets, ERP, data entry forms
- Users expect Excel-like UX (keyboard nav, fill-down, paste from clipboard)
- Data < 100k (combine with virtual scroll for more)

## When NOT to use

- Read-mostly data — Form 1 (offset) is simpler
- Mobile — Excel UX doesn't translate to touch
- Casual users who don't know Excel shortcuts

## Pseudocode

```
function DataGrid<T>(props):
  state items: list of T = []
  state selection: { row: int, col: int } = { row: 0, col: 0 }
  state editingCell: optional { row: int, col: int } = null
  state clipboard: any = null
  state undoStack: list of Change = []

  function onCellClick(row, col):
    selection = { row, col }

  function onCellDoubleClick(row, col):
    editingCell = { row, col }

  function onKey(event):
    if not editingCell:
      switch event.key:
        case ArrowUp:    selection = { row: selection.row - 1, col: selection.col }
        case ArrowDown:  selection = { row: selection.row + 1, col: selection.col }
        case ArrowLeft:  selection = { row: selection.row, col: selection.col - 1 }
        case ArrowRight: selection = { row: selection.row, col: selection.col + 1 }
        case Enter:      editingCell = selection
        case Ctrl+C:     clipboard = getCell(selection)
        case Ctrl+V:     setCell(selection, clipboard)
        case Delete:     setCell(selection, null)
        case Ctrl+Z:     undo()

  function setCell(loc, newValue):
    oldValue = items[loc.row][loc.col]
    items[loc.row][loc.col] = newValue
    undoStack.push({ loc, oldValue, newValue })
    debouncedSave()           # batch save to backend, e.g. 500ms

  function commitEdit(newValue):
    setCell(editingCell, newValue)
    editingCell = null

  function undo():
    if undoStack.empty: return
    change = undoStack.pop()
    items[change.loc.row][change.loc.col] = change.oldValue
    debouncedSave()

  render:
    keyboard listener on grid container
    grid:
      rows.map((row, i) =>
        cols.map((col, j) =>
          if editingCell == { row: i, col: j }:
            <input value={row[col]} onCommit={commitEdit} autofocus />
          else:
            <cell selected={selection eq { row: i, col: j }}
                  onClick={() => onCellClick(i, j)}
                  onDoubleClick={() => onCellDoubleClick(i, j)} />
        )
      )
```

## Representative libraries

- **Handsontable** (`handsontable/handsontable`, 21k+) — best Excel feel, formulas, merged cells, partial commercial license
- **Glide Data Grid** (`glideapps/glide-data-grid`, 6k+) — Canvas-rendered, 1M+ rows smooth, React-only
- **AG Grid Community** (`ag-grid/ag-grid`, 14k+) — full-featured grid; Excel export needs Enterprise
- **react-data-grid** (`adazzle/react-data-grid`, 7k+) — lighter open-source alternative
- **Slick Grid** (`6pac/SlickGrid`, 6k+) — classic, jQuery-era veteran still maintained

## Pitfalls / Anti-patterns

- ❌ PATCH on every cell change → network spam
  - **Fix**: debounce 500ms; batch on blur, save button, or page leave
- ❌ No undo stack → user mistakes are irreversible
  - **Fix**: maintain Change history; support Ctrl+Z; consider redo too
- ❌ Formula references with cycles (A=B+1, B=A+1) → infinite loop
  - **Fix**: cycle detection in formula evaluator before recompute
- ❌ Locale-dependent number parsing — "1,000" interpreted as 1.0 in some locales
  - **Fix**: explicit format spec, not browser locale
- ❌ Paste from Excel preserves formatting → unwanted style bleed
  - **Fix**: paste as plain text, parse manually
- ❌ Concurrent edits from multiple users → last-write-wins data loss
  - **Fix**: optimistic locking with version field, or CRDT-style merge (rare in data grids)
- ❌ Validation only on save → user discovers errors after typing whole row
  - **Fix**: per-cell async validation with debounce, inline error indicator
