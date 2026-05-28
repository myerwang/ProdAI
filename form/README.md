---
topic: form
applies_to: [frontend, backend, mobile]
forms: 9
decision: choose by (field set static-or-dynamic, form length, edit granularity, branching, input purpose)
status: stable
last_reviewed: 2026-05-28
---

# Form Patterns — Index

Nine production-grade form forms. Each form lives in its own file with metadata,
pseudocode, pitfalls, and measured high-star references (≥5,000★, per `../AGENTS.md` §3.6).

"Form" here = **collecting and submitting user input** (create/edit a record, run a
query, upload a file). The **table** topic covers *displaying* data; this topic covers
*entering* it. Two of the nine (`06_inline_edit`, `07_autosave_draft`) are **capabilities**
that layer onto the other forms, not standalone shapes — they are tagged as such.

The **cross-cutting layer** — engine choice (controlled vs uncontrolled), schema
validation, sync vs async/server validation, cross-field validation — is covered *inside*
each form rather than as separate files. See "Cross-cutting" below.

---

## Decision Tree (read first)

```
what is the user doing?
├─ submitting a record (create / edit)
│   ├─ field set is FIXED & short                          → 01_single_form
│   ├─ field set is FIXED but long / staged                → 02_multistep_wizard
│   ├─ field COUNT is dynamic (repeatable rows/groups)     → 03_dynamic_field_array
│   ├─ which fields SHOW depends on prior answers          → 04_conditional_survey_form
│   └─ the form DEFINITION is data (JSON Schema / server)  → 05_schema_driven_form
├─ editing one field/record in place (no full form)        → 06_inline_edit       (capability)
├─ must never lose progress (long / flaky network)         → 07_autosave_draft    (capability, layer on 01/02)
├─ entering query criteria to filter a list (not submit)   → 08_search_filter_form
└─ uploading file(s) / binary                              → 09_file_upload_form

orthogonal: pick a validation strategy for ANY of the above → see "Cross-cutting"
```

The first real decision is **is the field set known at build time?** If yes → 01/02. If
the *count* varies → 03. If *which fields* vary → 04. If the *whole definition* is data
(authored elsewhere or served by the backend) → 05.

---

## Summary

| # | Form | File | Kind | Field set | Top OSS (verified 2026-05-28) |
|---|---|---|---|---|---|
| 1 | Single Form | [01_single_form.md](./01_single_form.md) | shape | fixed, short | react-hook-form ~45k★ / formik ~34k★ |
| 2 | Multi-step Wizard | [02_multistep_wizard.md](./02_multistep_wizard.md) | shape | fixed, staged | ant-design ~98k★ / material-ui ~98k★ |
| 3 | Dynamic Field Array | [03_dynamic_field_array.md](./03_dynamic_field_array.md) | shape | dynamic count | react-hook-form ~45k★ / react-dnd ~22k★ |
| 4 | Conditional / Survey | [04_conditional_survey_form.md](./04_conditional_survey_form.md) | shape | dynamic visibility | formbricks ~12k★ / formily ~13k★ |
| 5 | Schema-driven | [05_schema_driven_form.md](./05_schema_driven_form.md) | shape | defined by data | react-jsonschema-form ~16k★ / formily ~13k★ |
| 6 | Inline Edit | [06_inline_edit.md](./06_inline_edit.md) | capability | one field/record | ant-design ~98k★ / material-ui ~98k★ |
| 7 | Autosave / Draft | [07_autosave_draft.md](./07_autosave_draft.md) | capability | layers on 01/02 | react-hook-form ~45k★ |
| 8 | Search / Filter | [08_search_filter_form.md](./08_search_filter_form.md) | shape | query criteria | ant-design ~98k★ / material-ui ~98k★ |
| 9 | File Upload | [09_file_upload_form.md](./09_file_upload_form.md) | shape | file(s) / binary | uppy ~31k★ / filepond ~16k★ |

---

## Cross-cutting (applies inside every form)

These are not separate forms; each form file applies them as needed.

- **Engine: controlled vs uncontrolled** — controlled re-renders on every keystroke
  (simple, but costly for big forms); uncontrolled (ref/subscription-based) re-renders
  only on submit/validation (react-hook-form's model). Default to uncontrolled for large
  or high-frequency forms.
- **Schema validation** — define the shape once and derive both runtime validation and
  the type: `zod` ~43k★ / `yup` ~24k★ / `valibot` ~8.7k★. Wire the schema to the engine via
  a resolver/adapter. Single source of truth for validation rules.
- **Sync vs async/server validation** — sync rules (required, format, range) run on the
  client; uniqueness / existence checks must hit the server, debounced, with the field
  marked "validating". Never trust client validation alone — re-validate on the server at
  submit (defense in depth).
- **Cross-field validation** — rules spanning fields (`end > start`, `confirm == password`)
  belong to the schema/form level, not the field level.

References for the cross-cutting layer (verified 2026-05-28 via GitHub API):

- [colinhacks/zod](https://github.com/colinhacks/zod) — ~43k★: TS-first schema validation, the de-facto resolver target
- [jquense/yup](https://github.com/jquense/yup) — ~24k★: object schema validation, classic Formik pairing
- [fabian-hiller/valibot](https://github.com/fabian-hiller/valibot) — ~8.7k★: modular, tree-shakable schema validation (repo now under `open-circle/`)

---

## Common Combinations

- **Onboarding flow**: 02 (wizard) + 07 (autosave each step) + 04 (branch by plan/role)
- **CMS / admin "settings" page**: 01 (single) + 07 (autosave dirty fields)
- **Invoice / order builder**: 03 (line-item field array) inside 01 or 02
- **Public questionnaire**: 04 (conditional) authored via 05 (schema), rendered generically
- **Backend-owned form (no redeploy to change fields)**: 05 (server-driven schema) + 04 (rules)
- **Data table row quick-edit**: 06 (inline edit) — distinct from `table/05_editable_data_grid` (grid/cell-level)
- **List page filters**: 08 (search/filter) feeding a `table/` form
- **Media / document submission**: 09 (file upload) embedded as a field inside 01/02

---

## Boundaries with other topics

- **`table/`** — *displaying* rows. Inline cell editing across a whole grid is
  `table/05_editable_data_grid`; editing a single record/field in a detail view is `06_inline_edit` here.
- **Rich-text / WYSIWYG editors** — a *field type* (one input within a form), not a form
  shape. Out of scope here.
- **Visual query-builder UIs** (nested AND/OR trees) — a niche sub-pattern of `08`; the
  dedicated OSS for it currently sits below the 5,000★ bar, so `08` cites the general
  advanced-search form pattern (Ant Design / MUI) as its primary reference.

---

## How to extend

This folder was created upfront as a recognized multi-form taxonomy (`../AGENTS.md` §2.3).
When a sub-pattern grows to a 3rd related document, deepen it:

```
02_multistep_wizard.md     ← form overview (current)
multistep_wizard/          ← when deep-dives accumulate
├── branching_steps.md
├── step_state_machine.md
└── ...
```

See `../CONTRIBUTING.md` §3 for the on-demand vs upfront-taxonomy rules.

---

## History

- **2026-05-28**: Initial taxonomy. 9 forms, created upfront per AGENTS.md §2.3. Primary
  axis = UX/architecture shape; state-management & validation covered cross-cutting inside
  each form. All references measured via GitHub API (≥5,000★ bar).
