# ProdAI

> 🇨🇳 [中文](./README.md) | 🇯🇵 [日本語](./README.ja.md) | 🇬🇧 [English](./README.en.md)

**Production-grade AI co-development knowledge base** — Let AI reference this repo to
deliver standardized, reliable, production-grade solutions even to non-technical users.

## What it is

ProdAI is not a tutorial or an awesome list. It is a **production-grade solutions
handbook for AI to read**:

- When AI co-development hits a "this needs to be standardized" or "this should be
  abstracted" moment, **AI proactively consults ProdAI** → gets a battle-tested solution
- Content is in **pseudocode** — AI translates into any language (React / Vue / Rust /
  Python / SQL ...) on demand
- Completely independent from any specific business project — generic knowledge,
  reusable across projects

## What it is NOT

- ❌ A backup of business project code
- ❌ Tutorial / course material
- ❌ Personal blog / notebook
- ❌ Awesome links list

## Folder structure (grows on demand)

```
ProdAI/
├── README.md / README.ja.md / README.en.md   # Tri-lingual intro (always in sync)
├── AGENTS.md                                  # AI behavior rules (must read)
├── CONTRIBUTING.md                            # Acceptance standards
└── (topic folders grow as needed in practice)
```

Folders are **created on demand**: only when a third related document arrives.
No pre-created empty folders.

## Must-reads for AI collaborators

1. **`AGENTS.md`** — Behavior rules (forbidden content / triggers / write flow /
   pseudocode format)
2. **`CONTRIBUTING.md`** — Acceptance standards (what kind of content can enter ProdAI)

## Acceptance standards (summary)

- ✅ Abstracted generic patterns, design decision trees, industry research, lessons
- ✅ **Pseudocode** examples (AI-readable, convertible to any language)
- ✅ Clear "when to use / when NOT to use" guidance
- ✅ Must include a "pitfalls / anti-patterns" section
- ❌ Anything business-project-related: project names, table names, API paths, URLs,
   credentials, personal information
- ❌ Specific tech stack syntax (React / Rust concrete syntax)

See `CONTRIBUTING.md` for details.

## Contribution flow

1. AI identifies "this is a ProdAI candidate" during co-development
2. Abstract it (strip business names, convert to pseudocode)
3. Get user confirmation
4. Write to appropriate location (create folder as needed)
5. Sync all three READMEs
6. Commit + push (**push must announce to user**)

## License

TBD (decide before going public)

## Version

- 2026-05-19 — Initial skeleton
