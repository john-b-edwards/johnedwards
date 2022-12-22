---
date: "2022-12-21"
description: Getting better {tidymodels} performance with {stacks}
draft: false
keywords:
- machine_learning
math: true
slug: stacks
pagination: true
twitterImg: https://github.com/john-b-edwards/johnedwards/raw/main/public/images/stacks/social.png
tags:
- r
- tidymodels
title: Hacking by stacking—how to get better {tidymodels} performance with {stacks}
toc: true
header:
  caption: ''
  image: 'https://github.com/john-b-edwards/johnedwards/raw/main/public/images/stacks/social.png'
---

## Introduction

There is no more frustrating feeling than finishing just shy of a podium position in a Kaggle competition. Those precious competition points were right there! If only you had just a slightly better log loss! Alas, you exhausted every tool in your data science toolkit, and that was the best you could do. Unless... there was another way to get even better performance out of your models. Something so absurdly simple to implement that it felt almost like hacking the leaderboard. Enter the `{stacks}` package.

The `{stacks}` package is a part of the `{tidymodels}` ecosystem. It allows you to ensemble your models so quickly and so simply it can feel like cheating. Ensembling is a powerful machine learning technique that can boost model performance with additional resources, and it can help you squeeze a few extra points of log loss out of your predictions.

## How does ensembling work?

The typical data science workflow goes something like this:

