---
date: "2023-10-23"
description: Visualizing Big Data Bowl tracking data
draft: false
keywords:
- machine_learning
- football
math: true
slug: big-data-bowl-makie
pagination: true
twitterImg: https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_51_0.png
tags:
- julia
title: Animating Plays in Julia with Makie.jl
toc: true
bsky_thread: https://bsky.app/profile/johnbedwards.io/post/3lcjtzoy2xv2l
---

![](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/thumb_play.gif)

# Introduction

No matter what kind of project you want to tackle, you're going to want the ability to understand what's happening on a given play with the Big Data Bowl dataset. Easier said than done! You can go to YouTube and try to scrub through hours of film to see plays, but 1) that takes a ton of time and 2) means interfacing with the game in a meaningful way beyond manipulating a spreadsheet, which no self-respecting analytics nerd would ever do,,,

This problem has seen a *lot* of different approaches in previous Big Data Bowls. You can check out [Nick Wan's](https://www.kaggle.com/code/nickwan/animated-gif-for-plays-python) and [Hunter Kempf's](https://www.kaggle.com/code/huntingdata11/animated-and-interactive-nfl-plays-in-plotly) notebooks on plotting and animating plays in Python, or [Pablo Landeros'](https://www.kaggle.com/code/pablollanderos33/generate-play-animations-with-ggplot) notebook on plotting and animating plays in R. But I'm not here to reinvent the wheel. I'm not some SQUARE who uses DATED PROGRAMMING LANGUAGES like some of you SHEEPLE. Besides, I'm here to push an agenda: #BringJuliaToKaggle

That's right -- we're going to be working in [Julia](https://julialang.org/) to produce our play visualizations. My hope is that this notebook can serve as a leaping off point for anyone who wants to do Big Data Bowl projects in Julia, and my secondary, ulterior motive is to [continue to spread Julia content on Kaggle](https://www.kaggle.com/code/metlover/cnn-ann-approach-using-flux-jl-17-24339-rmse) in the hopes we get dedicated support soon.

# Loading in Data

To begin, let's pull our data into our directory. I've already set things up with the [Kaggle API](https://github.com/Kaggle/kaggle-api) to make this relatively painless. We'll pull down not just our competition data but a few other useful visuals to help us trick out our project.


```julia
using DataFrames, CSV
function build_directory()
    run(`kaggle competitions download -c nfl-big-data-bowl-2024 --force`)
    run(`mkdir -p data`)
    run(`unzip nfl-big-data-bowl-2024.zip -d data`)
    run(`mkdir -p visuals`)
    run(`curl.exe -o visuals/bdb-logo.png https://operations.nfl.com/media/3577/big-data-bowl-transparent.png --ssl-no-revoke`)
    run(`curl.exe -o visuals/nfl-logo.png https://upload.wikimedia.org/wikipedia/en/thumb/a/a2/National_Football_League_logo.svg/745px-National_Football_League_logo.svg.png --ssl-no-revoke`)
    run(`curl.exe -o visuals/nfl-teams.csv https://raw.githubusercontent.com/nflverse/nflverse-pbp/master/teams_colors_logos.csv --ssl-no-revoke`)
    teams = CSV.read("visuals/nfl-teams.csv",DataFrame)
    for row in 1:nrow(teams)
        abbr = teams.team_abbr[row]
        run(`curl.exe -o visuals/$abbr-wordmark.png https://raw.githubusercontent.com/nflverse/nflverse-pbp/master/wordmarks/$abbr.png --ssl-no-revoke`)
    end
end
build_directory();
```

With our data loaded in, we can take a glimpse at what the tracking data actually looks like.


```julia
week1 = CSV.read("data/tracking_week_1.csv",DataFrame)
```


| **Row**      | **gameId** | **playId** | **nflId** | **displayName** | **frameId** | **time**                   | **jerseyNumber** | **club** | **playDirection** | **x**    | **y**    | **s**    | **a**    | **dis**  | **o**    | **dir**  | **event**           |
|:------------:|:----------:|:----------:|:---------:|:---------------:|:-----------:|:--------------------------:|:----------------:|:--------:|:-----------------:|:--------:|:--------:|:--------:|:--------:|:--------:|:--------:|:--------:|:-------------------:|
| **Int64**    | Int64      | String7    | String31  | Int64           | String31    | String3                    | String15         | String7  | Float64           | Float64  | Float64  | Float64  | Float64  | String7  | String31 | String31 |
| **1**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 1           | 2022-09-08 20:24:05.200000 | 76               | BUF      | left              | 88.37    | 27.27    | 1.62     | 1.15     | 0.16     | 231.74   | 147.9    | NA                  |
| **2**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 2           | 2022-09-08 20:24:05.299999 | 76               | BUF      | left              | 88.47    | 27.13    | 1.67     | 0.61     | 0.17     | 230.98   | 148.53   | pass_arrived        |
| **3**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 3           | 2022-09-08 20:24:05.400000 | 76               | BUF      | left              | 88.56    | 27.01    | 1.57     | 0.49     | 0.15     | 230.98   | 147.05   | NA                  |
| **4**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 4           | 2022-09-08 20:24:05.500000 | 76               | BUF      | left              | 88.64    | 26.9     | 1.44     | 0.89     | 0.14     | 232.38   | 145.42   | NA                  |
| **5**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 5           | 2022-09-08 20:24:05.599999 | 76               | BUF      | left              | 88.72    | 26.8     | 1.29     | 1.24     | 0.13     | 233.36   | 141.95   | NA                  |
| **6**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 6           | 2022-09-08 20:24:05.700000 | 76               | BUF      | left              | 88.8     | 26.7     | 1.15     | 1.42     | 0.12     | 234.48   | 139.41   | pass_outcome_caught |
| **7**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 7           | 2022-09-08 20:24:05.799999 | 76               | BUF      | left              | 88.87    | 26.64    | 0.93     | 1.69     | 0.09     | 235.77   | 134.32   | NA                  |
| **8**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 8           | 2022-09-08 20:24:05.900000 | 76               | BUF      | left              | 88.91    | 26.59    | 0.68     | 1.74     | 0.07     | 240      | 131.01   | NA                  |
| **9**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 9           | 2022-09-08 20:24:06.000000 | 76               | BUF      | left              | 88.94    | 26.57    | 0.42     | 1.74     | 0.04     | 243.56   | 122.29   | NA                  |
| **10**       | 2022090800 | 56         | 35472     | Rodger Saffold  | 10          | 2022-09-08 20:24:06.099999 | 76               | BUF      | left              | 88.95    | 26.58    | 0.14     | 1.83     | 0.01     | 246.07   | 85.87    | NA                  |
| **11**       | 2022090800 | 56         | 35472     | Rodger Saffold  | 11          | 2022-09-08 20:24:06.200000 | 76               | BUF      | left              | 88.92    | 26.6     | 0.26     | 1.9      | 0.03     | 252.65   | 326.63   | NA                  |
| **12**       | 2022090800 | 56         | 35472     | Rodger Saffold  | 12          | 2022-09-08 20:24:06.299999 | 76               | BUF      | left              | 88.9     | 26.63    | 0.51     | 2.45     | 0.04     | 257.66   | 315.55   | NA                  |
| **13**       | 2022090800 | 56         | 35472     | Rodger Saffold  | 13          | 2022-09-08 20:24:06.400000 | 76               | BUF      | left              | 88.84    | 26.68    | 0.81     | 2.03     | 0.08     | 262.09   | 311.72   | NA                  |
| **&vellip;** | &vellip;   | &vellip;   | &vellip;  | &vellip;        | &vellip;    | &vellip;                   | &vellip;         | &vellip; | &vellip;          | &vellip; | &vellip; | &vellip; | &vellip; | &vellip; | &vellip; | &vellip; | &vellip;            |
| **1407428**  | 2022091200 | 3826       | NA        | football        | 42          | 2022-09-12 23:05:57.099999 | NA               | football | left              | 57.13    | 8.44     | 2.17     | 3.63     | 0.29     | NA       | NA       | NA                  |
| **1407429**  | 2022091200 | 3826       | NA        | football        | 43          | 2022-09-12 23:05:57.200000 | NA               | football | left              | 56.99    | 8.66     | 2.01     | 2.68     | 0.26     | NA       | NA       | NA                  |
| **1407430**  | 2022091200 | 3826       | NA        | football        | 44          | 2022-09-12 23:05:57.299999 | NA               | football | left              | 56.85    | 8.87     | 1.97     | 1.7      | 0.25     | NA       | NA       | NA                  |
| **1407431**  | 2022091200 | 3826       | NA        | football        | 45          | 2022-09-12 23:05:57.400000 | NA               | football | left              | 56.76    | 9.04     | 1.83     | 1.14     | 0.19     | NA       | NA       | NA                  |
| **1407432**  | 2022091200 | 3826       | NA        | football        | 46          | 2022-09-12 23:05:57.500000 | NA               | football | left              | 56.63    | 9.28     | 2.15     | 0.33     | 0.27     | NA       | NA       | NA                  |
| **1407433**  | 2022091200 | 3826       | NA        | football        | 47          | 2022-09-12 23:05:57.599999 | NA               | football | left              | 56.52    | 9.5      | 2.34     | 1.12     | 0.24     | NA       | NA       | NA                  |
| **1407434**  | 2022091200 | 3826       | NA        | football        | 48          | 2022-09-12 23:05:57.700000 | NA               | football | left              | 56.4     | 9.72     | 2.53     | 1.23     | 0.26     | NA       | NA       | NA                  |
| **1407435**  | 2022091200 | 3826       | NA        | football        | 49          | 2022-09-12 23:05:57.799999 | NA               | football | left              | 56.22    | 9.89     | 2.56     | 1.25     | 0.25     | NA       | NA       | tackle              |
| **1407436**  | 2022091200 | 3826       | NA        | football        | 50          | 2022-09-12 23:05:57.900000 | NA               | football | left              | 56.06    | 10.08    | 2.5      | 1.14     | 0.24     | NA       | NA       | NA                  |
| **1407437**  | 2022091200 | 3826       | NA        | football        | 51          | 2022-09-12 23:05:58.000000 | NA               | football | left              | 55.89    | 10.27    | 2.38     | 1.7      | 0.25     | NA       | NA       | NA                  |
| **1407438**  | 2022091200 | 3826       | NA        | football        | 52          | 2022-09-12 23:05:58.099999 | NA               | football | left              | 55.73    | 10.44    | 2.07     | 2.83     | 0.24     | NA       | NA       | NA                  |
| **1407439**  | 2022091200 | 3826       | NA        | football        | 53          | 2022-09-12 23:05:58.200000 | NA               | football | left              | 55.57    | 10.57    | 1.86     | 3.0      | 0.2      | NA       | NA       | NA                  |


The available tracking data is split up into several different components:
* `gameId` -- A unique identifier assigned to each game
* `playId` -- A unique identifier assigned to each play
* `frameId` -- The frame of the play
For each unique combination of `gameId`, `playId`, and `frameId`, we have the `x` and `y` coordinates of each player (identified with any of `nflId`, `displayName`, or `jerseyNumber`) as well as the football. The `x` and `y` coordinates are laid out as demonstrated in this graphic, helpfully provided by the organizers by the Big Data Bowl:

![](https://www.googleapis.com/download/storage/v1/b/kaggle-user-content/o/inbox%2F3258%2F820e86013d48faacf33b7a32a15e814c%2FIncreasing%20Dir%20and%20O.png?generation=1572285857588233&alt=media)

We can use this plot as a guide to generating our own graphic in Julia.

# Groundskeeping

Our first step is to build the field on which plays will take place. We will use `Makie.jl` as the plotting library for our visuals -- it's the most flexible library for plotting in Julia, and should allow us to manipulate and animate the play we want. Specifically, we'll import `CairoMakie`, which will allow us to render high quality graphics using the Cairo backend.

What's the first step to building a football field? Laying down the grass, of course -- we'll make a green background that fits the coordinate system described in the graphic above. Note that this can take a while to render initially -- after compiling everything, it should run much faster thanks to Julia magic 🪄.


```julia
using CairoMakie
function base_figure()
    f = Figure(backgroundcolor = :darkgreen, resolution = (1200,533))
    ax = Axis(f[1,1], limits = (0,120,0,53.3), backgroundcolor = :darkgreen)
    hidedecorations!(ax)
    hidespines!(ax) 
    return f, ax
end
f, ax = base_figure()
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_5_0.png)
    


Next, we'll be good groundskeepers and paint the lines.


```julia
using GeometryBasics
f, = base_figure()
poly!(
    Rect(0,0,120,53.3),
    strokecolor = :white, 
    strokewidth = 3, 
    color = :darkgreen
)
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_7_0.png)
    


This football field could use some lines. Let's bust out our tape measure and start marking out the field!


```julia
vlines!(
    10:5:110,
    color = :white
)
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_9_0.png)
    


That's looking good, but how will we know where the spot the ball? After all, not like there's a chip in the football or anything,,, this field clearly needs some line numbers.


```julia
text!(
    18:10:98,
    repeat([5],9); 
    text = string.([10:10:50; 40:-10:10]),
    font="C:/Users/edwar/Documents/GitHub/bdb24/visuals/besley_heavy.otf",
    fontsize=30,
    color=:white
    )
text!(
    18:10:98,
    repeat([43.5],9); 
    text = string.([10:10:50; 40:-10:10]),
    font="C:/Users/edwar/Documents/GitHub/bdb24/visuals/besley_heavy.otf",
    fontsize=30,
    color=:white
)
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_11_0.png)
    


Great! Now, we need some arrows, [just to make sure everyone knows which way to run the ball.](https://www.youtube.com/watch?v=iiMVv-0uek0&ab_channel=NFLFilms) We can use some ASCII text art to quickly throw some on our plot.


```julia
text!(
    16.5:10:46.5, 
    repeat([6.5],4);
    text = repeat(["◂"],4),
    fontsize=20,
    color=:white
)
text!(
    16.5:10:46.5, 
    repeat([45],4);
    text = repeat(["◂"],4),
    fontsize=20,
    color=:white
)
text!(
    72.5:10:102.5, 
    repeat([6.5],4);
    text = repeat(["▸"],4),
    fontsize=20,
    color=:white
)
text!(
    72.5:10:102.5, 
    repeat([45],4);
    text = repeat(["▸"],4),
    fontsize=20,
    color=:white
)
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_13_0.png)
    


