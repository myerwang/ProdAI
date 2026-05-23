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

### 1.6 High-Star GitHub Reference (mandatory, ≥1 per form)

Every **form / pattern** must cite **at least one OSS repo with > 5,000 GitHub stars**.

- **No armchair invention (闭门造车).** Research and the early collection phase must be
  grounded in high-star OSS — verify "what the industry does" against real implementations
  before writing it down.
- Each doc must have a `## References` section, one entry per line:
  ```markdown
  ## References
  - [owner/repo](https://github.com/owner/repo) — ~98k★ (verified 2026-05-23): one-line use
  ```
- Star counts drift. **Measure via GitHub API / official page** before writing, and append
  `verified <date>`. **Fabricated URLs or guessed star counts are a §1-level violation.**
- A reference that drops below 5,000★ gets swapped for a higher-star alternative at the next review.
- If no OSS implementation above 5,000★ exists for a form, it does not meet the bar
  (not yet an "industry-standard production pattern") — skip it or consult the user.

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
4. **Attach measured high-star references** — Per §1.6, find a > 5,000★ OSS for each form,
   measure stars via GitHub API, write a `## References` section with `verified <date>`. No fabrication.
5. **User confirms** — Get explicit OK
6. **Write to appropriate location** (see §3 folder structure)
7. **Update tri-lingual READMEs** if structure changed (README.md / README.ja.md / README.en.md)
8. **Commit** — Automatic OK
9. **Push** — Must announce to user (auto-push is not allowed)

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

### 3.1 Recognized multi-form taxonomy → create the folder upfront, collect ALL forms (the `table/` way)

The on-demand rule above is for **scattered, one-off documents**. When a topic is recognized
from the start as a mature **multi-form production taxonomy**, do NOT dribble single files —
**create the topic folder immediately and comprehensively collect the popular production-grade
forms in one pass**, exactly like `table/` (8 forms). This is the §0 "do it right in one shot" mindset.

Trigger (all three must hold):
1. You can enumerate **≥3 production-grade forms** upfront
2. Each form has a real **OSS implementation with > 5,000★** (satisfies §1.6)
3. There is a **decision axis** between forms (a decision tree can be written)

What to produce (mirror `table/`):
- `<topic>/README.md` — index + decision tree + summary table (a "Best Lib" column per form)
- `<topic>/NN_<form>.md` — one file per form, each complete with metadata / when-to-use /
  pseudocode / pitfalls / `## References` (≥1 measured > 5,000★, verified date)

Example: JWT is not a topic — it is **one form of the `auth` topic**. So don't create a lone
`jwt.md`; create `auth/` and collect session-cookie / JWT / OAuth2-OIDC / API key / WebAuthn
(passkeys) etc. together.

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
- **2026-05-23** Added §1.6 (mandatory high-star GitHub reference, > 5,000★ per form, no armchair invention) and a measured-reference step in §2.2. Backfilled References into all 8 table forms.
- **2026-05-23** Added §3.1 (recognized multi-form taxonomy → create folder upfront and collect all forms, the `table/` way). E.g. JWT goes into a comprehensive `auth/` topic, not a lone file.
