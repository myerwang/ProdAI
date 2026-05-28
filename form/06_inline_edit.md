---
form: inline_edit
topic: form
applies_to: [frontend, backend]
field_set: one field/record in place
kind: capability
decision: edit a single field/record in place, no full form, optimistic save
status: stable
last_reviewed: 2026-05-28
---

# Form 6: Inline Edit (edit-in-place)

A **capability**, not a full-form shape. The user clicks a value to edit it in place,
saves one field (or one small record) on blur/Enter, and stays on the page. Profile fields,
detail-panel quick edits, single-cell tweaks.

## When to use

- Editing one field or a tiny record without navigating to a form page
- Display-mode is the default; edit-mode is entered on demand
- Low-friction quick edits (rename, toggle, change a status)

## When NOT to use

- Editing many cells across a grid at once → `table/05_editable_data_grid` (grid-level)
- A whole record with many related fields → Form 1 (single form)
- Edits must be validated together / submitted atomically → Form 1/2

## Boundary vs table editable grid

```
table/05_editable_data_grid : MANY rows × cells, spreadsheet-like, bulk edit, grid engine
06_inline_edit (here)       : ONE field/record in a detail/list view, edit-in-place toggle
```

## Conclusion

Per-field display↔edit toggle + save on blur/Enter, cancel on Escape + **optimistic update
with rollback** on failure + per-field validation before commit. Each field saves
independently (PATCH one field), so failures are isolated.

## Pseudocode

```
function InlineField(props):                  # props: value, validate, save (PATCH one field)
  state mode: "view" | "edit" = "view"
  state draft = props.value
  state status: "idle" | "saving" | "error" = "idle"

  function begin():
    draft = props.value
    mode = "edit"

  function cancel():                           # Escape
    draft = props.value
    mode = "view"

  async function commit():                     # blur / Enter
    if draft == props.value: mode = "view"; return    # no-op, skip network
    err = props.validate(draft)
    if err: status = "error"; return            # stay in edit, show error

    prev = props.value
    props.onOptimistic(draft)                   # optimistic: UI shows new value now
    mode = "view"; status = "saving"
    try:
      saved = await props.save(draft)           # PATCH { field: draft }
      props.onOptimistic(saved)                 # reconcile with server echo
      status = "idle"
    catch e:
      props.onOptimistic(prev)                  # ROLLBACK to previous value
      status = "error"; mode = "edit"           # let user retry

  render:
    if mode == "view":
      clickable value (props.value) + edit affordance
      if status == "error": small retry hint
    else:
      input bound to draft
        on Enter → commit, on Escape → cancel, on blur → commit
      if status == "saving": subtle spinner
```

## Concurrency / staleness

- Inline edits race with other users and with stale page data. Send the field's previous
  value or a version/etag so the server can reject a write based on stale data
  (optimistic concurrency) instead of silently clobbering.
- On conflict, surface "this changed since you loaded it" and refetch.

## References

High-star OSS implementations (stars verified 2026-05-28 via GitHub API; ≥5,000★ bar):

- [ant-design/ant-design](https://github.com/ant-design/ant-design) — ~98k★: `Typography` editable text + editable Table cell pattern
- [mui/material-ui](https://github.com/mui/material-ui) — ~98k★: inline-editable fields / DataGrid single-cell edit
- [react-hook-form/react-hook-form](https://github.com/react-hook-form/react-hook-form) — ~45k★: per-field validation/state for the edit-mode input
- [mantinedev/mantine](https://github.com/mantinedev/mantine) — ~31k★: editable text / inline input components

## Pitfalls / Anti-patterns

- ❌ Optimistic update with no rollback → UI shows a value the server rejected
  - **Fix**: snapshot previous value, restore on failure, re-enter edit
- ❌ Saving on blur AND Enter fires twice → double PATCH
  - **Fix**: guard with a "no change since last commit" check; debounce/lock during save
- ❌ Saving when the value didn't change → pointless network + audit noise
  - **Fix**: skip commit when `draft == value`
- ❌ No Escape-to-cancel → user can't back out of a mistaken edit
  - **Fix**: Escape restores draft and returns to view mode
- ❌ Last-write-wins clobbering concurrent edits
  - **Fix**: send version/etag; server rejects stale writes; prompt refetch
- ❌ Using inline edit for a many-field record → fragmented saves, no atomicity
  - **Fix**: switch to Form 1 (single form) and submit together