We're cooking so far! But how will our kickers know where to kick from? This field is in need of hash marks.


```julia
vlines!(
    10:1:110;
    ymin = 0 / 53.3, # these are a percentage of the plotting window, not absolute coordinates
    ymax = 0.6 / 53.3,
    color = :white
)
vlines!(
    10:1:110;
    ymin = 52.7 / 53.3,
    ymax = 53.3 / 53.3,
    color = :white
)
vlines!(
    10:1:110;
    ymin = 20.2 / 53.3,
    ymax = 20.8 / 53.3,
    color = :white
)
vlines!(
    10:1:110;
    ymin = 33.2 / 53.3,
    ymax = 32.6 / 53.3,
    color = :white
)
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_15_0.png)
    


Now that's a proper, self-respecting football field! But we're still missing one thing... that's right, an enormous, obnoxious logo smack dab in the center of the field. Fortunately, adding it is a cinch with Makie.


```julia
using FileIO
img = load("visuals/bdb-logo.png")
image!(
    [50,70],[18,38],rotr90(img)
)
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_17_0.png)
    


Our field looks ready to go! We can wrap everything in a function for convenience's sake and just call on that whenever we need to plot on top of a field.


```julia
f, = base_figure()
function plot_field(logo=false)
    # background
    # remove plot decorations
    # exterior lines
    poly!(
        Rect(0,0,120,53.3),
        strokecolor = :white, 
        strokewidth = 3, 
        color = :darkgreen
    )
    # interior lines
    vlines!(
        10:5:110,
        color = :white
    )
    # hash marks
    vlines!(
        10:1:110;
        ymin = 0 / 53.3, # these are a percentage of the plotting window, not absolute coordinates
        ymax = 0.6 / 53.3,
        color = :white
    )
    vlines!(
        10:1:110;
        ymin = 52.7 / 53.3,
        ymax = 53.3 / 53.3,
        color = :white
    )
    vlines!(
        10:1:110;
        ymin = 20.2 / 53.3,
        ymax = 20.8 / 53.3,
        color = :white
    )
    vlines!(
        10:1:110;
        ymin = 33.2 / 53.3,
        ymax = 32.6 / 53.3,
        color = :white
    )
    # field numbers
    text!(
        18:10:98,
        repeat([5],9); 
        text = string.([10:10:50; 40:-10:10]),
        font="C:/Users/edwar/Documents/GitHub/bdb24/visuals/besley_heavy.otf",
        fontsize=30,
        color=:white
        )
    text!(
        18:10:98,
        repeat([43.5],9); 
        text = string.([10:10:50; 40:-10:10]),
        font="C:/Users/edwar/Documents/GitHub/bdb24/visuals/besley_heavy.otf",
        fontsize=30,
        color=:white
    )
    # field arrows
    text!(
        16.5:10:46.5, 
        repeat([6.5],4);
        text = repeat(["◂"],4),
        fontsize=20,
        color=:white
    )
    text!(
        16.5:10:46.5, 
        repeat([45],4);
        text = repeat(["◂"],4),
        fontsize=20,
        color=:white
    )
    text!(
        72.5:10:102.5, 
        repeat([6.5],4);
        text = repeat(["▸"],4),
        fontsize=20,
        color=:white
    )
    text!(
        72.5:10:102.5, 
        repeat([45],4);
        text = repeat(["▸"],4),
        fontsize=20,
        color=:white
    )
    if logo
        img = load("visuals/bdb-logo.png")
        image!(
            [50,70],[18,38],rotr90(img)
        )
    end
    return f, ax
end
plot_field()
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_19_0.png)
    


# Play Action

Our next step is to plot the players on the field for a given frame. For simplicity's sake, let's start out just plotting a random frame from a random play in week 1.


```julia
frame = week1[(week1.gameId .== 2022091100) .& (week1.playId .== 1608) .& (week1.frameId .== 1),:]
```


| **Row**   | **gameId** | **playId** | **nflId** | **displayName**       | **frameId** | **time**                   | **jerseyNumber** | **club** | **playDirection** | **x**   | **y**   | **s**   | **a**   | **dis** | **o**    | **dir**  | **event** |
|:---------:|:----------:|:----------:|:---------:|:---------------------:|:-----------:|:--------------------------:|:----------------:|:--------:|:-----------------:|:-------:|:-------:|:-------:|:-------:|:-------:|:--------:|:--------:|:---------:|
| **Int64** | Int64      | String7    | String31  | Int64                 | String31    | String3                    | String15         | String7  | Float64           | Float64 | Float64 | Float64 | Float64 | String7 | String31 | String31 |
| **1**     | 2022091100 | 1608       | 38607     | Demario Davis         | 1           | 2022-09-11 14:14:42.400000 | 56               | NO       | left              | 56.66   | 25.97   | 0.0     | 0.0     | 0.0     | 80.84    | 240.31   | NA        |
| **2**     | 2022091100 | 1608       | 39975     | Cordarrelle Patterson | 1           | 2022-09-11 14:14:42.400000 | 84               | ATL      | left              | 68.78   | 23.69   | 0.0     | 0.0     | 0.0     | 259.23   | 300.66   | NA        |
| **3**     | 2022091100 | 1608       | 40017     | Tyrann Mathieu        | 1           | 2022-09-11 14:14:42.400000 | 32               | NO       | left              | 52.97   | 17.53   | 0.0     | 0.0     | 0.02    | 67.67    | 250.77   | NA        |
| **4**     | 2022091100 | 1608       | 41232     | Jake Matthews         | 1           | 2022-09-11 14:14:42.400000 | 70               | ATL      | left              | 62.45   | 20.62   | 0.0     | 0.0     | 0.0     | 272.22   | 196.4    | NA        |
| **5**     | 2022091100 | 1608       | 41257     | Bradley Roby          | 1           | 2022-09-11 14:14:42.400000 | 21               | NO       | left              | 52.82   | 34.89   | 0.11    | 0.45    | 0.01    | 26.97    | 53.25    | NA        |
| **6**     | 2022091100 | 1608       | 42345     | Marcus Mariota        | 1           | 2022-09-11 14:14:42.400000 | 1                | ATL      | left              | 62.8    | 23.72   | 0.0     | 0.0     | 0.0     | 256.19   | 167.66   | NA        |
| **7**     | 2022091100 | 1608       | 42553     | Christian Ringo       | 1           | 2022-09-11 14:14:42.400000 | 57               | NO       | left              | 60.5    | 24.63   | 0.0     | 0.0     | 0.0     | 87.24    | 96.17    | NA        |
| **8**     | 2022091100 | 1608       | 44823     | Marshon Lattimore     | 1           | 2022-09-11 14:14:42.400000 | 23               | NO       | left              | 60.29   | 14.52   | 0.0     | 0.0     | 0.0     | 359.05   | 14.58    | NA        |
| **9**     | 2022091100 | 1608       | 44851     | Marcus Maye           | 1           | 2022-09-11 14:14:42.400000 | 6                | NO       | left              | 46.55   | 26.75   | 0.67    | 0.9     | 0.07    | 88.19    | 246.27   | NA        |
| **10**    | 2022091100 | 1608       | 44862     | Justin Evans          | 1           | 2022-09-11 14:14:42.400000 | 30               | NO       | left              | 59.35   | 30.64   | 0.0     | 0.0     | 0.0     | 107.49   | 281.42   | NA        |
| **11**    | 2022091100 | 1608       | 45550     | Elijah Wilkinson      | 1           | 2022-09-11 14:14:42.400000 | 68               | ATL      | left              | 62.39   | 22.4    | 0.0     | 0.0     | 0.0     | 241.48   | 51.29    | NA        |
| **12**    | 2022091100 | 1608       | 46083     | Marcus Davenport      | 1           | 2022-09-11 14:14:42.400000 | 92               | NO       | left              | 60.41   | 18.29   | 0.0     | 0.0     | 0.0     | 52.63    | 326.14   | NA        |
| **13**    | 2022091100 | 1608       | 46197     | Kentavius Street      | 1           | 2022-09-11 14:14:42.400000 | 91               | NO       | left              | 60.44   | 21.83   | 0.0     | 0.0     | 0.0     | 81.96    | 118.2    | NA        |
| **14**    | 2022091100 | 1608       | 47797     | Chris Lindstrom       | 1           | 2022-09-11 14:14:42.400000 | 63               | ATL      | left              | 62.09   | 25.1    | 0.0     | 0.0     | 0.0     | 250.06   | 27.22    | NA        |
| **15**    | 2022091100 | 1608       | 47814     | Kaleb McGary          | 1           | 2022-09-11 14:14:42.400000 | 76               | ATL      | left              | 62.29   | 26.68   | 0.0     | 0.0     | 0.0     | 262.33   | 295.83   | NA        |
| **16**    | 2022091100 | 1608       | 48723     | Parker Hesse          | 1           | 2022-09-11 14:14:42.400000 | 46               | ATL      | left              | 62.81   | 29.37   | 0.0     | 0.0     | 0.0     | 250.23   | 209.71   | NA        |
| **17**    | 2022091100 | 1608       | 52489     | Bryan Edwards         | 1           | 2022-09-11 14:14:42.400000 | 89               | ATL      | left              | 63.57   | 33.01   | 0.0     | 0.0     | 0.0     | 278.91   | 178.09   | NA        |
| **18**    | 2022091100 | 1608       | 53433     | Kyle Pitts            | 1           | 2022-09-11 14:14:42.400000 | 8                | ATL      | left              | 62.56   | 27.83   | 0.0     | 0.0     | 0.0     | 221.39   | 299.25   | NA        |
| **19**    | 2022091100 | 1608       | 53457     | Payton Turner         | 1           | 2022-09-11 14:14:42.400000 | 98               | NO       | left              | 60.41   | 27.64   | 0.0     | 0.0     | 0.0     | 101.17   | 260.18   | NA        |
| **20**    | 2022091100 | 1608       | 53489     | Pete Werner           | 1           | 2022-09-11 14:14:42.400000 | 20               | NO       | left              | 56.45   | 22.09   | 0.0     | 0.0     | 0.0     | 94.74    | 203.69   | NA        |
| **21**    | 2022091100 | 1608       | 53543     | Drew Dalman           | 1           | 2022-09-11 14:14:42.400000 | 67               | ATL      | left              | 61.79   | 23.64   | 0.0     | 0.0     | 0.0     | 264.7    | 305.59   | NA        |
| **22**    | 2022091100 | 1608       | 54473     | Drake London          | 1           | 2022-09-11 14:14:42.400000 | 5                | ATL      | left              | 62.43   | 14.69   | 0.0     | 0.0     | 0.0     | 297.2    | 104.64   | NA        |
| **23**    | 2022091100 | 1608       | NA        | football              | 1           | 2022-09-11 14:14:42.400000 | NA               | football | left              | 61.55   | 23.73   | 0.0     | 0.0     | 0.0     | NA       | NA       | NA        |



We can represent all of the players on the field as little dots (as is the convention in showing this kind of tracking data). We can treat this as a scatterplot in Makie.


```julia
f, = base_figure()
plot_field()
scatter!(frame.x, frame.y)
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_23_0.png)
    


