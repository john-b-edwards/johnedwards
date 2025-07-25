---
title: "cbbreadr: a new package for college basketball data"
description: Loading college basketball data into R rapidly using cbbreadr
date: "2025-07-25"
keywords: 
- college_basketball
- basketball
slug: cbbreadr_launch
tags: 
- r
draft: false
toc: true
bsky_thread: https://bsky.app/profile/johnbedwards.io/post/3lus7es3ro22w
---

![](https://github.com/john-b-edwards/cbbd-data/raw/master/data/outputs/cbbreadr_hex.png?raw=true)

I'm pleased to announce my new college basketball R package, `{cbbreadr}`, has been approved and is now available on CRAN! You can now install with `install.packages()` from base R like so:

```r
install.packages("cbbreadr")
```

If you're too hip, cool, and trendy for CRAN and would prefer the development version, you can install the most up-to-date version of `{cbbreadr}` from Github like so:

```r
devtools::install_github("john-b-edwards/cbbreadr")
```

## What is `{cbbreadr}`?

Longtime social media followers of mine may know that I am both [a big college basketball analytics fan](https://johnbedwards.io/blog/march_madness_2025_round_one/) and a [member of the nflverse team](https://github.com/nflverse). Within the nflverse, our major contribution to NFL analytics has been building data pipelines to quickly and efficiently deliver parsed NFL play-by-play and other data sources via a variety of software packages. As you might be able to tell from the name of the package, `{cbbreadr}` is a version of the `{nflreadr}` package that, instead of loading NFL data, loads up-to-date college basketball data.

## Where do you get your data from?

The data available in the  `{cbbreadr}` package comes from the wonderful [CollegeBasketballData.com](https://collegebasketballdata.com/) API, maintained by Bill Radjewski (of [CollegeFootballData.com](https://collegefootballdata.com/) fame). Bill scrapes and parses his CBB data from multiple sources, then cleans it and makes it available via several API endpoints. I have built several pipelines that grab this data, turn it into rectangular data when needed, and then makes this data available in R via `cbbreadr::load_*` where `*` is the specified data resource. I do some additional cleaning and renaming to make this data easier to work with.

## How frequently is the data in `{cbbreadr}` updated?

Most of the data in `{cbbreadr}` is updated on a daily basis. For example, overnight overnight, the play-by-play data from the previous day's games becomes available in the API, so I grab it, parse it, and make it available via `load_plays()`. Some data does not to be updated on a daily basis, however--for example, I only update the data available via `load_conferences()` yearly. To see when a resource was most recently updated, or the status of the pipelines, see the [cbb-data status page](https://github.com/john-b-edwards/cbbd-data).

## What can you do with `{cbbreadr}`?

In theory, a great many things! The availability of substitution, shot-charting, and play-by-play data over many seasons allows for a lot of different projects that can be done with this data.

As a quick example, we can use `load_plays()` to plot Cooper Flagg's shot chart from the 2024-2025 season:

```{r}
# load plays (this will take a few seconds)
cbbreadr::load_plays(2025) |>
  # filter to shots that Cooper Flagg took & we have shot tracking data
  dplyr::filter(
    shooter_athlete_name == "Cooper Flagg" &
      !is.na(shot_x_coordinate) &
      !is.na(shot_y_coordinate)
  ) |>
  # scale coordinates to be in feet and flip cross-court coordinates so we're all on one court
  dplyr::mutate(
    x = (shot_x_coordinate / 10),
    x = ifelse(x > 47, 94 - x, x),
    y = (shot_y_coordinate / 10),
    y = ifelse(shot_x_coordinate / 10 > 47, 50 - y, y)
  ) |>
  ggplot2::ggplot(ggplot2::aes(x = x, y = y)) +
  # hardwood court
  ggplot2::geom_rect(
    data = data.frame(x = 0, y = 0),
    ggplot2::aes(xmin = 0, xmax = 47, ymin = 0, ymax = 50),
    fill = "#dfbb85"
  ) +
  # hexbins for Flagg's shots
  ggplot2::geom_hex(bins = 20) +
  # three point line
  ggforce::geom_arc(
    data = data.frame(x = c(0), y = c(0)),
    ggplot2::aes(x0 = 5.25, y0 = 25, r = 20.75, start = 0, end = pi),
    color = "white",
    inherit.aes = FALSE
  ) +
  ggplot2::geom_path(
    data = data.frame(x = c(0, 5.25), y = c(25 + 20.75, 25 + 20.75)),
    color = "white"
  ) +
  ggplot2::geom_path(
    data = data.frame(x = c(0, 5.25), y = c(25 - 20.75, 25 - 20.75)),
    color = "white"
  ) +
  # free throw line
  ggplot2::geom_path(
    data = data.frame(
      x = c(0, 19, 19, 0),
      y = c(25 - 6, 25 - 6, 25 + 6, 25 + 6)
    ),
    color = "white"
  ) +
  ggforce::geom_arc(
    data = data.frame(x = c(0), y = c(0)),
    ggplot2::aes(x0 = 19, y0 = 25, r = 6, start = 0, end = pi),
    color = "white",
    inherit.aes = FALSE
  ) +
  # interior paint
  ggforce::geom_arc(
    data = data.frame(x = c(0), y = c(0)),
    ggplot2::aes(x0 = 4, y0 = 25, r = 4, start = 0, end = pi),
    color = "white",
    inherit.aes = FALSE
  ) +
  ggplot2::geom_path(
    data = data.frame(x = c(4, 4), y = c(21, 29)),
    color = "white"
  ) +
  # remove all other theme elements
  ggplot2::theme_void() +
  # scale gradient to a heatmap
  ggplot2::scale_fill_gradient(low = "#FFFFFF", high = "#FF0000") +
  # force limits to show exactly half the court
  ggplot2::ylim(c(0, 50)) +
  ggplot2::xlim(c(0, 47)) +
  # add labels
  ggplot2::labs(
    title = "Cooper Flagg Shot Chart, 2024-2025",
    subtitle = "From games where shot data was available",
    caption = "Source: cbbreadr",
    fill = "Shots"
  )
```

![sample cbbreadr plot](https://github.com/john-b-edwards/johnedwards/blob/main/public/images/cbbreadr/sample_plot.png?raw=true)

There are a great many more things you can do with this package; I hope to demonstrate some of them for you later, but I also hope others can use this package to build tutorials and do college basketball research as well. If you encounter any errors or issues with the package, please leave a note [here](https://github.com/john-b-edwards/cbbreadr), and if you are building anything with this package--from the most most barebones of scatterplots to the biggest and best machine-learning model college basketball has ever seen--please let me know about it on [Bluesky](https://bsky.app/profile/johnbedwards.io), I would love to see what you can create with the package!