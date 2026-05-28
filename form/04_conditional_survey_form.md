---
form: conditional_survey_form
topic: form
applies_to: [frontend, backend]
field_set: dynamic visibility
decision: which fields show/are required depends on prior answers (branching / skip logic)
status: stable
last_reviewed: 2026-05-28
---

# Form 4: Conditional / Survey Form (branching, skip logic)

Which fields appear (and which become required) depends on earlier answers. Surveys,
questionnaires, eligibility flows, dynamic application forms.

## When to use

- Field visibility/requiredness is driven by other fields' values
- "If answered X, show questions Y and Z; otherwise skip"
- Questionnaires with skip logic, progressive disclosure, eligibility branching

## When NOT to use

- All fields always show (just long) → Form 2 (wizard)
- Only the *count* of a repeated group varies → Form 3 (field array)
- The branching rules are authored as data by non-developers → drive Form 5 (schema) with
  the rules engine described here

## Conclusion

Declarative visibility rules (data, not nested `if` in markup) + hidden fields are
**excluded from validation and from the submitted payload** + recompute reactively when
dependencies change.

## Pseudocode

```
# rules as DATA, evaluated against current values — not hard-coded JSX conditionals
fields = [
  { id: "employed", type: "bool" },
  { id: "employer", type: "string", visibleWhen: { employed: true }, requiredWhen: { employed: true } },
  { id: "salary",   type: "number", visibleWhen: { employed: true } },
  { id: "jobSeeking", type: "bool", visibleWhen: { employed: false } },
]

function ConditionalForm(props):
  form = useForm({ defaultValues: props.initial })
  values = form.watch()                            # reactive snapshot

  function isVisible(field):
    return matches(field.visibleWhen, values)      # null/absent ⇒ always visible

  function isRequired(field):
    return isVisible(field) and matches(field.requiredWhen, values)

  visibleFields = fields.filter(isVisible)

  function onSubmit(raw):
    # CRITICAL: strip hidden fields before validate + submit
    payload = pick(raw, visibleFields.map(f => f.id))
    errors = validate(payload, rulesFor(visibleFields))   # only visible-field rules
    if errors: form.setErrors(errors); return
    await props.submit(payload)

  render:
    for field in visibleFields:                    # re-renders when deps change
      renderField(field) with required = isRequired(field)
    submit button → form.handleSubmit(onSubmit)
```

## Hidden-field semantics (the crux)

```
when a field becomes hidden:
  - it MUST NOT block submit on a "required" rule
  - its value MUST NOT be sent (or must be nulled) — stale answers to skipped questions
  - re-showing it later: decide keep-old-value vs reset-to-default (be consistent)

re-validate ONLY currently-visible fields; server applies the SAME rule engine
```

## Server-side parity

- The branching/visibility rules must be evaluated on the server too — a client can submit
  any payload. Re-run the same rule set server-side to decide which fields are required and
  to reject answers to questions that should have been skipped.
- Keep the rule definition shared (data), so client and server evaluate identically.

## References

High-star OSS implementations (stars verified 2026-05-28 via GitHub API; ≥5,000★ bar):

- [formbricks/formbricks](https://github.com/formbricks/formbricks) — ~12k★: open-source survey platform with conditional logic / branching
- [alibaba/formily](https://github.com/alibaba/formily) — ~13k★: reactive field "reactions" (visibility/required driven by other fields)
- [react-hook-form/react-hook-form](https://github.com/react-hook-form/react-hook-form) — ~45k★: `watch` + conditional render + `shouldUnregister` to drop hidden fields

## Pitfalls / Anti-patterns

- ❌ Hidden-but-required field blocks submit → user stuck on a question they can't see
  - **Fix**: requiredness is conditional on visibility; validate visible fields only
- ❌ Submitting stale answers to skipped questions → bad data downstream
  - **Fix**: strip/null hidden fields from the payload before submit
- ❌ Deeply nested `if` conditionals in markup → unmaintainable, untestable rules
  - **Fix**: rules as data, one `isVisible(field, values)` evaluator
- ❌ Visibility computed once (not reactive) → fields don't appear when deps change
  - **Fix**: recompute on every dependency change (watch)
- ❌ Trusting client visibility only → server accepts answers to skipped questions
  - **Fix**: evaluate the same rules server-side; reject inconsistent payloads
- ❌ Circular dependencies (A shows B, B shows A) → infinite recompute / flicker
  - **Fix**: validate the rule graph is acyclic at authoring time