This is a good start, but this really doesn't show us much about the frame in question. Who belongs to which team? Who is each player? Where's the ball???

We can identify who belongs to each team by coloring them uniquely. This can be a little tricky -- some teams share primary colors, some teams share secondary colors -- picking which color to use for each team is tricky! I've gone through and tried to pick out a dark and light variant of a color from each team's color scheme, as well as a color for text that shows up well on both the light and dark colors. You can check out my work on this stored as a .csv [here](https://gist.githubusercontent.com/john-b-edwards/7197daa7128088f2cb5bddef4e09bfa7/raw/f15750ff7beda7328fe01d877a2b6134d428fa0d/nfl_colors.csv). We'll follow this set of rules for determining the colors to be used:

* If `club` is the football, color the dot brown (already covered in the .csv)
* If `club` is the home team, color the dot the light-variant of the team's color.
* If `club` is the road team, color the dot the dark-variant of the team's color.


```julia
team_colors = CSV.read(download("https://gist.githubusercontent.com/john-b-edwards/7197daa7128088f2cb5bddef4e09bfa7/raw/8af4fb83609104fa14950cde5cc31d4d95a247b9/nfl_colors.csv"), DataFrame)
games = CSV.read("data/games.csv",DataFrame)
home_away = games[:,[:gameId, :homeTeamAbbr]]
frame = innerjoin(frame,team_colors,on = :club => :team)
frame = innerjoin(frame,home_away,on = :gameId)
frame.color = ifelse.(frame.homeTeamAbbr .== frame.club, frame.dark, frame.light)
frame
```


| **Row**   | **gameId** | **playId** | **nflId** | **displayName**       | **frameId** | **time**                   | **jerseyNumber** | **club** | **playDirection** | **x**   | **y**   | **s**   | **a**   | **dis** | **o**    | **dir**  | **event** | **dark** | **light** | **text** | **homeTeamAbbr** | **color** |
|:---------:|:----------:|:----------:|:---------:|:---------------------:|:-----------:|:--------------------------:|:----------------:|:--------:|:-----------------:|:-------:|:-------:|:-------:|:-------:|:-------:|:--------:|:--------:|:---------:|:--------:|:---------:|:--------:|:----------------:|:---------:|
| **Int64** | Int64      | String7    | String31  | Int64                 | String31    | String3                    | String15         | String7  | Float64           | Float64 | Float64 | Float64 | Float64 | String7 | String31 | String31 | String7   | String7  | String7   | String3  | String7          |
| **1**     | 2022091100 | 1608       | 39975     | Cordarrelle Patterson | 1           | 2022-09-11 14:14:42.400000 | 84               | ATL      | left              | 68.78   | 23.69   | 0.0     | 0.0     | 0.0     | 259.23   | 300.66   | NA        | #4b4b4b  | #A5ACAF   | #A71930  | ATL              | #4b4b4b   |
| **2**     | 2022091100 | 1608       | 41232     | Jake Matthews         | 1           | 2022-09-11 14:14:42.400000 | 70               | ATL      | left              | 62.45   | 20.62   | 0.0     | 0.0     | 0.0     | 272.22   | 196.4    | NA        | #4b4b4b  | #A5ACAF   | #A71930  | ATL              | #4b4b4b   |
| **3**     | 2022091100 | 1608       | 42345     | Marcus Mariota        | 1           | 2022-09-11 14:14:42.400000 | 1                | ATL      | left              | 62.8    | 23.72   | 0.0     | 0.0     | 0.0     | 256.19   | 167.66   | NA        | #4b4b4b  | #A5ACAF   | #A71930  | ATL              | #4b4b4b   |
| **4**     | 2022091100 | 1608       | 45550     | Elijah Wilkinson      | 1           | 2022-09-11 14:14:42.400000 | 68               | ATL      | left              | 62.39   | 22.4    | 0.0     | 0.0     | 0.0     | 241.48   | 51.29    | NA        | #4b4b4b  | #A5ACAF   | #A71930  | ATL              | #4b4b4b   |
| **5**     | 2022091100 | 1608       | 47797     | Chris Lindstrom       | 1           | 2022-09-11 14:14:42.400000 | 63               | ATL      | left              | 62.09   | 25.1    | 0.0     | 0.0     | 0.0     | 250.06   | 27.22    | NA        | #4b4b4b  | #A5ACAF   | #A71930  | ATL              | #4b4b4b   |
| **6**     | 2022091100 | 1608       | 47814     | Kaleb McGary          | 1           | 2022-09-11 14:14:42.400000 | 76               | ATL      | left              | 62.29   | 26.68   | 0.0     | 0.0     | 0.0     | 262.33   | 295.83   | NA        | #4b4b4b  | #A5ACAF   | #A71930  | ATL              | #4b4b4b   |
| **7**     | 2022091100 | 1608       | 48723     | Parker Hesse          | 1           | 2022-09-11 14:14:42.400000 | 46               | ATL      | left              | 62.81   | 29.37   | 0.0     | 0.0     | 0.0     | 250.23   | 209.71   | NA        | #4b4b4b  | #A5ACAF   | #A71930  | ATL              | #4b4b4b   |
| **8**     | 2022091100 | 1608       | 52489     | Bryan Edwards         | 1           | 2022-09-11 14:14:42.400000 | 89               | ATL      | left              | 63.57   | 33.01   | 0.0     | 0.0     | 0.0     | 278.91   | 178.09   | NA        | #4b4b4b  | #A5ACAF   | #A71930  | ATL              | #4b4b4b   |
| **9**     | 2022091100 | 1608       | 53433     | Kyle Pitts            | 1           | 2022-09-11 14:14:42.400000 | 8                | ATL      | left              | 62.56   | 27.83   | 0.0     | 0.0     | 0.0     | 221.39   | 299.25   | NA        | #4b4b4b  | #A5ACAF   | #A71930  | ATL              | #4b4b4b   |
| **10**    | 2022091100 | 1608       | 53543     | Drew Dalman           | 1           | 2022-09-11 14:14:42.400000 | 67               | ATL      | left              | 61.79   | 23.64   | 0.0     | 0.0     | 0.0     | 264.7    | 305.59   | NA        | #4b4b4b  | #A5ACAF   | #A71930  | ATL              | #4b4b4b   |
| **11**    | 2022091100 | 1608       | 54473     | Drake London          | 1           | 2022-09-11 14:14:42.400000 | 5                | ATL      | left              | 62.43   | 14.69   | 0.0     | 0.0     | 0.0     | 297.2    | 104.64   | NA        | #4b4b4b  | #A5ACAF   | #A71930  | ATL              | #4b4b4b   |
| **12**    | 2022091100 | 1608       | 38607     | Demario Davis         | 1           | 2022-09-11 14:14:42.400000 | 56               | NO       | left              | 56.66   | 25.97   | 0.0     | 0.0     | 0.0     | 80.84    | 240.31   | NA        | #101820  | #FFFFFF   | #D3BC8D  | ATL              | #FFFFFF   |
| **13**    | 2022091100 | 1608       | 40017     | Tyrann Mathieu        | 1           | 2022-09-11 14:14:42.400000 | 32               | NO       | left              | 52.97   | 17.53   | 0.0     | 0.0     | 0.02    | 67.67    | 250.77   | NA        | #101820  | #FFFFFF   | #D3BC8D  | ATL              | #FFFFFF   |
| **14**    | 2022091100 | 1608       | 41257     | Bradley Roby          | 1           | 2022-09-11 14:14:42.400000 | 21               | NO       | left              | 52.82   | 34.89   | 0.11    | 0.45    | 0.01    | 26.97    | 53.25    | NA        | #101820  | #FFFFFF   | #D3BC8D  | ATL              | #FFFFFF   |
| **15**    | 2022091100 | 1608       | 42553     | Christian Ringo       | 1           | 2022-09-11 14:14:42.400000 | 57               | NO       | left              | 60.5    | 24.63   | 0.0     | 0.0     | 0.0     | 87.24    | 96.17    | NA        | #101820  | #FFFFFF   | #D3BC8D  | ATL              | #FFFFFF   |
| **16**    | 2022091100 | 1608       | 44823     | Marshon Lattimore     | 1           | 2022-09-11 14:14:42.400000 | 23               | NO       | left              | 60.29   | 14.52   | 0.0     | 0.0     | 0.0     | 359.05   | 14.58    | NA        | #101820  | #FFFFFF   | #D3BC8D  | ATL              | #FFFFFF   |
| **17**    | 2022091100 | 1608       | 44851     | Marcus Maye           | 1           | 2022-09-11 14:14:42.400000 | 6                | NO       | left              | 46.55   | 26.75   | 0.67    | 0.9     | 0.07    | 88.19    | 246.27   | NA        | #101820  | #FFFFFF   | #D3BC8D  | ATL              | #FFFFFF   |
| **18**    | 2022091100 | 1608       | 44862     | Justin Evans          | 1           | 2022-09-11 14:14:42.400000 | 30               | NO       | left              | 59.35   | 30.64   | 0.0     | 0.0     | 0.0     | 107.49   | 281.42   | NA        | #101820  | #FFFFFF   | #D3BC8D  | ATL              | #FFFFFF   |
| **19**    | 2022091100 | 1608       | 46083     | Marcus Davenport      | 1           | 2022-09-11 14:14:42.400000 | 92               | NO       | left              | 60.41   | 18.29   | 0.0     | 0.0     | 0.0     | 52.63    | 326.14   | NA        | #101820  | #FFFFFF   | #D3BC8D  | ATL              | #FFFFFF   |
| **20**    | 2022091100 | 1608       | 46197     | Kentavius Street      | 1           | 2022-09-11 14:14:42.400000 | 91               | NO       | left              | 60.44   | 21.83   | 0.0     | 0.0     | 0.0     | 81.96    | 118.2    | NA        | #101820  | #FFFFFF   | #D3BC8D  | ATL              | #FFFFFF   |
| **21**    | 2022091100 | 1608       | 53457     | Payton Turner         | 1           | 2022-09-11 14:14:42.400000 | 98               | NO       | left              | 60.41   | 27.64   | 0.0     | 0.0     | 0.0     | 101.17   | 260.18   | NA        | #101820  | #FFFFFF   | #D3BC8D  | ATL              | #FFFFFF   |
| **22**    | 2022091100 | 1608       | 53489     | Pete Werner           | 1           | 2022-09-11 14:14:42.400000 | 20               | NO       | left              | 56.45   | 22.09   | 0.0     | 0.0     | 0.0     | 94.74    | 203.69   | NA        | #101820  | #FFFFFF   | #D3BC8D  | ATL              | #FFFFFF   |
| **23**    | 2022091100 | 1608       | NA        | football              | 1           | 2022-09-11 14:14:42.400000 | NA               | football | left              | 61.55   | 23.73   | 0.0     | 0.0     | 0.0     | NA       | NA       | NA        | #825736  | #825736   | #825736  | ATL              | #825736   |



Now that we've applied our color rules, we can scatter out dots on the field, and tag them as the appropriate color.


```julia
f, = base_figure()
plot_field()
scatter!(
    frame.x, 
    frame.y; 
    color = String.(frame.color), 
    markersize = 40, 
    strokewidth=2
)
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_27_0.png)
    


To better identify the players on the field, we can also overlay their uniform numbers on top of the play.


```julia
text!(
    frame.x[frame.club .!= "football"],
    frame.y[frame.club .!= "football"];
    text = String.(frame.jerseyNumber[frame.club .!= "football"]),
    font = "Helvetica",
    fontstyle = :bold,
    fontsize=20,
    color=String.(frame.text[frame.club .!= "football"]),
    align = (:center, :center)
)
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_29_0.png)
    


