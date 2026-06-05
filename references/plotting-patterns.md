# Plotting & report patterns

These are the tidyplots idioms and the report shell shared across flow analyses. The point of
keeping them here is consistency: every report reads the same way, so only the *comparison*
changes between experiments.

## Setup

Load `tidyverse` and `tidyplots`; use tidyplots' **default theme**. Do not source
`plotting_fxns.R` and do not call `set_theme(theme_Publication())` — that theme has been
retired. If a house style is wanted later it can be applied once with `set_theme(...)` in the
setup chunk, but absent a request, leave the default.

```r
library(tidyverse)
library(tidyplots)
library(knitr)
library(kableExtra)
```

## Standard bar chain (raw values)

`x` = the x-axis factor, `color` = the comparison factor, faceting (`split_plot`) = the
marker/analyte. One panel per marker.

```r
df %>%
  tidyplot(x = <x_var>, y = <value>, color = <color_var>) %>%
  add_mean_bar(alpha = 0.7) %>%
  add_sd_errorbar() %>%
  add_data_points() %>%
  adjust_colors(new_colors = COLORS) %>%
  adjust_title("<Metric> — Raw") %>%
  adjust_x_axis_title("<x label>") %>%
  adjust_y_axis_title("<value> ± SD") %>%
  split_plot(by = <marker>, ncol = N) %>%
  adjust_size(width = 80, height = 40) %>%
  save_and_show("<metric>_raw")
```

`adjust_size` units are millimetres per panel; pair with chunk-header `fig.width`/`fig.height`
(inches) for the HTML layout — roughly `fig.width ≈ 4 × ncol`, `fig.height ≈ 4 × nrow`.

## Exporting every plot to PDF

All plots are written to a `plots/` directory as PDFs in addition to appearing in the HTML.
This is deliberately not a knitr device trick (`dev = "pdf"` / `fig.path`), because those
size the PDF from the chunk's `fig.width`/`fig.height` and ignore the physical size set by
`adjust_size()` — which is the whole reason the chains end with `adjust_size()`. Instead,
define one helper in the setup chunk and end every plot chain with it:

```r
PLOT_DIR <- "plots"
dir.create(PLOT_DIR, showWarnings = FALSE, recursive = TRUE)

save_and_show <- function(p, name) {
  tidyplots::save_plot(p, file.path(PLOT_DIR, paste0(name, ".pdf")))  # exact adjust_size dims
  p                                                                    # return so knitr embeds it
}
```

`tidyplots::save_plot()` honours the plot's stored dimensions (and handles `split_plot()`
patchworks), so the PDF matches the on-screen figure. Because the helper returns `p`, the
plot is still the chunk's value and knitr renders it into the report — one pipe step gives
you both outputs. Pass a short, unique, filename-safe `name` per plot (e.g.
`"pstat_mfi_raw"`, `"pstat_mfi_fold_change"`); these become the PDF filenames. `dir.create`
runs against the knit working directory (the `root.dir` you set), so `plots/` lands in the
project root.

## Fold-change normalization

Fold-change is computed *per replicate* against the control level, holding every factor that
is **not** the normalization axis constant. The normalization axis is whatever the control
level lives on (usually the x-axis factor, e.g. cytokine `none` or treatment `DMSO`).

```r
fc_keys <- c("<marker>", "<color_var>", "replicate")   # the non-normalized factors

baseline <- df %>%
  filter(<x_var> == CTRL_LEVEL) %>%
  select(all_of(fc_keys), baseline = <value>)

df <- df %>%
  left_join(baseline, by = fc_keys) %>%
  mutate(fold_change = <value> / baseline,
         log2FC      = log2(fold_change)) %>%
  select(-baseline)
```

Plotting fold-change reuses the standard chain with two changes:

- Reference line: `add_reference_lines(y = 1)` for raw fold-change, `add_reference_lines(y = 0)`
  for log2FC.
- Drop the control level from the x-axis for the FC panel (`filter(<x_var> != CTRL_LEVEL)`) —
  its fold-change is trivially 1 and just compresses the scale.

If the control level is on the *color* axis instead (e.g. comparing drugs to DMSO across
conditions), swap which factor is filtered and keyed accordingly — `fc_keys` is always
"everything except the normalization axis".

## Geometry variants

For distributions rather than group means, swap the bar layer:

```r
%>% add_violin() %>% add_data_points()      # or
%>% add_boxplot() %>% add_data_points()
```

Frequencies and raw MFI are often clearer as bars; per-cell distributions (when you have many
events per well) read better as violins. Default to bars unless the user wants spread shown.

## Colors

Provide an explicit named vector for the comparison levels you care about, and auto-fill any
remaining level from a palette so the script doesn't break when a new group appears:

```r
COLORS <- c(VLP = "#1b9e77", recomb = "#d95f02")   # example
new_lv <- setdiff(levels(df[[color_var]]), names(COLORS))
if (length(new_lv)) {
  auto <- setdiff(RColorBrewer::brewer.pal(8, "Set2"), unname(COLORS))
  COLORS <- c(COLORS, setNames(auto[seq_along(new_lv)], new_lv))
}
```

## Report shell

`html_document` output with:

```yaml
output:
  html_document:
    toc: true
    toc_float: true
    toc_depth: 4
    theme: flatly
    highlight: default
    code_folding: hide
```

Section layout (numbered, `#`-level):

1. **Overview** — one paragraph: what's compared, which markers, how fold-change is defined.
2. **Setup** — libraries + the `USER PARAMETERS` block.
3. **Data Loading & Wrangling** — load, derive constants from the data, clean metadata,
   reshape each measurement family to long, join, compute fold-change, summarise. Print small
   `kbl()` row-count tables so the reader can sanity-check the reshape.
4. *(optional)* **QC** — e.g. % viable, with a cutoff and a filter step, if the panel has it.
5. **One section per measurement family** — use `{.tabset}` to hold Raw / Fold-change / log2FC
   subtabs.
6. **Session Info** — `sessionInfo()`.

Derive marker lists and factor levels from the data where practical (e.g. detect markers from
column headers) so the report adapts when the panel changes, while still letting the user pin
an explicit subset/order through the parameter block.
