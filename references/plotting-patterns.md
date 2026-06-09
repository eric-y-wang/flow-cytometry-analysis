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
  split_plot(by = <marker>, ncol = NCOL) %>%
  adjust_size(width = PANEL_W, height = PANEL_H)
save_plot(p, file.path(PLOT_DIR, paste0("<metric>_raw__", EXPERIMENT, ".pdf")), view_plot = FALSE)
p
```

(assign the chain to `p` first — see "Exporting every plot to PDF" below.)

## Figure sizing — derive `fig.*` from the panel size

`adjust_size()` units are **millimetres per panel** and they size the *PDF*. The chunk-header
`fig.width`/`fig.height` are **inches** and size the *HTML* figure. These are two separate
controls and will silently drift if you hardcode them independently (e.g. the same plot ending
up taller in one tab than another). Instead, make the panel size the single source of truth and
derive the chunk dims from it. In the setup chunk:

```r
PANEL_W <- 60    # per-panel width  (mm), fed to adjust_size(); bump if x-axis bars look cramped
PANEL_H <- 40    # per-panel height (mm)
NCOL    <- N     # facet columns; rows are derived from the panel count

fig_dim <- function(n_panels, ncol = NCOL, w_mm = PANEL_W, h_mm = PANEL_H,
                    legend_in = 1.3, title_in = 0.6,
                    width_scale = 1.25, height_scale = 1.25) {
  nrow <- ceiling(n_panels / ncol)
  # mm -> inches (/25.4); pad width for the color legend and height for the title.
  # width_scale / height_scale give the chunk device extra room so the HTML figure isn't
  # squished relative to the plot's true size (panels + legend + axis titles).
  c(w = (ncol * w_mm / 25.4 + legend_in) * width_scale,
    h = (nrow * h_mm / 25.4 + title_in) * height_scale)
}
```

`PANEL_W`/`PANEL_H` control the panel content density (raise if many x-axis categories crowd
the bars); `width_scale`/`height_scale` control how much padding the chunk device gets around
the rendered plot (raise if the HTML figure looks squished or the legend/axis is clipped).
They are independent knobs — the panel dims change the PDF too, the scales only affect the HTML
`fig.width`/`fig.height`.

Compute the dims once per measurement family after the reshape (`DIM <- fig_dim(n_distinct(df$marker))`)
and reference them in each plot chunk's header — `fig.width` and `fig.height` accept R
expressions, so the header reads `{r raw, fig.width=DIM["w"], fig.height=DIM["h"]}`. Use the
same `NCOL` in `split_plot(ncol = NCOL)` and `PANEL_W`/`PANEL_H` in `adjust_size()` so the PDF
and the embedded HTML figure always rescale together when you change a panel size, the column
count, or the number of markers.

## Exporting every plot to PDF

All plots are written to a `plots/` directory as PDFs in addition to appearing in the HTML.
This is deliberately not a knitr device trick (`dev = "pdf"` / `fig.path`), because those
size the PDF from the chunk's `fig.width`/`fig.height` and ignore the physical size set by
`adjust_size()` — which is the whole reason the chains end with `adjust_size()`. Instead,
create `PLOT_DIR` once in the setup chunk:

```r
PLOT_DIR <- "plots"
dir.create(PLOT_DIR, showWarnings = FALSE, recursive = TRUE)
```

Then in each plot chunk, assign the finished chain to `p`, call `save_plot()` on it, and put
`p` on its own line so knitr embeds it:

```r
p <- df %>% tidyplot(...) %>% ... %>% adjust_size(width = PANEL_W, height = PANEL_H)
save_plot(p, file.path(PLOT_DIR, paste0("<name>__", EXPERIMENT, ".pdf")), view_plot = FALSE)
p
```

`view_plot = FALSE` is essential: `save_plot()` defaults to `TRUE` and its final
`print(input)` draws the plot to the active device, which knitr captures — then the returned
`p` is auto-printed too, embedding every figure in the HTML **twice**.

`tidyplots::save_plot()` honours the plot's stored dimensions (and handles `split_plot()`
patchworks), so the PDF matches the on-screen figure. The trailing bare `p` is the chunk's
value, so knitr renders it into the report — saving and showing in two lines. Pass a short,
unique, filename-safe `name` per plot (e.g. `"pstat_mfi_raw"`, `"pstat_mfi_fold_change"`) and
append the `EXPERIMENT` slug so analyses never overwrite each other; these become the PDF
filenames. `dir.create` runs against the knit working directory (the `root.dir` you set), so
`plots/` lands in the project root.

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