Hrm, that's problematic -- the text representing all the players overlaps significantly, causing a big mess in the middle of the scene. 

It's slightly less efficient, but let's re-work our code to iterately place each player's dot and then text, to fix this overlapping text issue. This is slightly less efficient from a computational lens, but this is also Julia, where everything runs lightning fast, so who the #$@! cares???


```julia
f, = base_figure()
plot_field()
for n in 1:nrow(frame)
    scatter!(
        frame.x[n], 
        frame.y[n]; 
        color = String.(frame.color[n]), 
        markersize = 40, 
        strokewidth=2
    )
    if frame.club[n] != "football"
        text!(
            frame.x[n],
            frame.y[n];
            text = String.(frame.jerseyNumber[n]),
            font = "Helvetica",
            fontstyle = :bold,
            fontsize=20,
            color=String.(frame.text[n]),
            align = (:center, :center)
        )
    end
end
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_31_0.png)
    


Visually, that looks much better -- we're going for a "player tokens" look, and this captures that vibe quite well.

Speaking of tokens -- that football looks pretty whack. Who ever heard of a round football? This isn't Europe, we call that soccer in these here parts. 🇺🇸


```julia
function football()
    # from https://upload.wikimedia.org/wikipedia/commons/2/2d/American_football_icon_simple_flat.svg
    football_string = "M260.23 242.48C470.17 49.91 703.98 47.53 770.09 50.38c54.62 2.3575 100.81 8.0024 121.59 12.397 34.724 7.3432 25.195 7.327 46.702 33.008 21.507 25.681 34.249 26.987 35.022 68.388.63801 34.21 5.9807 206.33-60.35 357.28-66.33 150.96-113.03 188.58-173.28 243.85-60.25 55.26-197.57 142.87-350.57 174.11-79.27 16.19-298.03 7.65-312.03-10.34-5.637-8.26-14.161-17.48-20.932-22.5-15.188-16.56-25.367-51.22-26.948-79.87-1.581-28.65-13.148-189.89 60.548-346.12 73.692-156.24 170.39-238.1 170.39-238.1z"
    symbol = BezierPath(football_string, fit = true)
    return(symbol)
