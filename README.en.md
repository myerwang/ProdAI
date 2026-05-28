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

## Nature (key positioning)

- **NOT a skill library** — not a Claude Code skill / agent / callable tool collection; provides no execution capability
- **Non-invasive** — does not actively inject behavior into AI; enforces no workflow
- **Passive reference only** — AI consults this **autonomously** when uncertain or needing standardization during co-development; the author does not mandate consultation
- **Continuously evolves with tech** — not an unchanging bible; outdated approaches are marked `status: deprecated` or replaced

## Folder structure (grows on demand)

```
ProdAI/
├── README.md / README.ja.md / README.en.md   # Tri-lingual intro (always in sync)
├── AGENTS.md                                  # AI behavior rules (must read)
├── CONTRIBUTING.md                            # Acceptance standards
├── table/                                     # Table design patterns (8 forms)
│   ├── README.md                              # Index + decision tree
│   ├── 01_offset_table/                       # Standard offset pagination
│   ├── 02_cursor_pagination/                  # Cursor / keyset pagination
│   ├── 03_infinite_scroll/                    # Infinite scroll
│   ├── 04_virtual_scroll/                     # Virtual scroll
│   ├── 05_editable_data_grid/                 # Excel-like data grid
│   ├── 06_tree_table/                         # Tree table (nested)
│   ├── 07_pivot_table/                        # Pivot table
│   └── 08_server_side_row_model/              # Server-side row model (>1M)
├── auth/                                      # Auth patterns (6 forms)
│   ├── README.md                              # Index + decision tree
│   ├── 01_session_cookie.md                   # Server session + cookie (stateful)
│   ├── 02_jwt.md                              # JWT stateless token
│   ├── 03_oauth2_oidc.md                      # OAuth2 / OIDC delegated / federated
│   ├── 04_api_key.md                          # API Key / PAT (machine-to-machine)
│   ├── 05_webauthn_passkey.md                 # WebAuthn / Passkeys passwordless
│   └── 06_refresh_token_rotation.md           # Refresh token rotation / session lifecycle
├── form/                                      # Form patterns (9 forms)
│   ├── README.md                              # Index + decision tree + cross-cutting layer
│   ├── 01_single_form.md                      # Single-page form (controlled/uncontrolled + schema)
│   ├── 02_multistep_wizard.md                 # Multi-step wizard / stepper
│   ├── 03_dynamic_field_array.md              # Dynamic field array (add/remove rows)
│   ├── 04_conditional_survey_form.md          # Conditional / survey (skip logic)
│   ├── 05_schema_driven_form.md               # Schema-driven / server-driven form
│   ├── 06_inline_edit.md                      # Inline edit (capability)
│   ├── 07_autosave_draft.md                   # Autosave / draft (capability)
│   ├── 08_search_filter_form.md               # Search / filter form
│   └── 09_file_upload_form.md                 # File upload field (→ pointer to upload/)
└── upload/                                     # Upload patterns (8 forms)
    ├── README.md                              # Index + decision tree + summary
    ├── 01_server_proxied_multipart.md         # Server-proxied multipart (≤5MB / must traverse app)
    ├── 02_presigned_direct_put.md             # Presigned URL direct PUT (5–100MB)
    ├── 03_multipart_parallel_parts.md         # Multipart parallel parts (>100MB / abortable)
    ├── 04_resumable_tus.md                    # tus resumable protocol (flaky / cross-session)
    ├── 05_client_preprocessing.md             # Client preprocessing (resize / HEIC / EXIF / hash)
    ├── 06_background_offline_queue.md         # Background offline queue (SW + IndexedDB / PWA)
    ├── 07_streaming_server_ingestion.md       # Streaming server ingestion (proxy + memory-bound)
    └── 08_post_upload_pipeline.md             # Post-upload pipeline (thumbnails / scan / transcode)
```