![A flowchart showing a typical data science process. At the top is a box for "exploratory data analysis"", flowing down into a box saying "Generating and evaluating candidate models", which flows into several boxes of individual model candidates titled "Candidate A", "Candidate B", and so on with the log loss of each model listed. The candidate with the lowest log loss flows down further into boxes titled "Selected as best candidate" and "Predict on test set and submit" in order, whereas all other candidates flow into red boxes with the label "Rejected".](https://github.com/john-b-edwards/johnedwards/raw/main/public/images/stacks/typical_ds_workflow.png)

We generate several candidate models (perhaps the same model types but with different hyperparameters, or a combination of different model types and different hyperparameters) then select the model that performs the best on out-of-sample data, and use that model to generate and submit our test predictions.

This process is somewhat inefficient from a performance lens. We discard models that perform worse than the best performing candidate, but what if those models contain additional information that the best performing candidate does not? What we can do instead is take all of the information contained in our candidate models and then weight the predictions of individual candidates to maximize how much information we are extracting from our workflow.

![A flowchart showing the data science process for creating an ensemble. At the top is a box for "exploratory data analysis"", flowing down into a box saying "Generating and evaluating candidate models", which flows into several boxes of individual model candidates titled "Candidate A", "Candidate B", and so on with the log loss of each model listed. Each candidate box flows into a box showing the weight of each candidate (70% for the model with the lowest log loss, 20% for the second lowest, etc.). Each weight box flows into a single box that says "Candidates ensembled", which in turn flows into a box that says "Predict and submit on test set".](https://github.com/john-b-edwards/johnedwards/raw/main/public/images/stacks/ensemble_workflow.png)

Ensembling can yield slightly better results than the best performing individual model. It relies on the [wisdom of the crowd principle](https://en.wikipedia.org/wiki/The_Wisdom_of_Crowds) in that several informed models will make a better decision than the single-best performing model.

Ensembling can be tricky and daunting at first glance. How can you put all of these candidates together? With the `{stacks}` package, you can seamlessly integrate ensembling into your `{tidymodels}` workflow and get see better results readily!

## How does {stacks} ensemble models?

Imagine that you have a bunch of models that you have trained on the same dataset, where the only difference is that each model has been fed a different set of hyperparameters (so some models may be overfit, some may be underfit, some may be better at predicting outliers, and so on). How do you best combine those predictions to squeeze a little extra predictive power out of them? You have several obstacles in your way:

* **How do you handle the fact that your model predictions are likely to be highly correlated?** Even with the differences in hyperparameters, your models are all learning off the same set of data. If you fit a secondary model on top of your predictions, you are fitting a model on highly correlated features.
* **How do you know which candidates are predictive and which are overfit?** A model that has a training accuracy of 0.90 seems like a model that should be weighed more than a model with a training accuracy of 0.85, but if the first model is overfit and the second model is not, it can cause worse performance if you choose the first for your ensemble rather than the second.
* **How do you properly predict on all of these models to create your final predictions?** Say you make an ensemble with 10 models: those are 10 models that you have to store, load, predict, then ensemble those predictions together. That is a lot more complicated than just dealing with one model!

`{stacks}` helps us address all of these problems while working seamlessly within the tidymodels ecosystem.

### Highly correlated predictions

When dealing with highly correlated features in data science, a common approach is [LASSO](https://en.wikipedia.org/wiki/Lasso_(statistics)) regression, where the coefficient for a given feature is assigned a shrinkage parameter. When fitting the model, the coefficient for the feature will tend towards zero unless it provides significant predictive power. In a dataset with many highly correlated features, LASSO will reduce how many features are ultimately included in the model by setting the coefficients for many features to zero, allowing only a few remaining features to drive the predictive power of the model. `{stacks}` by default uses LASSO to reduce the number of highly correlated candidate models included in the ensemble.

### Telling apart overfit candidates from predictive candidates

Like training any other kind of model, when `{stacks}` trains an ensemble model, it checks on out-of-sample performance to ensure that the model is not being overfit. For this, `{stacks}` relies on [bootstrapping](https://en.wikipedia.org/wiki/Bootstrapping_(statistics)).  `{stacks}` will randomly sample rows from the training set, train off of predictions from those rows, then evaluate the ensemble on how it performed on the rows that were *not* sampled from the training dataset. This results in a more robust ensemble that filters out overfit models and leans more heavily on those that perform well out of sample.

### Managing multiple models easily

With `{stacks}`, the workflow to manage these models is absurdly simple. Simply make a few changes to your `{tidymodels}` workflow and you can easily integrate `{stacks}`. The wonderful thing is that a "stack" (what `{stacks}` calls an ensemble) looks and behaves just like any other model. How does this work? Let us dive into an example.

## How to ensemble with {stacks}

To demonstrate how easy it is to ensemble with `{stacks}`, we can walk through an example of a typical data science workflow with `{tidymodels}`, then show how with a few lines of code we can modify our code to allow for ensembling. For this example, we will use the [BoardGameGeek dataset](https://www.kaggle.com/competitions/sliced-s01e01/data) from the first episode of Sliced on Kaggle.

We can simply kitchen-sink our model for the purposes of this exercise. Our pre-processing will consist of the following steps:
* Deselect ID variables (`game_id`, `names`, `designer`)
* Parse our `category` variable into one-hot encoded variables
* Parse our `mechanic` variable into one-hot encoded variables

```r
# install.packages(c("tidyverse","tidymodels"))
library(tidyverse)
library(tidymodels)

train <-
  read_csv("train.csv",
           col_types = list(category11 = col_character(),
                            category12 = col_character()))

pre_proc <- recipe(geek_rating ~ ., data = train |>
                     select(-c(game_id, names, designer))) |>
  step_dummy_multi_choice(starts_with("category")) |>
  step_dummy_extract(mechanic, sep = ", ")

pre_proc |>
  prep() |>
  bake(new_data = NULL) |>
  head()
#> # A tibble: 6 x 150
#>   game_id names      min_p~1 max_p~2 avg_t~3 min_t~4 max_t~5  year num_v~6   age
#>     <dbl> <fct>        <dbl>   <dbl>   <dbl>   <dbl>   <dbl> <dbl>   <dbl> <dbl>
#> 1   17526 Hecatomb         2       4      30      30      30  2005     360    15
#> 2     156 Wildlife ~       2       6      60      60      60  1985     535    10
#> 3    2397 Backgammon       2       2      30      30      30 -3000    9684     8
#> 4    8147 Maka Bana        2       6      60      45      60  2003     658    10
#> 5   92190 Super Dun~       2       6     120     120     120  2011    2338    10
#> 6    1668 Modern Na~       2       6      90      90      90  1989     401    12
#> # ... with 140 more variables: owned <dbl>, designer <fct>, geek_rating <dbl>,
#> #   category1_Abstract.Strategy <int>, category1_Action...Dexterity <int>,
#> #   category1_Adventure <int>, category1_Age.of.Reason <int>,
#> #   category1_American.Civil.War <int>, category1_American.Indian.Wars <int>,
#> #   category1_American.Revolutionary.War <int>, category1_American.West <int>,
#> #   category1_Ancient <int>, category1_Animals <int>, category1_Arabian <int>,
#> #   category1_Aviation...Flight <int>, category1_Bluffing <int>, ...
```

We will use a simple repeated 5-fold cross validation to evaluate our parameters, then create a specification for an XGBoost regression model (we are employing a regression model because we want to optimize our root-mean squared error for the `geek_rating` feature). For the XGBoost model, we will tune it using a latin hypercube to grid search as much of the space covered by the potential values for XGBoost hyperparameters as possible.

```r
folds <- vfold_cv(train, v = 5, repeats = 3)

xgb_spec <- boost_tree(
  trees = tune(),
  tree_depth = tune(),
  min_n = tune(),
  loss_reduction = tune(),
  sample_size = tune(),
  mtry = tune(),
  learn_rate = tune()
) |>
  set_engine("xgboost") |>
  set_mode("regression")

xgb_grid <- grid_latin_hypercube(
  trees(),
  tree_depth(),
  min_n(),
  loss_reduction(),
  sample_size = sample_prop(),
  finalize(mtry(), train),
  learn_rate(),
  size = 100
)
```

We can initialize our workflow and begin to tune it.

```r
xgb_wf <- workflow() |>
  add_recipe(pre_proc) |>
  add_model(xgb_spec)

set.seed(123)
xgb_res <- tune_grid(
  xgb_wf,
  resamples = folds,
  grid = xgb_grid,
  control = control_grid(),
  metrics = metric_set(rmse)
)

```

We can find the model that performed the best for our target metric, select those parameters, then fit a final version of our workflow on those parameters. 

```r
best_rmse <- select_best(xgb_res, "rmse")

final_mod <- finalize_workflow(
  xgb_wf, 
  best_rmse
) |>
  fit(data = train)
```

We can predict on our test set and submit our values to Kaggle to see how we did.

```r
test <-
  readr::read_csv("test.csv",
                  col_types = list(category11 = col_character(),
                                   category12 = col_character()))

test$geek_rating <- predict(final_mod, test)$.pred

test |>
  select(game_id, geek_rating) |>
  write.csv("basic_submission.csv",row.names = F)
```

How did we do?

![A screenshot from Kaggle.com, showing the RMSE of our predictions: a score on the private leaderboard of 0.1617 and a score on the public leaderboard of 0.16119.](https://github.com/john-b-edwards/johnedwards/raw/main/public/images/stacks/standard_submission.png)

We did pretty solid! Our model performed well on both the public and private leaderboards. But we can do better. Let us adjust our code to show how to stack effectively with `{stacks}`

```r
# install.packages("stacks")
library(stacks)
```

We are going to pass in `stacks::control_stack_grid()` for the `control` argument in `tune_grid`.

```r
xgb_res <- tune_grid(
  xgb_wf,
  resamples = folds,
  grid = xgb_grid,
  control = control_stack_grid(),
  metrics = metric_set(rmse)
)
```

What does `stacks::control_stack_grid()` do differently than `tune::control_grid()`? Checking the internals makes it obvious:

```r
stacks::control_stack_grid
#> function () 
#> {
#>     tune::control_grid(save_pred = TRUE, save_workflow = TRUE)
#> }
#> <bytecode: 0x00000000271c5320>
#> <environment: namespace:stacks>
```

When we generate candidate models, by default, `tune::control_grid()` does not save the predictions generated by the candidate model to be more efficient with memory. Because `{stacks}` fits an ensemble model using the predictions of each candidate model as features, `{stacks}` requires that we save the workflow and predictions for each candidate model. Note also that because `stacks::control_stack_grid` is just a wrapper for `tune::control_grid`, instead of `control_stack_grid()`, you can simply pass in `tune::control_grid(save_pred = T, save_workflow = T)` to the `control` argument along with any other additional arguments you may want for tuning (such as `verbose = T` to track the progress of tuning).

With our candidates generated, we can now build our stack. We do this by initializing a stack with the `stacks()` function, then pipe in our candidate models using `add_candidates()`.

```r
xgb_stack <- stacks() |>
  add_candidates(xgb_res)
```

We fit our ensemble with the `blend_predictions()` function.

```r
xgb_stack <- xgb_stack |>
  blend_predictions()
```

What is `blend_predictions()` doing? As mentioned before, `{stacks}` fits a LASSO model on the predictions of candidate models to determine what candidates to include in the model. This step happens when we call `blend_predictions()`.

What does our ensemble look like?

```r
xgb_stack
#> -- A stacked ensemble model -------------------------------------
#> 
#> Out of 83 possible candidate members, the ensemble retained 4.
#> Penalty: 1e-06.
#> Mixture: 1.
#> 
#> The 4 highest weighted members are:
#> # A tibble: 4 x 3
#>   member        type          weight
#>   <chr>         <chr>          <dbl>
#> 1 xgb_res_1_085 boost_tree 0.785    
#> 2 xgb_res_1_067 boost_tree 0.116    
#> 3 xgb_res_1_019 boost_tree 0.106    
#> 4 xgb_res_1_098 boost_tree 0.0000350
#> 
#> Members have not yet been fitted with `fit_members()`.
```

We can see that our ensemble retained four models. One of the models is weighted very heavily in our ensemble (with a weight of 0.785), but other models also have significant influence on the ultimate results.

With the coefficients decided for our stack, we can execute a final fit on our model using the `fit_members()` function, using only the candidates that our model selected (there is no need to fit a model using candidates whose coefficient we know will be zero).

```r
xgb_stack <- xgb_stack |>
  fit_members()
```

Our stack is freshly served, with a pat of butter on top! We can now predict on our test set using the new stack and submit our predictions.


```r
test$geek_rating <- predict(xgb_stack, test)$.pred

test |>
  select(game_id, geek_rating) |>
  write.csv("stacked_submission.csv",row.names = F)
```

How did we do?

![A screenshot from Kaggle.com, showing the RMSE of our prediction from our stacked predictions and our basic predictions. For the stacked predictions, the private score is 0.15816, and the public score is 0.13775. For the basic predictions, the private score is 0.1617 and the public score is 0.16119.](https://github.com/john-b-edwards/johnedwards/raw/main/public/images/stacks/stacked_submission.png)

By stacking our candidate models instead of discarding all but the single best-performing model, our model performs significantly better on both the private and public leaderboards! Look at how little extra code we had to write. This was the extent of the changes we made to our original workflow:
* We loaded in `{stacks}`
* We saved predictions and workflows when tuning
* We added candidates to a stack and fit an ensemble
* We predicted on the ensemble instead of a single model

With minimal effort, we were able to signficantly boost our model performance. That is the beauty of `{stacks}`!

## Extending this example

`{stacks}` has a fair bit more functionality than demonstrated in the above example. We can do a lot of different things with it! For example...

### Incorporating different *kinds* of models

We demonstrated how to ensemble using candidate XGBoost models. If we wanted to incorporate different model types, such as random forest or a support vector machine, all we would need to do is generate candidate models for them as well (in a similar manner to how we generated multiple candidate models for XGBoost using `tune_grid()`) then bake them in using `add_candidates()`. You can use largely any model type that is available in `{parsnip}` for this! 

```r
full_stack <- stacks() |>
  add_candidates(xgb_res) |>
  add_candidates(rf_ref) |>
  add_candidates(svm_ref)
```

### Sing a different tune

Sometimes you may wish to generate candidates for your models not by grid searching, but through a [Gaussian process](https://towardsdatascience.com/gaussian-processes-smarter-tuning-for-your-ml-models-c72c7d4f5833). You can do this by using `tune_bayes()` functionality in `{tidymodels}`, and either pass in `stacks::control_stack_bayes()` or just ensure that `tune::control_bayes()` has `save_pred` and `save_workflow` set to true.

```r
res <- wf |>
  tune_bayes(
    resamples = folds,
    param_info = params,
    initial = 10,
    iter = 100,
    metrics = metrics,
    control = control_bayes(
      no_improve = 10,
      verbose = F,
      time_limit = 240,
      save_pred = T, # set this to true!
      save_workflow = T # set this to true too!
    )
  )
```

### Tuning your ensemble

Sometimes you may wish to control *how* the ensemble creates predictions. The `mixture` argument in `blend_predictions()` is set to 1 (for LASSO) by default, but you can also pass in `mixture = 0` to fit a ridge regression, pass in a value between 0 and 1 for an elastic fit, or even tune the mixture penalty by passing in a range of values to search over (i.e. `mixture = seq(0,1,by=0.1)`). The penalty parameter is tuned by default in `blend_predictions()`, but you can pass in your own set of penalty values to tune over as well. You can also change the number of boostrap samples used by `blend_predictions()` with the `times` parameter.

```r
stacks() |>
  add_candidates(xgb_res) |>
  blend_predictions(
    penalty = 10^(-9:-1), # tune over these penalty values
    mixture = 0, # do a ridge regression instead of a LASSO regression
    times = 50 # bootstrap 50 samples from the train set
  )
```

### Treating your ensemble like a tidymodel

Since `blend_predictions()` and `fit_members()` are essentially wrappers over a penalized linear regression within `{tidymodels}`, you can speed up how long it takes to run `blend_predictions()` or `fit_members()` by running in parallel using `doParallel::registerDoParallel()`. You can also control how these functions grid search for model parameters using `tune::control_grid()`.

```r
# fit an ensemble in parallel
doParallel::registerDoParallel()
stacks() |>
  add_candidates(xgb_res) |>
  blend_predictions() |>
  fit_members() |>
doParallel::stopImplicitCluster()

# print out results of fitting each ensemble
stacks() |>
  add_candidates(xgb_res) |>
  blend_predictions(control = tune_grid(verbose = T)) |>
  fit_members(control = tune_grid(verbose = T))
```

## When *not* to ensemble

`{stacks}` seems so simple and arbitrary to implement (something I find myself saying about much of the `{tidymodels}` ecosystem, which is a credit to its developers) that it is fair to ask: why *wouldn't* you use `{stacks}`?

* **You value interpretability.** In some Kaggle competitions, the difference of a few points of log loss can be the difference between thousands of dollars or none at all. For a competition like that, improving your model's performance even marginally can be extremely impactful. In the real world, the impact of a machine learning model is more often measured by buy-in than by RMSE. It is difficult enough to explain on its own how XGBoost works and the impact of individual features, and it is even more difficult to explain the impact of an individual feature across multiple models in an ensemble. For circumstances where you are looking to learn about a system by exploring feature importance or when you are rolling out a new model in a situation where non-technical audiences need to understand how you are generating predictions, you may opt not to ensemble to preserve explainability and interpretability.

* **You have limited computing resources.** Tuning is computationally expensive by itself. `{tidymodels}` by default claws back some of these resources by not saving the predictions and workflows for each candidate model, but in order for `{stacks}` to function we need to store those in memory. We also have to fit the ensemble model on top of the existing models. Depending on the size of your data, the number of candidate models you train, or the amount of computing resources you have at your disposal, this process can run very slowly or even become practically infeasible.

* **You do not have a large amount of data to work with.** Much like random forests and gradient boosted trees, going with an ensemble when you do not have a lot of data to work with is generally overkill. 

## Wrapping up

That is `{stacks}` in a nutshell! I appreciate how easily this package integrates with my workflow and how reliable it is for getting a little bit of a boost on Kaggle leaderboards. Credit goes to Simon Couch, Max Kuhn, and the wonderful folks at Posit (previously RStudio) for developing `{stacks}`, with additional credit to the `{tidymodels}` developer team for their work in creating the framework that `{stacks}` operates in. You can read more about using `{stacks}` and `{tidymodels}` below. Happy tuning!

* [{stacks} documentation](https://stacks.tidymodels.org/index.html)
* [{tidymodels} documentation](https://www.tidymodels.org/)
* [posit website](https://posit.co/)