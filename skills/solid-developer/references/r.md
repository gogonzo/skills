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

The R framing: **do not demand a fat object when you only call one method on it.**
Accept the narrowest thing the function actually uses — usually a function, a
data frame, or a vector — not a whole model/fit/connection object.

```r
# Violation: evaluate() takes a full model object, but only ever calls predict().
# Any caller must now construct something with a `predict` method just to test
# a scoring rule — mocks in tests have to fake the entire model surface.
evaluate <- function(model, data) {
  checkmate::assert_data_frame(data)
  pred <- predict(model, data)        # <- the only thing we use `model` for
  mean((pred - data$y)^2)
}

# Fix: depend on the narrow capability. Callers pass a prediction function;
# tests pass `function(d) d$x * 2` with no model object at all.
evaluate <- function(predict_fn, data) {
  checkmate::assert_function(predict_fn)
  checkmate::assert_data_frame(data)
  checkmate::assert_subset("y", names(data))
  mean((predict_fn(data) - data$y)^2)
}

# Real use:
evaluate(\(d) predict(fit, d), data)

# Test use — no model needed:
evaluate(\(d) d$x, data.frame(x = 1:3, y = c(1, 2, 4)))
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

## Assertions — non-negotiable

Every exported function validates its inputs on entry. R is dynamically typed;
without assertions, a wrong-shape argument becomes a confusing error ten frames
deep, and DIP-style injection ("pass me a function") is unsafe without a shape
check. Prefer `checkmate` for expressive, fast assertions; `stopifnot()` is fine
for trivial cases.

```r
fit_model <- function(data, formula, weights = NULL) {
  checkmate::assert_data_frame(data, min.rows = 1)
  checkmate::assert_formula(formula)
  checkmate::assert_numeric(weights, len = nrow(data), null.ok = TRUE)
  # ... body can now trust its inputs
}
```

Rules:
- Assert at the public boundary of a function, not in every internal helper.
- Assert *shape and invariant*, not just type (`min.rows`, `len`, `any.missing = FALSE`).
- Post-conditions on the return value of risky operations (`assert_data_frame(out)`).
- In tests, still assert — a test that silently passes `NULL` somewhere is a lie.

## Picking the object system

- **Plain functions + S3** — the default. Use for almost everything. S3 dispatch
  satisfies OCP without ceremony (see the `score()` example above).
- **R6** — use when you genuinely need **mutable state** (a cache, a growing
  buffer, a connection pool) or a **singleton** (a logger, a configured client).
  R6 objects are reference-semantic, which is a tool — and a footgun — so reach
  for it only when copy-on-modify semantics are actually wrong for the problem.
- **S4** — use when you need **multiple dispatch** (method chosen on more than
  one argument's class) or **strong typing** with validity checks
  (`setValidity()`). Bioconductor-style packages are the typical home. Do not
  use S4 just for "proper classes" — the overhead is real.
- **Do not mix** object systems for the same concept. Pick one per type and
  stay consistent.

## Encapsulation — minimise the public surface

- The `NAMESPACE` file (generated from roxygen `@export`) *is* your public
  API. Anything without `@export` is private — that is the correct default.
  Add `@export` only when an external user genuinely needs to call the symbol.
- Internal helpers live in the package but are not exported. Users who reach
  for them via `pkg:::helper` are on their own; do not treat `:::` access as
  a supported contract.
- For R6 classes, use `private = list(...)` for fields and methods that
  shouldn't be reachable from outside. `active` bindings are for controlled
  read-only exposure.
- For S4, mark slots you don't want inspected as internal by convention
  (leading dot, e.g. `.cache`) and do not document them.
- `@keywords internal` + omitting `@export` hides helpers from the package
  index while still giving you testable, documented functions.
- Never `@export` a helper just so tests can reach it. Tests in a package can
  call internal functions directly — exporting for test convenience leaks
  internals to every user forever.

## Other R-specific notes

- `options()`, environment variables, and package-level state are hidden
  dependencies. For public APIs, pass them as arguments (DIP).
- `library()` inside a function is a side effect — never do it; declare
  dependencies in `DESCRIPTION` and call `pkg::fn()`.
- Prefer `vapply()` over `sapply()` — you assert the return shape at the call
  site, which is an assertion in disguise.
