# Stata → R: Data Cleaning for Epidemiology

A side-by-side reference for epidemiologists moving cleaning workflows from **Stata** to **R**.
Every Stata snippet has a matching R snippet, so you can find what you already know and read off the translation.

This guide is **tidyverse-first** (plus [`haven`](https://haven.tidyverse.org/) for `.dta` files and labels). The tidyverse reads close to Stata's verb-based logic — `gen`, `keep if`, `collapse` map almost one-to-one onto `mutate`, `filter`, `summarise`. Where base R or `data.table` is genuinely worth knowing, it's flagged.

> **One pipe note up front.** R chains steps with a pipe. This guide uses the native `|>` (R ≥ 4.1). If you read older code you'll see `%>%` from `magrittr` — for everything here they behave the same. Think of the pipe as the equivalent of running commands in sequence on the dataset in memory.

---

## Contents

1. [Setup](#setup)
2. [The big cheat sheet](#the-big-cheat-sheet)
3. [Reading data in](#reading-data-in)
4. [Inspecting your data](#inspecting-your-data)
5. [Renaming variables](#renaming-variables)
6. [Creating & modifying variables (`gen`, `replace`, `egen`)](#creating--modifying-variables)
7. [Recoding & conditional logic (`recode`, `case_when`)](#recoding--conditional-logic)
8. [Variable & value labels](#variable--value-labels)
9. [Keeping & dropping (`keep`, `drop`, `if`)](#keeping--dropping)
10. [Sorting](#sorting)
11. [Missing values (read this one)](#missing-values)
12. [Dates](#dates)
13. [Duplicates](#duplicates)
14. [The `by` / `bysort` prefix](#the-by--bysort-prefix)
15. [Aggregating (`collapse`)](#aggregating)
16. [Reshaping (`reshape`)](#reshaping)
17. [Merging datasets (`merge`)](#merging-datasets)
18. [Tabulations & frequencies](#tabulations--frequencies)
19. [`destring`, `encode`, `decode`](#destring-encode-decode)
20. [Saving](#saving)
21. [A worked mini cleaning pipeline](#a-worked-mini-cleaning-pipeline)
22. [Gotchas for Stata users](#gotchas-for-stata-users)
23. [Further resources](#further-resources)

---

## Setup

```r
install.packages(c("tidyverse", "haven", "janitor", "labelled", "skimr"))

library(tidyverse)   # dplyr, tidyr, readr, stringr, etc.
library(haven)       # read/write .dta, keep Stata labels
library(janitor)     # clean_names(), tabyl()
library(labelled)    # work with variable/value labels
```

In Stata your dataset lives in memory and commands act on it implicitly. In R you hold data in **named objects** (data frames / tibbles) and pass them explicitly. The mental shift: instead of `gen bmi = ...` acting on "the" dataset, you write `df <- df |> mutate(bmi = ...)` — you're reassigning the object.

---

## The big cheat sheet

| Task | Stata | R (tidyverse) |
|---|---|---|
| Load Stata file | `use "d.dta", clear` | `df <- read_dta("d.dta")` |
| Load CSV | `import delimited "d.csv", clear` | `df <- read_csv("d.csv")` |
| Inspect structure | `describe` | `glimpse(df)` |
| Summary stats | `summarize` | `summary(df)` / `skim(df)` |
| Rename | `rename old new` | `rename(df, new = old)` |
| New variable | `gen y = x*2` | `mutate(df, y = x*2)` |
| Replace conditionally | `replace y = 0 if x<5` | `mutate(df, y = if_else(x<5, 0, y))` |
| Recode | `recode x (1=0)(2=1)` | `mutate(df, x = recode(x, `1`=0, `2`=1))` |
| Keep columns | `keep id age sex` | `select(df, id, age, sex)` |
| Drop columns | `drop weight` | `select(df, -weight)` |
| Keep rows | `keep if age>=18` | `filter(df, age>=18)` |
| Drop rows | `drop if missing(age)` | `filter(df, !is.na(age))` |
| Sort | `sort age` | `arrange(df, age)` |
| Sort descending | `gsort -age` | `arrange(df, desc(age))` |
| Drop dups | `duplicates drop` | `distinct(df)` |
| Group + aggregate | `collapse (mean) bp, by(arm)` | `df |> group_by(arm) |> summarise(bp = mean(bp))` |
| Wide → long | `reshape long` | `pivot_longer(...)` |
| Long → wide | `reshape wide` | `pivot_wider(...)` |
| Merge 1:1 | `merge 1:1 id using "b.dta"` | `left_join(a, b, by = "id")` |
| Frequency | `tab sex` | `count(df, sex)` / `tabyl(df, sex)` |
| Save | `save "out.dta", replace` | `write_dta(df, "out.dta")` |

Everything below expands these with the patterns you actually hit in epi work.

---

## Reading data in

```stata
* Stata
use "cohort.dta", clear
import delimited "lab_results.csv", clear
import excel "survey.xlsx", firstrow clear
```

```r
# R
df  <- read_dta("cohort.dta")          # haven; keeps labels
lab <- read_csv("lab_results.csv")     # readr; guesses column types
sur <- readxl::read_excel("survey.xlsx")
```

**Tidy your variable names on the way in.** Stata names are already syntactic; R is happy with `Age (years)` as a column name but it'll make you miserable. `janitor::clean_names()` standardises everything to `snake_case`:

```r
df <- read_csv("messy.csv") |> clean_names()
# "Age (years)" -> age_years, "HbA1c%" -> hb_a1c_percent
```

---

## Inspecting your data

| Stata | R | Notes |
|---|---|---|
| `browse` | `View(df)` | Opens the spreadsheet viewer (RStudio) |
| `list in 1/10` | `head(df, 10)` | First rows |
| `describe` | `glimpse(df)` | Types + first values, one row per variable |
| `codebook age` | `df |> count(age)` or `skim(df$age)` | |
| `summarize` | `summary(df)` | |
| `summarize, detail` | `skimr::skim(df)` | Richer; percentiles, missingness, mini-histograms |
| `inspect x` | `skim(df$x)` | |
| `count` | `nrow(df)` | Number of observations |

`skim()` is the closest single thing to `summarize, detail` across a whole dataset, and it reports missingness per variable — which you want constantly in epi.

---

## Renaming variables

```stata
* Stata
rename old_name new_name
rename (v1 v2 v3) (age sex bmi)
rename q* answer*           // pattern
```

```r
# R — note the order is new = old (opposite of Stata's read order, same logic)
df <- df |> rename(new_name = old_name)
df <- df |> rename(age = v1, sex = v2, bmi = v3)

# pattern renames
df <- df |> rename_with(~ str_replace(.x, "^q", "answer"), starts_with("q"))
```

---

## Creating & modifying variables

`gen`/`replace`/`egen` all collapse into one verb: **`mutate()`**.

```stata
* Stata
gen bmi = weight / (height/100)^2
gen agegrp = .
replace agegrp = 1 if age < 40
replace agegrp = 2 if age >= 40 & age < 65
replace agegrp = 3 if age >= 65

egen mean_bp = mean(sbp)            // scalar broadcast to all rows
egen max_visit = max(visit), by(id) // within-group
```

```r
# R
df <- df |>
  mutate(
    bmi    = weight / (height/100)^2,
    agegrp = case_when(
      age < 40            ~ 1,
      age >= 40 & age < 65 ~ 2,
      age >= 65           ~ 3
    ),
    mean_bp = mean(sbp, na.rm = TRUE)   # recycled across all rows, like egen mean
  )

# egen ... , by(id)  -> group_by then mutate
df <- df |>
  group_by(id) |>
  mutate(max_visit = max(visit, na.rm = TRUE)) |>
  ungroup()
```

> **Always `ungroup()`** after a grouped `mutate`/`summarise` unless you deliberately want the grouping to persist. Forgetting this is the #1 silent bug for Stata users, because Stata's `by` only lasts for one command.

A single conditional `replace` is cleanest with `if_else()`:

```stata
replace dose = 0 if dose == .
```

```r
df <- df |> mutate(dose = if_else(is.na(dose), 0, dose))
```

---

## Recoding & conditional logic

```stata
* Stata
recode smoke (1=0)(2=1)(9=.)
recode age (0/17=1)(18/64=2)(65/max=3), gen(agecat)
```

```r
# R — recode() for value-to-value swaps
df <- df |> mutate(smoke = recode(smoke, `1` = 0, `2` = 1, `9` = NA_real_))

# case_when() for ranges (the recode (a/b=) syntax)
df <- df |>
  mutate(agecat = case_when(
    age <= 17           ~ 1,
    age >= 18 & age <= 64 ~ 2,
    age >= 65           ~ 3,
    TRUE                ~ NA_real_   # the catch-all "everything else"
  ))
```

`case_when()` is your workhorse. It reads top to bottom and stops at the first match — the same way you'd stack `replace ... if` lines, but safer because the conditions are explicit and you can see them all at once.

---

## Variable & value labels

This matters more in epi than almost anywhere, because Stata datasets carry labels everywhere. `haven` preserves them on import as a `labelled` class.

```stata
* Stata
label variable sbp "Systolic blood pressure (mmHg)"
label define sexlbl 0 "Male" 1 "Female"
label values sex sexlbl
```

```r
# R, with the labelled package
df <- df |>
  set_variable_labels(sbp = "Systolic blood pressure (mmHg)") |>
  set_value_labels(sex = c(Male = 0, Female = 1))

# See them
var_label(df$sbp)
val_labels(df$sex)
```

**The key difference:** in R you usually convert labelled variables to proper **factors** for analysis, rather than carrying numeric-codes-with-labels the way Stata does:

```r
df <- df |> mutate(sex = as_factor(sex))   # haven: uses value labels as levels
# now sex prints as Male/Female and behaves correctly in models & tables
```

Factors are R's native categorical type — they're what `tab`, regression, and plotting expect. If you remember one thing: **Stata value labels ≈ R factors.**

---

## Keeping & dropping

```stata
* Stata
keep id age sex sbp dbp
drop notes comment
keep if age >= 18
drop if missing(sbp)
keep if inrange(visit, 1, 3)
```

```r
# R — columns: select() | rows: filter()
df <- df |> select(id, age, sex, sbp, dbp)
df <- df |> select(-notes, -comment)
df <- df |> filter(age >= 18)
df <- df |> filter(!is.na(sbp))
df <- df |> filter(between(visit, 1, 3))
```

`select()` also takes helpers Stata can't easily do: `starts_with("lab_")`, `ends_with("_v2")`, `contains("date")`, `where(is.numeric)`.

---

## Sorting

```stata
* Stata
sort id visit
gsort -sbp          // descending
gsort id -date
```

```r
# R
df <- df |> arrange(id, visit)
df <- df |> arrange(desc(sbp))
df <- df |> arrange(id, desc(date))
```

---

## Missing values

This is the single biggest place Stata habits cause **wrong results in R**, so read carefully.

In Stata, missing `.` is treated as **larger than any number**. So `keep if age > 90` silently *keeps* the missing ages. In R, missing is `NA` and comparisons with it return `NA` (not TRUE), so `filter(age > 90)` *drops* the missings. The behaviours are opposite — audit any `if` condition you port over.

```stata
* Stata
gen flag = missing(sbp)
egen nmiss = rowmiss(sbp dbp map)
mvdecode _all, mv(9999)     // turn 9999 into missing
```

```r
# R
df <- df |> mutate(flag = is.na(sbp))

# count missings across columns in each row
df <- df |> mutate(nmiss = rowSums(is.na(across(c(sbp, dbp, map)))))

# replace a sentinel value with NA across the whole frame
df <- df |> mutate(across(everything(), ~ na_if(.x, 9999)))
```

Functions like `mean()`, `sum()`, `sd()` return `NA` if **any** value is missing — you must pass `na.rm = TRUE` explicitly. Stata drops missings from these by default, so this trips up everyone at least once:

```r
mean(df$sbp)                 # NA if any sbp missing  <- surprise
mean(df$sbp, na.rm = TRUE)   # what you actually wanted
```

---

## Dates

```stata
* Stata
gen edate = date(date_str, "DMY")
format edate %td
gen fu_days = exit_date - enrol_date
```

```r
# R — lubridate (part of tidyverse) parses by stating the order
library(lubridate)
df <- df |>
  mutate(
    edate   = dmy(date_str),                 # day-month-year
    fu_days = as.numeric(exit_date - enrol_date)   # difference in days
  )
```

`ymd()`, `mdy()`, `dmy()` mirror Stata's `"YMD"`/`"MDY"`/`"DMY"` masks. Date subtraction gives a `difftime`; wrap in `as.numeric()` to get plain days, the way Stata stores them as integers.

---

## Duplicates

```stata
* Stata
duplicates report id
duplicates tag id visit, gen(dup)
duplicates drop                 // drop fully identical rows
bysort id (date): keep if _n == 1   // first record per id
```

```r
# R
df |> count(id) |> filter(n > 1)                 # report dups
df <- df |> group_by(id, visit) |> mutate(dup = n() - 1) |> ungroup()
df <- df |> distinct()                           # drop identical rows

# first record per id (sort within group, take row 1)
df <- df |> arrange(id, date) |> distinct(id, .keep_all = TRUE)
```

---

## The `by` / `bysort` prefix

Stata's `bysort` is a temporary prefix on one command. R makes the grouping an explicit, persistent step — so you `group_by()`, do the work, then `ungroup()`.

```stata
* Stata
bysort id (date): gen visit_n = _n
bysort id: egen total_visits = count(visit)
bysort id (date): gen baseline_bp = sbp[1]
```

```r
# R
df <- df |>
  arrange(id, date) |>
  group_by(id) |>
  mutate(
    visit_n      = row_number(),    # Stata's _n within group
    total_visits = n(),             # Stata's _N within group
    baseline_bp  = first(sbp)       # sbp[1] after sorting
  ) |>
  ungroup()
```

Cheat sheet for the Stata `_n` / `_N` family:

| Stata | R (inside `group_by`) |
|---|---|
| `_n` | `row_number()` |
| `_N` | `n()` |
| `x[1]` | `first(x)` |
| `x[_N]` | `last(x)` |
| `x[_n-1]` | `lag(x)` |
| `x[_n+1]` | `lead(x)` |

---

## Aggregating

`collapse` becomes `group_by()` + `summarise()`.

```stata
* Stata
collapse (mean) sbp dbp (sd) sbp (count) n=id, by(arm sex)
```

```r
# R
summary_df <- df |>
  group_by(arm, sex) |>
  summarise(
    sbp_mean = mean(sbp, na.rm = TRUE),
    dbp_mean = mean(dbp, na.rm = TRUE),
    sbp_sd   = sd(sbp, na.rm = TRUE),
    n        = n(),
    .groups  = "drop"          # equivalent to ungrouping afterwards
  )
```

Note this creates a **new, smaller object** (`summary_df`) instead of overwriting your data the way `collapse` destructively does in memory. That's usually a feature — your row-level data survives.

---

## Reshaping

```stata
* Stata: wide -> long
reshape long bp_, i(id) j(visit)

* long -> wide
reshape wide bp, i(id) j(visit)
```

```r
# R: wide -> long
long <- df |>
  pivot_longer(
    cols = starts_with("bp_"),
    names_to = "visit",
    names_prefix = "bp_",
    values_to = "bp"
  )

# long -> wide
wide <- long |>
  pivot_wider(
    id_cols = id,
    names_from = visit,
    values_from = bp,
    names_prefix = "bp_"
  )
```

`pivot_longer`/`pivot_wider` are more verbose than `reshape` but far less cryptic — you name each piece (`names_to`, `values_to`) instead of memorising `i()`/`j()`.

---

## Merging datasets

Stata's `merge` becomes a **join**. The big conceptual upgrade: you state the relationship by *choosing the join type*, and there's no `_merge` variable to clean up afterwards.

```stata
* Stata
merge 1:1 id using "labs.dta"
merge m:1 site using "site_info.dta"
keep if _merge == 3      // keep matched only
```

```r
# R
both    <- left_join(cohort, labs, by = "id")        # keep all of cohort
matched <- inner_join(cohort, labs, by = "id")       # _merge == 3 only
all_rows<- full_join(cohort, labs, by = "id")        # keep everything
```

| Stata intent | R join |
|---|---|
| keep master + matched (`_merge` 1 & 3) | `left_join(master, using)` |
| matched only (`_merge == 3`) | `inner_join()` |
| everything (`_merge` 1, 2 & 3) | `full_join()` |
| only in master | `anti_join(master, using)` |

The `1:1` / `m:1` distinction isn't declared in the join itself — but you can have R **check** it, which catches the silent fan-out merges that bite people:

```r
both <- left_join(cohort, labs, by = "id", relationship = "one-to-one")
# errors loudly if the keys aren't unique as you claimed
```

---

## Tabulations & frequencies

```stata
* Stata
tab sex
tab1 sex smoke diabetes
tab sex arm, row col
tab agegrp, sum(sbp)
```

```r
# R — count() is quick; tabyl() is the epi-friendly one
count(df, sex)
df |> select(sex, smoke, diabetes) |> map(table)   # tab1 over several vars

# cross-tab with row/col percentages
df |>
  tabyl(sex, arm) |>
  adorn_totals(c("row", "col")) |>
  adorn_percentages("row") |>
  adorn_pct_formatting()

# tab var, sum(other)  -> group means
df |> group_by(agegrp) |> summarise(mean_sbp = mean(sbp, na.rm = TRUE), n = n())
```

`janitor::tabyl()` is the closest thing to Stata's `tab` for epidemiology — it gives counts, percentages, and tidy 2×2 tables you can pipe straight into `epitools` for risk/odds ratios.

---

## destring, encode, decode

```stata
* Stata
destring age, replace        // string "45" -> numeric 45
encode sex_str, gen(sex)     // string -> labelled numeric
decode sex, gen(sex_str)     // labelled numeric -> string
tostring id, replace
```

```r
# R
df <- df |> mutate(age = as.numeric(age))            # destring
df <- df |> mutate(age = parse_number(age))          # destring, ignoring stray text/units
df <- df |> mutate(sex = factor(sex_str))            # encode (string -> factor)
df <- df |> mutate(sex_str = as.character(sex))      # decode-ish
df <- df |> mutate(id = as.character(id))            # tostring
```

`parse_number()` is handy when a "numeric" column arrived with junk like `"45 kg"` or `"<0.01"` — it pulls the number out where `as.numeric()` would just give `NA`.

---

## Saving

```stata
* Stata
save "clean.dta", replace
export delimited "clean.csv", replace
```

```r
# R
write_dta(df, "clean.dta")     # back to Stata, labels preserved
write_csv(df, "clean.csv")
saveRDS(df, "clean.rds")       # native R format — fastest, keeps all types/labels
```

For a pure-R workflow prefer `.rds`: it round-trips factors, labels, and types perfectly and loads with `df <- readRDS("clean.rds")`. Use `write_dta()` when a collaborator still needs Stata.

---

## A worked mini cleaning pipeline

Here's the same end-to-end clean in both languages, so you can see how a chain of Stata commands becomes one piped R block.

```stata
* Stata
import delimited "raw_cohort.csv", clear
rename (v1 v2 v3 v4) (id age sex_str sbp)
destring sbp, replace
encode sex_str, gen(sex)
recode sbp (0=.)                          // implausible zeros -> missing
keep if age >= 18
gen agegrp = .
replace agegrp = 1 if age < 40
replace agegrp = 2 if age >= 40 & age < 65
replace agegrp = 3 if age >= 65
duplicates drop id, force
save "clean_cohort.dta", replace
```

```r
# R — one readable pipeline
clean <- read_csv("raw_cohort.csv") |>
  clean_names() |>
  rename(id = v1, age = v2, sex_str = v3, sbp = v4) |>
  mutate(
    sbp    = parse_number(sbp),
    sex    = factor(sex_str),
    sbp    = if_else(sbp == 0, NA_real_, sbp),   # implausible zeros -> NA
    agegrp = case_when(
      age < 40             ~ 1,
      age >= 40 & age < 65 ~ 2,
      age >= 65            ~ 3
    )
  ) |>
  filter(age >= 18) |>
  distinct(id, .keep_all = TRUE)

write_dta(clean, "clean_cohort.dta")
```

The whole logic sits in one block you can read top to bottom — and your raw object is untouched, so re-running is safe.

---

## Gotchas for Stata users

- **Missing comparisons are reversed.** `keep if x > 5` keeps missings in Stata; `filter(x > 5)` drops them in R. Audit every ported condition.
- **`mean()`/`sum()` need `na.rm = TRUE`.** They return `NA` on any missing, unlike Stata which drops missings silently.
- **Grouping persists.** `group_by()` stays active across commands until you `ungroup()`. Stata's `by` lasts one command only.
- **You don't edit "the" dataset — you reassign objects.** If you forget the `df <- ...`, your change vanishes. Nothing happens "in memory" implicitly.
- **`=` vs `==`.** `=` assigns, `==` tests equality. Stata is more forgiving here; R is not.
- **Factors ≠ strings ≠ labelled numerics.** Stata blurs these; R keeps them distinct. For analysis you almost always want **factors**. Convert labelled imports with `as_factor()`.
- **1-based, and `NA` is contagious.** `x[1]` is the first element (not `x[0]`), and any arithmetic touching `NA` yields `NA`.
- **No `_merge` to babysit.** Pick the join type instead, and use `relationship =` to assert 1:1 / m:1 and get an error if it's violated.

---

## Further resources

- **Epidemiologist R Handbook** — `https://epirhandbook.com` — the field-specific reference; cleaning, case studies, outbreak analysis.
- **R for Data Science (2e)** — `https://r4ds.hadley.nz` — the canonical tidyverse text.
- **`haven` docs** — `https://haven.tidyverse.org` — Stata/SPSS/SAS import and labels.
- **`janitor`** — `https://sfirke.github.io/janitor/` — `tabyl()`, `clean_names()`, dup tools.
- **dplyr ↔ Stata** cheat sheet thinking: every `bysort: command` is `group_by() |> verb() |> ungroup()`.

---

*Contributions welcome — open an issue or PR if you hit a Stata pattern that's missing.*
