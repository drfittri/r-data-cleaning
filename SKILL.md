---
name: r-data-cleaning
description: >
  R data cleaning skill for making Excel files analysis-ready. Use when the user
  wants to clean data.xlsx (or any .xlsx file) for R analysis, generate a cleaning
  script, standardize columns, recode factors, parse free-text fields into binary
  indicators, or prepare data for gtsummary / logistic regression workflows.
  Trigger when user says: "clean data", "prepare data for R", "make analysis-ready",
  "generate cleaning script", "recode variables", or references data.xlsx with R.
---

# R Data Cleaning Skill

Generate a self-contained `.R` script that cleans an Excel file for analysis.

## Step 1 – Inspect (single R call, compact output)

```bash
Rscript -e '
suppressPackageStartupMessages({ library(readxl); library(janitor); library(dplyr) })
f <- "data.xlsx"
sheets <- excel_sheets(f)
cat("file:", f, "\nsheets:", paste(sheets, collapse=" | "), "\n\n")
for (s in sheets) {
  d_raw <- read_excel(f, sheet = s)
  if (ncol(d_raw) == 0 || nrow(d_raw) == 0) { cat("sheet", s, ": EMPTY\n\n"); next }
  d <- d_raw |> remove_empty(c("rows","cols")) |> clean_names()
  cat("--- sheet:", s, " dim:", nrow(d), "x", ncol(d),
      " dupes:", sum(duplicated(d)), " empty_r/c removed:",
      nrow(d_raw)-nrow(d), "/", ncol(d_raw)-ncol(d), "---\n")
  for (col in names(d)) {
    v <- d[[col]]; nu <- length(unique(v[!is.na(v)])); nm <- sum(is.na(v))
    tp <- if (is.logical(v)) "lgl" else if (is.numeric(v)) "num" else if (inherits(v,"POSIXct")) "date" else "chr"
    cat(sprintf("  %-25s %-4s NA:%-4d uniq:%-4d", col, tp, nm, nu))
    if (tp %in% c("chr","lgl") || nu <= 15) {
      uv <- unique(v[!is.na(v)])
      uv <- if (length(uv)>10) c(head(uv,10),"...") else uv
      cat(" [", paste(uv, collapse=" | "), "]")
    }
    cat("\n")
  }
  cat("\n")
}'
```

One line per column: name, type (`chr`/`num`/`date`/`lgl`), NA count, unique
count, values for character/logical/low-cardinality. Also reports: sheet names,
empty rows/cols removed, duplicate row count. No row-level data enters context.

**If header is not row 1** (inspection shows garbage column names), re-run with
`skip = N` to find the real header row.

## Step 2 – Classify columns

From the inspection output, classify each column:

| Type | Signal | Action |
|---|---|---|
| categorical | chr, few unique values (e.g. Male/Female) | `case_when` + `factor()` |
| binary yes/no | chr, values YES/NO | `case_when` + `factor(c("No","Yes"))` |
| free-text multi-value | chr, comma-separated (e.g. "HPT, CAD") | normalize → `str_detect` → binary flags |
| numeric-as-string | chr, values look numeric + occasional text | `str_extract` → `as.numeric()` |
| censored numeric | chr, values like `"<0.01"`, `">500"`, `"TNTC"` | extract number + censoring flag |
| numeric code | num, but actually categories (1=Male, 2=Female) | `factor()` with explicit labels |
| logical | lgl (TRUE/FALSE) | convert to `factor(c("No","Yes"))` |
| date | date or chr with date-like values | `lubridate::dmy()` / `ymd()` |
| ID / code | chr, unique per row, may have leading zeros | keep as character, never `as.numeric()` |
| true numeric | num, continuous | keep as-is |
| derived | multiple source columns combine into one | `case_when` across columns |

## Step 3 – Generate the `.R` file

Read `references/patterns.md` for code templates. Apply the matching pattern for
each column's classification. Output structure:

1. **Packages** — `library(pacman); p_load(tidyverse, lubridate, janitor, here, readxl, writexl, haven, labelled, gtsummary, gt, flextable, summarytools)`
2. **Load** — `read_excel(sheet=, skip=)` → `remove_empty()` → `clean_names()`
3. **Structural** — remove duplicates, fix ID columns, handle multi-sheet if needed
4. **Clean** — one `mutate()` block per column or group, using `## ──` section headers
5. **Checks** — `glimpse(data)` then `summary(data)`

## Rules

- **Non-destructive:** load raw data into `raw <- read_excel(...)`, then `data <- raw %>% clean_names()` so the original is always recoverable in the session
- **Traceable:** every value mapping must be explicit (`case_when`, regex, `fct_collapse`) — no silent coercion or unnamed recodes
- **Long column names:** if original names are full survey questions or complex metadata, rename to short IDs (`c1`, `c2` or meaningful snake_case), then store originals as variable labels via `labelled::set_variable_labels()`
- Output a single `.R` file (default: `clean_data.R`)
- Self-contained: runs with `source("clean_data.R")` if `data.xlsx` is present
- No unresolved TODOs — every column must have actual values from inspection
- Factor reference level: absence/control first (`"No"`, `"Female"`, `"None"`, `"0"`)
- If an existing `.R` file has cleaning code, offer to replace that section only
- Add analysis packages (`pROC`, `broom`, `car`, etc.) only if user requests analysis too
