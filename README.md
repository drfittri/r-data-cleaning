# R Data Cleaning Skill

An AI agent skill that generates production-ready R cleaning scripts from messy Excel files — built for the statistical analysis freelancer who needs speed without sacrificing control.

## What This Is

This is a **skill file** for AI coding agents (Claude Code, opencode, Cursor, etc.). When loaded, it teaches the agent to:

1. **Inspect** any Excel file (`data.xlsx`) — column-by-column, sheet-by-sheet
2. **Classify** every column (categorical, binary, free-text, numeric-as-string, censored, date, ID, etc.)
3. **Generate** a self-contained `clean_data.R` script that turns raw data into analysis-ready output

The generated script is non-destructive, fully traceable, and ready to `source()`.

## Philosophy

> Fast workflow, human in the loop.

Every cleaning decision is **explicit** — `case_when()` mappings, regex patterns, factor levels. No silent coercion. No unnamed recodes. The analyst reviews, adjusts, and runs. The agent does the grinding; you keep the judgment.

## What It Handles

| Data Problem | Approach |
|---|---|
| Categorical strings (Male/Female) | `case_when()` → `factor()` |
| Binary yes/no fields | Normalize → `factor(c("No","Yes"))` |
| Free-text multi-value ("HPT, CAD, DM") | `str_detect()` → binary indicator flags |
| Numeric stored as string | `str_extract()` → `as.numeric()` |
| Censored lab values ("<0.01", ">500") | Extract number + censoring flag column |
| Numeric codes (1=Male, 2=Female) | `factor()` with explicit labels |
| Excel TRUE/FALSE → factor | `if_else()` → `factor(c("No","Yes"))` |
| Date cleanup (text + serial numbers) | `lubridate::dmy()` / `as.Date(origin=)` |
| ID columns with leading zeros | Preserve as character, never coercing to numeric |
| Long survey question column names | Rename to short IDs + `labelled::set_variable_labels()` |
| Multi-sheet files, skipped headers | Sheet selection, `skip=` parameter |
| Derived/composite columns | `case_when()` across source columns |

## File Structure

```
r-data-cleaning/
├── SKILL.md              # Main skill instructions (loaded by the AI agent)
├── references/
│   └── patterns.md       # 16 reusable R code patterns for common cleaning tasks
└── README.md             # This file
```

## How to Use

### As an AI Agent Skill

Install this skill in your agent of choice, then prompt with:

> "Clean data.xlsx and generate an R cleaning script"

The agent will inspect your file, classify columns, and write `clean_data.R`.

### Manual Reference

The patterns in `references/patterns.md` are also useful standalone — copy-paste the template that matches your data problem.

## Requirements

The generated scripts use these R packages (installed via `pacman`):

- `tidyverse` — data wrangling
- `readxl` / `writexl` — Excel I/O
- `janitor` — `clean_names()`, `remove_empty()`
- `lubridate` — date parsing
- `labelled` — variable labels for `gtsummary`
- `gtsummary`, `flextable`, `summarytools` — inspection and Table 1

## Example Output

Given a messy clinical dataset with free-text comorbidity fields, censored lab values, and numeric-coded demographics, the agent produces a script like:

```r
# ── Packages ──────────────────────────────────────────────────────────────────
library(pacman)
p_load(tidyverse, lubridate, janitor, here, readxl, writexl, labelled, gtsummary, summarytools)

# ── Load ──────────────────────────────────────────────────────────────────────
raw <- read_excel(here("data.xlsx"), sheet = "Sheet1")
data <- raw |> remove_empty(c("rows", "cols")) |> clean_names() |> distinct()

# ── Demographics ──────────────────────────────────────────────────────────────
data <- data |> mutate(
  sex = factor(sex, levels = c(1, 2), labels = c("Male", "Female")),
  age = as.numeric(age)
)

# ── Comorbidities: free-text parsing ──────────────────────────────────────────
data <- data |> mutate(
  comorb_norm = comorbidities |> as.character() |> str_to_upper() |> str_trim(),
  dm   = if_else(str_detect(comorb_norm, "\\bDM\\b|DIABETES"), "Yes", "No"),
  hpt  = if_else(str_detect(comorb_norm, "\\bHPT\\b|HYPERTENSION"), "Yes", "No"),
  across(c(dm, hpt), ~ factor(.x, levels = c("No", "Yes")))
)

# ── Lab: censored values ──────────────────────────────────────────────────────
data <- data |> mutate(
  crp_cens = case_when(str_detect(crp_raw, "^<") ~ "left", TRUE ~ "none"),
  crp = as.numeric(str_extract(crp_raw, "\\d+\\.?\\d*"))
)

# ── Checks ────────────────────────────────────────────────────────────────────
glimpse(data)
summary(data)
```

## About the Author

I'm a statistical analysis freelancer. I built this skill because I got tired of writing the same cleaning boilerplate for every project — and because I believe the analyst should always be in the loop. The agent handles the repetitive pattern-matching; the human validates every decision before it touches real data.

**Need statistical analysis help?** Reach out — I work with clinical, epidemiological, and social science datasets. Fast turnaround, transparent methods, full reproducibility.

---

*This skill pairs well with the [r-data-analysis](https://github.com/drfittri/r-data-analysis) skill for automated Table 1, univariate regression, and logistic regression workflows.*
