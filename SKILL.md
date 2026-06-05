---
name: flow-cytometry-analysis
description: >-
  Build a reproducible R Markdown / HTML report from FlowJo-exported flow cytometry CSVs.
  Trigger whenever the user wants to analyze, summarize, or plot flow cytometry data —
  especially wide CSV exports with columns like `Sample:`, `GROUPNAME`, `$WELLID`, and
  gating-path columns such as `... | Median (<marker>)`, `... | Geometric Mean (...)`, or
  `.../<population>+ | Freq. of Parent`. Use it for MFI / marker-frequency / pSTAT
  activation analysis, comparing treatments, conditions, cytokine sources, or timepoints,
  and for fold-change vs an untreated/control group. Reach for this skill even when the user
  just points at a `tables/` folder of FlowJo exports plus a `metadata.csv` and asks for
  plots, without saying the word "skill".
---

# Flow Cytometry Analysis

Generate an R Markdown report that ingests FlowJo wide-format CSV exports, joins them to an
experimental `metadata.csv`, reshapes to long (tidy) format, and renders tidyplots figures
(raw values, fold-change, and/or log2 fold-change vs a control group). The FlowJo ingestion
is a fixed, well-understood backbone; the *comparison* (what's on the x-axis, what's colored,
what's faceted, what the control group is, which plot geometry) changes from experiment to
experiment and is what you configure each run.

## When you load this skill

Two reference files hold the heavy detail — read them when you reach the relevant step;
don't try to hold it all at once:

- **`references/flowjo-csv-format.md`** — exact column conventions, the parsing regexes for
  pulling marker/population names out of gating-path headers, the `"n/a"` gotcha, and how to
  handle one-analyte-per-group layouts. Read this at step 3.
- **`references/plotting-patterns.md`** — the tidyplots plotting chains, the fold-change
  normalization join, geometry variants, and the output Rmd shell. Read this at steps 4–5.

The starting artifact is **`assets/flow_analysis_template.Rmd`** — a parameterized skeleton
you copy into the project and adapt. Most of the job is wiring real column names and the
chosen comparison into that template, not writing an Rmd from scratch.

## Workflow

### 1. Locate the inputs

Find the `metadata.csv` and the measurement CSV(s) — typically in a `tables/` subfolder of
the project. Read only the **headers** (first row, or first ~10 rows) of each file to see
the column families. Don't read whole measurement files into context; they can be large and
you only need the column names plus a few example rows to recognize the format.

### 2. Establish the experimental design with the user

These are the configurable axes. They determine the entire analysis, and the same FlowJo
export can be sliced many ways — so never guess them silently. Capture:

- **experiment name** — a short, filename-safe label for this analysis (e.g. `il2_dose_response`, `2026-06-05_pSTAT`). **Always ask the user for this** — do not guess it — even if they supplied every other parameter. It is appended to every output the skill produces (the `.Rmd`, the HTML report, every plot PDF, and the processed CSVs) so multiple analyses can coexist in one project without overwriting each other. Sanitize the user's answer to a filename-safe slug (lowercase, spaces/special chars → `_`) and store it as `EXPERIMENT`.
- **x-axis factor** — the experimental variable along the x-axis (e.g. cytokine, condition, timepoint).
- **comparison / color factor** — what distinguishes the colored bars/groups (e.g. treatment, cytokine source, genotype).
- **faceting variable** — usually the marker/analyte; each panel is one marker.
- **control level** — the baseline level used for fold-change (e.g. `none`, `DMSO`, `unstim`). Skip fold-change if there's no meaningful control.
- **value types to show** — raw, fold-change, and/or log2 fold-change.
- **plot geometry** — bar of group means (default) or violin/box for distributions.

**If the user supplied these (in their request or earlier in the conversation), restate your
reading of all of them and proceed.** Otherwise — for any the user did **not**
specify — you must ask rather than assume; a wrong x-axis or control level silently produces
a plausible-looking but wrong report. The **experiment name** is the one parameter you must
always ask for explicitly, since it cannot be inferred from the data.

Ask with the `AskUserQuestion` tool so the user can pick rather than type. Make answering
easy by pre-filling realistic options inferred from the data: read the `metadata.csv` column
names and their distinct values (from step 1) and offer those as the candidate factors and
levels. For example, if `metadata.csv` has a `treatment` column with values `DMSO`/`drugA`,
offer `treatment` as a candidate comparison factor and `DMSO` as a candidate control level.
Group the open questions into one `AskUserQuestion` call (it allows up to four) rather than
asking one at a time. Only the parameters the user left unspecified need to be asked (the
experiment name always being one of them); don't re-ask what they already gave you. Once
answered, restate the final parameter set — including the experiment name — and proceed to
step 3.

### 3. Map the measurement columns

Read `references/flowjo-csv-format.md`, then classify each measurement column: MFI
(`Median (...)` / `Geometric Mean (...)`) or frequency (`... | Freq. of Parent`), and which
marker/analyte each one carries. Note which metadata column names the per-group analyte when
a CSV measures one analyte per `GROUPNAME` (common for pSTAT panels) — you'll join on it to
keep only the real row per sample.

### 4. Copy and adapt the template

Copy `assets/flow_analysis_template.Rmd` to the project root, naming it with the experiment
slug: `flow_cytometry_analysis_<EXPERIMENT>.Rmd`. Then:

- Set `root.dir` in the setup chunk to the project directory.
- Set `EXPERIMENT` in the `USER PARAMETERS` block to the filename-safe experiment slug from
  step 2. It feeds `save_and_show()` (plot PDFs land in `plots/<name>__<EXPERIMENT>.pdf`),
  the processed-CSV filenames, and the report title.
- Fill the rest of the `USER PARAMETERS` block from step 2 (control level, x/color/facet vars,
  level orderings, colors, geometry).
- Wire the exact column names and parsing regexes from step 3 into the reshape chunks.
- Add one analysis section per measurement family (one for each CSV / metric type), keeping
  the numbered-section structure and the `{.tabset}` split for raw / fold-change / log2FC.

### 5. Apply the plotting conventions

Follow `references/plotting-patterns.md`. The standard chain is
`tidyplot(x, y, color) %>% add_mean_bar(alpha=0.7) %>% add_sd_errorbar() %>%
add_data_points() %>% adjust_colors(...) %>% split_plot(by=<marker>, ncol=N) %>%
adjust_size(...) %>% save_and_show("<name>")`. Use the fold-change normalization join
described there for FC/log2FC panels. Every plot chain ends with `save_and_show()` so it is
exported to `plots/` as a PDF (at the `adjust_size()` dimensions) *and* embedded in the HTML
— give each plot a short, unique name. Do **not** source `plotting_fxns.R` or set
`theme_Publication()` — use tidyplots' default theme.

### 6. Write outputs and tell the user how to knit

Emit long-format processed CSVs (`<measurements>_processed__<EXPERIMENT>.csv`) from a hidden
chunk so the wrangled data is inspectable. Knitting also writes every figure to a `plots/`
directory as a PDF via the `save_and_show()` helper (created automatically in the setup
chunk), with the experiment slug appended to each filename, so the user gets
publication-ready vector files alongside the HTML. Every output — the `.Rmd`, the rendered
HTML, the plot PDFs, and the processed CSVs — carries the experiment name, so reruns with a
different name never clobber a previous analysis. Then tell the user how to render, e.g.
`rmarkdown::render("flow_cytometry_analysis_<EXPERIMENT>.Rmd")`. If you can't run R in this
environment, say so and leave the knit to the user.
