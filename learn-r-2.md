# Learn R — A Practical Guide

> R is a language built for statistics, data analysis, and visualisation. It's quirky, vectorised by default, and once it clicks it's genuinely fun. Work through this file with RStudio open — copy each block, run it, then break it on purpose.

---

## Table of Contents

0. [Setup](#0-setup)
1. [The Absolute Basics](#1-the-absolute-basics)
2. [Data Types](#2-data-types)
3. [Vectors — the heart of R](#3-vectors--the-heart-of-r)
4. [Lists & Data Frames](#4-lists--data-frames)
5. [Control Flow](#5-control-flow)
6. [Functions](#6-functions)
7. [Packages](#7-packages)
8. [dplyr — data wrangling that reads like English](#8-dplyr--data-wrangling-that-reads-like-english)
9. [ggplot2 — plotting](#9-ggplot2--plotting)
10. [Reading & Writing Data](#10-reading--writing-data)
11. [A Mini Statistics Tour](#11-a-mini-statistics-tour)
12. [Common Gotchas](#12-common-gotchas)
13. [A Suggested Learning Path](#13-a-suggested-learning-path)
14. [Resources Worth Your Time](#14-resources-worth-your-time)

---

## 0. Setup

1. Install **R** from [cran.r-project.org](https://cran.r-project.org/)
2. Install **RStudio Desktop** (the IDE) from [posit.co](https://posit.co/download/rstudio-desktop/)
3. Open RStudio → `File > New File > R Script`
4. Run a line with `Ctrl/Cmd + Enter`

```r
# Your first line of R
print("hello, R")
```

---

## 1. The Absolute Basics

### Assignment
R uses `<-` (preferred) or `=` for assignment.

```r
x <- 10
y = 5        # works, but stylistically <- is more "R"
x + y        # 15
```

### Comments
Anything after `#` is a comment.

### Getting help
```r
?mean        # opens help for the mean() function
??regression # fuzzy search across help
```

---

## 2. Data Types

R has a handful of atomic types you'll meet constantly:

| Type        | Example          | Check with        |
|-------------|------------------|-------------------|
| numeric     | `3.14`           | `is.numeric()`    |
| integer     | `5L`             | `is.integer()`    |
| character   | `"hello"`        | `is.character()`  |
| logical     | `TRUE` / `FALSE` | `is.logical()`    |
| factor      | categorical      | `is.factor()`     |
| NA          | missing value    | `is.na()`         |

```r
class(3.14)        # "numeric"
class("hello")     # "character"
class(TRUE)        # "logical"
```

---

## 3. Vectors — the heart of R

A vector is a 1-D sequence of values of **the same type**. Almost everything in R is a vector under the hood.

```r
nums    <- c(1, 2, 3, 4, 5)
words   <- c("apple", "pear", "fig")
bools   <- c(TRUE, FALSE, TRUE)

length(nums)       # 5
nums * 2           # 2 4 6 8 10   <- vectorised!
nums[1]            # 1   (R is 1-indexed, not 0)
nums[2:4]          # 2 3 4
nums[nums > 2]     # 3 4 5
```

> **Critical tip:** Loops are often unnecessary in R. If you find yourself writing `for`, ask whether a vectorised operation would do.

### Sequences
```r
1:10               # 1 2 3 ... 10
seq(0, 1, by = 0.1)
rep("hi", 3)       # "hi" "hi" "hi"
```

---

## 4. Lists & Data Frames

### Lists — vectors that can mix types
```r
me <- list(name = "Ada", age = 36, langs = c("R", "Python"))
me$name            # "Ada"
me[["langs"]]      # "R" "Python"
```

### Data frames — the workhorse
A data frame is a list of equal-length vectors, displayed as a table.

```r
df <- data.frame(
  name  = c("Ada", "Bo", "Cee"),
  score = c(91, 85, 78),
  pass  = c(TRUE, TRUE, FALSE)
)

df              # print
str(df)         # structure
head(df, 2)     # first 2 rows
df$score        # column
df[1, ]         # first row
df[, "name"]    # name column
df[df$pass, ]   # rows where pass is TRUE
```

---

## 5. Control Flow

```r
# if / else
if (x > 0) {
  print("positive")
} else if (x == 0) {
  print("zero")
} else {
  print("negative")
}

# for loop
for (i in 1:5) {
  print(i^2)
}

# while
n <- 1
while (n < 100) n <- n * 2
n   # 128
```

---

## 6. Functions

```r
square <- function(x) {
  x^2
}
square(4)          # 16

# default args + multiple returns via list
stats <- function(v, na.rm = TRUE) {
  list(
    mean = mean(v, na.rm = na.rm),
    sd   = sd(v,   na.rm = na.rm)
  )
}
stats(c(1, 2, 3, NA))
```

### The `apply` family — vectorised iteration
```r
sapply(1:5, function(x) x^2)           # 1 4 9 16 25
lapply(1:3, function(x) rep("a", x))   # returns a list
```

---

## 7. Packages

R's superpower is its package ecosystem (CRAN + Bioconductor).

```r
install.packages("dplyr")    # once
library(dplyr)               # every session
```

The **tidyverse** is a collection most modern R users live in:

```r
install.packages("tidyverse")
library(tidyverse)   # loads dplyr, ggplot2, tidyr, readr, ...
```

---

## 8. dplyr — data wrangling that reads like English

The pipe `|>` (or `%>%` from magrittr) passes the left side as the first argument of the right.

```r
library(dplyr)

starwars |>
  filter(species == "Human") |>
  select(name, height, mass) |>
  mutate(bmi = mass / (height / 100)^2) |>
  arrange(desc(bmi)) |>
  head(5)
```

Core verbs to memorise:

| Verb         | Does                  |
|--------------|-----------------------|
| `filter()`   | keep rows             |
| `select()`   | keep columns          |
| `mutate()`   | add / modify columns  |
| `arrange()`  | sort                  |
| `summarise()`| collapse to one row   |
| `group_by()` | group for summarise   |

---

## 9. ggplot2 — plotting

ggplot builds plots in layers: data + aesthetic mappings + geometric layers.

```r
library(ggplot2)

ggplot(mtcars, aes(x = wt, y = mpg, colour = factor(cyl))) +
  geom_point(size = 3) +
  geom_smooth(method = "lm", se = FALSE) +
  labs(
    title  = "Weight vs MPG",
    x      = "Weight (1000 lbs)",
    y      = "Miles per gallon",
    colour = "Cylinders"
  ) +
  theme_minimal()
```

---

## 10. Reading & Writing Data

```r
library(readr)

df <- read_csv("data.csv")
write_csv(df, "out.csv")

# Excel
library(readxl)
read_excel("file.xlsx", sheet = 1)

# RDS — R's native binary format, fast and lossless
saveRDS(df, "df.rds")
df2 <- readRDS("df.rds")
```

---

## 11. A Mini Statistics Tour

```r
x <- rnorm(100, mean = 50, sd = 10)   # 100 normal samples

mean(x); median(x); sd(x); var(x)
summary(x)
quantile(x, c(0.25, 0.5, 0.75))

# t-test
t.test(x, mu = 50)

# linear model
fit <- lm(mpg ~ wt + hp, data = mtcars)
summary(fit)
coef(fit)
```

---

## 12. Common Gotchas

- **1-indexed**, not 0. `v[1]` is the first element.
- `NA` propagates silently. `mean(c(1, NA))` is `NA`. Use `na.rm = TRUE`.
- `=` inside function calls means argument assignment, not equality. Use `==` to compare.
- `TRUE` and `FALSE` are reserved, but `T` and `F` are just variables that *happen* to equal them and can be overwritten. Don't use `T`/`F`.
- Strings: single and double quotes are equivalent, but be consistent.
- `factor` levels can bite you when reading messy CSVs — `read_csv()` from readr avoids the worst of it.

---

## 13. A Suggested Learning Path

1. Get comfortable with vectors and indexing (Sections 1–4).
2. Pick a dataset you actually care about — sport, finance, music, anything.
3. Wrangle it with `dplyr`, plot it with `ggplot2`.
4. Write one function a day for a week.
5. Tackle a small project end to end: read → clean → analyse → visualise → write up in **R Markdown** or **Quarto**.

---

## 14. Resources Worth Your Time

- **R for Data Science (2e)** by Wickham, Çetinkaya-Rundel & Grolemund — free at [r4ds.hadley.nz](https://r4ds.hadley.nz/)
- **Advanced R** by Wickham — free at [adv-r.hadley.nz](https://adv-r.hadley.nz/) (for after the basics)
- **swirl** — interactive lessons inside R: `install.packages("swirl"); library(swirl); swirl()`
- **Posit Cheatsheets** — [posit.co/resources/cheatsheets](https://posit.co/resources/cheatsheets/)
- **CRAN Task Views** — curated package lists by topic

---

*Open RStudio. Run the first block. Don't read the next section until the current one runs without errors.*
