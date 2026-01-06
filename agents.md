# R Coding Guidelines


## How to Respond


- Provide complete, runnable code blocks (not fragments)
- Explain your reasoning BEFORE the code, not after
- When modifying existing code, show the full function/section, not just changed lines
- If uncertain about requirements, ask ONE clarifying question before proceeding
- Match the existing code style in the file


---


## PRIMARY DIRECTIVE: Keep Each Step Simple


**Pipes are fine. Complexity within steps is not.**


Each step in a pipeline should do ONE simple thing. Don't write steps that transform many variables at once or contain complex nested logic.


### ✅ GOOD: Simple pipes with one operation per step


```r
result <- data |>
  filter(x > 0) |>
  mutate(y = log(x)) |>
  group_by(group) |>
  summarise(mean_y = mean(y), .groups = "drop")
```


### ❌ BAD: Complex transformations crammed into single steps


```r
result <- data |>
  mutate(
    x_clean = ifelse(x < 0, NA, x),
    x_log = log(x_clean + 1),
    x_scaled = (x_log - mean(x_log, na.rm = TRUE)) / sd(x_log, na.rm = TRUE),
    y_clean = case_when(
      y < quantile(y, 0.01) ~ quantile(y, 0.01),
      y > quantile(y, 0.99) ~ quantile(y, 0.99),
      TRUE ~ y
    ),
    y_log = log(y_clean),
    z_factor = factor(z, levels = c("low", "med", "high")),
    outcome = x_scaled * y_log + as.numeric(z_factor)
  )
```


### ✅ GOOD: Break complex transformations into readable steps


```r
# Clean x
data_clean <- data |>
  mutate(x_clean = ifelse(x < 0, NA, x),
         x_log = log(x_clean + 1))


# Scale x
x_mean <- mean(data_clean$x_log, na.rm = TRUE)
x_sd <- sd(data_clean$x_log, na.rm = TRUE)
data_scaled <- data_clean |>
  mutate(x_scaled = (x_log - x_mean) / x_sd)


# Winsorize y
y_bounds <- quantile(data_scaled$y, c(0.01, 0.99))
data_winsor <- data_scaled |>
  mutate(y_clean = pmax(pmin(y, y_bounds[2]), y_bounds[1]),
         y_log = log(y_clean))


# Create final outcome
result <- data_winsor |>
  mutate(z_factor = factor(z, levels = c("low", "med", "high")),
         outcome = x_scaled * y_log + as.numeric(z_factor))
```


### When to add verification output


Add `cat()`, `glimpse()`, or `summary()` after steps that:
- Filter rows (check how many remain)
- Create variables that could produce NA/Inf (check for problems)
- Merge datasets (check row counts)
- Involve complex logic (spot-check results)


---


## SECOND DIRECTIVE: Colorblind-Friendly Visualization


**ALL visualizations MUST use colorblind-friendly color palettes.**


### Required palettes


```r
# Discrete/categorical (ALWAYS use one of these)
scale_fill_viridis_d(option = "viridis")   # Preferred
scale_color_viridis_d(option = "viridis")  # Preferred
scale_fill_brewer(palette = "Set2")        # Alternative


# Continuous (ALWAYS use one of these)
scale_fill_viridis_c(option = "viridis")   # Preferred
scale_color_viridis_c(option = "viridis")  # Preferred
```


### Never use


- Default ggplot2 colors
- Rainbow palettes
- Red/green combinations


---


## Code Style


- Use tidyverse packages for all data manipulation
- Use `<-` for assignment (not `=`)
- Use `snake_case` for variable names
- Keep lines under 80 characters
- Use `here::here()` for file paths (never `setwd()`)
- Always `set.seed()` for reproducibility


---


## Project Structure


```
project/
├── R/              # Function definitions
├── data/raw/       # Never modify files here
├── data/processed/ # Cleaned data goes here
├── output/         # Results, tables, figures
├── scripts/        # Analysis scripts (numbered: 01_clean.R, 02_analyze.R)
└── tests/          # testthat tests
```


- Use `here::here()` for all paths
- Scripts should be self-contained and runnable from project root


---


## Script Structure


```r
# Load packages ----
library(tidyverse)
library(here)


set.seed(12345)


# Load data ----
data <- read_csv(here("data", "raw", "datafile.csv"))


# Inspect ----
glimpse(data)
summary(data)


# Analysis (step-by-step with verification) ----
# ...


# Session info ----
cat("\n=== Session Info ===\n")
sessionInfo()
```


