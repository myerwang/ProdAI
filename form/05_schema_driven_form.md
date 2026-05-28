---
form: schema_driven_form
topic: form
applies_to: [frontend, backend]
field_set: defined by data
decision: the form definition is data (JSON Schema / config), possibly served by backend
status: stable
last_reviewed: 2026-05-28
---

# Form 5: Schema-driven Form (generated from JSON Schema / config)

The form is not hand-coded; a **data definition** (JSON Schema, UI schema, or config)
describes the fields, and a generic renderer turns it into a working form. Includes the
**server-driven** variant where the backend owns and serves the definition.

## When to use

- Forms must be added/changed *without a frontend redeploy* (server-driven)
- Many similar forms — author them as data instead of coding each
- Non-developers (admins) author forms via a builder that emits a schema
- The validation schema already exists and you want the UI generated from it

## When NOT to use

- A handful of bespoke forms with custom layout/interactions → Form 1/2 (hand-coded is simpler)
- One-off form → schema indirection is overkill
- Highly custom per-field UX that fights the generic renderer → hand-code it

## Conclusion

Separate **data schema** (types + validation) from **UI schema** (widgets, order, layout).
A widget registry maps field types → components. Server-driven = fetch the schema at
runtime; **always re-validate against the schema server-side**.

## Pseudocode

```
# data schema: shape + validation (e.g. JSON Schema)
dataSchema = {
  fields: {
    email:   { type: "string", format: "email", required: true },
    plan:    { type: "enum", values ["free","pro"], required: true },
    seats:   { type: "int", min: 1 },
  }
}
# ui schema: presentation only (separate concern)
uiSchema = {
  order: ["email", "plan", "seats"],
  widgets: { plan: "radio", seats: "stepper" },
}

# widget registry: field type/widget → component (the only place UI is hand-coded)
registry = {
  "string": TextWidget, "enum": SelectWidget, "radio": RadioWidget,
  "int": NumberWidget, "stepper": StepperWidget, ...
}

function SchemaForm(props):
  # server-driven variant: schema arrives at runtime
  schema = props.schema ?? await fetchSchema(props.formId)
  form = useForm({ resolver: schemaResolver(schema.dataSchema) })

  function renderField(name):
    def = schema.dataSchema.fields[name]
    widgetName = schema.uiSchema.widgets[name] ?? def.type
    Widget = registry[widgetName] ?? FallbackWidget
    return Widget({ name, def, control: form })

  render:
    for name in schema.uiSchema.order:
      renderField(name)
    submit button → form.handleSubmit(values => props.submit(props.formId, values))
```

## Server-driven variant

```
# backend owns the definition; changing a form = changing data, no client deploy
GET  /forms/{id}/schema   → { dataSchema, uiSchema, version }
POST /forms/{id}/submit   → server validates payload against dataSchema[version]

client caches schema by (id, version); invalidates when version changes
unknown field type from server → FallbackWidget + log, never crash the form
```

## Data schema vs UI schema (keep separate)

- **Data schema** = source of truth for validation + types; shared with the server.
- **UI schema** = order, grouping, which widget, labels, help text. Presentation only.
- Mixing them means a layout tweak risks changing validation, and vice versa.

## References

High-star OSS implementations (stars verified 2026-05-28 via GitHub API; ≥5,000★ bar):

- [rjsf-team/react-jsonschema-form](https://github.com/rjsf-team/react-jsonschema-form) — ~16k★: render + validate a form from JSON Schema + separate UI schema
- [alibaba/formily](https://github.com/alibaba/formily) — ~13k★: JSON-schema-driven forms with a widget registry and reactions
- [colinhacks/zod](https://github.com/colinhacks/zod) — ~43k★: code-first schema you can also drive generic rendering from (schema → UI)

## Pitfalls / Anti-patterns

- ❌ Merging data schema and UI schema → layout changes break validation
  - **Fix**: two separate documents, one renderer that consumes both
- ❌ Unknown field type from a server schema crashes the whole form
  - **Fix**: FallbackWidget + log; degrade gracefully, never blank-screen
- ❌ Trusting the client to enforce a server-served schema
  - **Fix**: server validates the payload against the same schema+version it served
- ❌ No schema versioning → a deployed schema change breaks in-flight drafts
  - **Fix**: version schemas; validate drafts against the version they were started on
- ❌ Reaching for schema-driven for 2–3 bespoke forms → indirection tax with no payoff
  - **Fix**: hand-code when the form count is small and layouts are custom (Form 1/2)
- ❌ Generic renderer can't express a needed custom interaction → forcing it via hacks
  - **Fix**: allow per-field custom widget escape hatches in the registry
