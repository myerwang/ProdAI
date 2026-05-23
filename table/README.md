---
topic: table
applies_to: [frontend, backend, sql, mobile]
data_scale: [small, medium, large, xlarge]
decision: choose by (data_scale, mutability, ordering_needs, edit_mode)
status: stable
last_reviewed: 2026-05-19
---

# Table Patterns — Index

Eight production-grade table forms. Each form lives in its own subfolder with
pseudocode, libraries, and pitfalls.

---

## Decision Tree (read first)

```
data scale?
├─ < 1k                                 → 01_offset_table
├─ 1k - 100k
│   ├─ append-only & time-sorted        → 02_cursor_pagination
│   ├─ mobile UI                         → 03_infinite_scroll
│   ├─ Excel-like edit                   → 05_editable_data_grid
│   ├─ nested data                       → 06_tree_table
│   └─ mutating CRUD, jump pages         → 01_offset_table
├─ 100k - 1M
│   ├─ many rows visible at once         → 04_virtual_scroll + (01 or 02)
│   ├─ nested + huge                     → 06_tree_table + lazy children
│   └─ mostly read, any sort             → 04_virtual_scroll + 02
└─ > 1M                                  → 08_server_side_row_model

special needs?
├─ multi-dim analysis                    → 07_pivot_table
└─ extreme custom UI                     → headless framework (TanStack Table)
```

---

## Summary

| # | Form | Folder | Scale | Mutating | Sort | Best Lib |
|---|---|---|---|---|---|---|
| 1 | Offset Table | [01_offset_table/](./01_offset_table/) | < 100k | ✓ | any | Ant / MUI / shadcn |
| 2 | Cursor Table | [02_cursor_pagination/](./02_cursor_pagination/) | any | △ | fixed | TanStack (self-wire) |
| 3 | Infinite Scroll | [03_infinite_scroll/](./03_infinite_scroll/) | any | △ | fixed | TanStack Virtual / FlatList |
| 4 | Virtual Scroll | [04_virtual_scroll/](./04_virtual_scroll/) | 10k–1M | ✓ | any | react-window / virtua |
| 5 | Editable Data Grid | [05_editable_data_grid/](./05_editable_data_grid/) | < 100k edit | ✓✓ | any | Handsontable / Glide / AG Grid |
| 6 | Tree Table | [06_tree_table/](./06_tree_table/) | varies | ✓ | varies | AG Grid / Ant Design |
| 7 | Pivot Table | [07_pivot_table/](./07_pivot_table/) | server-aggregated | — | — | AG Grid Enterprise |
| 8 | Server-Side Row Model | [08_server_side_row_model/](./08_server_side_row_model/) | > 1M | ✓ | any | AG Grid Enterprise |

---

## Common Combinations

- **Mobile log viewer**: 03 (infinite scroll) + 04 (virtual scroll)
- **Admin transaction log**: 02 (cursor) + 04 (virtual when page large)
- **Huge org chart**: 06 (tree) + 04 (virtual) + lazy children
- **BI dashboard**: 07 (pivot) for summary + 01 (offset) for drill-down detail
- **Financial trading blotter**: 08 (server-side) is the only feasible answer

---

## References convention

Per `../AGENTS.md` §3.6, **every form's README must cite ≥1 GitHub OSS repo with
> 5,000★** in a `## References` section, with the star count **measured** (GitHub API)
and a `verified <date>` stamp. No armchair invention — research is grounded in real
high-star implementations. Stars drift; a reference dropping below 5,000★ is swapped at
the next review. The "Best Lib" column above is a quick pointer; the per-form `## References`
sections hold the verified detail.

## How to extend

When a 3rd document about a sub-pattern of one form arrives, create deeper folders:

```
02_cursor_pagination/
├── README.md          ← form overview (current)
├── sql_keyset.md      ← when 2nd doc arrives, put at top level
└── advanced/          ← when 3rd related arrives, create folder
    ├── multi_sort_cursor.md
    └── ...
```

See `../CONTRIBUTING.md` §3 for the on-demand folder rule.

---

## History

- **2026-05-19**: Initial taxonomy. 8 forms indexed, each in its own folder.
- **2026-05-23**: Added a verified `## References` section to all 8 forms (≥1 OSS > 5,000★,
  stars measured via GitHub API). Dropped/relabeled sub-5k claims (SlickGrid ~2k,
  virtua ~3.6k, PivotTable.js ~4.4k); fixed `react-data-grid` URL to `Comcast/`.
