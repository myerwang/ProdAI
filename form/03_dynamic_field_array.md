---
form: dynamic_field_array
topic: form
applies_to: [frontend, backend]
field_set: dynamic count
decision: a group of fields repeats an arbitrary number of times (add/remove/reorder)
status: stable
last_reviewed: 2026-05-28
---

# Form 3: Dynamic Field Array (repeatable groups)

A sub-group of fields repeats an unknown number of times: invoice line items, multiple
phone numbers, team members, key-value pairs. The user adds, removes, and reorders rows.

## When to use

- The *count* of a field/group is variable and user-driven
- Rows are homogeneous (same field shape repeated)
- You may need nested arrays (rows that themselves contain arrays)

## When NOT to use

- Fixed, known number of fields → Form 1 (single)
- *Which* fields show varies (heterogeneous by answer) → Form 4 (conditional)
- The repeated rows are really tabular data being edited en masse → consider
  `table/05_editable_data_grid`

## Conclusion

Stable per-row identity (not array index) + array-level helpers (append/remove/move) +
validation addressed by path (`items[2].price`). Reorder via drag-drop updates order only.

## Pseudocode

```
function FieldArrayForm(props):
  form = useForm({ schema, defaultValues: { items: props.initial ?? [] } })
  array = useFieldArray(form, "items")          # gives fields[], append, remove, move

  emptyRow = { _key: newId(), name: "", qty: 1, price: 0 }

  render:
    for (row, index) in array.fields:           # row._key is STABLE, survives reorder
      group keyed by row._key:
        field("items[" + index + "].name")
        field("items[" + index + "].qty")
        field("items[" + index + "].price")
        button "remove" → array.remove(index)
        drag handle → on drop(from, to): array.move(from, to)
    button "add row" → array.append(emptyRow)

    # derived/aggregate value recomputes from the array
    total = sum(form.watch("items"), row => row.qty * row.price)
    show "Total: " + total

    submit button → form.handleSubmit(props.submit)
```

## Why stable keys matter

```
# WRONG: key = array index
remove(1) shifts indices → React reuses DOM nodes → values land on the wrong row,
focus jumps, uncontrolled inputs show stale data

# RIGHT: key = stable per-row id assigned at append time (row._key)
remove/reorder preserves identity → correct values, focus, and validation per row
```

## Validation

- Validate per-path (`items[i].price > 0`) and surface the error on that row.
- Array-level rules (min 1 row, max 20 rows, unique names across rows) live at the array
  schema level, not per field.
- Show row errors inline next to the offending row, plus a form-level summary if many.

## References

High-star OSS implementations (stars verified 2026-05-28 via GitHub API; ≥5,000★ bar):

- [react-hook-form/react-hook-form](https://github.com/react-hook-form/react-hook-form) — ~45k★: `useFieldArray` (append/remove/move/swap, stable `id` per row)
- [jaredpalmer/formik](https://github.com/jaredpalmer/formik) — ~34k★: `FieldArray` helper (push/remove/move)
- [react-dnd/react-dnd](https://github.com/react-dnd/react-dnd) — ~22k★: drag-and-drop backend for reordering rows
- [alibaba/formily](https://github.com/alibaba/formily) — ~13k★: `ArrayField` / array-table with built-in add/remove/sort

## Pitfalls / Anti-patterns

- ❌ Using the array index as the React key → values/focus jump on remove/reorder
  - **Fix**: assign a stable id per row at append time, key by that
- ❌ Recomputing aggregates (totals) in render from a controlled mirror that lags
  - **Fix**: derive from the watched array value; keep one source of truth
- ❌ No min/max guard → user submits 0 rows or 10,000 rows
  - **Fix**: array-level schema rules (min/max), disable "add" at max
- ❌ Validating the whole array on every keystroke in any row → O(rows) work per key
  - **Fix**: validate the touched path; debounce array-level checks
- ❌ Sending the full array on every change to the server → chatty, race-prone
  - **Fix**: submit once; if autosaving (Form 7), send diffs/debounce
- ❌ Nested arrays addressed by fragile string concatenation that breaks on reorder
  - **Fix**: rely on the library's path API and stable ids at every level
