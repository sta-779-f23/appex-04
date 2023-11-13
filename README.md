# Continuous exposures and g-computation

``` r
library(tidyverse)
library(broom)
library(touringplans)
library(splines)

seven_dwarfs <- seven_dwarfs_train_2018 |>
  filter(wait_hour == 9)
```

For this set of exercises, we’ll use g-computation to calculate a causal
effect for continuous exposures.

In the touringplans data set, we have information about the posted
waiting times for rides. We also have a limited amount of data on the
observed, actual times. The question that we will consider is this: Do
posted wait times (`wait_minutes_posted_avg`) for the Seven Dwarves Mine
Train at 8 am affect actual wait times (`wait_minutes_actual_avg`) at 9
am? Here’s our DAG:

First, let’s wrangle our data to address our question: do posted wait
times at 8 affect actual weight times at 9? We’ll join the baseline data
(all covariates and posted wait time at 8) with the outcome (average
actual time). We also have a lot of missingness for
`wait_minutes_actual_avg`, so we’ll drop unobserved values for now.

You don’t need to update any code here, so just run this.

``` r
eight <- seven_dwarfs_train_2018 |>
  filter(wait_hour == 8) |>
  select(-wait_minutes_actual_avg)

nine <- seven_dwarfs_train_2018 |>
  filter(wait_hour == 9) |>
  select(park_date, wait_minutes_actual_avg)

wait_times <- eight |>
  left_join(nine, by = "park_date") |>
  drop_na(wait_minutes_actual_avg)
```

# Your Turn 1

For the parametric G-formula, we’ll use a single model to fit a causal
model of Posted Waiting Times (`wait_minutes_posted_avg`) on Actual
Waiting Times (`wait_minutes_actual_avg`) where we include all
covariates, much as we normally fit regression models. However, instead
of interpreting the coefficients, we’ll calculate the estimate by
predicting on cloned data sets.

Two additional differences in our model: we’ll use a natural cubic
spline on the exposure, `wait_minutes_posted_avg`, using `ns()` from the
splines package, and we’ll include an interaction term between
`wait_minutes_posted_avg` and `park_extra_magic_morning`. These
complicate the interpretation of the coefficient of the model in normal
regression but have virtually no downside (as long as we have a
reasonable sample size) in g-computation, because we still get an easily
interpretable result.

First, let’s fit the model.

1.Use `lm()` to create a model with the outcome, exposure, and
confounders identified in the DAG. 2. Save the model as
`standardized_model`

``` r
_______ ___ _______(
  wait_minutes_actual_avg ~ ns(_______, df = 5)*park_extra_magic_morning + _______ + _______ + _______, 
  data = seven_dwarfs
)
```

# Your Turn 2

Now that we’ve fit a model, we need to clone our data set. To do this,
we’ll simply mutate it so that in one set, all participants have
`wait_minutes_posted_avg` set to 30 minutes and in another, all
participants have `wait_minutes_posted_avg` set to 60 minutes.

1.  Create the cloned data sets, called `thirty` and `sixty`.
2.  For both data sets, use `standardized_model` and `augment()` to get
    the predicted values. Use the `newdata` argument in `augment()` with
    the relevant cloned data set. Then, select only the fitted value.
    Rename `.fitted` to either `thirty_posted_minutes` or
    `sixty_posted_minutes` (use the pattern
    `select(new_name = old_name)`).
3.  Save the predicted data sets as`predicted_thirty` and
    `predicted_sixty`.

``` r
_______ <- seven_dwarfs |>
  _______

_______ <- seven_dwarfs |>
  _______

predicted_thirty <- standardized_model |>
  _______(newdata = _______) |>
  _______

predicted_sixty <- standardized_model |>
  _______(newdata = _______) |>
  _______
```

# Your Turn 3

Finally, we’ll get the mean differences between the values.

1.  Bind `predicted_thirty` and `predicted_sixty` using `bind_cols()`
2.  Summarize the predicted values to create three new variables:
    `mean_thirty`, `mean_sixty`, and `difference`. The first two should
    be the means of `thirty_posted_minutes` and `sixty_posted_minutes`.
    `difference` should be `mean_sixty` minus `mean_thirty`.

``` r
_______ |>
  _______(
    mean_thirty = _______,
    mean_sixty = _______,
    difference = _______
  )
```

That’s it! `difference` is our effect estimate, marginalized over the
spline terms, interaction effects, and confounders.

## Stretch goal: Boostrapped intervals

Like propensity-based models, we need to do a little more work to get
correct standard errors and confidence intervals. In this stretch goal,
use rsample to bootstrap the estimates we got from the G-computation
model.

Remember, you need to bootstrap the entire modeling process, including
the regression model, cloning the data sets, and calculating the
effects.

``` r
set.seed(1234)
library(rsample)

fit_gcomp <- function(split, ...) { 
  .df <- analysis(split) 
  
  # fit outcome model. remember to model using `.df` instead of `seven_dwarfs`
  
  
  # clone datasets. remember to clone `.df` instead of `seven_dwarfs`
  
  
  # predict actual wait time for each cloned dataset
  
  
  # calculate ATE
  bind_cols(predicted_thirty, predicted_sixty) |>
    summarize(
      mean_thirty = mean(thirty_posted_minutes),
      mean_sixty = mean(sixty_posted_minutes),
      difference = mean_sixty - mean_thirty
    ) |>
    # rsample expects a `term` and `estimate` column
    pivot_longer(everything(), names_to = "term", values_to = "estimate")
}

gcomp_results <- bootstraps(seven_dwarfs, 1000, apparent = TRUE) |>
  mutate(results = map(splits, ______))

# using bias-corrected confidence intervals
boot_estimate <- int_bca(_______, results, .fn = fit_gcomp)

boot_estimate
```

------------------------------------------------------------------------

# Take aways

- To fit the parametric G-formula, fit a standardized model with all
  covariates. Then, use cloned data sets with values set to each level
  of the exposure you want to study.
- Use the model to predict the values for that level of the exposure and
  compute the effect estimate you want
