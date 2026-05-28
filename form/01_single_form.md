---
form: single_form
topic: form
applies_to: [frontend, backend]
field_set: fixed, short
decision: fixed field set, fits on one screen, single submit
status: stable
last_reviewed: 2026-05-28
---

# Form 1: Single-page Form (controlled/uncontrolled + schema-validated)

The 90% case. A fixed set of fields on one screen, validated against a schema, submitted
once. Create-or-edit a single record.

## When to use

- The field set is known at build time and fits comfortably on one screen
- One logical submit (create or update one record)
- Validation rules are mostly static (required / format / range / cross-field)

## When NOT to use

- Field set is long enough to overwhelm → Form 2 (wizard)
- The *number* of a field repeats (line items, phone numbers) → Form 3 (field array)
- Which fields appear depends on answers → Form 4 (conditional)
- The form definition is authored as data / served by backend → Form 5 (schema-driven)
- You just edit one field in place → Form 6 (inline edit)

## Conclusion

Uncontrolled engine (subscription-based) for anything non-trivial + a single schema as the
source of truth for both validation and types + server re-validation at submit.

## Pseudocode

```
# one schema = validation rules + inferred types, reused on client and server
schema = defineSchema({
  name:  string, required, maxLen 100
  email: string, required, format email
  age:   int, optional, range 0..150
})

function SingleForm(props):
  # uncontrolled: fields register with the engine, no re-render per keystroke
  form = useForm({ schema, defaultValues: props.initial, mode: "onBlur" })

  function onSubmit(values):
    # client already validated via schema; server MUST re-validate (defense in depth)
    set form.submitting true
    try:
      result = await props.submit(values)     # POST/PUT to backend
      form.reset(result)                       # sync clean state from server echo
      props.onSuccess(result)
    catch err:
      if err.fieldErrors:                      # server-side validation failures
        form.setErrors(err.fieldErrors)        # map onto the matching fields
      else:
        props.onError(err)
    finally:
      set form.submitting false

  render:
    form bound to onSubmit:
      field("name")  + error("name")
      field("email") + error("email")
      field("age")   + error("age")
      submit button:
        disabled when form.submitting or (form.submitCount > 0 and not form.valid)
        label = form.submitting ? "Saving…" : "Save"
```

## Controlled vs uncontrolled (the core engine choice)

```
controlled:
  every keystroke updates form state → whole form (or subtree) re-renders
  simplest mental model; fine for small forms
  cost: O(fields) re-render per keystroke on large forms

uncontrolled (recommended default for non-trivial):
  inputs hold their own value (ref); engine subscribes
  re-render only on validation / submit / watched-field change
  scales to large forms without per-keystroke jank
```

## Validation strategy

- `mode`: validate `onSubmit` (least noisy), `onBlur` (balanced, recommended), or
  `onChange` (most eager, costly). Re-validate eagerly *after* first failed submit.
- Sync rules (required/format/range) from the schema run on the client.
- Async rules (uniqueness, existence) hit the server, debounced, field marked "validating".
- **Always re-validate on the server at submit** and map `fieldErrors` back onto fields.

## References

High-star OSS implementations (stars verified 2026-05-28 via GitHub API; ≥5,000★ bar):

- [react-hook-form/react-hook-form](https://github.com/react-hook-form/react-hook-form) — ~45k★: uncontrolled, subscription-based engine; resolver adapters for zod/yup/valibot
- [jaredpalmer/formik](https://github.com/jaredpalmer/formik) — ~34k★: controlled-model form state, classic yup pairing
- [colinhacks/zod](https://github.com/colinhacks/zod) — ~43k★: schema = validation + inferred types, single source of truth
- [TanStack/form](https://github.com/TanStack/form) — ~6.5k★: headless, framework-agnostic form state with typed validation
- [logaretm/vee-validate](https://github.com/logaretm/vee-validate) — ~11k★: the Vue-side equivalent (schema-driven validation)

## Pitfalls / Anti-patterns

- ❌ Controlled inputs on a large form → per-keystroke re-render jank
  - **Fix**: uncontrolled engine; isolate expensive subtrees
- ❌ Trusting client validation only → invalid/malicious data persisted
  - **Fix**: re-validate on the server with the *same* schema; never skip
- ❌ Duplicating validation rules in two places (client form + server) that drift apart
  - **Fix**: one schema shared by both, or generate one from the other
- ❌ No mapping of server `fieldErrors` back to fields → user sees a generic toast, can't fix
  - **Fix**: return structured field errors and `setErrors` onto the matching fields
- ❌ Resetting to empty after save → loses server-computed fields (ids, timestamps)
  - **Fix**: `reset()` from the server echo, not from the submitted values
- ❌ Submit button enabled while submitting → double-submit / duplicate records
  - **Fix**: disable while `submitting`; idempotency key on the backend for safety
