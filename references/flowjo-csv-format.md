# FlowJo wide-export CSV format

FlowJo's table editor exports one row per sample (well) and one column per
statistic-per-gate. This file documents the conventions so adapting the template to a new
export is mechanical. All regexes below are the ones already validated on real exports.

## Index columns (always present)

| Column | Meaning | Handling |
|---|---|---|
| `Sample:` | FCS filename, e.g. `[ A1 Well_001.fcs ]` | Drop it — the well/group columns are the reliable keys. |
| `GROUPNAME` | Plate or group the sample belongs to | Rename to `plate`. |
| `$WELLID` | Well position, e.g. `A1`, `H9` | Rename to `well`. Backtick-escape the `$` in R: `` `$WELLID` ``. |

Join every reshaped measurement table to `metadata.csv` on `plate` + `well`.

## Measurement column grammar

Two families, distinguished by the text after the `|`:

- **MFI / intensity:** `<gate/path> | <stat> (<marker>)`
  - `<stat>` is one of `Median`, `Geometric Mean`, `Mean`.
  - Example: `Lymphocytes/Single Cells | Median (Comp-FITC-A :: CD4)`
  - Example: `Lymphocytes/Single Cells | Geometric Mean (Comp-Alexa Fluor 647-A :: null)`
- **Frequency:** `<gate/path>/<population> | Freq. of Parent`
  - Example: `Lymphocytes/Single Cells/pSTAT1+ | Freq. of Parent`
  - Example: `Lymphocytes/Single Cells/Live/CD4+/IFNg+ | Freq. of Parent`

## Parsing regexes

Extract the marker/population name from the header so you can pivot to long format and
`factor()` it. Use `stringr::str_extract`:

```r
# MFI marker inside Median( ... )
str_extract(col, "(?<=Median \\()[^)]+")
# MFI marker inside Geometric Mean( ... )
str_extract(col, "(?<=Geometric Mean \\()[^)]+")

# Frequency population — last path component before " | Freq"
str_trim(str_extract(col, "(?<=/)([^/|]+)(?=\\s*\\| Freq)"))

# Nested sub-population under a named parent gate (e.g. under CD4+)
str_trim(str_extract(col, "(?<=CD4\\+/)([^/|]+)(?=\\s*\\| Freq)"))
```

When a single export mixes `Median` and `Geometric Mean` columns for different channels
(common — BV421 reported as Median, AF647 as Geometric Mean), select the value per row with
`case_when()` on a metadata column (e.g. `fluorophore`) rather than trying one regex for all.

## Capturing the gating hierarchy (parent / grandparent)

Keep the full gate path and its ancestry alongside each marker so the processed CSV can later
be plotted against gating context (e.g. comparing a population to its `Freq. of Parent`
denominator). The text before `" | "` is the slash-delimited gate path; index it from the leaf
end:

```r
extract_gate <- function(col) str_trim(str_extract(col, "^[^|]+"))
gate_level   <- function(path, n) str_split_i(path, fixed("/"), -n)   # n = 1 is the leaf
```

The leaf differs by family, so the parent sits at a different depth:

- **Frequency** (`Lymphocytes/Single Cells/pSTAT1+ | Freq. of Parent`) — the leaf *is* the
  gated population. `population = gate_level(path, 1)` (`pSTAT1+`),
  `parent = gate_level(path, 2)` (`Single Cells`, the freq denominator),
  `grandparent = gate_level(path, 3)` (`Lymphocytes`).
- **MFI** (`Lymphocytes/Single Cells | Median (mBaoJin)`) — the marker is in the parens and the
  path is all ancestor gates. `parent = gate_level(path, 1)` (`Single Cells`, the gate the stat
  is measured on), `grandparent = gate_level(path, 2)` (`Lymphocytes`).

`str_split_i()` (stringr ≥ 1.5) returns `NA` past the root, so shallow gates are safe. Keep
`gate_path` / `parent` / `grandparent` in the final `select()` so they survive into the
processed CSV.

## The `"n/a"` gotcha

Channels that weren't measured for a given group hold the **literal string `"n/a"`**, not an
empty cell. `as.numeric("n/a")` is `NA` with a warning, but more importantly the column
imports as character and silently poisons arithmetic. Always clean first:

```r
parse_num <- function(x) as.numeric(na_if(as.character(x), "n/a"))
```

Apply `parse_num` to every measurement value before any numeric operation.

## One-analyte-per-group layout

Some panels measure a different analyte per `GROUPNAME` (e.g. one pSTAT per plate). The wide
CSV then has the measured analyte populated and the other analyte columns blank/`NA` for that
row. To keep only the real measurement:

1. Pivot all candidate analyte columns to long (`pivot_longer`).
2. Extract the analyte name from the column header (regex above).
3. `inner_join` to metadata on `plate`, `well`, **and** the metadata column that names the
   per-group analyte (e.g. `pSTAT`). This keeps exactly the row whose extracted analyte
   matches what that group was supposed to measure, dropping the blanks.

```r
df <- raw %>%
  rename(plate = GROUPNAME, well = `$WELLID`) %>%
  select(-`Sample:`) %>%
  pivot_longer(-c(plate, well), names_to = "col_name", values_to = "value",
               values_transform = list(value = as.character)) %>%
  mutate(value = parse_num(value),
         analyte_col = str_extract(col_name, "pSTAT[0-9]+")) %>%
  filter(!is.na(analyte_col)) %>%
  inner_join(metadata %>% mutate(analyte_col = as.character(pSTAT)),
             by = c("plate", "well", "analyte_col"))
```

## `metadata.csv` convention

Columns: `GROUPNAME`, `$WELLID`, plus arbitrary experimental factor columns (condition,
treatment, replicate, cytokine, source, genotype, timepoint, and — for one-analyte-per-group
layouts — a column naming the analyte and/or its detection channel). Rename
`GROUPNAME → plate` and `$WELLID → well`, cast factors with explicit `levels =` so x-axis and
facet ordering is intentional, and cast `replicate` to integer.
