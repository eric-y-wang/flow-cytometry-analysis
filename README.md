# flow-cytometry-analysis

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that turns
FlowJo-exported flow cytometry CSVs into a reproducible R Markdown / HTML report.

## How to use

1. **Perform FlowJo analysis (gating, statistics, etc.)**
2. **Create folder named tables in your analysis directory**
3. **Export results tables to /tables:** Create a results table as you normally would to export data. Add the keywords $GROUPNAME and $WELLID to the results table. Save as a .csv file. Generally as best practice include the type of measurement in the file name (e.g. mfi.csv or freq.csv) 
4. **Create metadata table to /tables:** Once you have your results .csv files, copy and paste the GROUPNAME and WELLID columns into a new .csv. Fill in the metadata with appropriate information (treatment, cage, mouse, etc.). These variables will be ingested by Claude Code and used for plotting.
5. **Perform skill with Claude:** Point Claude Code to your analysis directory and ask it to perform analysis with this skill. You can give information ahead of table on what comparisons you want to make and how you want things plotted. Otherwise, it will ask you for guidance while running the skill.

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
   (raw MFI / marker frequency / absolute cell counts, fold-change, log2FC), one
   panel per marker, and writes each plot to a `plots/` directory as a vector PDF
   *and* embeds it in the HTML report.

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
   `Geometric Mean (...)`), frequency (`... | Freq. of Parent`), or count
   (`... | Count`) and extract its marker/population name. For counts, also
   confirm the volume-column mapping (see [Absolute cell counts](#absolute-cell-counts)).
4. **Copy and adapt the template** — copy `flow_analysis_template.Rmd` into the
   project and wire in real column names, regexes, and the chosen comparison.
5. **Apply the plotting conventions** — use the standard tidyplots bar/violin
   chains and the fold-change normalization join from `plotting-patterns.md`.
6. **Write outputs and knit** — emit long-format `*_processed.csv` files,
   render to HTML, and export every figure to `plots/` as a PDF.

## Necessary input files

| File | Required | Description |
|---|---|---|
| `tables/metadata.csv` | **Yes** | Experimental design table. Must contain `GROUPNAME` (→ `plate`) and `$WELLID` (→ `well`) index columns plus arbitrary factor columns (condition, treatment, replicate, cytokine, source, genotype, timepoint, …). For one-analyte-per-group panels, include a column naming the per-group analyte and/or its detection channel. For **absolute counts**, also include the three volume columns `flow_vol`, `stained_sample_vol`, `total_sample_vol` (see [Absolute cell counts](#absolute-cell-counts)). |
| `tables/<measurements>.csv` | **Yes** (one or more) | FlowJo wide-format table export(s). One row per sample/well with index columns `Sample:`, `GROUPNAME`, `$WELLID`, and measurement columns following the grammar `<gate/path> \| <stat> (<marker>)` for MFI, `<gate/path>/<population> \| Freq. of Parent` for frequencies, or `<gate/path>/<population> \| Count` for event counts. A **counts** export additionally carries a `$VOL` index column (the µL the cytometer acquired per well) used for the absolute-cell-number calculation — see [Absolute cell counts](#absolute-cell-counts). |

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

## Absolute cell counts

A FlowJo `| Count` column is the number of **events the cytometer recorded** in a
gate — not an absolute number of cells. It depends on how much volume the
instrument happened to aspirate and on how much the sample was diluted during
staining. The skill converts each raw count into the cell number in the whole
sample using two volume corrections:

```
abs_count = count
            × (flow_vol / $VOL)                        # acquisition-volume correction
            × (total_sample_vol / stained_sample_vol)  # staining-dilution correction
```

| Term | Source | Meaning |
|---|---|---|
| `count` | counts CSV | Raw events FlowJo counted in the gate. |
| `$VOL` | counts CSV (per well) | **Acquired volume** — the µL the cytometer actually ran. Varies well-to-well, since the instrument rarely runs exactly the nominal volume. |
| `flow_vol` | metadata.csv | **Volume resuspended for flow cytometry** — the volume the stained sample was resuspended in before running. |
| `stained_sample_vol` | metadata.csv | **Volume of sample used for staining** — the aliquot taken for staining. |
| `total_sample_vol` | metadata.csv | **Volume the entire sample is resuspended in** — the full sample the staining aliquot was drawn from. |

**What each factor does.** The first factor (`flow_vol / $VOL`) scales the events
seen in the acquired volume up to the full volume the sample was resuspended in
for flow — i.e. it corrects for the cytometer only sampling part of the tube. The
second factor (`total_sample_vol / stained_sample_vol`) scales from the aliquot
that was stained back up to the entire harvested sample. When all three metadata
volumes are equal (e.g. all `100`), the second factor is `1` and the calculation
reduces to a pure acquisition-volume correction.

**Volume labels are verified before use.** Because a mislabeled volume produces
absolute numbers that look plausible but are silently wrong, the skill restates
which column it intends to use for each of the four quantities and asks you to
confirm before computing anything. If the expected columns are missing or named
differently, it maps them by meaning and asks, never substituting a default for a
core factor — and if `flow_vol` or `$VOL` is absent entirely it will fall back to
plotting raw event counts (clearly labeled as events, not cells) rather than
guess.

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
