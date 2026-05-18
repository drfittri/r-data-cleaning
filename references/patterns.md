# R Cleaning Patterns

## 1. Categorical → Factor

```r
data <- data %>%
  mutate(
    col = case_when(
      str_to_upper(col) == "VALUE1" ~ "Label1",
      str_to_upper(col) == "VALUE2" ~ "Label2",
      TRUE ~ NA_character_
    ),
    col = factor(col, levels = c("Label1", "Label2"))
  )
```

For binary grouping (e.g. ethnicity): use `TRUE ~ "Other"` instead of `NA`.

## 2. Binary Yes/No

```r
data <- data %>%
  mutate(
    col = case_when(
      str_to_upper(col) == "YES" ~ "Yes",
      str_to_upper(col) == "NO"  ~ "No",
      TRUE ~ NA_character_
    ),
    col = factor(col, levels = c("No", "Yes"))
  )
```

## 3. Free-text multi-value → Binary flags

Normalize first, then `str_detect` per keyword. Use `\\b` for whole-word match,
`|` for synonyms.

```r
data <- data %>%
  mutate(
    col = col %>% as.character() %>% str_to_upper() %>%
      str_replace_all("\\s+", " ") %>% str_trim(),
    flag_a = if_else(str_detect(col, "\\bKEYWORD\\b"), "Yes", "No", missing = NA_character_),
    flag_b = if_else(str_detect(col, "SYN1|SYN2"), "Yes", "No", missing = NA_character_),
  ) %>%
  mutate(across(c(flag_a, flag_b), ~ factor(.x, levels = c("No", "Yes"))))
```

## 4. Numeric stored as string

```r
data <- data %>%
  mutate(
    col = case_when(
      is.na(col) ~ NA_character_,
      str_to_upper(col) == "SPECIAL" ~ "0",
      TRUE ~ str_extract(col, "^\\d+\\.?\\d*")
    ),
    col = as.numeric(col)
  )
```

Strip units: `str_remove(col, "\\s*(mmol/L|mg/dL|%)$")`

## 5. Consolidate columns → one factor

```r
data <- data %>%
  mutate(
    new = case_when(
      !is.na(col_a) ~ "A",
      !is.na(col_b) ~ "B",
      TRUE ~ "None"
    ),
    new = factor(new, levels = c("None", "A", "B"))
  )
```

## 6. Binary indicator from multiple columns

```r
data <- data %>%
  mutate(
    event = if_else(!is.na(col_a) | !is.na(col_b), "Yes", "No"),
    event = factor(event, levels = c("No", "Yes"))
  )
```

## 7. Count / ordinal from flags

```r
data <- data %>%
  mutate(
    n = (col_a == "Yes") + (col_b == "Yes") + (col_c == "Yes"),
    n_cat = case_when(is.na(src) ~ NA_character_, n == 0 ~ "0", n == 1 ~ "1", n >= 2 ~ "≥2"),
    n_cat = factor(n_cat, levels = c("0", "1", "≥2"))
  )
```

## 8. Date cleanup

```r
# Text dates
data <- data %>% mutate(date_col = dmy(date_col))

# Excel serial numbers
data <- data %>% mutate(date_col = as.Date(as.numeric(date_col), origin = "1899-12-30"))
```

## 9. Factor level renaming

```r
data <- data %>%
  mutate(col = fct_recode(col, "Nice Label" = "0", "Other Label" = "1"))
```

## 10. Long column names → short IDs + variable labels

When original names are full survey questions or complex metadata:

```r
orig_names <- names(data)  # preserve before clean_names
data <- data %>% clean_names()
# If names are still unwieldy, rename to short IDs
names(data) <- paste0("c", seq_along(data))
# Store originals as labels for gtsummary / sjPlot
data <- data %>% labelled::set_variable_labels(.labels = setNames(as.list(orig_names), names(data)))
```

## 11. Comma-separated → separate_rows approach

Alternative to `str_detect` flags when you need to reshape or count items:

```r
data <- data %>% mutate(.row_id = row_number())
long <- data %>%
  select(.row_id, multi_col) %>%
  mutate(multi_col = str_to_upper(str_trim(multi_col))) %>%
  separate_rows(multi_col, sep = "\\s*,\\s*")
# Derive per-row summaries, then join back
flags <- long %>%
  mutate(val = 1L) %>%
  pivot_wider(names_from = multi_col, values_from = val, values_fill = 0L)
data <- data %>% left_join(flags, by = ".row_id") %>% select(-.row_id)
```

Use `str_detect` (pattern 3) for simple binary flags; use `separate_rows`
when you need per-item counts, frequencies, or the long-form itself.

## 12. Structural: multi-sheet, skip rows, remove empty, deduplicate

```r
# Multi-sheet: read specific sheet
raw <- read_excel("data.xlsx", sheet = "Sheet2")

# Header not at row 1: skip metadata rows
raw <- read_excel("data.xlsx", skip = 3)

# Remove fully empty rows/columns
data <- raw %>% remove_empty(c("rows", "cols")) %>% clean_names()

# Remove duplicate rows
data <- data %>% distinct()
# Or flag them: data %>% mutate(.is_dupe = duplicated(.) | duplicated(., fromLast = TRUE))
```

## 13. Censored / limit-of-detection values

Lab values like `"<0.01"`, `">500"`, `"TNTC"`. Extract numeric + censoring flag.

```r
data <- data %>%
  mutate(
    col_cens = case_when(
      str_detect(col, "^<")  ~ "left",
      str_detect(col, "^>")  ~ "right",
      str_detect(col, "TNTC|TOO") ~ "right",
      TRUE ~ "none"
    ),
    col_cens = factor(col_cens, levels = c("none", "left", "right")),
    col = as.numeric(str_extract(col, "\\d+\\.?\\d*"))
  )
```

## 14. Numeric codes → labelled factor

When data stores categories as numbers (1=Male, 2=Female):

```r
data <- data %>%
  mutate(
    col = factor(col, levels = c(1, 2), labels = c("Male", "Female"))
  )
```

## 15. Logical TRUE/FALSE → factor

R reads Excel TRUE/FALSE as `lgl`. Convert for gtsummary:

```r
data <- data %>%
  mutate(col = factor(if_else(col, "Yes", "No"), levels = c("No", "Yes")))
```

## 16. ID columns: preserve as character

IDs with leading zeros (`"001"`) must stay character. Force on load or fix after:

```r
# On load: col_types argument
raw <- read_excel("data.xlsx", col_types = c("text", "guess", "guess"))

# After load: convert back
data <- data %>% mutate(id = sprintf("%03d", as.integer(id)))
```