---


## Documentation


- Use roxygen2 for all functions
- Required tags: `@param`, `@return`, `@examples`
- Include at least one runnable example


```r
#' Calculate group means with robust SEs
#'
#' @param data A data frame containing the variables
#' @param outcome Character. Name of the outcome variable
#' @param group Character. Name of the grouping variable
#'
#' @return A tibble with group means and standard errors
#'
#' @examples
#' calc_group_means(mtcars, "mpg", "cyl")
calc_group_means <- function(data, outcome, group) {
  # ...
}
```


---


## Testing


- Write tests for any function with more than 5 lines
- Use `testthat` framework
- Include edge cases: empty inputs, NA values, single-row data


```r
test_that("function handles empty data gracefully", {
  expect_error(my_function(data.frame()), "Data must have")
})


test_that("function handles NA values", {
  result <- my_function(data_with_nas)
  expect_false(any(is.na(result$estimate)))
})
```


---


## Error Handling


- Fail fast with informative messages
- Validate inputs at function start
- Error messages should say what was expected AND what was received
- Use `tryCatch()` for external operations (file I/O, API calls)


```r
my_function <- function(data, var) {
  # Input validation first
  stopifnot(
    "data must be a data frame" = is.data.frame(data),
    "data cannot be empty" = nrow(data) > 0,
    "var must be in data" = var %in% names(data)
  )
  
  # For external operations
  result <- tryCatch(
    read_csv(here("data", "file.csv")),
    error = function(e) {
      stop("Failed to read data file: ", e$message)
    }
  )
}
```


---


## Statistical Analysis


- Check assumptions before running models
- Use `broom::tidy()` for clean model output
- Use `estimatr::lm_robust()` for robust standard errors
- Report confidence intervals alongside point estimates
- Use `modelsummary` for publication-ready tables


---


## Causal Inference Workflow


When estimating treatment effects:


1. Show balance table before/after matching
2. Report both ATE and ATT when relevant
3. Include sensitivity analysis (e.g., Rosenbaum bounds)
4. Always cluster standard errors at treatment assignment level
5. Use `fixest` for high-dimensional fixed effects


---


## Packages


### Essential
- `tidyverse`, `here`, `broom`, `modelsummary`


### Statistical
- `estimatr`, `marginaleffects`, `fixest`, `sandwich`, `lmtest`


### Visualization
- `viridis` (required), `patchwork`, `ggeffects`


### Political Science
- `survey` (survey data), `plm` (panel data), `MatchIt` (matching)
- `did` (diff-in-diff), `sf`/`spdep` (spatial)


### Dependency rules
- Minimize dependencies—prefer base R or tidyverse over niche packages
- If suggesting a new package, note installation: `# install.packages("pkg")`
- Don't use `library()` inside functions—use `::` notation


---


## When Modifying My Code


- Preserve my variable naming conventions
- Don't refactor unrelated code unless asked
- If you see a bug unrelated to my question, mention it but don't fix it silently
- Match the existing indentation and spacing style


---


## Never Do These


- Don't use `attach()` – causes namespace conflicts
- Don't use `T`/`F` – use `TRUE`/`FALSE`
- Don't use `=` for assignment – use `<-`
- Don't use `1:length(x)` – use `seq_along(x)`
- Don't use `sapply()` – use `vapply()` or `map_*()` for type safety
- Don't use `subset()` in functions – use `filter()`
- Don't suppress warnings without documenting why
- Don't hardcode file paths
- Don't leave `print()` statements in production functions
- Don't forget to set seeds
- **Never cram complex multi-variable transformations into a single step**
- **Never use default color palettes**


---


## Reproducibility


Every analysis script must end with session info:


```r
cat("\n=== Session Info ===\n")
sessionInfo()
```


For manuscripts or critical analyses:


```r
# Lock package versions
renv::snapshot()
```


---


## Output Checklist


- [ ] Each pipe step does ONE simple thing
- [ ] Complex transformations broken into separate steps
- [ ] Verification output after risky operations (filters, merges, NA-prone transforms)
- [ ] Tidyverse functions used
- [ ] Colorblind-friendly palettes for all plots
- [ ] Package loading at top
- [ ] Seed set for reproducibility
- [ ] Steps clearly numbered
- [ ] Functions documented with roxygen2
- [ ] Tests written for custom functions
- [ ] Session info at end of script
