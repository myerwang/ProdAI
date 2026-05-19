# Contributing to ProdAI

How a new document gets accepted into ProdAI. Applies to both humans and AI collaborators.

For full AI behavior rules see `AGENTS.md`.

---

## 1. Acceptance Standards (Quality Bar)

A document is **eligible for ProdAI** only if it meets **all** of:

### 1.1 Abstraction

- ❌ No project names, product names, code names
- ❌ No business-domain vocabulary (industry-specific terms)
- ❌ No DB table names, API paths, enum values, internal component names
- ❌ No infrastructure identifiers (cloud project ID, service names, URLs, hostnames)
- ❌ No credentials, tokens, secrets (in any form, even dev env)
- ❌ No personal / customer / partner names

✅ Only generic patterns, principles, decision trees, industry knowledge.

### 1.2 Pseudocode (not specific syntax)

Code examples must be **pseudocode that AI can translate to any language**:

```
# OK ✅
function CursorList<T>(props):
  state pages: list of list of T
  effect on (deps):
    result = await fetcher({ cursor, limit })
    pages = [result.items]
```

```
// NG ❌ — React-specific syntax
const [pages, setPages] = useState<T[][]>([]);
useEffect(() => { ... }, [deps]);
```

```
// NG ❌ — Rust-specific syntax
async fn list_items(pool: &PgPool) -> Result<Vec<Item>> { ... }
```

### 1.3 Clear Decision

The first sections must answer:

- **When to use this?** (data scale, business pattern, constraints)
- **When NOT to use this?** (anti-cases)
- **The recommended choice** (1 sentence)

Then explain rationale, alternatives, tradeoffs.

### 1.4 Pitfalls Section

Every doc must contain:

```markdown
## Pitfalls / Anti-patterns

- ❌ Mistake 1: ...
  - Why it fails: ...
  - Fix: ...
- ❌ Mistake 2: ...
```

This is the **safety net for AI** to avoid misapplying the pattern.

### 1.5 Metadata Header

```yaml
---
topic: pagination
applies_to: [frontend, backend, sql]
data_scale: [medium, large, xlarge]
decision: when N > 100k → cursor; when frequent mutation → offset
status: stable           # draft | stable | deprecated
last_reviewed: 2026-05-19
---
```

---

## 2. Adding Process

### 2.1 Trigger (when to add)

- Pattern observed in 2+ projects
- Recurring decision point ("offset vs cursor", "REST vs GraphQL", etc.)
- Industry knowledge worth preserving (OSS comparison, benchmark)
- Lesson learned (anti-pattern that bit you)

### 2.2 Steps

1. **AI proposes** — "This looks like a ProdAI candidate (topic: X)"
2. **Strip business info** — Replace all business names with generic placeholders
3. **Convert to pseudocode** — No language-specific syntax
4. **User confirms** — Get explicit OK
5. **Write to appropriate location** (see §3 folder structure)
6. **Update tri-lingual READMEs** if structure changed (README.md / README.ja.md / README.en.md)
7. **Commit** — Automatic OK
8. **Push** — Must announce to user (auto-push is not allowed)

---

## 3. Folder Structure (grows on-demand)

Do NOT pre-create empty folders. Rule: **3rd related document triggers a folder**.

```
Phase A: 1st document — direct under root
ProdAI/
└── pagination_design.md

Phase B: 2nd related document — still under root
ProdAI/
├── pagination_design.md
└── cursor_explained.md

Phase C: 3rd related document — create folder
ProdAI/
└── pagination/
    ├── README.md
    ├── design.md
    ├── cursor.md
    └── offset.md
```

Folder names: topic-based, lowercase, snake_case if needed (`table`, `database`, `pagination`, `ai_collaboration`, `state_management`).

---

## 4. README Sync Checklist

When adding/removing/restructuring files at root or top-level folders, **all 3 READMEs must be updated in the same commit**:

- [ ] `README.md` (Chinese)
- [ ] `README.ja.md` (Japanese)
- [ ] `README.en.md` (English)

Content must match (not just translation — structural equivalence).

---

## 5. Push Rules

- `git commit` — AI can do automatically
- `git push origin main` — AI **must announce** ("pushing to main now") or let user lead
- `git push -f` / branch delete / tag — User confirmation **required**

ProdAI is a knowledge repo. Even though it's not a production system, casual leaks of business info would be irreversible. Hence the announce-required rule.

---

## 6. Violations

If business info leaks into ProdAI:

1. AI must immediately report to user
2. Delete the offending content
3. If already pushed: use `git filter-repo` or BFG to scrub history (with user's direction)
4. Add a "lesson learned" entry to this file or `AGENTS.md`

---

## 7. License Considerations

ProdAI is **currently private**. Before making public:

- Decide license (MIT / Apache 2.0 / CC BY-SA / etc.)
- Audit all content for any residual business info
- Decide attribution model

---

## History

- **2026-05-19** Initial version. Defined acceptance standards, adding process, folder structure rules.
