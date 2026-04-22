# SOLID in R

R is functional-first with dispatch via S3/S4/R6. Apply SOLID at the *package* boundary — internal analysis scripts are often fine without heavy OO.

## S — Single Responsibility

```r
# Violation: one function loads, cleans, fits, and plots.
analyze <- function(path) {
  d <- read.csv(path)
  d <- d[complete.cases(d), ]
  m <- lm(y ~ x, d)
  plot(m); m
}

# Fix: one verb per function. Pipe at the call site.
load_data   <- function(path) read.csv(path)
clean_data  <- function(d)    d[complete.cases(d), ]
fit_model   <- function(d)    lm(y ~ x, d)
plot_model  <- function(m)    plot(m)

m <- load_data("f.csv") |> clean_data() |> fit_model()
plot_model(m)
```

## O — Open/Closed

```r
# Violation: every new metric edits the switch.
score <- function(kind, x, y) {
  if (kind == "rmse") sqrt(mean((x - y)^2))
  else if (kind == "mae") mean(abs(x - y))
}

# Fix: S3 dispatch — new metrics register a method without editing `score`.
score <- function(metric, x, y) UseMethod("score")
score.rmse <- function(metric, x, y) sqrt(mean((x - y)^2))
score.mae  <- function(metric, x, y) mean(abs(x - y))

score(structure(list(), class = "rmse"), x, y)
```

## L — Liskov Substitution

```r
# Violation: a "RobustModel" subclass silently ignores weights its parent accepts.

# Fix: if a subclass cannot honour the parent's contract, it is not a subclass.
# Prefer composition:
fit <- function(data, strategy) strategy(data)        # strategy is a function
fit(data, strategy = fit_ols)
fit(data, strategy = fit_robust)                      # no inheritance lie
```

## I — Interface Segregation

```r
# Violation: a "model" object is expected to implement predict(), summary(),
# plot(), tidy(), glance(), augment() — consumers that only need predict()
# still pay for the whole surface.

# Fix: small, role-shaped generics. A consumer calls only the generic it needs.
predict_only <- function(model, newdata) predict(model, newdata)
# Callers that never call summary() never require it to exist.
```

## D — Dependency Inversion

```r
# Violation: analysis function reaches out to the database and the system clock.
run_report <- function(date_col) {
  con <- DBI::dbConnect(RPostgres::Postgres())
  rows <- DBI::dbGetQuery(con, "SELECT ...")
  rows[rows[[date_col]] < Sys.Date(), ]
}

# Fix: inject the data source and the clock.
run_report <- function(fetch_rows, today = Sys.Date, date_col) {
  rows <- fetch_rows()
  rows[rows[[date_col]] < today(), ]
}

# main.R — wire real dependencies here.
con <- DBI::dbConnect(RPostgres::Postgres())
run_report(
  fetch_rows = function() DBI::dbGetQuery(con, "SELECT ..."),
  date_col   = "created_at"
)
```

Notes for R:
- Prefer plain functions and S3 dispatch. Reach for R6/S4 only when you have real
  mutable state or multiple dispatch.
- `options()` and package-level state are hidden dependencies — pass them as
  arguments when the function is part of a public API.
