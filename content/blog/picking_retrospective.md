---
date: "2022-01-10"
description: Lessons from picking the 2022 CFB Season
draft: false
keywords:
- college_football
- football
- machine_learning
math: true
slug: picking_2022
pagination: true
tags:
- misc
title: Lessons from picking the 2022 CFB Season
toc: false
---

![social-image](https://github.com/john-b-edwards/johnedwards/raw/f724922b0d89b87c1253a9f6974ff2098281216f/themes/hugo-theme-codex/static/football.jpg)

As we reach the end of the 2021-22 college football season (the national championship game is unfolding on my TV as I type this), I wanted to take a look back at my performance in the [College Football Data predictions contest](https://predictions.collegefootballdata.com/leaderboard). Minimum 400 games picked (not count the NCG, but it should not affect my performance) (I picked 726 games total, for reference), I finished:

* 1st in straight-up picks
* 3rd in ATS picks
* 1st in absolute error
* 4th in mean squared error

So I did pretty well, especially for someone who picked as many games as I did! How did I do it? I employed a number of different techniques -- while I do not want to get into the technical weeds of my process, I figured people might be interested in what strategies I employed in generating my predictions. Here are few techniques I employed that could be useful in your own picking games.

## Use a model

This might seem like a "duh" -- if you are reading this blog, you are probably at least aware of sports analytics -- but I think it is important to discuss a little _why_ to use a model. 

If I told you that entering a week 10 game, the home team had averaged 5.6 yards per carry for the season, and the road team had allowed an average of 4.5 yards per carry for the season, how would you weight that information into your estimate of which team is better? What if I told you that the away team is starting a freshman QB who was a 4-star recruit, while the home team is starting a 5th-year senior? 

In picking games, there is rarely a magic bullet. Sometimes you can get close with team strength approximations like SRS, Elo, LRMC, but even these are incomplete. When looking at a variety of factors, you *need* a model to do the grunt work for you of weighing and adjusting a variety of factors. If you are so inclined, you can take a model estimate and adjust it up or down based on your own interpretation of factors, but do so at your own risk! It can be difficult to out-smart a well-built model in the long run (for evidence of this, check the ATS record on both the CFBD prediction game and the position of "The Market" on the leaderboard of the [NFL Game Data prediction game](https://nflgamedata.com/predict/leaderboard.php)).

Additionally, for picking games in bulk, a model is a necessity. I could not possibly break down all 726 games I put together in any useful manner. A model allows you to automate inputs and outputs, so you can pick more games more efficiently and accurately.

## Listen to Vegas

As good as your model might be, there are a number of factors that can be especially difficult to predict. What is the impact of being down a starting quarterback? What happens if the defensive line is dealing with a COVID outbreak? What if it rains? Tracking all of these factors is extremely difficult -- maybe you can keep track of the depth charts of all 120+ college football teams given a full week to prepare, but you likely have things like "a social life" and "work" to attend to in the meanwhile.

The solution is simple -- let someone else handle it! Vegas bookmakers generally act rationally when adjusting spreads and odds. As shown above, Vegas' predictions are extremely accurate, which makes them difficult to beat. You can flip this disadvantage into an advantage by incorporating bookmaker's odds into your model.

Another benefit is that when something happens that would affect the outcome of a game -- such as a starting quarterback getting hurt at practice -- Vegas responds quickly and accurately. If you regularly update your predictions and effectively incorporate Vegas odds into your model, you can see a significant improvement in accuracy than if you made a single prediction for each game and never updated it.

## Blend Preseason and In-Season

One of the biggest issues pickers encounter when modeling picks is that it can be difficult to pick games early in the season. In the first game of a season, we know very little about how good or bad each team is, and a model that is trained on in-season data will handle early-season predictions overconfidently, if it can handle them at all.

The best approach I found with this was creating two models -- one based entirely on preseason data, and one based on a mixture of preseason data and a rolling-average of in-season data. The preseason model reflected the uncertainty of early-season games, while the in-season model reflected increasing confidence as learned more about each team. I then weighted my predictions based on how deep in the season I was -- in week 1, my predictions were based entirely off of the preseason model estimates, but in week 4, my predictions were a mix of preseason and in-season model estimates. By the end of the season, I was almost wholly relying on the in-season model.

## Automate Everything

My goal this season was to pick every single game. As I said before, that would not have been possible without a model, but it also would not have been possible without a solid pipeline for model picks -- bringing data in, cleaning it, running it through a model, then uploading those predictions to the CFBD site. Bringing in new data is trivial thanks to the [CFBD API](https://api.collegefootballdata.com/api/docs/?url=/api-docs.json), and with a little Selenium I was able to update my process completely. 

Did I actually pick every game? No -- an internet outage while I was on vacation and an external data source implementing a breaking change mid-season meant two weeks for me were de-facto washes. But still, the work I put in to automate the entire process makes producing and uploading picks an arbitrary task. Being able to update my picks so quickly also means I can be more reactive to changes in the game's outlook, just like the Vegas line.

## Conclusion

Picking games accurately is still extremely hard, especially against the spread! Next year, I have some new strategies I want to try to improve my ATS accuracy, which should hopefully improve my performance in other metrics overall as well. For the moment though, I am extremely happy with my performance to date, and I am already looking forward to next season. Go Jackets!