end 

f, = base_figure()
plot_field()

for n in 1:nrow(frame)
    if frame.club[n] != "football"
    scatter!(
        frame.x[n], 
        frame.y[n]; 
        color = String.(frame.color[n]), 
        markersize = 40, 
        strokewidth=2
    )
    text!(
        frame.x[n],
        frame.y[n];
        text = String.(frame.jerseyNumber[n]),
        font = "Helvetica",
        fontstyle = :bold,
        fontsize=20,
        color=String.(frame.text[n]),
        align = (:center, :center)
    )
    else
        scatter!(
            frame.x[n], 
            frame.y[n]; 
            marker = football(),
            color = String.(frame.color[n]), 
            rotations = pi / 4,
            markersize = 25, 
            strokewidth=2
        )
    end
end
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_33_0.png)
    


Looking great so far! We can wrap this in a function to make plotting the players a little easier.


```julia
function plot_players(frame)
    for n in 1:nrow(frame)
        if frame.club[n] != "football"
        scatter!(
            frame.x[n], 
            frame.y[n]; 
            color = String.(frame.color[n]), 
            markersize = 40, 
            strokewidth=2
        )
        text!(
            frame.x[n],
            frame.y[n];
            text = String.(frame.jerseyNumber[n]),
            font = "Helvetica",
            fontstyle = :bold,
            fontsize=20,
            color=String.(frame.text[n]),
            align = (:center, :center)
        )
        else
            scatter!(
                frame.x[n], 
                frame.y[n]; 
                marker = football(),
                color = String.(frame.color[n]), 
                rotations = pi / 4,
                markersize = 25, 
                strokewidth=2
            )
        end
    end
end
f, = base_figure()
plot_field()
plot_players(frame)
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_35_0.png)
    


# Decorating the Field

Next step -- those end-zones are looking awful empty. Let's try filling them up with some wordmarks, courtesy of the kind folks at the `{nflverse}` (I hear they're a great bunch, very handsome, that John Edwards fella in particular).

We'll first figure out who the team with the ball is on the play, then figure out which direction the play is going offensively -- left to right. From there, we'll plot the wordmark of the defensive team in the end-zone in the direction of the play, and the possession team in the opposite end-zone. Let's pull in the `plays.csv` file and join it to figure out which team has the ball. We can then read in wordmarks for the possessing and defensive teams:


```julia
plays = CSV.read("data/plays.csv",DataFrame)
pos_team = plays[:,[:gameId, :playId, :possessionTeam, :defensiveTeam, :yardlineSide, :yardlineNumber, :yardsToGo]]
frame = innerjoin(frame,pos_team,on = [:gameId, :playId])
using Images
wordmark_pos = load("visuals/" * frame.possessionTeam[1] * "-wordmark.png")
wordmark_def = load("visuals/" * frame.defensiveTeam[1] * "-wordmark.png")
```


<img src="data:image/png;base64,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">


That wordmark is looking good! Next step -- we'll figure out the play direction, and based on that, plot the wordmarks (along with the team colors) in the correct endzones.


```julia
f, = base_figure()
plot_field()
# plot the end-zones
# we do this first to make sure the end-zones are layered under the players!
if frame.playDirection[1] == "left"
    poly!(
        Rect(0,0,10,53.3),
        strokecolor = :white, 
        strokewidth = 3, 
        color = String(frame.color[frame.defensiveTeam .== frame.club][1])
    )
    poly!(
        Rect(110,0,10,53.3),
        strokecolor = :white, 
        strokewidth = 3, 
        color = String(frame.color[frame.possessionTeam .== frame.club][1])
    )
    image!(
        [120,110],[53,0],wordmark_pos
    )
    image!(
        [0,10],[0,53],wordmark_def
    )
