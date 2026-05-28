---
form: multistep_wizard
topic: form
applies_to: [frontend, backend]
field_set: fixed, staged
decision: fixed but long field set, split into ordered steps with per-step validation
status: stable
last_reviewed: 2026-05-28
---

# Form 2: Multi-step Wizard (stepper)

A long but fixed field set, split into ordered steps. Per-step validation, progress
indicator, back/next navigation, review-before-submit. One logical submit at the end.

## When to use

- The full field set is fixed but too long/intimidating for one screen
- Steps have a natural order (account → profile → payment → review)
- You want to validate incrementally and stop the user advancing on errors
- Checkout, onboarding, KYC, multi-section applications

## When NOT to use

- The field set fits on one screen → Form 1 (single)
- *Which* steps appear depends on earlier answers and the graph is non-linear → see
  "Branching" below (still a wizard, but model the order as a state machine), or Form 4
- The whole definition is data/served by backend → Form 5 driving a wizard layout

## Conclusion

One form state object spanning all steps + per-step validation gate on "next" + the final
submit sends the *whole* accumulated payload. Keep step definitions declarative (a list).

## Pseudocode

```
steps = [
  { id: "account", fields: ["email", "password"] },
  { id: "profile", fields: ["name", "country"] },
  { id: "review",  fields: [] },                      # read-only summary step
]

function Wizard(props):
  form = useForm({ schema: fullSchema, defaultValues: props.initial })
  state stepIdx: int = 0
  state maxVisited: int = 0                            # for clickable progress

  current = steps[stepIdx]

  async function next():
    # validate ONLY the current step's fields, not the whole form
    ok = await form.validateFields(current.fields)
    if not ok: return                                  # stay, show errors
    stepIdx = min(stepIdx + 1, length(steps) - 1)
    maxVisited = max(maxVisited, stepIdx)

  function prev():
    stepIdx = max(stepIdx - 1, 0)                       # never re-validate going back

  async function finish():
    if not await form.validate(): return                # full validation at the end
    values = form.getValues()                           # whole accumulated payload
    await props.submit(values)

  render:
    progress(steps, stepIdx, maxVisited)                # click only to ≤ maxVisited
    switch current.id:
      "account": fields email, password
      "profile": fields name, country
      "review":  read-only summary of getValues()
    footer:
      if stepIdx > 0: button "Back" → prev
      if stepIdx < last: button "Next" → next
      else: button "Submit" → finish (disabled while submitting)
```

## Branching steps (non-linear order)

```
# when the next step depends on answers, model order as a transition function,
# NOT array index arithmetic
function nextStepId(currentId, values):
  if currentId == "account":
    return values.isBusiness ? "company" : "profile"
  ...
# keep a visited stack so "Back" can unwind correctly
```

If branching gets complex (many conditional fields *within* steps), compose with Form 4.

## State persistence

- Keep one form state object for the whole wizard; do **not** unmount-and-lose data when
  switching steps. Either keep all step components mounted (hidden) or persist values to
  the form state before unmounting a step.
- For long/important flows, layer Form 7 (autosave/draft) so a refresh/crash doesn't wipe
  progress.

## References

High-star OSS implementations (stars verified 2026-05-28 via GitHub API; ≥5,000★ bar):

- [ant-design/ant-design](https://github.com/ant-design/ant-design) — ~98k★: Steps + Form, the canonical step/wizard UI
- [mui/material-ui](https://github.com/mui/material-ui) — ~98k★: Stepper (horizontal/vertical, linear/non-linear)
- [react-hook-form/react-hook-form](https://github.com/react-hook-form/react-hook-form) — ~45k★: one form state spanning steps, partial `trigger`/validate per step
- [mantinedev/mantine](https://github.com/mantinedev/mantine) — ~31k★: Stepper component + `useForm` for per-step state

## Pitfalls / Anti-patterns

- ❌ Validating the whole form on every "Next" → user blocked by errors in steps they
  haven't reached
  - **Fix**: validate only the current step's fields on "Next"; full validate only at submit
- ❌ Unmounting a step wipes its values → data lost when navigating back
  - **Fix**: single form state for all steps; persist before unmount or keep mounted+hidden
- ❌ Re-validating when going **back** → annoying errors on fields the user is leaving
  - **Fix**: never validate on "Back"
- ❌ Index arithmetic (`stepIdx + 1`) for branching flows → wrong step, broken "Back"
  - **Fix**: transition function + visited stack
- ❌ Submitting each step to the server separately, then a half-finished record lingers
  - **Fix**: accumulate client-side, submit once at the end; or use explicit server-side
    draft state with a clear "completed" transition
- ❌ Progress bar lets users jump to unvisited future steps → land on empty/invalid step
  - **Fix**: allow jumping only to already-visited steps (`maxVisited`)
