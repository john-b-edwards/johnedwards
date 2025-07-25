---
title: "Gin and (mono)tonic constraints"
description: How to improve model performance and buy-in with one easy parameter!
date: "2025-05-15"
keywords: 
- machine_learning
slug: monotonic_constraints
tags: 
- r
draft: true
toc: true
---

## Introduction

About a month aog, the exercise app Strava rolled out its new ["Performance Predictions"](https://support.strava.com/hc/en-us/articles/35272903405965-Performance-Predictions) feature to users. The tool takes in as inputs all of the runs a user has tracked on the app and runs them through a machine learning model to project how fast the user could reasonably expect to complete a race of a given distance. Your projected times can be compared to their previous projections to show how much progress you have made in the course of your training. I thought this was a pretty cool feature--[as a runner myself](https://johnbedwards.io/blog/marathon_training/) and as someone who is actively training for a half marathon, I thought this tool would be an interesting way to visualize my progress.

However, I was a little surprised when, after training solidly for a full month, my projected half-marathon time actually got **worse**. Even more surprising--my projected time for all of my other races got significantly better!

<blockquote class="bluesky-embed" data-bluesky-uri="at://did:plc:cu3xuh2dtx4dpqg57hnh3hmo/app.bsky.feed.post/3lp3ybkvkjc2f" data-bluesky-cid="bafyreic2df4p7tzxovjorkewycen6ctsnoga6bax2fulnxigg2apkcmz2u" data-bluesky-embed-color-mode="system">

<p lang="en">

Someone needs to teach the Strava data scientists about monotonicity<br><br><a href="https://bsky.app/profile/did:plc:cu3xuh2dtx4dpqg57hnh3hmo/post/3lp3ybkvkjc2f?ref_src=embed">[image or embed]</a>

</p>

— John Edwards (<a href="https://bsky.app/profile/did:plc:cu3xuh2dtx4dpqg57hnh3hmo?ref_src=embed">@johnbedwards.io</a>) <a href="https://bsky.app/profile/did:plc:cu3xuh2dtx4dpqg57hnh3hmo/post/3lp3ybkvkjc2f?ref_src=embed">May 13, 2025 at 7:46 PM</a>

</blockquote>

<script async src="https://embed.bsky.app/static/embed.js" charset="utf-8"></script>

The documentation for the tool cautions about these changes in timing as such:

> For example, an athlete training for a marathon -- running more weekly volume and focusing on longer intervals -- may see significant improvement in their half-marathon and marathon predictions but not see equivalent improvement in their predictions for the shorter distances. Similarly, an athlete focused on shorter distances -- emphasizing speed and power in their training -- may see more improvement in their **5K** and **10K** predictions than they do in the longer distances where those capabilities are less important.

However... no mention is made of the athlete, who, when training for a half-marathon, sees their projected time in the 5k, 10k, and marathon get better while somehow, their half-marathon time gets worse! What's going on?

## An introduction to monotonicity

My skeet gives away the game here--what Strava's model is missing is a ***monotonic constraint.***

Monotonicity is a term from mathematics, used to describe a function that only trends in one direction. Consider the function $f(x) = x$ -- no matter what, as $x$ increases, $f(x)$ does not decrease. The function is monotonic.

![](public/images/monotonic/monotonic_1.png)

Similarly, $f(x) = -x$ is **also** monotonic. No matter what, as $x$ increases, $f(x)$ *does not increase*.

![](public/images/monotonic/monotonic_2.png)

Now consider $f(x) = x ^ 2$. For values of $x$ beginning at $0$ as $x$ increases, $f(x)$ increases. However, for values of $x$ beginning at $0$ as $x$ **decreases**, $f(x)$ *still increases.* This function is **not** monotonic.

![](public/images/monotonic/monotonic_3.png)

Note my awkward language: I have said "as $x$ increases, $f(x)$ does not decrease". Why not say "as $x$ increases, $f(x)$ increases"? Because for a function $f(x)$ where $f(x)$ does not change over some range of $x$, $f(x)$ may still be monotonic so long as there is not some range of $x$ over which $f(x)$ increases and a different range of $x$ over which $f(x)$ decreases.

Take this step function, $f(x) = \lfloor x \rfloor$. There are many places in this function where for two different values of $x$, $f(x)$ is the same. However, there is no place in which for two different values of $x$ such that for $x_1 > x_2$, $f(x_1) < f(x_2)$. For any two values of $x$, the larger value of $x$ will have the same or larger value of $f(x)$. Thus, this step function **is** monotonic.

![](public/images/monotonic/monotonic_4.png)

## Montonocity in machine learning

How can data scientists leverage monotonic constraints in building more effective machine learning models? I will walk through a few real-world examples where applying a monotonic constraint causes a model to perform better!

These examples will all be in R, as R has a number of packages available for working with monotonically constrained model building. However, R does not have a monopoly on motonocity--there are a variety of packages across a variety of languages capable of building monotonically constrained models!

### Correlated Variables

To begin, let's pull in a dataset from Nick Wan's data science gameshow, Sliced. [In season 1, episode 3](https://www.kaggle.com/competitions/sliced-s01e03-DcSXes/overview), competitors were asked to model the profit generated by a product based on data from a superstore.

We'll attempt to model profit using two variables

### Berkson's Paradox

### Avoiding Overfitting