else
    poly!(
        Rect(0,0,10,53.3),
        strokecolor = :white, 
        strokewidth = 3, 
        color = String(frame.color[frame.possessionTeam .== frame.club][1])
    )
    poly!(
        Rect(110,0,10,53.3),
        strokecolor = :white, 
        strokewidth = 3, 
        color = String(frame.color[frame.defensiveTeam .== frame.club][1])
    )
    image!(
        [120,110],[53,0],wordmark_def
    )
    image!(
        [0,10],[0,53],wordmark_pos
    )
end
# okay, now we can plot the players
plot_players(frame)
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_41_0.png)
    


Oh _helllllllll_ yeah. Alright, we're missing one more thing. We want this to look at least a little like what it does on TV, so to make things clearer, we're going to use the play data and plot the line of scrimmage in blue and the line to gain in yellow. We'll wrap this into the same logic that determines which way to place the end-zones (and which way the ball is going):


```julia
f, = base_figure()
plot_field()
yardline_100 = ifelse(frame.yardlineSide[1] == frame.possessionTeam[1],50 - (50 - frame.yardlineNumber[1]), 50 + (50 - frame.yardlineNumber[1]))
if frame.playDirection[1] == "left"
    poly!(
        Rect(0,0,10,53.3),
        strokecolor = :white, 
        strokewidth = 3, 
        color = String(frame.color[frame.defensiveTeam .== frame.club][1])
    )
    poly!(
        Rect(110,0,10,53.3),
        strokecolor = :white, 
        strokewidth = 3, 
        color = String(frame.color[frame.possessionTeam .== frame.club][1])
    )
    image!(
        [120,110],[53,0],wordmark_pos
    )
    image!(
        [0,10],[0,53],wordmark_def
    )
    # line of scrimmage in blue, dashed
    vlines!(
        110 - yardline_100,
        color = :blue,
        linestyle = :dash
    )
    # line to gain -- in yellow, of course
    vlines!(
        110 - yardline_100 - frame.yardsToGo[1],
        color = :yellow,
        linestyle = :dash
    )
else
    poly!(
        Rect(0,0,10,53.3),
        strokecolor = :white, 
        strokewidth = 3, 
        color = String(frame.color[frame.possessionTeam .== frame.club][1])
    )
    poly!(
        Rect(110,0,10,53.3),
        strokecolor = :white, 
        strokewidth = 3, 
        color = String(frame.color[frame.defensiveTeam .== frame.club][1])
    )
    image!(
        [120,110],[53,0],wordmark_def
    )
    image!(
        [0,10],[0,53],wordmark_pos
    )
    vlines!(
        10 + yardline_100,
        color = :blue,
        linestyle = :dash
    )
    vlines!(
        10 + yardline_100 + frame.yardsToGo[1],
        color = :yellow,
        linestyle = :dash
    )
end
plot_players(frame)
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_43_0.png)
    


# Putting it all together

Terrific, looks like our field is ready to go. Like the other parts of the plot, let's wrap it in a function.


```julia
function plot_ez_and_lines(frame)
    wordmark_pos = load("visuals/" * frame.possessionTeam[1] * "-wordmark.png")
    wordmark_def = load("visuals/" * frame.defensiveTeam[1] * "-wordmark.png")  
    yardline_100 = ifelse(frame.yardlineSide[1] == frame.possessionTeam[1],50 - (50 - frame.yardlineNumber[1]), 50 + (50 - frame.yardlineNumber[1]))
    if frame.playDirection[1] == "left"
        
        poly!(
            Rect(0,0,10,53.3),
            strokecolor = :white, 
            strokewidth = 3, 
            color = String(frame.color[frame.defensiveTeam .== frame.club][1])
        )
        poly!(
            Rect(110,0,10,53.3),
            strokecolor = :white, 
            strokewidth = 3, 
            color = String(frame.color[frame.possessionTeam .== frame.club][1])
        )
        image!(
            [120,110],[53,0],wordmark_pos
        )
        image!(
            [0,10],[0,53],wordmark_def
        )
        vlines!(
            110 - yardline_100,
            color = :blue,
            linestyle = :dash
        )
        vlines!(
            110 - yardline_100 - frame.yardsToGo[1],
            color = :yellow,
            linestyle = :dash
        )
    else
        poly!(
            Rect(0,0,10,53.3),
            strokecolor = :white, 
            strokewidth = 3, 
            color = String(frame.color[frame.possessionTeam .== frame.club][1])
        )
        poly!(
            Rect(110,0,10,53.3),
            strokecolor = :white, 
            strokewidth = 3, 
            color = String(frame.color[frame.defensiveTeam .== frame.club][1])
        )
        image!(
            [120,110],[53,0],wordmark_def
        )
        image!(
            [0,10],[0,53],wordmark_pos
        )
        vlines!(
            10 + yardline_100,
            color = :blue,
            linestyle = :dash
        )
        vlines!(
            10 + yardline_100 + frame.yardsToGo[1],
            color = :yellow,
            linestyle = :dash
        )
    end
