---
date: "2021-06-01"
description: The math heavy portion of how LRMC works
draft: false
keywords:
- math
- matrices
- college_football
- college_basketball
- football
- basketball
- lrmc
math: true
pagination: true
slug: lrmc_pt_1
tags:
- math
- old-posts
title: Applying LRMC Rankings to College Football, Part One
toc: false
bsky_thread: https://bsky.app/profile/johnbedwards.io/post/3lcjtzoxy222l
---

_The following [was originally published on the CFBD Blog](https://blog.collegefootballdata.com/talking-tech-applying-lrmc-rankings-to-college-football-part-one/) and has been reproduced here with edits for clarity._

[As Bill covered earlier this season](https://blog.collegefootballdata.com/talking-tech-calculating-srs-in-a-pandemic/), calculating strength-of-schedule adjusted metrics like SRS are a little tricky given how few non-conference games teams play this year. While I think his technique for conference-based SRS is a great attempt at a really difficult problem, I think there is some value in discussing alternative approaches to evaluating team strength this year that do not struggle with the same singularity issues as SRS. I hope to cover one of them in this post and a following post: Logistic-Regression/Markov Chain ratings, or as they are more commonly known, [LRMC](https://www2.isye.gatech.edu/~jsokol/lrmc/about/). 

This post covers the mathematical concepts behind LRMC. If you are unfamiliar with Markov chains this should be a useful read for you! You may feel inclined to skip this post and proceed right to part two with the code (which you can do here), but I highly recommend making sure you have a good understanding of the math first!

## An Overview of LRMC

LRMC is a rating systems devised by Georgia Tech professors Dr. Joel Sokol and Dr. Paul Kvam (largely maintained by Dr. Sokol) designed to predict the results of college basketball games[^1][^2]. As the name implies, the system uses two different modeling techniques to generate ratings, logistic regression and a Markov chain.

To best explain how LRMC works, we can use an example. Imagine a 3 team league consisting of Team A, Team B, and Team C who each play 4 games, a home-and-home series, against the other teams in the league. Here are the results of those games for a given season.

| Home Team | Home Team Score | Away Team Score | Away Team |
|-----------|:---------------:|:---------------:|----------:|
| Team A    | 65              | 62              | Team B    |
| Team B    | 65              | 59              | Team A    |
| Team A    | 64              | 58              | Team C    |
| Team C    | 68              | 64              | Team A    |
| Team B    | 70              | 63              | Team C    |
| Team C    | 73              | 57              | Team B    |

Let's give you the task of figuring out who the best team was. Not terribly easy, given that every team finished exactly 2-2! 

You decide to start your rankings alphabetically, as a prior, and iterate through them to come up with the best answer: so Team A is ranked 1st, Team B 2nd, and Team C 3rd. However, is it fair to rank Team A over Team B? After all, Team B beat Team A by a slightly larger margin on Team B's home court than Team A beat Team B by on Team A's home court. What about Team C over Team B? After all, Team C recorded the largest margin of victory over Team B. But then couldn't we also give some credence to Team A, who beat Team C at home while playing them close on the road? Quickly, we're entering a bit of a loop in terms of evaluating teams. Which is good! This loop is how we'll arrive at an answer. But first, it would be good to know how much weight we should place on certain margins of victories for different teams. This is where the logistic regression part of LRMC comes in.

## Using Logistic Regression to Pick Teams

Taking advantage of college basketball's home-and-home scheduling system for conference games, Sokol and Kvam compiled the margins of victories in those games for the winning teams and ran a logistic regression[^3] on those margins of victories to determine those team's odds of winning the road game of that session (in other words, the logistic function $$ f(x_{a,b}) \approx s_x $$ where <em>x</em><sub><em>a,b</em></sub> is the point differential for Team A that occurred when they hosted Team B and <em>s</em><sub><em>x</em></sub> are the odds Team A will beat Team B on the road). From there, Kvam and Sokol found the value of <em>x</em> for which <em>s</em><sub><em>x</em></sub> = 50%, then used that value to determine the home field advantage and adjust their equation for <em>s</em><sub><em>x</em></sub> to find <em>r</em><sub><em>x</em></sub>, the odds of Team A winning on a neutral court (or, the true talent win probability for Team A).[^4]

I'll do the hard work for you; here are values of  <em>r</em><sub><em>x</em></sub> for each of the games (the odds the home team is better than the away team based on the point differential of each game)[^5][^6]:

| Home Team | *r<sub>x</sub>*   | Away Team |
|-----------|:------:|----------:|
| Team A    | 46%    | Team B    |
| Team B    | 49%    | Team A    |
| Team A    | 51%    | Team C    |
| Team C    | 46%    | Team A    |
| Team B    | 50%    | Team C    |
| Team C    | 61%    | Team B    |

With these probabilities established, we can now do some looping, using the second part of LRMC: the Markov chain. 

## An Introduction to Markov Chains

Markov chains are an exceedingly useful modeling technique designed to simulate random events that have a sort of stickiness to them, as a value moves from one state to another. 

As an introduction, here is a sample Markov chain problem: Imagine that you love to eat two different foods, and have one of either tacos or pizza every day. However, if you ate pizza the day before, there's a 70% chance you feel like switching it up the next day and have tacos, and a 30% chance you feel like having another slice the next day. If you ate tacos the day before, there's a 50% chance you'll have more tacos today, and a 50% chance you'll have pizza. Here's a far reaching question: given that you ate tacos yesterday, what will you want to eat 5 days from now?

To answer this question, we can use something called a transition matrix. Think of a transition matrix like a magic box – we put numbers in the right side and it spits an output from the top. This matrix is represented as _P_. 

$$
P = \begin{bmatrix}
   0.5 & 0.5\\\\\\
   0.7 & 0.3
\end{bmatrix}
$$

If you're confused about what _P_ stands for, just think about it this way: the matrix is just restating the probabilities we discussed in the opening paragraph of this section.

|           Given that I...   | ...there is a __ chance I will eat tacos today. | ...there is a __ chance I will eat pizza today. |
|---------------------|----------------------|----------------------|
| **...ate tacos yesterday...** | 50%                  | 50%                  |
|**...ate pizza yesterday...** | 70%                  | 30%                  |

Our input is essentially a binary representation of the row names of _P_, with a 1 indicating what we ate and a 0 representing what we did not eat. We'll represent this matrix as:

$$
x^{(0)} = \begin{bmatrix} 1 & 0 \end{bmatrix}
$$

To obtain the odds of what we eat tomorrow, we just need to multiply our matrix by _P_.

$$
x^{(0)} \cdot P = x^{(1)}
$$

$$
\begin{bmatrix} 1 & 0 \end{bmatrix} \cdot \begin{bmatrix}
   0.5 & 0.5\\\\\\
   0.7 & 0.3
\end{bmatrix} = x^{(1)}
$$

$$
\begin{bmatrix} 1 \cdot 0.5 + 0 \cdot 0.7 & 1 \cdot 0.5 + 0 \cdot 0.3\end{bmatrix} = x^{(1)}
$$

$$
\begin{bmatrix} 0.5 & 0.5 \end{bmatrix} = x^{(1)}
$$

Little surprise here: as we previously said, when we ate tacos yesterday, there is a 50% chance we eat tacos today, and a 50% chance we eat pizza today.

What if we want to know what we eat tomorrow? Then the output of _x_<sup>(0)</sup> times _P_ becomes _x_<sup>(1)</sup> and we repeat the process over again.

$$
x^{(1)} \cdot P = x^{(2)}
$$

$$
\begin{bmatrix} 0.5 & 0.5 \end{bmatrix} \cdot \begin{bmatrix}
   0.5 & 0.5\\\\\\
   0.7 & 0.3
\end{bmatrix} = x^{(2)}
$$

$$
\begin{bmatrix} 0.5 \cdot 0.5 + 0.5 \cdot 0.7 & 0.5 \cdot 0.5 + 0.5 \cdot 0.3\end{bmatrix} = x^{(2)}
$$


$$
\begin{bmatrix} 0.6 & 0.4 \end{bmatrix} = x^{(2)}
$$

So given that we ate tacos yesterday, there is a 60% chance we eat tacos tomorrow, and a 40% chance we eat pizza tomorrow.

This is a Markov chain in action! We have taken an initial state vector (_x_<sup>(0)</sup>), multiplied it by our transition matrix (_P_), then taken the outcome of that (_x_<sup>(1)</sup>) and multiplied it again by P to find _x_<sup>(2)</sup> – the so-called "chaining". For the algebraically inclined, you will find that given an initial state _x_<sup>(0)</sup> and a constant transition matrix P, to find _x_<sup>(_k_)</sup>, you can skip some steps and just use this formula[^7]:

$$
x^{(0)} \cdot P^{k} = x^{(k)}
$$

So on the 5th day, given that you ate tacos yesterday, your odds of eating tacos versus pizza are $$x^{(5)} = \begin{bmatrix} 0.583 & 0.417 \end{bmatrix}$$ or a 58% chance of eating tacos versus a 42% chance of eating pizza.

You'll notice that as the value of k increases, the probability of us landing in a given state approaches some finite value. For example:

For _k_ = 1, there is a 50% chance we will eat tacos. 

For _k_ = 2, there is a 60% chance we will eat tacos.

For _k_ = 3, there is a 58% chance we will eat tacos.

For _k_ = 4, there is a 58.4% chance we will eat tacos.

Note that we approach this same state even if we had eaten pizza yesterday: 

$$
\text{if } x^{(0)} = \begin{bmatrix} 1 & 0 \end{bmatrix} \text{ then } x^{(5)} = \begin{bmatrix} 0.583 & 0.417 \end{bmatrix}
$$

$$
\text{if } x^{(0)} = \begin{bmatrix} 0 & 1 \end{bmatrix} \text{ then } x^{(5)} = \begin{bmatrix} 0.584 & 0.416 \end{bmatrix}
$$

In other words, this system approaches a "steady state", where in the long run, our odds of eating pizzas versus tacos on a given day are two constant values. Not every Markov chain approaches a steady state[^8], but ours does – how do we find out what it is?

We have two ways of finding that steady state. We can simply calculate what our final state vector will look like at some time in the far-off future, like _x_<sup>(0)</sup> • _P_<sup>1000</sup>. However, we can solve for an exact answer by calculating the eigenvector of our transition matrix _P_[^9]. In either approach, you will arrive at a steady state of [0.583, 0.417][^10].

## Putting it all together

Let's return to our teams and their records. Previously, we calculated <em>r</em><sub><em>x</em></sub> for each game, the odds that the home team would beat the away team on a neutral court given the point differential in that game. We can take these probabilities and convert them into a transition matrix to generate our rankings. 

Borrowing from the analogy first deployed by Callaghan, Mucha, and Porter in proposing the application of this method to sports[^11], imagine a room full of monkeys who randomly choose a game and determine whether or not the home team or the road team is better using <em>r</em><sub><em>x</em></sub>. A transition matrix will not just reflect the odds of which team is better, but the odds of choosing a particular game as well. To generate our transition matrix, we sum up <em>r</em><sub><em>x</em></sub> for games involving a particular team (let's say, Team X) by team. In the row of Team X, we place each team's sum of <em>r</em><sub><em>x</em></sub> in each team's corresponding column (including Team X). Then, we divide all of the values in the row by the number of games Team X played.

|            | Team A | Team B | Team C |
|------------|--------|--------|--------|
| **Team A** | 50%    | 26%    |  24%   |
| **Team B** | 24%    | 48%    |  28%   |
| **Team C** | 26%    | 22%    |  52%   |

Let's say [1,0,0] represents an initial state vector, a monkey with a random prior belief that Team A is better than Team B or Team C. To evaluate their prior, they watch a random game that Team A played. There is a 50% chance that they pick a game where Team A faced Team B, and a 50% chance that they pick a game where Team A faced Team C. If they pick a game where Team A faced Team B, there is a 52% chance they walk away from that game thinking that Team B is better than Team A; if they pick a game where Team A faced Team C, there is a 48% chance that they walk away thinking Team C is better than Team A. Thus, there is a 50% * 48% + 50% * 52% = 50% chance they stay with Team A in our initial state, a 50% * 52% = 26% chance they think Team B is better, and a 50% * 48% = 24% chance they think Team C is better. 

Or, more succinctly:

$$
\begin{bmatrix} 1 & 0 & 0 \end{bmatrix} \cdot \begin{bmatrix}
   0.50 & 0.26 & 0.24\\\\\\
   0.24 & 0.48 & 0.28\\\\\\
   0.26 & 0.22 & 0.52
\end{bmatrix} = \begin{bmatrix} 0.5 & 0.26 & 0.24 \end{bmatrix}
$$

So which team is best? Our transition matrix approaches an output vector of [0.33, 0.32, 0.35], indicating that Team C is likely the best team in our league, followed by Team A with Team B just a hair behind. 

## Conclusion

By pairing simple modeling tools such as logistic regression and Markov chains together, LRMC becomes a powerful predictive algorithm: Kvam and Sokol found that LRMC consistently outperformed considerably more complicated algorithms in picking college basketball games[^1]. LRMC values are also fairly easily regressed to estimate values such as expected point spreads and win probabilities, so while one might not get a nice and tidy "This team would beat an average team by X points" value like SRS, LRMC yields useful values regardless. But ultimately, the beauty of LRMC results from its simplicity: with just four basic inputs (home team, away team, point differential, and court location), it performs well enough to be included as an important input for college basketball aggregate rankings such as 538's NCAA March Madness projections[^12].  

Having established our understanding of the math behind LRMC, we can now implement an algorithmic ranking of LRMC in college football. You can find part two of this article here.

[^1]: [Kvam, P. and J. Sokol, "A logistic regression/Markov chain model for NCAA basketball", Naval Research Logistics (2006), 53, pp. 788-803](https://www2.isye.gatech.edu/~jsokol/ncaa.pdf)
[^2]: [Brown, M. and J. Sokol, "An Improved LRMC Method for NCAA Basketball Prediction", Journal of Quantitative Analysis in Sports (2010), Vol. 6, Iss. 3, Article 4](https://econpapers.repec.org/article/bpjjqsprt/v_3a6_3ay_3a2010_3ai_3a3_3an_3a4.htm)
[^3]: A logistic regression model, for the unacquainted, is similar to a linear regression model except instead of predicting a continuous value, such as a point differential, it instead gives a probability that an output falls into one of two classes – in this instance, the odds of a win or a loss.
[^4]: Sokol goes into this calculation in his papers, but I will briefly explain it here: Imagine that based on the logistic regression of a history of home-and-home games, _f_(13) = <em>s</em><sub>13</sub> = 50%. Given that the 13 point spread results not just from Team A's expected point differential against Team B on a neutral court (represented as _d_) but also from the home court advantage (represented as _h_), then 13 = _d_ + _h_. For Team A to have 50% odds of winning versus Team B, the expected point spread must be even, and the home court advantage should go equally against Team A for being on the road, hence, 0 = _d_ - _h_, or _d_ = _h_. Plugging this into 13 = _d_ + _h_, we find 13 = 2 _h_, or _h_ = 6.5. From there: $$ f(x_{a,b} + h) \approx r_x $$ <em>h</em> here is just a hypothetical value; the empirical nature of home court advantage will vary depending on the sport, time period, and other circumstances.
[^5]: These figures are calculated using a quick-and-dirty logistic regression model of college basketball scores from 2008 to 2020, without data cleaning. The model has coefficients _a_ = 0.05, _b_ = -0.65, and _h_ = 6.5  such that the odds of Team X beating Team Y on a neutral court given that Team X previously beat Team Y at home by x points is $$e^{0.05(x + 6.5) - 0.065)} / (1 - e^{(0.05(x + 6.5) - 0.065))}$$ Again, these figures will vary from sport to sport as well as situation to situation.
[^6]:  Note that for Team B @ Team A and Team A @ Team C, despite that the home team won both of those games, the odds are that the home team is worse than the away team because the home team. Why is this? In those games, the home team won by a very close margin; our logistic regression model believes the home court advantage in points to be greater than the margin of victory for those home teams, so it believes that the away team is better. Think of it like covering the spread.
[^7]: $$\text{if } x^{(1)} = x^{(0)} \cdot P \text{ and } x^{(2)} = x^{(1)} \cdot P \text{ then } x^{(2)} x^{(1)} \cdot P^2$$
[^8]: Consider a Markov chain with states A, B, and C, where if you are in state A, there is a 50% chance you will move to state B and a 50% chance you will remain in state A; if you are in state B, there is a 100% chance you will move to state C; if you are in state C, there is a 100% chance you will move to state B. In the long run, any initial state will end up in either state B or C, but which of state B or state C they will be in at a given time is almost wholly random! This Markov chain lacks a steady state.
[^9]: For a detailed instruction on how to calculate eigenvectors of transition matrices, check out [this video](https://www.youtube.com/watch?v=d9-L2AmngKE). 
[^10]: Note that our steady state might look identical to _x_<sup>(5)</sup>, but they are not the same – _x_<sup>(5)</sup> carried out is [0.5832, 0.4168], our steady state carried out is [0.58333..., 0.41666...].
[^11]: [Callaghan, T., P.J. Mucha, and M.A. Porter, "The Bowl Championship Series: A Mathematical Review", Notices of the American Mathmatical Society (2004), 51, pp. 887-893](https://www.ams.org/journals/notices/200408/fea-mucha.pdf)
[^12]: [Silver, N.,  "How FiveThirtyEight’s March Madness Bracket Works", FiveThirtyEight, (2015).](https://fivethirtyeight.com/features/march-madness-predictions-2015-methodology/)