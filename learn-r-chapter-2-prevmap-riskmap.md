# Learn R — Chapter 2: Prevalence and Risk Mapping

> This chapter assumes you have the R basics from `learn-r-2.md`: vectors, data frames, packages, plotting, and simple models. Here the goal is applied: learn the shape of a disease mapping workflow using `PrevMap` and `RiskMap`.

---

## Table of Contents

0. [What These Packages Are For](#0-what-these-packages-are-for)
1. [Setup](#1-setup)
2. [The Data Shape You Need](#2-the-data-shape-you-need)
3. [A First `PrevMap` Workflow](#3-a-first-prevmap-workflow)
4. [Predicting a Prevalence Surface with `PrevMap`](#4-predicting-a-prevalence-surface-with-prevmap)
5. [A First `RiskMap` Workflow](#5-a-first-riskmap-workflow)
6. [Predicting Targets with `RiskMap`](#6-predicting-targets-with-riskmap)
7. [How to Read the Outputs](#7-how-to-read-the-outputs)
8. [Common Gotchas](#8-common-gotchas)
9. [Practice Exercises](#9-practice-exercises)
10. [Resources Worth Your Time](#10-resources-worth-your-time)

---

## 0. What These Packages Are For

`PrevMap` and `RiskMap` are for **model-based geostatistics**: modelling outcomes observed at point locations, then predicting risk or prevalence at unsampled locations.

Typical public-health examples:

- malaria prevalence measured in villages
- parasite infection counts from surveys
- disease presence/absence by household
- environmental measurements across space

The important idea is:

```text
observed outcome = covariate effects + spatially correlated residual pattern + noise
```

The spatially correlated part matters because nearby places often have similar risk.

---

## 1. Setup

Install the packages once:

```r
install.packages(c(
  "PrevMap",
  "RiskMap",
  "sf",
  "ggplot2",
  "dplyr"
))
```

Load them in each session:

```r
library(PrevMap)
library(RiskMap)
library(sf)
library(ggplot2)
library(dplyr)
```

> **Reality check:** geostatistical models can take minutes or hours on real data. Start with small examples, short MCMC settings, and only increase computation once the code runs end to end.

---

## 2. The Data Shape You Need

For prevalence mapping, your data usually has one row per sampled location:

```r
survey <- data.frame(
  village = c("A", "B", "C", "D"),
  x       = c(0, 1, 2, 3),
  y       = c(0, 1, 0, 2),
  positive = c(4, 8, 3, 12),
  tested   = c(30, 35, 28, 40),
  elevation = c(120, 150, 110, 180)
)

survey$prevalence <- survey$positive / survey$tested
survey
```

Key columns:

- `positive`: number of positive cases
- `tested`: number of people tested, also called the binomial denominator
- `x`, `y`: projected coordinates, not just arbitrary labels
- covariates such as `elevation`, rainfall, vegetation index, temperature, or urban/rural status

Plot the raw data before modelling:

```r
ggplot(survey, aes(x = x, y = y, colour = prevalence, size = tested)) +
  geom_point() +
  scale_colour_viridis_c() +
  coord_equal() +
  theme_minimal()
```

---

## 3. A First `PrevMap` Workflow

`PrevMap` is especially focused on spatial prevalence data. For binomial outcomes, the core model is fitted with `binomial.logistic.MCML()`.

This example simulates a small toy survey. It is not a scientific analysis; it is just a safe sandbox for learning the function arguments.

```r
set.seed(42)

n <- 35

dat <- data.frame(
  x = runif(n, 0, 10),
  y = runif(n, 0, 10),
  elevation = rnorm(n)
)

dat$tested <- sample(25:80, n, replace = TRUE)

# A fake risk surface: prevalence changes with elevation and east-west location.
linear_risk <- -1 + 0.5 * dat$elevation + 0.12 * dat$x
dat$prob <- plogis(linear_risk)
dat$positive <- rbinom(n, size = dat$tested, prob = dat$prob)

head(dat)
```

Set up short MCMC controls for learning:

```r
control <- control.mcmc.MCML(
  n.sim = 1000,
  burnin = 200,
  thin = 8
)
```

Fit the model:

```r
fit_prev <- binomial.logistic.MCML(
  formula = positive ~ elevation,
  units.m = ~ tested,
  coords = ~ x + y,
  data = dat,
  par0 = c(0, 0, 1, 2, 0.1),
  control.mcmc = control,
  kappa = 0.5,
  start.cov.pars = c(2, 0.1),
  method = "nlminb",
  messages = TRUE,
  plot.correlogram = FALSE
)

summary(fit_prev)
coef(fit_prev)
```

What the main arguments mean:

- `formula = positive ~ elevation` models positives as a function of covariates.
- `units.m = ~ tested` tells `PrevMap` the binomial denominator.
- `coords = ~ x + y` gives the spatial coordinates.
- `par0` gives starting values for regression and covariance parameters.
- `kappa` fixes the Matern smoothness parameter.
- `control.mcmc` controls the Monte Carlo part of the likelihood.

If the model fails, simplify first: remove covariates, reduce the spatial domain, or try different starting values.

---

## 4. Predicting a Prevalence Surface with `PrevMap`

After fitting, create a prediction grid:

```r
grid_prev <- expand.grid(
  x = seq(0, 10, length.out = 40),
  y = seq(0, 10, length.out = 40)
)

grid_covariates <- data.frame(
  elevation = 0
)

grid_covariates <- grid_covariates[rep(1, nrow(grid_prev)), , drop = FALSE]
```

Predict prevalence:

```r
pred_prev <- spatial.pred.binomial.MCML(
  object = fit_prev,
  grid.pred = as.matrix(grid_prev),
  predictors = grid_covariates,
  control.mcmc = control,
  type = "marginal",
  scale.predictions = "prevalence",
  quantiles = c(0.025, 0.975),
  standard.errors = TRUE,
  thresholds = 0.2,
  scale.thresholds = "prevalence",
  messages = TRUE
)
```

Turn the predictions into a plotting data frame:

```r
map_prev <- cbind(
  grid_prev,
  prevalence = pred_prev$prevalence$predictions,
  lower = pred_prev$prevalence$quantiles[, 1],
  upper = pred_prev$prevalence$quantiles[, 2],
  exceed_20 = pred_prev$exceedance.prob[, 1]
)

ggplot(map_prev, aes(x = x, y = y, fill = prevalence)) +
  geom_raster() +
  geom_point(data = dat, aes(x = x, y = y), inherit.aes = FALSE, size = 1) +
  scale_fill_viridis_c() +
  coord_equal() +
  theme_minimal()
```

Plot exceedance probability:

```r
ggplot(map_prev, aes(x = x, y = y, fill = exceed_20)) +
  geom_raster() +
  scale_fill_viridis_c(limits = c(0, 1)) +
  coord_equal() +
  labs(fill = "Pr(prevalence > 0.20)") +
  theme_minimal()
```

Exceedance maps are often more useful for decisions than raw prevalence maps because they answer a direct question: "Where is risk probably above the intervention threshold?"

---

## 5. A First `RiskMap` Workflow

`RiskMap` provides a newer interface for generalized linear Gaussian process models through `glgpm()`. It supports Gaussian, binomial, and Poisson outcomes.

Start with the built-in `italy_sim` example:

```r
data(italy_sim)

head(italy_sim)
str(italy_sim)
```

Fit a simple Gaussian spatial model:

```r
fit_risk_gaussian <- glgpm(
  formula = y ~ gp(x1, x2),
  data = italy_sim,
  family = "gaussian",
  crs = 32634,
  messages = FALSE
)

summary(fit_risk_gaussian)
coef(fit_risk_gaussian)
```

The formula `y ~ gp(x1, x2)` says:

- `y` is the response.
- `gp(x1, x2)` adds a spatial Gaussian process using the coordinate columns.
- `family = "gaussian"` means the response is continuous.

For binomial data, the pattern is similar, but you need a count response and a denominator:

```r
# Skeleton only: replace names with your real data columns.
fit_risk_binomial <- glgpm(
  formula = positive ~ elevation + gp(x, y),
  data = survey,
  family = "binomial",
  distr_offset = survey$tested,
  crs = 3857,
  control_mcmc = set_control_sim(
    n_sim = 2000,
    burnin = 500,
    thin = 10
  ),
  messages = TRUE
)
```

Here `distr_offset` is the binomial denominator, equivalent in spirit to `units.m` in `PrevMap`.

---

## 6. Predicting Targets with `RiskMap`

The typical `RiskMap` prediction flow is:

```text
fit model -> predict components over grid -> combine components into target -> plot
```

Create a prediction grid. If you already have a study-area polygon as an `sf` object, `create_grid()` can build a point grid inside it:

```r
# Example shape from the sf package.
nc <- st_read(system.file("shape/nc.shp", package = "sf"), quiet = TRUE)
nc <- st_transform(nc, 3857)

grid_sf <- create_grid(nc, spat_res = 20)
```

Predict over a grid:

```r
# Skeleton: grid_pred must use the same CRS as the fitted RiskMap object.
pred_grid <- pred_over_grid(
  object = fit_risk_binomial,
  grid_pred = st_geometry(grid_sf),
  predictors = data.frame(
    elevation = mean(survey$elevation, na.rm = TRUE)
  )[rep(1, nrow(grid_sf)), , drop = FALSE],
  control_sim = set_control_sim(
    n_sim = 2000,
    burnin = 500,
    thin = 10
  ),
  type = "marginal",
  messages = TRUE
)
```

Convert predicted components into the target you want:

```r
target_grid <- pred_target_grid(
  object = pred_grid,
  include_covariates = TRUE,
  include_nugget = FALSE,
  f_target = list(
    prevalence = plogis
  ),
  pd_summary = list(
    mean = mean,
    lower = function(x) quantile(x, 0.025),
    upper = function(x) quantile(x, 0.975)
  )
)

plot(target_grid, which_target = "prevalence", which_summary = "mean")
```

The exact object fields can vary by model family and package version, so use these every time:

```r
class(pred_grid)
names(pred_grid)
class(target_grid)
names(target_grid)
```

---

## 7. How to Read the Outputs

Ask these questions before trusting a map:

1. Does the raw data show enough spatial coverage?
2. Are coordinates in a projected CRS with sensible distance units?
3. Do fitted covariate effects have plausible signs and sizes?
4. Is the spatial range believable for the disease process?
5. Are uncertainty intervals wide in places far from sampled points?
6. Do exceedance maps change substantially when you change model assumptions?

Useful outputs:

```r
summary(fit_prev)
coef(fit_prev)

summary(fit_risk_binomial)
coef(fit_risk_binomial)
```

Useful plots:

- raw prevalence points
- fitted mean prevalence surface
- lower and upper uncertainty surfaces
- exceedance probability surface
- residual or variogram checks, where available

---

## 8. Common Gotchas

- **Longitude/latitude is not always enough.** Many spatial models behave better with projected coordinates such as UTM, especially when distance matters.
- **Coordinates must match the CRS.** Do not label longitude/latitude as EPSG:3857 or UTM unless they really are.
- **Prediction covariates must match training covariates.** If the model used `elevation`, the prediction grid needs an `elevation` value for every prediction point.
- **MCMC settings used for learning are not final analysis settings.** Small `n.sim` or `n_sim` values are fine for debugging, not publication.
- **Starting values matter.** If optimization fails, try simpler models and better initial values.
- **Maps hide uncertainty.** Always inspect uncertainty or exceedance probability, not just the mean surface.

---

## 9. Practice Exercises

1. Run the `PrevMap` toy example and change the number of prediction grid points from `40` to `80`.
2. Add a second covariate to the simulated `PrevMap` data and refit the model.
3. Change the exceedance threshold from `0.2` to `0.4` and compare the map.
4. Run the `RiskMap` `italy_sim` example and inspect `summary()` and `coef()`.
5. Convert one toy data frame into an `sf` object with `st_as_sf()`.
6. Write a short paragraph explaining the difference between a prevalence map and an exceedance probability map.

---

## 10. Resources Worth Your Time

- `?binomial.logistic.MCML`
- `?spatial.pred.binomial.MCML`
- `?glgpm`
- `?pred_over_grid`
- `?pred_target_grid`
- Giorgi and Diggle, **Model-based Geostatistics for Global Public Health**
- `RiskMap` package site: <https://claudiofronterre.github.io/RiskMap/>
- MBG-R online book: <https://www.mbgr.org/>

---

*Run every block slowly. First make the toy examples work, then replace the toy data with your own survey data one column at a time.*