end
```


    plot_ez_and_lines (generic function with 1 method)


Now with just a couple of lines, we can plot any frame from any play:


```julia
new_frame = week1[(week1.gameId .== 2022091109) .& (week1.playId .== 2404) .& (week1.frameId .== 5),:]
new_frame = innerjoin(new_frame,team_colors,on = :club => :team)
new_frame = innerjoin(new_frame,home_away,on = :gameId)
new_frame.color = ifelse.(new_frame.homeTeamAbbr .== new_frame.club, new_frame.dark, new_frame.light)
new_frame = innerjoin(new_frame,pos_team,on = [:gameId, :playId])
f, = base_figure()
plot_field()
plot_ez_and_lines(new_frame)
plot_players(new_frame)
f
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_47_0.png)
    


We'll want the ability to functionalize this further, to make it easier to plot the frames of individual plays. To do this, let's prepare a DataFrame with all of the information we'll need to plot plays, and turn it into a function to load it into memory.


```julia
function plot_df(weeks = 1:9, data_dir = "data")
    # read in data
    plays = CSV.read(data_dir * "/plays.csv",DataFrame)
    team_colors = CSV.read(download("https://gist.githubusercontent.com/john-b-edwards/7197daa7128088f2cb5bddef4e09bfa7/raw/8af4fb83609104fa14950cde5cc31d4d95a247b9/nfl_colors.csv"), DataFrame)
    games = CSV.read(data_dir * "/games.csv",DataFrame)
    home_away = games[:,[:gameId, :homeTeamAbbr]]
    dfs = [CSV.read(data_dir * "/tracking_week_" * string(week) * ".csv",DataFrame) for week in weeks]
    df = vcat(dfs...)
    df = innerjoin(df,team_colors,on = :club => :team)
    df = innerjoin(df,home_away,on = :gameId)
    df.color = ifelse.(df.homeTeamAbbr .== df.club, df.dark, df.light)
    df = innerjoin(df,pos_team,on = [:gameId, :playId])
    return(df)
end
main_df = plot_df()
```


| **Row**      | **gameId** | **playId** | **nflId** | **displayName** | **frameId** | **time**                   | **jerseyNumber** | **club** | **playDirection** | **x**    | **y**    | **s**    | **a**    | **dis**  | **o**    | **dir**  | **event**           | **dark** | **light** | **text** | **homeTeamAbbr** | **color** | **possessionTeam** | **defensiveTeam** | **yardlineSide** | **yardlineNumber** | **yardsToGo** |
|:------------:|:----------:|:----------:|:---------:|:---------------:|:-----------:|:--------------------------:|:----------------:|:--------:|:-----------------:|:--------:|:--------:|:--------:|:--------:|:--------:|:--------:|:--------:|:-------------------:|:--------:|:---------:|:--------:|:----------------:|:---------:|:------------------:|:-----------------:|:----------------:|:------------------:|:-------------:|
| **Int64**    | Int64      | String7    | String31  | Int64           | String31    | String3                    | String15         | String7  | Float64           | Float64  | Float64  | Float64  | Float64  | String7  | String31 | String31 | String7             | String7  | String7   | String3  | String7          | String3   | String3            | String3           | Int64            | Int64              |
| **1**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 1           | 2022-09-08 20:24:05.200000 | 76               | BUF      | left              | 88.37    | 27.27    | 1.62     | 1.15     | 0.16     | 231.74   | 147.9    | NA                  | #C60C30  | #ffffff   | #00338D  | LA               | #ffffff   | BUF                | LA                | BUF              | 25                 | 10            |
| **2**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 2           | 2022-09-08 20:24:05.299999 | 76               | BUF      | left              | 88.47    | 27.13    | 1.67     | 0.61     | 0.17     | 230.98   | 148.53   | pass_arrived        | #C60C30  | #ffffff   | #00338D  | LA               | #ffffff   | BUF                | LA                | BUF              | 25                 | 10            |
| **3**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 3           | 2022-09-08 20:24:05.400000 | 76               | BUF      | left              | 88.56    | 27.01    | 1.57     | 0.49     | 0.15     | 230.98   | 147.05   | NA                  | #C60C30  | #ffffff   | #00338D  | LA               | #ffffff   | BUF                | LA                | BUF              | 25                 | 10            |
| **4**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 4           | 2022-09-08 20:24:05.500000 | 76               | BUF      | left              | 88.64    | 26.9     | 1.44     | 0.89     | 0.14     | 232.38   | 145.42   | NA                  | #C60C30  | #ffffff   | #00338D  | LA               | #ffffff   | BUF                | LA                | BUF              | 25                 | 10            |
| **5**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 5           | 2022-09-08 20:24:05.599999 | 76               | BUF      | left              | 88.72    | 26.8     | 1.29     | 1.24     | 0.13     | 233.36   | 141.95   | NA                  | #C60C30  | #ffffff   | #00338D  | LA               | #ffffff   | BUF                | LA                | BUF              | 25                 | 10            |
| **6**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 6           | 2022-09-08 20:24:05.700000 | 76               | BUF      | left              | 88.8     | 26.7     | 1.15     | 1.42     | 0.12     | 234.48   | 139.41   | pass_outcome_caught | #C60C30  | #ffffff   | #00338D  | LA               | #ffffff   | BUF                | LA                | BUF              | 25                 | 10            |
| **7**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 7           | 2022-09-08 20:24:05.799999 | 76               | BUF      | left              | 88.87    | 26.64    | 0.93     | 1.69     | 0.09     | 235.77   | 134.32   | NA                  | #C60C30  | #ffffff   | #00338D  | LA               | #ffffff   | BUF                | LA                | BUF              | 25                 | 10            |
| **8**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 8           | 2022-09-08 20:24:05.900000 | 76               | BUF      | left              | 88.91    | 26.59    | 0.68     | 1.74     | 0.07     | 240      | 131.01   | NA                  | #C60C30  | #ffffff   | #00338D  | LA               | #ffffff   | BUF                | LA                | BUF              | 25                 | 10            |
| **9**        | 2022090800 | 56         | 35472     | Rodger Saffold  | 9           | 2022-09-08 20:24:06.000000 | 76               | BUF      | left              | 88.94    | 26.57    | 0.42     | 1.74     | 0.04     | 243.56   | 122.29   | NA                  | #C60C30  | #ffffff   | #00338D  | LA               | #ffffff   | BUF                | LA                | BUF              | 25                 | 10            |
| **10**       | 2022090800 | 56         | 35472     | Rodger Saffold  | 10          | 2022-09-08 20:24:06.099999 | 76               | BUF      | left              | 88.95    | 26.58    | 0.14     | 1.83     | 0.01     | 246.07   | 85.87    | NA                  | #C60C30  | #ffffff   | #00338D  | LA               | #ffffff   | BUF                | LA                | BUF              | 25                 | 10            |
| **11**       | 2022090800 | 56         | 35472     | Rodger Saffold  | 11          | 2022-09-08 20:24:06.200000 | 76               | BUF      | left              | 88.92    | 26.6     | 0.26     | 1.9      | 0.03     | 252.65   | 326.63   | NA                  | #C60C30  | #ffffff   | #00338D  | LA               | #ffffff   | BUF                | LA                | BUF              | 25                 | 10            |
| **12**       | 2022090800 | 56         | 35472     | Rodger Saffold  | 12          | 2022-09-08 20:24:06.299999 | 76               | BUF      | left              | 88.9     | 26.63    | 0.51     | 2.45     | 0.04     | 257.66   | 315.55   | NA                  | #C60C30  | #ffffff   | #00338D  | LA               | #ffffff   | BUF                | LA                | BUF              | 25                 | 10            |
| **13**       | 2022090800 | 56         | 35472     | Rodger Saffold  | 13          | 2022-09-08 20:24:06.400000 | 76               | BUF      | left              | 88.84    | 26.68    | 0.81     | 2.03     | 0.08     | 262.09   | 311.72   | NA                  | #C60C30  | #ffffff   | #00338D  | LA               | #ffffff   | BUF                | LA                | BUF              | 25                 | 10            |
| **&vellip;** | &vellip;   | &vellip;   | &vellip;  | &vellip;        | &vellip;    | &vellip;                   | &vellip;         | &vellip; | &vellip;          | &vellip; | &vellip; | &vellip; | &vellip; | &vellip; | &vellip; | &vellip; | &vellip;            | &vellip; | &vellip;  | &vellip; | &vellip;         | &vellip;  | &vellip;           | &vellip;          | &vellip;         | &vellip;           | &vellip;      |
| **12187387** | 2022110700 | 3787       | NA        | football        | 33          | 2022-11-07 23:06:48.500000 | NA               | football | right             | 24.68    | 20.48    | 3.76     | 5.15     | 0.47     | NA       | NA       | NA                  | #825736  | #825736   | #825736  | NO               | #825736   | NO                 | BAL               | NO               | 11                 | 10            |
| **12187388** | 2022110700 | 3787       | NA        | football        | 34          | 2022-11-07 23:06:48.599999 | NA               | football | right             | 25.0     | 20.33    | 3.34     | 4.0      | 0.36     | NA       | NA       | NA                  | #825736  | #825736   | #825736  | NO               | #825736   | NO                 | BAL               | NO               | 11                 | 10            |
| **12187389** | 2022110700 | 3787       | NA        | football        | 35          | 2022-11-07 23:06:48.700000 | NA               | football | right             | 25.29    | 20.2     | 2.99     | 3.17     | 0.31     | NA       | NA       | NA                  | #825736  | #825736   | #825736  | NO               | #825736   | NO                 | BAL               | NO               | 11                 | 10            |
| **12187390** | 2022110700 | 3787       | NA        | football        | 36          | 2022-11-07 23:06:48.799999 | NA               | football | right             | 25.54    | 20.07    | 2.72     | 2.7      | 0.29     | NA       | NA       | NA                  | #825736  | #825736   | #825736  | NO               | #825736   | NO                 | BAL               | NO               | 11                 | 10            |
| **12187391** | 2022110700 | 3787       | NA        | football        | 37          | 2022-11-07 23:06:48.900000 | NA               | football | right             | 25.76    | 19.93    | 2.49     | 2.37     | 0.26     | NA       | NA       | NA                  | #825736  | #825736   | #825736  | NO               | #825736   | NO                 | BAL               | NO               | 11                 | 10            |
| **12187392** | 2022110700 | 3787       | NA        | football        | 38          | 2022-11-07 23:06:49.000000 | NA               | football | right             | 25.95    | 19.85    | 2.06     | 2.56     | 0.2      | NA       | NA       | NA                  | #825736  | #825736   | #825736  | NO               | #825736   | NO                 | BAL               | NO               | 11                 | 10            |
| **12187393** | 2022110700 | 3787       | NA        | football        | 39          | 2022-11-07 23:06:49.099999 | NA               | football | right             | 26.1     | 19.76    | 1.69     | 2.6      | 0.17     | NA       | NA       | NA                  | #825736  | #825736   | #825736  | NO               | #825736   | NO                 | BAL               | NO               | 11                 | 10            |
| **12187394** | 2022110700 | 3787       | NA        | football        | 40          | 2022-11-07 23:06:49.200000 | NA               | football | right             | 26.22    | 19.68    | 1.37     | 2.58     | 0.15     | NA       | NA       | tackle              | #825736  | #825736   | #825736  | NO               | #825736   | NO                 | BAL               | NO               | 11                 | 10            |
| **12187395** | 2022110700 | 3787       | NA        | football        | 41          | 2022-11-07 23:06:49.299999 | NA               | football | right             | 26.32    | 19.61    | 1.07     | 2.74     | 0.12     | NA       | NA       | NA                  | #825736  | #825736   | #825736  | NO               | #825736   | NO                 | BAL               | NO               | 11                 | 10            |
| **12187396** | 2022110700 | 3787       | NA        | football        | 42          | 2022-11-07 23:06:49.400000 | NA               | football | right             | 26.39    | 19.56    | 0.8      | 2.49     | 0.09     | NA       | NA       | NA                  | #825736  | #825736   | #825736  | NO               | #825736   | NO                 | BAL               | NO               | 11                 | 10            |
| **12187397** | 2022110700 | 3787       | NA        | football        | 43          | 2022-11-07 23:06:49.500000 | NA               | football | right             | 26.45    | 19.52    | 0.57     | 2.38     | 0.07     | NA       | NA       | NA                  | #825736  | #825736   | #825736  | NO               | #825736   | NO                 | BAL               | NO               | 11                 | 10            |
| **12187398** | 2022110700 | 3787       | NA        | football        | 44          | 2022-11-07 23:06:49.599999 | NA               | football | right             | 26.49    | 19.5     | 0.35     | 2.13     | 0.05     | NA       | NA       | NA                  | #825736  | #825736   | #825736  | NO               | #825736   | NO                 | BAL               | NO               | 11                 | 10            |