Folders are **created on demand**: a scattered one-off starts at root, and a folder forms
only when a third related document arrives. No pre-created empty folders.
**Exception**: a topic recognized upfront as a mature multi-form taxonomy (`table/`, `auth/`, …)
gets its folder immediately and collects the popular production-grade forms in one pass,
the `table/` way (see `AGENTS.md` §2.3).

## Current content

### [`table/`](./table/) — Table design patterns

Eight production-grade table forms: Offset / Cursor / Infinite Scroll / Virtual Scroll /
Editable Data Grid / Tree / Pivot / Server-Side Row Model. Each form includes pseudocode,
library recommendations, and pitfalls.

Start from [table/README.md](./table/README.md)'s decision tree.

### [`auth/`](./auth/) — Auth patterns

Six production-grade authentication forms: Session+Cookie / JWT / OAuth2-OIDC / API Key /
WebAuthn-Passkeys / Refresh Token Rotation. Each form includes pseudocode, pitfalls, and
measured high-star references.

Start from [auth/README.md](./auth/README.md)'s decision tree (begin with "stateful vs stateless").

### [`form/`](./form/) — Form patterns

Nine production-grade form forms: Single / Multi-step Wizard / Dynamic Field Array /
Conditional-Survey / Schema-driven / Inline Edit / Autosave-Draft / Search-Filter /
File Upload. Primary axis is UX/architecture shape; state management and validation are
covered as a cross-cutting layer inside each form. Each includes pseudocode, pitfalls, and
measured high-star references.

Start from [form/README.md](./form/README.md)'s decision tree (begin with "is the field set known at build time?").

### [`upload/`](./upload/) — Upload patterns

Eight production-grade upload forms: Server-Proxied Multipart / Presigned Direct PUT /
Multipart Parallel Parts / Resumable tus / Client Preprocessing / Background Offline Queue /
Streaming Server Ingestion / Post-Upload Pipeline. Covers images, files, video, and binary
blobs across four axes: frontend / backend / client preprocessing / server-side derivation.
`form/09_file_upload_form.md` is reduced to a pointer for "use as a form field"; the actual
transfer mechanics live in this directory. Each form includes pseudocode, pitfalls, and
measured high-star references.

Start from [upload/README.md](./upload/README.md)'s decision tree (file size first, then layer in orthogonal concerns).

## Must-reads for AI collaborators

1. **`AGENTS.md`** — Behavior rules (forbidden content / triggers / write flow /
   pseudocode format)
2. **`CONTRIBUTING.md`** — Acceptance standards (what kind of content can enter ProdAI)

## Acceptance standards (summary)

- ✅ Abstracted generic patterns, design decision trees, industry research, lessons
- ✅ **Pseudocode** examples (AI-readable, convertible to any language)
- ✅ Clear "when to use / when NOT to use" guidance
- ✅ Must include a "pitfalls / anti-patterns" section
- ✅ **Each form must cite ≥1 high-star (>5,000★) GitHub reference — no armchair invention** (measured stars + verified date)
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
- 2026-05-23 — New standard: each form must cite ≥1 high-star (>5,000★) GitHub reference, no armchair invention; backfilled References into all 8 table forms
- 2026-05-23 — Added the `auth/` topic (6 forms collected upfront, with measured high-star references); established the "recognized multi-form taxonomy → create folder upfront" procedure (AGENTS.md §2.3)
- 2026-05-28 — Added the `form/` topic (9 forms collected upfront: Single / Wizard / Field Array / Conditional-Survey / Schema-driven / Inline Edit / Autosave / Search-Filter / File Upload; state management & validation as a cross-cutting layer); references measured 2026-05-28
- 2026-05-28 — Added the `upload/` topic (8 forms collected upfront: Server-Proxied Multipart / Presigned Direct PUT / Multipart Parallel Parts / Resumable tus / Client Preprocessing / Background Offline Queue / Streaming Server Ingestion / Post-Upload Pipeline); `form/09_file_upload_form.md` shrunk to an `upload/` pointer; references measured 2026-05-28
