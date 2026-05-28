---
form: autosave_draft
topic: form
applies_to: [frontend, backend]
field_set: layers on 01/02
kind: capability
decision: long/important form must survive refresh/crash; persist progress automatically
status: stable
last_reviewed: 2026-05-28
---

# Form 7: Autosave / Draft (persistence capability)

A **capability** layered onto Form 1 or Form 2, not a standalone shape. The form persists
progress automatically (debounced) so a refresh, crash, or navigation doesn't lose work.
Drafts can be recovered later. Long forms, document editors, multi-session applications.

## When to use

- The form is long or valuable enough that losing input is unacceptable
- Users may leave and return across sessions/devices
- Network is flaky and you want progress saved as they go

## When NOT to use

- Short forms submitted in one sitting → just Form 1 (no autosave needed)
- Each save has side effects (emails, charges) → autosave is dangerous; use explicit submit
- Strict validation must pass before *any* persistence → drafts are partial/invalid by nature

## Conclusion

Debounce changes + save only **dirty** fields (a diff, not the whole form) + draft state is
**partial and may be invalid** (relax validation for drafts, enforce on final submit) +
clear save-status feedback (saving / saved / error) + on load, offer to resume a draft.

## Pseudocode

```
function AutosaveForm(props):
  form = useForm({ defaultValues: props.initial, mode: "onBlur" })
  state saveState: "idle" | "saving" | "saved" | "error" = "idle"
  state lastSavedAt: optional timestamp = null
  seqRef = 0

  # debounce: wait for a quiet window before saving
  debouncedSave = debounce(700ms, save)

  effect on (form values change AND form.isDirty):
    saveState = "saving"
    debouncedSave()

  async function save():
    seq = ++seqRef
    dirty = form.getDirtyValues()              # diff, not the whole payload
    if isEmpty(dirty): saveState = "idle"; return
    try:
      await props.saveDraft({ id: props.draftId, patch: dirty })  # PATCH draft, NOT submit
      if seq != seqRef: return                  # a newer save superseded this one
      form.markClean(dirty)                      # those fields are no longer dirty
      saveState = "saved"; lastSavedAt = now()
    catch e:
      if seq != seqRef: return
      saveState = "error"                        # keep dirty, retry on next change/backoff

  # resume flow
  effect on mount:
    draft = await props.loadDraft(props.draftId)
    if draft and draft.newerThan(props.initial):
      ask user: "Resume your saved draft?" → form.reset(draft) | discard

  async function finalSubmit(values):
    if not await form.validate(): return         # FULL validation only at submit
    await props.submit(values)
    await props.deleteDraft(props.draftId)        # clean up the draft

  render:
    ...fields (Form 1 or Form 2 body)...
    status indicator: saveState + "last saved " + relativeTime(lastSavedAt)
    submit button → finalSubmit
```

## Draft validation policy

```
draft save:    relax validation — partial, half-typed, invalid values are allowed to persist
final submit:  full schema validation must pass

flush pending debounce on: blur of last field, tab hidden (visibilitychange), beforeunload
storage choice:
  - server draft (recommended): cross-device, survives cache clear; needs draft table + TTL
  - local only (offline/simple): localStorage/IndexedDB; lost on cache clear, single device
```

## References

High-star OSS implementations (stars verified 2026-05-28 via GitHub API; ≥5,000★ bar):

- [react-hook-form/react-hook-form](https://github.com/react-hook-form/react-hook-form) — ~45k★: `watch` + `formState.dirtyFields` + `isDirty` to drive debounced dirty-only autosave
- [jaredpalmer/formik](https://github.com/jaredpalmer/formik) — ~34k★: `dirty` + values subscription for autosave-on-change patterns
- [TanStack/form](https://github.com/TanStack/form) — ~6.5k★: granular field subscriptions suited to debounced persistence

## Pitfalls / Anti-patterns

- ❌ Saving on every keystroke with no debounce → request storm, server load, races
  - **Fix**: debounce (e.g. ~700ms quiet window); coalesce changes
- ❌ Out-of-order saves: an older request lands after a newer one → stale data wins
  - **Fix**: sequence guard (seqRef) or server-side version; ignore superseded responses
- ❌ Enforcing full validation on draft saves → can't save half-finished work
  - **Fix**: relax validation for drafts; enforce only on final submit
- ❌ Saving the entire form each time → bandwidth waste, more conflict surface
  - **Fix**: save dirty diff only; mark clean on success
- ❌ Autosave wired to an endpoint with side effects (charge/email/notify)
  - **Fix**: drafts go to a side-effect-free draft store; side effects happen only on submit
- ❌ Pending debounced save lost when the tab closes
  - **Fix**: flush on visibilitychange/beforeunload
- ❌ Silent autosave with no feedback → user unsure if work is safe
  - **Fix**: explicit saving/saved/error indicator + last-saved time
