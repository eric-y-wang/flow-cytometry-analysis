# flow-cytometry-analysis

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that turns
FlowJo-exported flow cytometry CSVs into a reproducible R Markdown / HTML report.

## Overall function

Given a folder of FlowJo wide-format CSV exports plus an experimental
`metadata.csv`, the skill builds a parameterized R Markdown report that:

1. **Ingests** the FlowJo wide exports (one row per sample/well, one column per
   statistic-per-gate) and joins them to the experimental metadata on
   `plate` + `well`.
2. **Reshapes** the wide data to long/tidy format, parsing marker and population
   names out of the gating-path column headers and cleaning the literal `"n/a"`
   values FlowJo writes for unmeasured channels.
3. **Computes** per-replicate fold-change and log2 fold-change against a chosen
   control level.
4. **Renders** publication-quality [tidyplots](https://tidyplots.org/) figures
   (raw MFI / marker frequency, fold-change, log2FC), one panel per marker, and
   writes each plot to a `plots/` directory as a vector PDF *and* embeds it in
   the HTML report.

The FlowJo ingestion is a fixed, well-understood backbone. What changes per
experiment is the **comparison** — what's on the x-axis, what's colored, what's
faceted, and which group is the control — and that is what you configure each run.

## Outline of the skill

```
flow-cytometry-analysis/
├── SKILL.md                              # Entry point: trigger + 6-step workflow
├── assets/
│   └── flow_analysis_template.Rmd        # Parameterized Rmd skeleton to copy & adapt
└── references/
    ├── flowjo-csv-format.md              # Column conventions, parsing regexes, gotchas
    └── plotting-patterns.md              # tidyplots chains, fold-change join, report shell
```

The workflow defined in `SKILL.md` runs in six steps:

1. **Locate the inputs** — find `metadata.csv` and the measurement CSV(s)
   (typically in a `tables/` folder); read only the headers.
2. **Establish the experimental design** — confirm the six configurable axes with
   the user (x-axis factor, color/comparison factor, faceting marker, control
   level, value types to show, plot geometry).
3. **Map the measurement columns** — classify each as MFI (`Median (...)` /
   `Geometric Mean (...)`) or frequency (`... | Freq. of Parent`) and extract its
   marker/analyte name.
4. **Copy and adapt the template** — copy `flow_analysis_template.Rmd` into the
   project and wire in real column names, regexes, and the chosen comparison.
5. **Apply the plotting conventions** — use the standard tidyplots bar/violin
   chains and the fold-change normalization join from `plotting-patterns.md`.
6. **Write outputs and knit** — emit long-format `*_processed.csv` files,
   render to HTML, and export every figure to `plots/` as a PDF.

## Necessary input files

| File | Required | Description |
|---|---|---|
| `tables/metadata.csv` | **Yes** | Experimental design table. Must contain `GROUPNAME` (→ `plate`) and `$WELLID` (→ `well`) index columns plus arbitrary factor columns (condition, treatment, replicate, cytokine, source, genotype, timepoint, …). For one-analyte-per-group panels, include a column naming the per-group analyte and/or its detection channel. |
| `tables/<measurements>.csv` | **Yes** (one or more) | FlowJo wide-format table export(s). One row per sample/well with index columns `Sample:`, `GROUPNAME`, `$WELLID`, and measurement columns following the grammar `<gate/path> \| <stat> (<marker>)` for MFI or `<gate/path>/<population> \| Freq. of Parent` for frequencies. |

### Input format notes

- **Join keys:** every measurement table joins to `metadata.csv` on
  `plate` (`GROUPNAME`) + `well` (`$WELLID`).
- **`"n/a"` values:** unmeasured channels hold the literal string `"n/a"`, which
  must be coerced to `NA` before any arithmetic (the template's `parse_num()`
  handles this).
- **One-analyte-per-group layouts:** some panels measure a different analyte per
  group (e.g. one pSTAT per plate); the metadata column naming that analyte is
  used in an `inner_join` to keep only the real measurement per row.

See [`references/flowjo-csv-format.md`](references/flowjo-csv-format.md) for the
full column grammar, parsing regexes, and edge cases.

## Requirements

R with `tidyverse`, `tidyplots`, `knitr`, `kableExtra`, and `RColorBrewer`.
Render with:

```r
rmarkdown::render("flow_cytometry_analysis.Rmd")
```

## Outputs

- An HTML report (`flow_cytometry_analysis.html`) with a floating table of
  contents and Raw / Fold-change / log2FC tabs per measurement family.
- Vector PDF of every figure in `plots/`.
- Tidy long-format `tables/*_processed.csv` for downstream inspection.
