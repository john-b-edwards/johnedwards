---
date: "2024-11-18"
description: My writeup of my Royal Statistical Society Olmypic Prediction contest entry
draft: false
keywords:
- machine_learning
- olympics
slug: olympic_prediction_contest
tags:
- r
title: Predicting the Olympics
toc: true
---

![](https://upload.wikimedia.org/wikipedia/commons/thumb/b/bb/%C3%89preuve_Athl%C3%A9tisme_Jeux_Olympiques_2024_Stade_France_-_Saint-Denis_%28FR93%29_-_2024-08-02_-_138.jpg/1024px-%C3%89preuve_Athl%C3%A9tisme_Jeux_Olympiques_2024_Stade_France_-_Saint-Denis_%28FR93%29_-_2024-08-02_-_138.jpg)

*Stade de France; Chabe01, [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0), via Wikimedia Commons*

I had the pleasure of competing in the [Royal Statistical Society's 2024 Olympics prediction contest.](https://www-users.york.ac.uk/~bp787/RSS_pred_comp_2024.html) I also had the pleasure of winning! While I feel that my code is too messy to share at present and I lack the motivation to clean it up for public consumption, I did want to discuss at a high level my approach and lessons learned from this competition.

## Contest Overview

This contest involved predicting the medal table for the 2024 Olympics. If you watched the Olympics this year, at some point on the broadcast, you were likely to see a table that represented which country was leading the gold medal count, the total medal count, and so on: this is the medal table! The top of the final medal table for this year's Olympics looked something like this:

| Rank | NOC                | Gold | Silver | Bronze | Total |
| ---- | ------------------ | ---- | ------ | ------ | ----- |
| 1    | 🇺🇸 United States | 40   | 44     | 42     | 126   |
| 2    | 🇨🇳 China         | 40   | 27     | 24     | 91    |
| 3    | 🇯🇵 Japan         | 20   | 12     | 13     | 45    |
| 4    | 🇦🇺 Australia     | 18   | 19     | 16     | 53    |
| 5    | 🇫🇷 France        | 16   | 26     | 22     | 64    |

The goal of the contest was to predict the order of the "NOC"s (National Olympic Committees) within the medal table, graded by [Kendall's tau.](https://en.wikipedia.org/wiki/Kendall_rank_correlation_coefficient) To calculate Kendall's tau, you can perform a pairwise comparison of every item ranked:
* If you predict that Item 1 is higher than Item 2 and you are correct, you get +1 for that comparison. 
* If you predict that Item 1 is higher than Item 3 and you are wrong, you get -1 for that comparison. 
* If you predict that Item 2 is higher than Item 3 and Item 2 and Item 3 are tied, you get 0 for that comparison.

In the bulleted example above, our Kendall's tau is the average of our score, or (1 - 1 + 0) / 3 = 0.

For the RSS competition, the official ranking criteria was gold medals, with ties broken by silver medals, then bronze medals in that order. Any countries with the same number of gold, silver, and bronze medals were considered exactly tied.

The final submission needed to be just a ranking of the NOCs in our predicted order. So a submission would have looked something like:

1. United States
2. China
3. Japan

...

202. Zimbabwe
203. Belarus
204. Russia

## My Approach

### Modeling the structure of the Olympics

I tried to tackle this as a threefold problem:
1. I needed to capture data about what events were going on in the Olympics, who was competing in each event, and how medals were awarded for these events.
2. I needed to model the probability of individuals or teams winning different events.
3. I needed to use #1 and #2 to capture each country's expected gold, silver, and bronze medals, then rank countries based on this criteria.

I realized that if I could do #1 effectively, that would take much of the predictive burden off #2 and yield the best estimate for #3. For example, suppose all I knew was what countries were competing in what events and not what their odds of winning those respective events were. If I assumed even odds of each country winning each event, I would have achieved a solid Kendall's tau, as the countries that participated in the most events had the most opportunities to capture medals (something that played out in the actual final medal table).

### Web scraping

To pull down the events, I was able to find an [API for the summer Olympics](https://sph-s-api.olympics.com/summer/schedules/api/ENG/schedule/day/2024-07-29) that reflected the complete schedule for the games: what events were occurring, who was competing in them, what medal events there were, etc. After considerable wrangling, I was able to filter down to just the medal events and determine which countries were competing in each event.

A few events required some special wrangling:
* For artistic gymnastics, gymnasts could earn individual medals in different events (such as the vault or floor routine), and then their scores in these events contributed to winning an ["Individual All-Around"](https://en.wikipedia.org/wiki/Gymnastics_at_the_2024_Summer_Olympics_%E2%80%93_Women%27s_artistic_individual_all-around) medal _and_ a ["Team All-Around"](https://en.wikipedia.org/wiki/Gymnastics_at_the_2024_Summer_Olympics_%E2%80%93_Women%27s_artistic_team_all-around) medal. This structure is fairly unique to this event and it creates opportunities for counties to earn two extra medals for the medal table beyond the individual events.
* The participants for the Mixed Doubles Tennis event [were not announced until after the Olympics were underway,](https://www.itftennis.com/en/news-and-media/articles/mixed-doubles-entries-confirmed-for-paris-2024-olympic-tennis-event/) so at the time I compiled my submission, it was impossible to know which countries would be competing for a medal. I took an educated guess based on which countries had sent at least a men's and women's tennis player to the Olympics.
* Some athletes were not competing under an individual country but under either the [Individual Neutral Athletes NOC](https://olympics.com/ioc/paris-2024-individual-neutral-athletes) or the [IOC Refugee Olympic Team.](https://olympics.com/ioc/refugee-olympic-team) These NOCs were not listed under the sample submission on the website, so I ended up omitting them. 

### Modeling Individual Events

Now that I had all of the events and which countries were competing, my next task was to model the outcomes of each of the events as best I could. I took a twofold approach to get high-quality, well-calibrated estimates.

#### Betting Odds

There was a robust betting market [for the Olympics.](https://www.entaingroup.com/news-insights/latest-news/2024/olympics-2024-record-interest-in-betting-on-the-olympics-for-entain/) While some of these markets were skewed towards the United States or European countries where gambling is legal and popular (as bettors from these countries would tend to bet on their own countries to win), for smaller events with odds, these predictions were reasonably well calibrated (and in hindsight, the markets being skewed towards the United States turned out quite well for my Kendall's tau). For any events where there was betting information available (via FanDuel), I pulled in these odds and used them as my probabilistic estimates of a team or athlete winning.

#### Ranking with XGBoost

Only some events had betting information available, so I needed a fallback when it was missing. I pulled in the outcomes of previous Olympic events and grouped them by discipline as intuitively as I could (for example, all of the gymnastic events were pooled together). 

From there, I fit a simple XGBoost model to predict the "rank" of each country in each event, [using the `rank:pairwise` objective,](https://xgboost.readthedocs.io/en/latest/tutorials/learning_to_rank.html) given the country, the discipline of the event, and the number of competitors in each event. I fit on the past 30 years Olympic events as a means of capturing each country's recent culture and investment in each discipline. For each event, this produced an abstract continuous value representing which country the XGBoost model felt was likely to win (so if the XGBoost model produced a value of 10 for the United States and 5 for Japan, it felt that the United States was more likely to win the event than Japan).

After getting these values, I ran the rankings through [the Harville model for modeling ordinal outcomes](https://rdrr.io/cran/ohenery/man/ohenery.html) which allowed me to get a scaled probabilistic estimate of which country was likely to win an event. 

#### Extracting silver and bronze odds

After combining the betting odds and my own estimates, I had probabilities that a country would win gold but not silver or bronze. To grab these, I did some abominable math using conditional probabilities, the likes of which would probably cause Harville himself to spin in his grave.

To walk through a quick example of how these conditional probabilities worked: suppose there's a 3-team event where:
* Team A has 70% odds of getting 1st place
* Team B has 20% odds of getting 1st place
* Team C has 10% odds of getting 1st place. 

For this example, what are the odds that Team B gets second place?

* If Team A gets 1st place, and if we assume that the odds of getting first place among the remaining teams (Team B and Team C) are proportional to their odds of beating their remaining opponents (so Team B has 67% odds of getting first place in a competition between just B and C), then the odds that Team B gets 2nd place given that Team A gets first place are 67%. 
* With these same assumptions, the odds that Team B gets 2nd place given that Team C gets first place are 22%. 
* So the odds that Team B gets 2nd place are `pr(Team A 1st) * pr(Team B beats Team C) + pr(Team C 1st) * pr(Team B beats Team A)` = 47% + 2% = 49%. 

I expanded this logic for calculating the odds of getting a bronze medal as well.

As I said above, this is almost certainly some bad math: specifically the assumption that if B's odds of getting 1st place among A, B, and C are 20%, then B's odds of getting 1st place among B and C are 66%. However, it yields reasonable-looking and well-calibrated estimates for the odds of getting silver and bronze medals.

#### A faulty assumption!

I was so concerned with the bad math that I was doing above that I made a crucial error in my modeling process. In these probabilities, I assumed that if a country won a gold medal, they were excluded from winning silver and bronze. That is the case for team events, but it is certainly not the case for individual events! It is possible for a country to win multiple medals for the same event, and in rare cases, [they can win all three!](https://en.wikipedia.org/wiki/List_of_medal_sweeps_in_Olympic_athletics)

Fortunately, this mistake did not prove costly, for two reasons:
1. The primary ranking criteria was gold medals, and my estimates of gold medals were well calibrated.
2. For the first Olympics since the 2000 Olympics, no country recorded a podium sweep!

I got rather lucky that my biggest mistake in this process ultimately did not come back to haunt me.

### Putting it all together

I now had all of the Olympic events and each country's odds of winning a particular medal. By summing up each country's odds of winning an event across the Olympics, I obtained their number of expected gold medals (and then silver and bronze with the same logic). From there, I tried to get a little tricky with my methodology given the tiebreaker methodology. 

1. I ordered countries by their number of expected gold medals. 
2. I rounded each country's number of expected gold medals and grouped them by these rounded figures. 
3. Within these groups, I ordered them by their number of silver medals. 
4. I repeated this round-group-sort process for silver and bronze medals. 

This resulted in some countries being ranked above other countries with higher numbers of projected gold medals. For example, I projected Hungary for 5.4 gold medals and 5.3 silver medals, but I ranked Poland over them with 5.3 gold medals and 5.9 silver medals. The logic behind this is that these countries were likely to tie in terms of total gold medals, and given that they were likely to be tied, I wanted to rank the country with more silver medals higher.

## Final Results

After watching the medal tables with baited breath, I managed to come out on top with the best Kendall's tau!

| Forecaster                    | Rank | Tau   |
| ----------------------------- | ---- | ----- |
| John Edwards                  | 1    | 0.599 |
| mohamed                       | 2    | 0.564 |
| OlymPicks                     | 3    | 0.548 |
| Hammers_O’Callaghan           | 4    | 0.542 |
| ALEC                          | 5    | 0.538 |
| fa2410                        | 6    | 0.536 |
| AJM                           | 7    | 0.533 |
| ChristopherWharton            | 8    | 0.531 |
| JKH                           | 9    | 0.512 |
| Orla_S                        | 10   | 0.510 |
| KaitoGoto                     | 11   | 0.508 |
| JoePenn                       | 13   | 0.507 |
| JohnDSouza                    | 13   | 0.507 |
| JPN2020                       | 14   | 0.506 |
| HarrySnart                    | 15   | 0.499 |
| WilliamBowers                 | 16   | 0.494 |
| LexieB                        | 17   | 0.493 |
| ANIK                          | 18   | 0.485 |
| Rank_Deficient                | 19   | 0.479 |
| WeatherQuant                  | 20   | 0.458 |
| Kizmet24                      | 21   | 0.426 |
| tghaynes                      | 22   | 0.395 |
| Everyone_is_(equally)_awesome | 23   | 0.000 |

I suspect that the differentiating factor here for my submission was my work structuring the contest and navigating the tiebreaker logic. As I mentioned before, I made some mistakes with my modeling work, and I suspect that the event-level predictions were somewhat underfit.

Still, I was pretty pleased with my predictions. I felt like my biggest takeaway was the impact of the work I did to ensure my predictions were well-calibrated and captured the structure of Olympic events. I felt that doing a good amount of research and pulling in the individual events and participants yielded better results than if I had simply taken past Olympic medal tables and fed them into a model.

A big thank you to Ben Powell for running this competition! I'm looking forward to the next Olympics and perhaps another go-round where I remember that a country can take home more than one medal at some events!