# dtc-category-scanner

A Claude Code skill for scanning a DTC storefront’s category structure, estimating assortment width, and writing structured results back to Lark/Feishu sheets.

## What it does

This skill helps analyze a DTC ecommerce site and produce a standardized output for brand research.

It is designed to:
- identify the site’s real top-level merchandising categories
- calculate the **Top 5 core categories** by SPU count
- compute **average price** and **median price** for each selected category
- estimate total sitewide product count as **assortment width**
- generate a short **brand positioning + category structure summary**
- optionally write results back to a Lark summary sheet and a per-brand detail sheet
- optionally generate and write concatenated example images for each category

## Typical inputs

The skill expects some or all of the following:
- brand domain
- starting URL (homepage, collection page, or best sellers page)
- Lark summary sheet URL
- target row number or brand identifier in the summary sheet
- Lark/Wiki detail sheet URL if per-brand detail output is needed
- whether example images are required
- the active currency shown on the site

## Typical outputs

The workflow produces four main outputs:
1. **Top 5 category metrics**
   - category name
   - SPU count
   - average price
   - median price
2. **Assortment width**
   - total sitewide product count
3. **Brand summary**
   - a short positioning + category structure summary
4. **Optional detail outputs**
   - per-category example products
   - per-brand detail sheet rows
   - concatenated category example images

The workflow also recommends saving the assortment-width calculation method, source, and limitations to a local notes file, typically under `scraping_notes/{domain}.md`.

## Dependencies

This repository contains the skill itself, but it assumes the following capabilities already exist in your Claude Code environment:
- `web-access` — for site exploration, dynamic page access, and structured web extraction
- `lark-shared` — for Lark CLI auth, permissions, and safety rules
- `lark-sheets` — for reading/writing spreadsheet content
- `lark-cli` — required binary for Lark operations

If you want to use the Lark writeback part, make sure those skills and tools are available first.

## Repository structure

```text
.
├── SKILL.md
└── references/
    ├── category-metric-rules.md
    ├── examples.md
    └── lark-write-sop.md
```

## How the workflow works

At a high level, the skill follows this sequence:
1. explore the site’s actual category structure
2. normalize comparable top-level categories
3. collect products and compute category metrics
4. estimate total sitewide product count
5. generate a short brand/category summary
6. read Lark headers before any write
7. lock exact top 3 products for each selected category
8. optionally write brand detail rows to a dedicated sheet
9. write summary fields, numbers, and optional images back to Lark
10. read back the written range to verify correctness

## Notes on image writing

The skill uses an inline SOP for image handling.

The default image flow is:
- lock the exact top 3 products for each category
- download their hero images locally
- concatenate them into one local image per category
- write text and numeric cells first
- then write images cell-by-cell using `lark-cli sheets +write-image`

It intentionally avoids treating image writing as a separate required skill.

## Intended use cases

This skill is useful when you want to:
- compare DTC brands by category structure
- estimate how wide or narrow a site’s assortment is
- standardize category research across multiple brands
- build a repeatable workflow for Lark-based brand research operations

## Limitations

This is not a generic crawler for every ecommerce task.

It is optimized for:
- category-structure analysis
- Top 5 category metrics
- assortment width estimation
- structured Lark writeback

If you need large-scale product extraction beyond this workflow, extend it rather than assuming that every scraping use case is already covered.

## Source of truth

The skill behavior is defined in:
- `SKILL.md`
- files under `references/`

Use those files as the canonical operational spec.