Now, we can make a function to just plot any frame of any play we query.


```julia
function plot_frame(df, gameId, playId, frameId)
    f, = base_figure()
    plot_field()
    frame = df[(df.gameId .== gameId) .& (df.playId .== playId) .& (df.frameId .== frameId),:]
    plot_ez_and_lines(frame)
    plot_players(frame)
    return(f)
end
plot_frame(main_df, 2022100908, 210, 40)
```


    
![png](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/Makie_Visualization_51_0.png)
    


So we can now plot any frame from any play. How do we coalesce this into an animation of the play? Makie allows us to make a Gif of a play super easily, with the `record()` function. We'll simply filter down to a specific play, then iterate over all the frames of the play and record each plot, then string those together to create a play animation.


```julia
function animate_play(df, gameId, playId)
    frames = 1:maximum(df.frameId[(df.gameId .== gameId) .& (df.playId .== playId)])
    # we buffer the animation with some still frames to make looping a little less jumpy
    frames = [repeat([1],5);frames;repeat([maximum(df.frameId[(df.gameId .== gameId) .& (df.playId .== playId)])],5)]
    f, ax = base_figure()
    record(f, "play.gif", frames; framerate = 15) do frame
        empty!(ax)
        plot_field()
        plot_ez_and_lines(df[(df.gameId .== gameId) .& (df.playId .== playId) .& (df.frameId .== frame),:])
        plot_players(df[(df.gameId .== gameId) .& (df.playId .== playId) .& (df.frameId .== frame),:])
    end
end
animate_play(main_df, 2022100908, 210)
```


    "play.gif"


![gif](https://raw.githubusercontent.com/john-b-edwards/random_notebooks/main/Makie_Visualization_files/play.gif)

# Extensions

I'm going to put a bow on this notebook here -- however, there are a ton of additional extensions others can pursue with the logic laid out in this notebook. `Makie` is remarkably flexible and allows for a ton of customization, and Julia as a language provides a ton of opportunities for BDB work. Some examples of potential ways to extend this work:
* You can plot player velocity and/or acceleration using arrows
* You can plot the paths of individual players [Family Circus style](https://www.reddit.com/media?url=https%3A%2F%2Fi.redd.it%2Fw4dai1creyo81.jpg) on a play
* You can create [Voronoi Diagrams](https://github.com/JuliaGeometry/VoronoiCells.jl) of field control on plays
* You can [visualize the optimal path for a ball carrier on a play](https://www.kaggle.com/code/robynritchie/punt-returns-using-the-math-to-find-the-path)

I hope you found this notebook helpful, and I hope this sparked some interest in the wonderful language of Julia! Happy tackling!