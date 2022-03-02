---
date: "2022-03-01"
description: Predicting the point value of Scrabble turns
draft: false
keywords:
- board_games
- machine_learning
math: true
slug: predicting-scrabble-point-values
pagination: true
twitterImg: https://i.imgur.com/pL1MNW7.png
tags:
- julia
title: Using Flux.jl to model Scrabble turns
toc: true
---

![](https://i.imgur.com/pL1MNW7.png)

I recently participated in a [Kaggle community competition](https://www.kaggle.com/c/scrabble-point-value) where the objective was to predict the point value of a word played on the 20th turn of a Scrabble game given a dataset of Scrabble games played on [Woogles.io](woogles.io), an online Scrabble website. After learning a lot about neural networks and Julia, I managed to swing first place! Below is my write-up of my solution, which also represents my first serious foray into working with Julia. I was really happy with this, learned a lot, and have some ideas for follow up projects, so this may not be the last you hear of Scrabble on my blog. Regardless, I enjoyed this project and I hope you enjoy my write-up!

# Introduction

For this competition, we will attempt to estimate the point value of a Scrabble turn given the initial board state and the player's current rack of letters. We are specifically seeking the point value of the 20th turn of games in our test dataset.

## Literature

### Scrabble

*Scrabble* is a popular word board game. 2-4 players begin the game with a rack of seven randomly-drawn letter tiles, drawn from a bag of 100 letter tiles. The first player places the tiles either horizontally or vertically at the center of a 15x15 grid to form a valid word, as determined by a dictionary agreed upon prior to the start of the game. Players then take turns attempting to place any number of letter tiles from their rack on the board to form valid words, intersecting with existing words on the board. When a player plays a word, they score a number of points depending on the point values of the individual letters used to form the word along with any multipliers they receive from placing a tile on a bonus spot, marked on the board. Players may double their point total if they play all seven tiles from their rack on a given turn, a move known in the Scrabble community as a "bingo". Players can also challenge the validity of a word played by another player -- if the challenge is successful, the player who played the offending word loses their turn. After players play tiles from their rack, they replenish their rack with up to seven tiles from the bag, until there are no more tiles remaining. The game ends when no players can place valid words on the board.

There are several variations on the standard Scrabble ruleset. For example, some variations will place time limits on how long players have to place a word, and players may incur a penalty if they fail to place a word in the requisite time frame. Other formats will penalize unsuccessful challenges.

Scrabble is played all over the world and in a variety of languages. The website [Woogles.io](woogles.io) allows for English-language players to participate in online matchups against each other or against bots, under a variety of rulesets.

### CNNs and Board Games

The nature of Scrabble makes it difficult to capture the state of the game such that it is easily read by machine learning models that take in tabular data, such as a random forest or XGBoost. For example, a player's ability to play a word in Scrabble is highly dependent upon what letters are on the board and what location those letters are in. This ability is also dependent on the letters in their rack, such that the letters on the rack can be combined into a valid word in the Scrabble dictionary.

Previously, researchers have employed convolutional neural networks (CNN) with great success to teach computers to play board games. The nature of a board game with a grid-space, a limited move-set, and tokens of different values can be captured using tensor representations, which can then be fed into convolutional layers along with a cost function to determine an optimal move. Some approaches have relied upon feeding expert moves to the neural network (Oshri, 2015), while others have relied upon training the network to play against itself repeatedly (Chellapilla & Fogel, 1999) (Silver et al., 2017).

Agarwal (2019) takes these insights and applies them to the game of Scrabble, providing a framework for encoding Scrabble boards for a CNN and training on expert Scrabble games.

> A 15 × 15 Scrabble board `b` is encoded into a 15 × 15 × 28 feature vector `v(b)` where the third dimension pertain[s] to the different type[s] of features used in our encoding of the Scrabble board. For a particular feature, each board position is a given a value. We used the following features for our encoding:
> 1. Whether a position is blank or not
> 2. Whether a position contains a particular [letter of the] alphabet, leading to a total of 26 features for the letters A-Z
> 3. The score of the tile placed on a position. All positions not containing any tiles are given a score of zero except ... [bonus tiles] which are given a score of -1

### Julia

Julia is a high-performance programming language. Julia is designed such that it "looks like Python, feels like Lisp, and runs like Fortan". Julia has been effectively used in a variety of academic and professional contexts, especially machine learning tasks, owing to its high performance and easy interoptability with GPU computing. Julia's intuitive and rapid array processing features, coupled with access to the high performance  [`Flux.jl`](https://github.com/FluxML/Flux.jl) machine learning library, make it an ideal choice for working with and modeling board game data.

## Our approach

We will take a twofold approach for modeling. First, we will attempt to determine if a word is challenged and the success of the challenge exactly using the provided metadata. For remaining turns, we will rely on a neural network to estimate the expected score of the word.

It is noteworthy that our approach will vary from the referenced frameworks in several ways given the constraints of this competition. Below, each consideration is listed and paired with the adjustment to the methodology.

| Consideration | Adjustment |
| --------- | ------- |
| Outside of Agarwal (2019), previous approaches have explored games with either a set number of tokens (such as checkers or chess) or an indeterminate number of tokens (such as Go). Scrabble requires players to place letter tiles on the board from their "rack", which consists of seven or fewer letter tiles. We may also wish to incorporate information such as the turn of the game. | We will use a concatenation layer in our neural network to combine the rack and additional metadata into our model, as Ningrum et. al (2021) do in constructing a classifier to identify malignant melanomas from images of suspected melanomas and patient metadata.
| The above approaches have generally sought to determine the best move available to a computer player by means of optimizing a loss function. In our problem, we are not attempting to determine the best move, rather, we are attempting to predict the score of the next move. | We will train to minimize the RMSE between our estimate of the score of a turn and the actual score of the turn.|
| Whereas Oshri (2015) relied upon the data of expert players to train a computer player, our data consists of players of varying skill levels playing against an AI. | We will incorporate a player's rating prior to the beginning of the game into our model as a part of our training data.|

We will rely on the Julia language (Bezanson et. al, 2017) to parse our data and implement our neural network using the `Flux.jl` package. The following external packages were used in this notebook:
* [`BSON.jl`](https://github.com/JuliaIO/BSON.jl)
* [`CSV.jl`](https://github.com/JuliaData/CSV.jl)
* [`CUDA.jl`](https://github.com/JuliaGPU/CUDA.jl)
* [`Dataframes.jl`](https://github.com/JuliaData/DataFrames.jl)
* [`Dictionaries.jl`](https://github.com/andyferris/Dictionaries.jl)
* [`MixedModels.jl`](https://github.com/JuliaStats/MixedModels.jl)
* [`Plots.jl`](https://github.com/JuliaPlots/Plots.jl)
* [`StatsPlots.jl`](https://github.com/JuliaPlots/StatsPlots.jl)

Additionally, we are using a CUDA-enabled NVIDIA RTX 2070 Super GPU to speed up processing our data and training our model. We also rely on the [Kaggle API](https://github.com/Kaggle/kaggle-api) to download our data and submit predictions from our notebook.


# Grabbing data

To begin, we will grab data for the competition with the Kaggle API.

```julia
run(`kaggle competitions download -c scrabble-point-value --force`)
run(`mkdir -p data`)
run(`powershell -command "Expand-Archive -Force scrabble-point-value.zip data"`)
```

```
Downloading scrabble-point-value.zip to c:\Users\edwar\Documents\scrabble_jl

100%|██████████| 12.4M/12.4M [00:00<00:00, 60.4MB/s]
Process(`powershell -command 'Expand-Archive -Force scrabble-point-value.zip data'`, ProcessExited(0))
```

We can pull our data into Julia using `DataFrames.jl` and `CSV.jl`.

```julia
using DataFrames
using CSV
ENV["COLUMNS"] = "1000"; # for display purposes
```

The data provided for this competition consists of approximately 26,000 Scrabble games played against the "BetterBot" AI player on the Scrabble website Woogles.io. We are provided with:

## `turns_train.csv`
A training dataset of completed games from Woogles.io.
* Each game and player have unique identifiers.
* Games are stored in tabular format, with each row representing a turn. Each game is represented as a sequential series of turns.
* Each player's rack immediately prior to their turn is stored as a column within this dataset.
* Each player's move is also listed, along with the number of points they scored on the turn and their game score following their turn.


```julia
turns_train = CSV.read("data/turns_train.csv", DataFrame)
turns_train[turns_train.game_id .== 1, :]
```

|    | game_id | turn_number |  nickname |    rack | location |     move | points | score | turn_type |
|---:|--------:|------------:|----------:|--------:|---------:|---------:|-------:|------:|----------:|
|    |   Int64 |       Int64 |  String31 | String7 |  String3 | String15 |  Int64 | Int64 |  String15 |
|  1 |       1 |           1 |      BLUE | ?EEEFJN |       8G |      JEE |     20 |    20 |      Play |
|  2 |       1 |           2 | BetterBot | CEGILRT |       J6 |     GIRL |     20 |    20 |      Play |
|  3 |       1 |           3 |      BLUE | ?EFNRRU |       7I |      F.R |     19 |    39 |      Play |
|  4 |       1 |           4 | BetterBot | CEEITYZ |      10G |     CITY |     34 |    54 |      Play |
|  5 |       1 |           5 |      BLUE | ?ELNORU |       6J |      .UL |      6 |    45 |      Play |
|  6 |       1 |           6 | BetterBot | AEEIIIZ |       L6 |      .EI |     11 |    65 |      Play |
|  7 |       1 |           7 |      BLUE | ?ENOORU |      G10 |      .OO |      5 |    50 |      Play |
|  8 |       1 |           8 | BetterBot | AEIIPSZ |      H12 |     PIES |     34 |    99 |      Play |
|  9 |       1 |           9 |      BLUE | ?ENNORU |      15A | REUNiON. |     74 |   124 |      Play |
| 10 |       1 |          10 | BetterBot | AADEFIZ |      B12 |     FAZ. |     32 |   131 |      Play |
| 11 |       1 |          11 |      BLUE | ANOOQUV |      14B |      .OO |     16 |   140 |      Play |
| 12 |       1 |          12 | BetterBot | ADEITUX |      12B |     .AUX |     28 |   159 |      Play |
| 13 |       1 |          13 |      BLUE | ACNQUVY |      11D |       YA |     33 |   173 |      Play |
| 14 |       1 |          14 | BetterBot | ABDDEIT |      14F |     DI.D |     19 |   178 |      Play |
| 15 |       1 |          15 |      BLUE | CNPQUVW |       K5 |      P.. |     10 |   183 |      Play |
| 16 |       1 |          16 | BetterBot | AABEELT |      E11 |      ..E |     10 |   188 |      Play |
| 17 |       1 |          17 |      BLUE | ACNQUVW |       M8 |      NAW |      9 |   192 |      Play |
| 18 |       1 |          18 | BetterBot | AABELLT |       M3 |     BALL |     17 |   205 |      Play |
| 19 |       1 |          19 |      BLUE | BCNQUVW |       4L |      W.B |     16 |   208 |      Play |
| 20 |       1 |          20 | BetterBot | AAENTTT |       N9 |     TEAT |     15 |   220 |      Play |
| 21 |       1 |          21 |      BLUE | CKNQTUV |      12L |      CU. |     10 |   218 |      Play |
| 22 |       1 |          22 | BetterBot | AHIMNTT |       L1 |     THA. |     15 |   235 |      Play |
| 23 |       1 |          23 |      BLUE | DEKNQTV |       O8 |     VENT |     32 |   250 |      Play |
| 24 |       1 |          24 | BetterBot | EGIMMNT |       1J |   ME.ING |     27 |   262 |      Play |
| 25 |       1 |          25 |      BLUE | DEGKQSS |      K12 |      SEG |     10 |   260 |      Play |
| 26 |       1 |          26 | BetterBot | AHIMSTV |       H4 |    VITA. |     20 |   282 |      Play |
| 27 |       1 |          27 |      BLUE | ADIKOQS |       G5 |       QI |     24 |   284 |      Play |
| 28 |       1 |          28 | BetterBot | DEHMORS |       F6 |       HO |     27 |   309 |      Play |
| 29 |       1 |          29 |      BLUE | ?ADKOOS |      15K |    SOAKs |     32 |   316 |      Play |
| 30 |       1 |          30 | BetterBot | DEEIMRS |       E1 |   DERMIS |     25 |   334 |      Play |
|  ⋮ |       ⋮ |           ⋮ |         ⋮ |       ⋮ |        ⋮ |        ⋮ |      ⋮ |     ⋮ |         ⋮ |

## `turns_test.csv` 
Formatted identically to `turns_train.csv`, except that data is only provided up to the 20th turn. The move and points-scored values for the 20th turn are listed as NA.


```julia
turns_test = CSV.read("data/turns_test.csv", DataFrame, types = Dict(7=>Int64, 8=>Int64), silencewarnings=true)
turns_test[turns_test.game_id .== 2, :]
```

|    | game_id | turn_number |  nickname |    rack | location |     move |  points |   score | turn_type |
|---:|--------:|------------:|----------:|--------:|---------:|---------:|--------:|--------:|----------:|
|    |   Int64 |       Int64 |  String31 | String7 |  String3 | String15 |  Int64? |  Int64? |  String15 |
|  1 |       2 |           1 |   dezinga | AAEGNTX |       H8 |       AX |      18 |      18 |      Play |
|  2 |       2 |           2 | BetterBot | ?AEHNOY |       G7 |     AHOY |      27 |      27 |      Play |
|  3 |       2 |           3 |   dezinga | AAEGNTT |       F9 |      GAT |      24 |      42 |      Play |
|  4 |       2 |           4 | BetterBot | ?ADEENN |      12A |  ANNExED |      71 |      98 |      Play |
|  5 |       2 |           5 |   dezinga | ?AEFNOT |       C7 | FONTA.gE |      80 |     122 |      Play |
|  6 |       2 |           6 | BetterBot | DEJLTWW |      A11 |    J.WED |      48 |     146 |      Play |
|  7 |       2 |           7 |   dezinga | CINOOOR |       B6 |      COO |      18 |     140 |      Play |
|  8 |       2 |           8 | BetterBot | FILRRTW |       8A |     W..F |      42 |     188 |      Play |
|  9 |       2 |           9 |   dezinga | INORRSU |       7G |       .R |      12 |     152 |      Play |
| 10 |       2 |          10 | BetterBot | EEILRRT |       I4 |     RILE |       9 |     197 |      Play |
| 11 |       2 |          11 |   dezinga | EINORSU |      10B |       U. |       4 |     156 |      Play |
| 12 |       2 |          12 | BetterBot | EELOPRT |       5E |  REPT.LE |      36 |     233 |      Play |
| 13 |       2 |          13 |   dezinga | DEINORS |      15C |  SORDINE |      88 |     244 |      Play |
| 14 |       2 |          14 | BetterBot | IILMOSS |       L1 |    LIMOS |      26 |     259 |      Play |
| 15 |       2 |          15 |   dezinga | DIIOSUU |       3K |      U.U |      10 |     254 |      Play |
| 16 |       2 |          16 | BetterBot | AIIKNSU |       1L |     .INK |      24 |     283 |      Play |
| 17 |       2 |          17 |   dezinga | ADIIIOS |       NA |     -III |       0 |     254 |  Exchange |
| 18 |       2 |          18 | BetterBot | AAIORSU |       J7 |     AURA |      10 |     293 |      Play |
| 19 |       2 |          19 |   dezinga | ABDENOS |       4D |      DOB |      18 |     272 |      Play |
| 20 |       2 |          20 | BetterBot | EEGIMOS |       NA |       NA | missing | missing |        NA |

## `scores.csv`
Metadata about each player. Includes their Woogles.io rating prior to each game.

```julia
scores = CSV.read("data/scores.csv", DataFrame, types = Dict(4=>Float32), silencewarnings=true)
scores[scores.game_id .== 1,:]
```

|   | game_id |  nickname |   score |   rating |
|--:|--------:|----------:|--------:|---------:|
|   |   Int64 |  String31 | String3 | Float32? |
| 1 |       1 | BetterBot |     348 |   1740.0 |
| 2 |       1 |      BLUE |     333 |   1537.0 |

## `games.csv` 
Metadata regarding each game in `turns_train.csv` and `turns_test.csv`. Administrative information regarding each game is stored here, such as the time limit of the game, or if the game is counted towards rankings.


```julia
games = CSV.read("data/games.csv", DataFrame)
games[games.game_id .== 1, :]
```

|   | game_id |    first | time_control_name | game_end_reason |  winner |          created_at | lexicon | initial_time_seconds | increment_seconds | rating_mode | max_overtime_minutes | game_duration_seconds |
|--:|--------:|---------:|------------------:|----------------:|--------:|--------------------:|--------:|---------------------:|------------------:|------------:|---------------------:|----------------------:|
|   |   Int64 | String31 |          String15 |        String31 | String3 |            String31 | String7 |                Int64 |             Int64 |     String7 |                Int64 |              String31 |
| 1 |       1 |     BLUE |             rapid |        STANDARD |       0 | 2021-11-14 16:10:47 |   CSW19 |                  600 |                 0 |       RATED |                    1 |      461.512848138809 |

Additionally, the dictionary used for each game, encoded as `lexicon`, is listed here. There are four distinct lexicons used:
* **CSW19**—The 2019 edition of Collins' Scrabble Words. This dictionary is used in Scrabble competitions in English-speaking countries except the United States, Thailand, and Canada.
* **CSW21**—The 2021 edition of Collins' Scrabble Words, updated to include new or previously excluded words from CSW19 along with the removal of disputed words from CSW19.
* **NWL18**—The 2018 edition of the NASPA (formerly the North American Scrabble Players Association, now simply NASPA) World List. This dictionary is used in Scrabble competitions in North America.
* **NWL20**—The 2020 edition of the NASPA World List. Like CSW21, this dictionary was updated to include new or previously excluded words and remove disputed words.

## Dictionaries

These are the distinct words included in each dictionary. This data was pulled and parsed from NASPA's [Zyzzyva](https://scrabbleplayers.org/w/Zyzzyva) software.

```julia
NWL18 = readlines("ref/NWL18.txt")
NWL20 = readlines("ref/NWL20.txt")
CSW19 = readlines("ref/CSW19.txt")
CSW21 = readlines("ref/CSW21.txt")
```

```
279077-element Vector{String}:
 "AA"
 "AAH"
 "AAHED"
 "AAHING"
 "AAHS"
 "AAL"
 "AALII"
 "AALIIS"
 "AALS"
 "AARDVARK"
 "AARDVARKS"
 "AARDWOLF"
 "AARDWOLVES"
 ⋮
 "ZYMOTECHNICS"
 "ZYMOTIC"
 "ZYMOTICALLY"
 "ZYMOTICS"
 "ZYMURGIES"
 "ZYMURGY"
 "ZYTHUM"
 "ZYTHUMS"
 "ZYZZYVA"
 "ZYZZYVAS"
 "ZZZ"
 "ZZZS"
 ```

# Data Exploration

## Turn-Types

On Woogles.io, there are several types of turns that one can take over the course of a game.

```julia
combine(groupby(turns_train, :turn_type), nrow => :Freq)
```

|   |     turn_type |   Freq |
|--:|--------------:|-------:|
|   |      String15 |  Int64 |
| 1 |          Play | 495040 |
| 2 |           End |  16754 |
| 3 |      Exchange |   9943 |
| 4 |          Pass |   3338 |
| 5 |       Timeout |   2325 |
| 6 |     Challenge |    208 |
| 7 |            NA |    106 |
| 8 | Six-Zero Rule |    206 |


Each turn type represents the following action within a Scrabble game:

* "Play"—a standard turn of Scrabble in which letter tiles are played on a Scrabble board.
* "End"—indicates the end of a game. At this point, the point values of tiles remaining on a players' rack are summed and subtracted from a player's current score to determine their final score.
* "Exchange"—at any point during a game, a player may pass their turn in order to take any number of tiles from their rack and exchange them for an equivalent number of random tiles from the tile bag. As we will see, this encoding is also used to represent successful challenges on Woogles.io.
* "Pass"—a player decides not to play any words or exchange any tiles.
* "Timeout"—a player runs out of time on their turn and is assessed a point penalty.
* "Challenge"—this turn type represents _unsuccessful challenges_, not all challenges. When a player's word is unsuccessfully challenged, it is marked in this column, and the challenged player may be awarded additional points depending on the ruleset.
* "NA"—this appears to be a parsing error where players literally place the letters "NA" in that order. In the `turns_test` dataset, however, this indicator represents the type of the 20th turn, hidden from competitors of this Kaggle competition.
* "Six-Zero Rule"—an obscure Scrabble rule allows for the game to end immediately when six consecutive turns pass with zero points scored (through either challenges or passes).

We can quickly fix "NA" in our dataset.

```julia
turns_train.turn_type[turns_train.turn_type .== "NA"] .= "Play";
```

Does BetterBot use any exchanges?

```julia
combine(groupby(turns_train[turns_train.nickname .== "BetterBot",:], :turn_type), nrow => :Freq)
```

|   |     turn_type |   Freq |
|--:|--------------:|-------:|
|   |      String15 |  Int64 |
| 1 |          Play | 251563 |
| 2 |           End |   9471 |
| 3 |          Pass |   1199 |
| 4 |     Challenge |    208 |
| 5 | Six-Zero Rule |    103 |

It appears the answer is no. BetterBot will challenge, however, and will also pass the turn from time to time (most likely when it has no further moves).

We noted that unsuccessful challenges are listed in our dataset under the "Challenge" `turn_type`—what does a successful challenge look like?

```julia
turns_train[(turns_train.game_id .== 44) .& in.(turns_train.turn_number, [[12,13]]),:]
```

|   | game_id | turn_number | nickname |    rack | location |     move | points | score | turn_type |
|--:|--------:|------------:|---------:|--------:|---------:|---------:|-------:|------:|----------:|
|   |   Int64 |       Int64 | String31 | String7 |  String3 | String15 |  Int64 | Int64 |  String15 |
| 1 |      44 |          12 | Churrrro | ?BIINTW |      11C |     WTIN |     16 |   163 |      Play |
| 2 |      44 |          13 | Churrrro | ?BIINTW |       NA |       -- |    -16 |   147 |  Exchange |

In this turn, user "Churrrro" attempted to play the invalid word "WTIN". Their opponent successfully challenged the word, so on the immediately following turn, their tiles were returned to their rack, and the score of "WTIN" was deducted from their previous score.

Could a game end before turn 20?

```julia
combine(groupby(turns_train[turns_train.turn_type .== "End",:], :turn_number), nrow => :Freq)
```

|    | turn_number |  Freq |
|---:|------------:|------:|
|    |       Int64 | Int64 |
|  1 |          21 |     7 |
|  2 |          22 |    39 |
|  3 |          23 |   173 |
|  4 |          24 |   461 |
|  5 |          25 |  1011 |
|  6 |          26 |  1623 |
|  7 |          27 |  2136 |
|  8 |          28 |  2408 |
|  9 |          29 |  2254 |
| 10 |          30 |  1874 |
| 11 |          31 |  1485 |
| 12 |          32 |  1050 |
| 13 |          33 |   753 |
| 14 |          34 |   484 |
| 15 |          35 |   323 |
| 16 |          36 |   199 |
| 17 |          37 |   132 |
| 18 |          38 |   100 |
| 19 |          39 |    65 |
| 20 |          40 |    44 |
| 21 |          41 |    43 |
| 22 |          42 |    26 |
| 23 |          43 |    15 |
| 24 |          44 |     9 |
| 25 |          45 |     9 |
| 26 |          46 |    12 |
| 27 |          47 |     4 |
| 28 |          48 |     2 |
| 29 |          51 |     1 |
| 30 |          52 |     4 |
|  ⋮ |           ⋮ |     ⋮ |

Our train data (and presumably our test data) has been selected such that no game ends on or before turn twenty—we should not need to worry about encoding the end of games as a potential result for our test dataset.

A few cues about our data, coupled with our dictionary data, should allow deduce the precise scores recorded on challenge turns:

If a player takes two turns in a row, the subsequent turn is an administrative turn, usually a challenge.
* If the previously played word was invalid as judged by the game's dictionary, the point value on that turn is equivalent to -1 × the points scored on the previous turn.
* If the previously played word was valid as judged by the game's dictionary, the point value on that turn is equivalent to the point bonus awarded for being challenged unsuccessfully (usually zero).

We can also pick up on another important pattern and encode it into our predictions: When a player passes several turns in a row, there is a strong likelihood that they have abandoned the game and all subsequent points scored are zero.

```julia
turns_train[turns_train.game_id .== 1867,:]
```

|    | game_id | turn_number |  nickname |    rack | location |      move | points | score | turn_type |
|---:|--------:|------------:|----------:|--------:|---------:|----------:|-------:|------:|----------:|
|    |   Int64 |       Int64 |  String31 | String7 |  String3 |  String15 |  Int64 | Int64 |  String15 |
|  1 |    1867 |           1 |    marvin | EFOPRST |       8D |   FORPETS |     82 |    82 |      Play |
|  2 |    1867 |           2 | BetterBot | BEEEIIN |       F7 |      B.IE |      8 |     8 |      Play |
|  3 |    1867 |           3 |    marvin | GNNOQWW |      E10 |       WOW |     23 |   105 |      Play |
|  4 |    1867 |           4 | BetterBot | EEEIIMN |      D11 |      NEEM |     24 |    32 |      Play |
|  5 |    1867 |           5 |    marvin | EGIKNNQ |       J6 |       QI. |     32 |   137 |      Play |
|  6 |    1867 |           6 | BetterBot | DEEGIIT |       K3 |     DIGIT |     27 |    59 |      Play |
|  7 |    1867 |           7 |    marvin | EGKNNUY |       4H |    KEY.NG |     38 |   175 |      Play |
|  8 |    1867 |           8 | BetterBot | EEEINTZ |       M4 |      .EEZ |     24 |    83 |      Play |
|  9 |    1867 |           9 |    marvin | ?LMNORU |       5C |   UNMORaL |     73 |   248 |      Play |
| 10 |    1867 |          10 | BetterBot | ACEINRT |      14A |    ANE.IC |     32 |   115 |      Play |
| 11 |    1867 |          11 |    marvin | DLPSTTU |      A12 |      PL.T |     27 |   275 |      Play |
| 12 |    1867 |          12 | BetterBot | CDEEFRT |      C12 |      FR.T |     27 |   142 |      Play |
| 13 |    1867 |          13 |    marvin | DELRSTU |       H7 |  R.SULTED |     60 |   335 |      Play |
| 14 |    1867 |          14 | BetterBot | ACDEEIJ |       J2 |       JA. |     32 |   174 |      Play |
| 15 |    1867 |          15 |    marvin | ?AADHOU |       N5 |       AHA |     40 |   375 |      Play |
| 16 |    1867 |          16 | BetterBot | ABCDEEI |       D1 |     CABI. |     24 |   198 |      Play |
| 17 |    1867 |          17 |    marvin | ?ADIOTU |       1D |     .OATI |     21 |   396 |      Play |
| 18 |    1867 |          18 | BetterBot | DEEGHNS |       O7 |      GHEE |     37 |   235 |      Play |
| 19 |    1867 |          19 |    marvin | ?AADORU |       NA |         - |      0 |   396 |      Pass |
| 20 |    1867 |          20 | BetterBot | DINSVXY |       2F |        XI |     52 |   287 |      Play |
| 21 |    1867 |          21 |    marvin | ?AADORU |       NA |         - |      0 |   396 |      Pass |
| 22 |    1867 |          22 | BetterBot | DNORSVY |      13G |   V.NDORS |     32 |   319 |      Play |
| 23 |    1867 |          23 |    marvin | ?AADORU |       NA |         - |      0 |   396 |      Pass |
| 24 |    1867 |          24 | BetterBot | ELOSUVY |       L8 |    VOYEU. |     32 |   351 |      Play |
| 25 |    1867 |          25 |    marvin | ?AADORU |       NA |         - |      0 |   396 |      Pass |
| 26 |    1867 |          26 | BetterBot |  AILNOS |      14L |      SOIL |     23 |   374 |      Play |
| 27 |    1867 |          27 |    marvin | ?AADORU |       NA |         - |      0 |   396 |      Pass |
| 28 |    1867 |          28 | BetterBot |      AN |       3B |       NA. |     10 |   384 |      Play |
| 29 |    1867 |          29 | BetterBot |      NA |       NA | (AADORU?) |     14 |   398 |       End |
| 30 |    1867 |          30 |    marvin | AADORU? |       NA |    (time) |    -10 |   386 |   Timeout |

## Ratings

Players on Woogles.io are assigned a rating that indicates their skill at Scrabble, allowing them to be matched easily against similarly skilled opponents.

```julia
describe(scores)
```

|   | variable |    mean |    min |  median |    max | nmissing |                  eltype |
|--:|---------:|--------:|-------:|--------:|-------:|---------:|------------------------:|
|   |   Symbol |  Union… |    Any |  Union… |    Any |    Int64 |                    Type |
| 1 |  game_id | 12963.0 |      1 | 12963.0 |  25925 |        0 |                   Int64 |
| 2 | nickname |         | 10Dots |         |   zuzu |        0 |                String31 |
| 3 |    score |         |      0 |         |     NA |        0 |                 String3 |
| 4 |   rating |  1736.7 | 1114.0 |  1748.0 | 2130.0 |     2266 | Union{Missing, Float32} |

```julia
using StatsPlots
density(Float32.(skipmissing(scores.rating)),legend = false, title="Distribution of pre-game ratings")
```

![](https://i.imgur.com/DvuXTje.png)

How do our ratings look for BetterBot versus non-BetterBot players?

```julia
density(Float32.(skipmissing(scores.rating[scores.nickname .== "BetterBot"])),label = "BetterBot",title="BetterBot Ratings versus Human Ratings")
density!(Float32.(skipmissing(scores.rating[scores.nickname .!= "BetterBot"])),label = "Human Players")
```

![](https://i.imgur.com/By50OaG.png)

There are two distinct peaks for Better Bot ratings. Could this have something to do with the game difficulty?

```julia
game_ratings = innerjoin(games,scores, on = :game_id)
density(Float32.(skipmissing(game_ratings.rating[(game_ratings.nickname .== "BetterBot") .& (game_ratings.time_control_name .== "rapid")])),label = "Rapid",title="BetterBot Ratings by game type")
density!(Float32.(skipmissing(game_ratings.rating[(game_ratings.nickname .== "BetterBot") .& (game_ratings.time_control_name .== "regular")])),label = "Regular")
density!(Float32.(skipmissing(game_ratings.rating[(game_ratings.nickname .== "BetterBot") .& (game_ratings.time_control_name .== "blitz")])),label = "Blitz")
density!(Float32.(skipmissing(game_ratings.rating[(game_ratings.nickname .== "BetterBot") .& (game_ratings.time_control_name .== "ultrablitz")])),label = "Ultrablitz")
```

![](https://i.imgur.com/Ut2Dryl.png)

There appears to be some signal here. Could BetterBot perform better or worse based on the type of dictionary used?

```julia
density(Float32.(skipmissing(game_ratings.rating[(game_ratings.nickname .== "BetterBot") .& (game_ratings.lexicon .== "CSW19")])),label = "CSW19",title="BetterBot Ratings by Dictionary Type")
density!(Float32.(skipmissing(game_ratings.rating[(game_ratings.nickname .== "BetterBot") .& (game_ratings.lexicon .== "CSW21")])),label = "CSW21")
density!(Float32.(skipmissing(game_ratings.rating[(game_ratings.nickname .== "BetterBot") .& (game_ratings.lexicon .== "NWL18")])),label = "NWL18")
density!(Float32.(skipmissing(game_ratings.rating[(game_ratings.nickname .== "BetterBot") .& (game_ratings.lexicon .== "NWL20")])),label = "NWL20")
```

![](https://i.imgur.com/ReErTC3.png)

So, BetterBot's rating is generally a product of game type and dictionary type. To get the most accurate reflection of the strength of BetterBot, we may consider encoding BetterBot's ratings as the mean of the ratings for each game type and dictionary type.

Do we observe this in human players as well?

```julia
density(Float32.(skipmissing(game_ratings.rating[(game_ratings.nickname .!= "BetterBot") .& (game_ratings.lexicon .== "CSW19")])),label = "CSW19",title="Human Ratings by Dictionary Type")
density!(Float32.(skipmissing(game_ratings.rating[(game_ratings.nickname .!= "BetterBot") .& (game_ratings.lexicon .== "CSW21")])),label = "CSW21")
density!(Float32.(skipmissing(game_ratings.rating[(game_ratings.nickname .!= "BetterBot") .& (game_ratings.lexicon .== "NWL18")])),label = "NWL18")
density!(Float32.(skipmissing(game_ratings.rating[(game_ratings.nickname .!= "BetterBot") .& (game_ratings.lexicon .== "NWL20")])),label = "NWL20")
```

![](https://i.imgur.com/cuh2Z9o.png)

The answer appears to be no.

# Parsing boards and racks

## `move` and `location`

The `turns_` datasets allow us to reconstruct the entire Scrabble board at any point in the game relying on the data provided here.

![](https://i.imgur.com/vi8cFQm.png)

In our `turns_` datasets:

* All turns are ordered, which allows us to parse each game sequentially.
* Every game begins with an empty board.
* As mentioned above, any successful challenges are identified the subsequent turn as exchanges for negative points.
* As shown above, the Woogles.io board is laid out in a grid form, with columns given a corresponding letter and rows given a corresponding number. When a word is laid out vertically, the `location` value begins with a letter, and when a word is laid out horizontally, the `location` value begins with a number.
* When a word is placed, if the word overlaps with an already-placed tile, `move` represents the character with a "." character (e.g. playing the word "GRAPE" with the tiles "G","R","A", and "E" on the player's rack and the tile "P" already on the board is represented in `move` as "GRA.E").
* When a word is placed using a wildcard tile (appearing in traditional Scrabble as simply a blank tile, represented in `rack` as "?"), the letter that the wildcard tile becomes is represented as lowercase in `move` (e.g. playing the word "BUCK" with the tiles "B" and "C" along with a wildcard tile on a player's `rack` and the tile "U" already on the board is represented in `move` as "B.Ck").

From this information, we can reconstruct the Scrabble board as it was immediately prior to each play. We will represent each board as a 15x15 matrix of characters. An empty space is represented with an empty character (`' '`), a blank tile is represented as an lowercase letter for the word it is intended to represent (e.g. `'i'`), and all other tiles are represented as upper-case letters (e.g. `'F'`).

We will first write a function for parsing moves. The function `place_word` will take in a `board` (the 15x15 matrix of characters), `location`, and `move`, and return a new board with the `move` placed in the proper `location`.

```julia
function place_word(board, location, move)
    cols = ["A","B","C","D","E","F","G","H","I","J","K","L","M","N","O"]
    place_vertical = isletter(only(location[1:1]))
    horz_start_coord = findall(cols .== match(r"[A-Z]",location).match)[1]
    vert_start_coord = parse(Int64, match(r"\d+",location).match)[1]
    word_length = length(move)
    letters_to_place = collect.(move)
    if place_vertical
        vert_end_coord = vert_start_coord + word_length - 1
        vert_coords = vert_start_coord:vert_end_coord
        vert_coords = vert_coords[letters_to_place .!= '.']
        horz_coords = horz_start_coord
    else
        horz_end_coord = horz_start_coord + word_length - 1
        horz_coords = horz_start_coord:horz_end_coord
        horz_coords = horz_coords[letters_to_place .!= '.']
        vert_coords = vert_start_coord
    end
    letters_to_place = letters_to_place[letters_to_place .!= '.']
    board[vert_coords,horz_coords] = letters_to_place
    board
end
empty_board = fill(' ',(15,15))
place_word(empty_board, "E8", "FOO.AR")
```

```
15×15 Matrix{Char}:
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  'F'  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  'O'  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  'O'  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  'A'  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  'R'  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ```

We can iterate over the turns of a game to find the status of a board prior to any turn. Note that we will include logic to account for removing successful challenges from the board.

```julia
function create_boards(game)
    board = fill(' ',(15,15,size(game)[1]))
    for n in 2:size(game)[1]
        if game.turn_type[n-1] == "Play"
            board[:,:,n] = place_word(board[:,:,n-1],game.location[n-1],game.move[n-1])
        elseif (n > 3) && (game.turn_type[n-1] == "Exchange") & (game.turn_type[n-2] == "Play") & (game.nickname[n-1] == game.nickname[n-2])
            board[:,:,n] = board[:,:,n-2]
        else
            board[:,:,n] = board[:,:,n-1]
        end
    end
    return(board)
end
create_boards(turns_train[turns_train.game_id .== 1,:])[:,:,20]
```

```
15×15 Matrix{Char}:
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  'B'  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  'W'  'A'  'B'  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  'P'  ' '  'L'  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  'G'  'U'  'L'  'L'  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  'F'  'I'  'R'  'E'  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  'J'  'E'  'E'  'R'  ' '  'I'  'N'  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  'L'  ' '  ' '  'A'  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  'C'  'I'  'T'  'Y'  ' '  ' '  'W'  ' '  ' '
 ' '  ' '  ' '  'Y'  'A'  ' '  'O'  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  'F'  'A'  'U'  'X'  ' '  'O'  'P'  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  'A'  ' '  ' '  'E'  ' '  ' '  'I'  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  'Z'  'O'  'O'  ' '  'D'  'I'  'E'  'D'  ' '  ' '  ' '  ' '  ' '  ' '
 'R'  'E'  'U'  'N'  'i'  'O'  'N'  'S'  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ```

## Anticipating Challenges
With our ruleset, we should be able to perfectly predict any challenge on a specific turn. As mentioned above:

>If a player takes two turns in a row, the subsequent turn is an administrative turn, usually a challenge.
>* If the previously played word was invalid as judged by the game's dictionary, the point value on that turn is equivalent to -1 × the points scored on the previous turn.
>* If the previously played word was valid as judged by the game's dictionary, the point value on that turn is equivalent to the point bonus awarded for being challenged unsuccessfully (usually zero).

We can do this by first writing a function to extract words from the Scrabble board, then writing a second function to compare two boards and see if any of the new words added to the board have made the board invalid.

```julia
board = create_boards(turns_train[turns_train.game_id .== 1,:])[:,:,20]
```

```

15×15 Matrix{Char}:
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  'B'  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  'W'  'A'  'B'  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  'P'  ' '  'L'  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  'G'  'U'  'L'  'L'  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  'F'  'I'  'R'  'E'  ' '  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  'J'  'E'  'E'  'R'  ' '  'I'  'N'  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '  'L'  ' '  ' '  'A'  ' '  ' '
 ' '  ' '  ' '  ' '  ' '  ' '  'C'  'I'  'T'  'Y'  ' '  ' '  'W'  ' '  ' '
 ' '  ' '  ' '  'Y'  'A'  ' '  'O'  ' '  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  'F'  'A'  'U'  'X'  ' '  'O'  'P'  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  'A'  ' '  ' '  'E'  ' '  ' '  'I'  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ' '  'Z'  'O'  'O'  ' '  'D'  'I'  'E'  'D'  ' '  ' '  ' '  ' '  ' '  ' '
 'R'  'E'  'U'  'N'  'i'  'O'  'N'  'S'  ' '  ' '  ' '  ' '  ' '  ' '  ' '
 ```

```julia
function extract_words(board)
    words = Vector{String}()
    for n in 1:size(board,1)
        string = board[n,:] |>
            join |>
            strip |>
            uppercase
        strings = split(string,' ')
        strings = strings[length.(strings) .> 1]
        append!(words, strings)
    end
    for n in 1:size(board,2)
        string = board[:,n] |>
            join |>
            strip |>
            uppercase
        strings = split(string,' ')
        strings = strings[length.(strings) .> 1]
        append!(words, strings)
    end
    return(words)
end
extract_words(board)
```

```
27-element Vector{String}:
 "WAB"
 "GULL"
 "FIRE"
 "JEER"
 "IN"
 "CITY"
 "YA"
 "FAUX"
 "OP"
 "ZOO"
 "DIED"
 "REUNIONS"
 "FAZE"
 ⋮
 "ON"
 "AXE"
 "DO"
 "COO"
 "IN"
 "PIES"
 "FE"
 "GIRLY"
 "PUR"
 "LEI"
 "BALL"
 "NAW"
 ```

```julia
function valid_turn(board_pre, board_post, dict)
    words_pre = extract_words(board_pre)
    words_post = extract_words(board_post)
    new_words = words_post[.!(in.(words_post, [words_pre]))]
    valid_words = in.(new_words,[dict])
    return(all(valid_words))
end
# example of a valid word
boards = create_boards(turns_train[turns_train.game_id .== 1,:])
board_pre = boards[:,:,19]
board_post = boards[:,:,20]
valid_turn(board_pre, board_post, CSW19)
```

```
true
```

```julia
# example of an invalid word
boards = create_boards(turns_train[turns_train.game_id .== 44,:])
board_pre = boards[:,:,12]
board_post = boards[:,:,13]
valid_turn(board_pre, board_post, CSW19)
```

```
false
```

For convenience, we can also store our dictionaries in (what else?) a dictionary!

```julia
lexicons = Dict("CSW19" => CSW19, "CSW21" => CSW21, "NWL18" => NWL18, "NWL20" => NWL20)
```

```
Dict{String, Vector{String}} with 4 entries:
  "CSW19" => ["AA", "AAH", "AAHED", "AAHING", "AAHS", "AAL", "AALII", "AALIIS", "AALS", "AARDVARK"  …  "ZYMOTICALLY", "ZYMOTICS", "ZYMURGIES", "ZYMURGY", "ZYTHUM", "ZYTHUMS", "ZYZZYVA", "ZYZZYVAS", "ZZZ", "ZZZS"]
  "CSW21" => ["AA", "AAH", "AAHED", "AAHING", "AAHS", "AAL", "AALII", "AALIIS", "AALS", "AARDVARK"  …  "ZYMOTICALLY", "ZYMOTICS", "ZYMURGIES", "ZYMURGY", "ZYTHUM", "ZYTHUMS", "ZYZZYVA", "ZYZZYVAS", "ZZZ", "ZZZS"]
  "NWL18" => ["AA", "AB", "AD", "AE", "AG", "AH", "AI", "AL", "AM", "AN"  …  "WITHDRAWNNESSES", "WOEBEGONENESSES", "WONDERFULNESSES", "WORRISOMENESSES", "WORTHLESSNESSES", "WRONGHEADEDNESS", "XENOTRANSPLANTS", "XEROGRAPHICALLY", "XERORADIOGRAPHY", "ZOOGEOGRAPHICAL"]
  "NWL20" => ["AA", "AB", "AD", "AE", "AG", "AH", "AI", "AL", "AM", "AN"  …  "WITHDRAWNNESSES", "WOEBEGONENESSES", "WONDERFULNESSES", "WORRISOMENESSES", "WORTHLESSNESSES", "WRONGHEADEDNESS", "XENOTRANSPLANTS", "XEROGRAPHICALLY", "XERORADIOGRAPHY", "ZOOGEOGRAPHICAL"]
```

Now, we can write a block of code that will parse game boards and anticipate the success of challenges. We will join our `turns_train` dataframe with our `games` dataframe to determine the ruleset, then assign points for challenged turns based on whether we detect a challenge and if the word is valid.

```julia
turns_train_joined = innerjoin(turns_train, games, on = :game_id)
turns_train_joined.ndx = 1:size(turns_train_joined,1)
turns_train_joined.good_challenge .= false
turns_train_joined.bad_challenge .= false
game_ids = unique(turns_train_joined.game_id)
for game_id in game_ids
    game = turns_train_joined[turns_train_joined.game_id .== game_id,:]
    boards = create_boards(game)
    dict = lexicons[game.lexicon[1]]
    for n in 2:size(boards,3)
        if (game.turn_number[n] <= 20) && (game.nickname[n] == game.nickname[n-1]) && valid_turn(boards[:,:,n-1], boards[:,:,n], dict) # short circuiting for lightning fast parsing
            turns_train_joined.bad_challenge[game.ndx[n]] = true
        elseif (game.turn_number[n] <= 20) && (game.nickname[n] == game.nickname[n-1]) && !valid_turn(boards[:,:,n-1], boards[:,:,n], dict)
            turns_train_joined.good_challenge[game.ndx[n]] = true
        end
    end
end

turns_train_joined.xpoints .= 0.0
turns_train_joined.xpoints[turns_train_joined.bad_challenge] .= sum(turns_train_joined.points[turns_train_joined.turn_type .== "Challenge"]) / sum(turns_train_joined.turn_type .== "Challenge")
turns_train_joined.xpoints[turns_train_joined.good_challenge] = -1 .* turns_train_joined.points[turns_train_joined.ndx[turns_train_joined.good_challenge] .- 1]

@df turns_train_joined[turns_train_joined.good_challenge,:] scatter(
    :xpoints,
    :points,
    title = "Expected points lost on successful challenges",
    legend=false
)
```

![](https://i.imgur.com/CsKtXnk.png)

```julia
turns_train_joined = nothing
GC.gc()
```

We can see that in all relevant instances, we are perfectly predicting the number of points lost on challenges and can guestimate with a decent bit of accuracy points gained from unsuccessful challenges. This can be quite a boon for our RMSE later in this Kaggle competition.



Following our data exploration, we can apply several insights to our modeling work to further refine our estimates. We can apply the following techniques to adjust our estimates:
 * We can perfectly detect and predict the point values of successful challenge turns
 * We can perfectly detect unsuccessful challenges and approximate with a high degree of accuracy the point values of these turns
 * We can apply manual adjustments depending on the number of consecutively passed turns.
 * We can adjust our estimates with MixedModels.jl on a per-player basis.

# Preparing and implementing a neural network

## Data preparation

We have a few tricks up our sleeve to handle predictions, but we now need to dive into the grunt work of building a neural network and preparing our data for it. We will need to represent the game board and rack as tensors, then feed these tensors into constructed neural networks, tune these networks, and finally predict on our training data.

To kick off the grunt work, we will write a function to encode the board state into a matrix. In the first two dimensions, we will encode the positional information of each bit. In the third dimension, we will encode the following information:
* The specific letter tile at each location (1-26), with a 1 representing that tile existing at that location, and a 0 if it does not.
* Any wildcard tiles (27), with a 1 representing that a tile is a wildcard and a 0 if it is not or if the space is empty.
* Whether or not a tile exists at a given location (28), with a 1 representing a tile, and a 0 representing an open space.
* The point value of the letter tile at a given location (29), with a 0 representing an open space or a wildcard tile.
* Available letter-bonus tiles (30), with a 2 representing a double-letter-score bonus tile, a 3 representing a triple-letter-score bonus tile, and a 0 representing that a letter bonus is unavailable at that space.
* Available word-bonus tiles (31), with a 2 representing a double-word-score bonus tile, a 3 representing a triple-word-score bonus tile, and a 0 representing a word bonus is unavailable at that space.

```julia
using Dictionaries 
LETTER_VALS = Dictionary(Dict( 'E' => 1,
                               'A' => 1,
                               'I' => 1,
                               'O' => 1,
                               'N' => 1,
                               'R' => 1,
                               'T' => 1,
                               'L' => 1,
                               'S' => 1,
                               'U' => 1,
                               'D' => 2,
                               'G' => 2,
                               'B' => 3,
                               'C' => 3,
                               'M' => 3,
                               'P' => 3,
                               'F' => 4,
                               'H' => 4,
                               'V' => 4,
                               'W' => 4,
                               'Y' => 4,
                               'K' => 5,
                               'J' => 8,
                               'X' => 8,
                               'Q' => 10,
                               'Z' => 10 ))

LETTER_BONUSES = 
[   [   0    0    0    2    0    0    0    0    0     0     0     2     0     0     0  ]
    [   0    0    0    0    0    3    0    0    0     3     0     0     0     0     0  ]
    [   0    0    0    0    0    0    2    0    2     0     0     0     0     0     0  ]
    [   2    0    0    0    0    0    0    2    0     0     0     0     0     0     2  ]
    [   0    0    0    0    0    0    0    0    0     0     0     0     0     0     0  ]
    [   0    3    0    0    0    3    0    0    0     3     0     0     0     3     0  ]
    [   0    0    2    0    0    0    2    0    2     0     0     0     2     0     0  ]
    [   0    0    0    2    0    0    0    0    0     0     0     2     0     0     0  ]
    [   0    0    2    0    0    0    2    0    2     0     0     0     2     0     0  ]
    [   0    3    0    0    0    3    0    0    0     3     0     0     0     3     0  ]
    [   0    0    0    0    0    0    0    0    0     0     0     0     0     0     0  ]
    [   2    0    0    0    0    0    0    2    0     0     0     0     0     0     2  ]
    [   0    0    0    0    0    0    2    0    2     0     0     0     0     0     0  ]
    [   0    0    0    0    0    3    0    0    0     3     0     0     0     0     0  ]
    [   0    0    0    2    0    0    0    0    0     0     0     2     0     0     0  ]    ]

WORD_BONUSES =  
[   [   3    0    0    0    0    0    0    3    0     0     0     0     0     0     3  ]
    [   0    2    0    0    0    0    0    0    0     0     0     0     0     2     0  ]
    [   0    0    2    0    0    0    0    0    0     0     0     0     2     0     0  ]
    [   0    0    0    2    0    0    0    0    0     0     0     2     0     0     0  ]
    [   0    0    0    0    2    0    0    0    0     0     2     0     0     0     0  ]
    [   0    0    0    0    0    0    0    0    0     0     0     0     0     0     0  ]
    [   0    0    0    0    0    0    0    0    0     0     0     0     0     0     0  ]
    [   3    0    0    0    0    0    0    2    0     0     0     0     0     0     3  ]
    [   0    0    0    0    0    0    0    0    0     0     0     0     0     0     0  ]
    [   0    0    0    0    0    0    0    0    0     0     0     0     0     0     0  ]
    [   0    0    0    0    2    0    0    0    0     0     2     0     0     0     0  ]
    [   0    0    0    2    0    0    0    0    0     0     0     2     0     0     0  ]
    [   0    0    2    0    0    0    0    0    0     0     0     0     2     0     0  ]
    [   0    2    0    0    0    0    0    0    0     0     0     0     0     2     0  ]
    [   3    0    0    0    0    0    0    3    0     0     0     0     0     0     3  ]    ]

ALPHABET = 'A':'Z'

function encode_board(board)
    encoded_board = zeros(15,15,31,1)
    for n in 1:31
        if n <= 26
            board_tmp = uppercase.(board) .== ALPHABET[n]
        elseif n == 27
            board_tmp = in.(board, [lowercase.(ALPHABET)])
        elseif n == 28
            board_tmp = board .!= ' '
        elseif n == 29
            board_tmp = encoded_board[:,:,n,1]
            letters = board[board .!= ' ']
            vals = map(x -> get(LETTER_VALS,x,0), letters)
            board_tmp[board .!= ' '] = vals
        elseif n == 30
            board_tmp = copy(LETTER_BONUSES)
            board_tmp[board .!= ' '] .= 0
        else
            board_tmp = copy(WORD_BONUSES)
            board_tmp[board .!= ' '] .= 0
        end
        encoded_board[:,:,n,:] = board_tmp
    end
    encoded_board
end

encode_board(create_boards(turns_train[turns_train.game_id .== 1,:])[:,:,25])[:,:,1,:] # shows all 'A's on the board
```

```
15×15×1 Array{Float64, 3}:
[:, :, 1] =
 0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
 0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
 0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  1.0  0.0  0.0  0.0
 0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  1.0  0.0  0.0
 0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
 0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
 0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
 0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
 0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  1.0  0.0  0.0
 0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
 0.0  0.0  0.0  0.0  1.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  1.0  0.0
 0.0  0.0  1.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
 0.0  1.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
 0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
 0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0  0.0
 ```

Finally, we will write a function to parse our rack and metadata into a format readable by a neural network. We will include a count of the number of letter tiles in each rack split out by column, the turn number, the timing format of the game, the rating mode of the game, the lexicon used in the game, and the rating of each player.

```julia
CHARS = ['?', 'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z']

function encode_game_data(turns, games, scores)
    # initial parsing
    main_df = innerjoin(turns, games, on = :game_id)
    main_df = innerjoin(main_df, scores[:,[:game_id, :nickname, :rating]], on = [:game_id, :nickname])
    main_df.rating[ismissing.(main_df.rating)] .= sum(scores.rating[(scores.rating .!= "BetterBot") .& .!ismissing.(scores.rating)]) / sum((scores.rating .!= "BetterBot") .& .!ismissing.(scores.rating)) # impute missing scores as mean score
    main_df.rating = convert.(Float32, main_df.rating) 
    main_df.is_BetterBot = main_df.nickname .== "BetterBot"
    # encode lexicons, combining for convenience sake
    main_df.CSW = (main_df.lexicon .== "CSW19") .| (main_df.lexicon .== "CSW21")
    main_df.NWL = (main_df.lexicon .== "NWL18") .| (main_df.lexicon .== "NWL20")
    # one-hot encode categorical variables
    time_control_encode = select(main_df, [:time_control_name => ByRow(isequal(v))=> Symbol(v) for v in unique(main_df.time_control_name)])
    rating_mode_encode = select(main_df, [:rating_mode => ByRow(isequal(v))=> Symbol(v) for v in unique(main_df.rating_mode)])
    # categorically encode turn numbers
    turn_numbers_string = main_df.turn_number
    turn_numbers_string[turn_numbers_string .>= 30] .= 30
    turn_numbers_string = string.(turn_numbers_string)
    turn_numbers_string[turn_numbers_string .== "30"] .= "30+"
    main_df.turn_numbers_string = turn_numbers_string
    turn_number_encode = select(main_df, [:turn_numbers_string => ByRow(isequal(v))=> Symbol(v) for v in ["1","2","3","4","5","6","7","8","9","10","11","12","13","14","15",
                                                                                                          "16","17","18","19","20","21","22","23","24","25","26","27","28","29","30+"]])
    # convert rack into columns
    rack = main_df.rack
    racks = mapreduce(permutedims, vcat, collect.(rpad.(rack,7,'.')))
    rack_encode = zeros(Int64, size(racks,1), size(CHARS,1))
    for j = 1:size(racks,2)
        for i = 1:size(racks,1)
            for k in 1:size(CHARS,1)
                if CHARS[k] == racks[i,j]
                    rack_encode[i,k] += 1
                end
            end
        end
    end
    # combine
    rack_encode = DataFrame(rack_encode, :auto)
    rename!(rack_encode, Symbol.(CHARS))
    main_df = main_df[:,[:points, :rating, :CSW, :NWL, :is_BetterBot]]
    main_df = hcat(main_df, time_control_encode, rating_mode_encode, rack_encode, turn_number_encode)
    return(main_df)
end

encode_game_data(turns_train[turns_train.game_id .== 1,:], games, scores)
```

`...truncated...`

## Network Architecture

To model our board data, we will employ a convolutional neural network, which will process and flatten our board data. From there, we will concatenate our board data with our related game data and run it through additional layers to arrive at estimates for our figures.

```julia
using Flux
using Flux.Data: DataLoader
using CUDA

if CUDA.functional()
    device = gpu
else
    device = cpu
end
device
```

```
gpu (generic function with 1 method)
```

We will borrow some custom code from the [Flux.jl advanced docs](https://github.com/FluxML/Flux.jl/blob/master/docs/src/models/advanced.md) and [Model Zoo](https://github.com/FluxML/model-zoo) to implement some helper functions. The `Join` structure will allow us to concatenate chains within our models.

```julia
struct Join{T, F}
    combine::F
    paths::T
end

Join(combine, paths...) = Join(combine, paths)

Flux.@functor Join

(m::Join)(xs::Tuple) = m.combine(map((f, x) -> f(x), m.paths, xs)...)
(m::Join)(xs...) = m(xs)
```

`eval_mse` will allow us to rely on our GPU to quickly evaluate our model at each epoch and determine the best set of parameters.

```julia
function eval_mse(loader, model, device)
    l = 0f0
    ntot = 0
    for (w, x, y) in loader
        w, x, y = w |> device, x |> device, y |> device
        ŷ = model((w, x))
        l += Flux.Losses.mse(ŷ, y') * size(x)[end]
        ntot += size(x)[end]
    end
    return (l/ntot)
end
```

```
eval_mse (generic function with 1 method)
```

Last, we will encode our data.

```julia
# encode turns_train
turns_train_game_data = encode_game_data(turns_train, games, scores)
turns = size(turns_train,1)
turns_train_boards = fill(' ',15,15,turns)
turns_train.ndx = 1:size(turns_train,1)
game_ids = unique(turns_train.game_id)
for game_id in game_ids
    turns_train_boards[:,:,turns_train.ndx[turns_train.game_id .== game_id]] = create_boards(turns_train[turns_train.game_id .== game_id,:])
end

turns_train_boards_data = zeros(15,15,31,turns)
for n in 1:turns
    turns_train_boards_data[:,:,:,n] = encode_board(turns_train_boards[:,:,n])
end
# encode turns_test
turns_test_game_data = encode_game_data(turns_test, games, scores)
turns = size(turns_test,1)
turns_test_boards = fill(' ',15,15,turns)
turns_test.ndx = 1:turns
game_ids = unique(turns_test.game_id)
for game_id in game_ids
    turns_test_boards[:,:,turns_test.ndx[turns_test.game_id .== game_id]] = create_boards(turns_test[turns_test.game_id .== game_id,:])
end

turns_test_boards_data = zeros(15,15,31,turns)
for n in 1:turns
    turns_test_boards_data[:,:,:,n] = encode_board(turns_test_boards[:,:,n])
end
```

## Tuning

To determine the best set of parameters and model infrastructure, we allocate roughly 2/3rds of the `turns_train` data to a training set and the remaining 1/3rd to a test dataset. Why split up `turns_train` and not use the existing values in `turns_test`? We have a different distribution of turn numbers in `turns_test` compared to `turns_train`—whereas `turns_train` contains data from a variety of turns, `turns_test` only includes data from turns 1 through 19 (which, notably, does not exclude turn 20—the turn we care most about!). Using this approach to select parameters will result in a model that generalizes well on an out-of-sample dataset.

The tuning code is attached but not run in the interest of time and memory. We selected 90 as the optimal number of epochs.

```julia
tune = false
train = false;
```

```julia
turns_train_xs_a = turns_train_game_data[turns_train_game_data.points .>= 0,Not(:points)]
turns_train_xs_a = Array(turns_train_xs_a)'
turns_train_xs_b = Float32.(turns_train_boards_data[:,:,:,turns_train_game_data.points .>= 0])

turns_train_ys = turns_train_game_data.points[turns_train_game_data.points .>= 0]

if tune
    train_xs_a = turns_train_xs_a[:,1:end-100000]
    train_xs_b = turns_train_xs_b[:,:,:,1:end-100000]
    train_ys = turns_train_ys[1:end-100000]

    valid_xs_a = turns_train_xs_a[:,end-99999:end]
    valid_xs_b = turns_train_xs_b[:,:,:,end-99999:end]
    valid_ys = turns_train_ys[end-99999:end]
end
```

We use a low batch size for parsing our data, which, coupled with a low learning rate for our ADAM optimizer, allows the optimizer to find the best values for our parameter set (albeit more slowly than a larger batch size or with a higher value of eta).

```julia
batch_size = 32

if tune
    train_loader = DataLoader((train_xs_a, train_xs_b, train_ys), batchsize=batch_size, shuffle=true)
    valid_loader = DataLoader((valid_xs_a, valid_xs_b, valid_ys), batchsize=batch_size)
end

eta = 1e-4
opt = ADAM(eta)
```

```
ADAM(0.0001, (0.9, 0.999), 1.0e-8, IdDict{Any, Any}())
```

Our model architecture is deliberately simple: we run our board through a convolutional layer, apply batch normalization, then apply a pooling layer. After flattening it down into a nx1 layer, we can join it to our rack/game table (of dimension mx1) and then run it through two dense layers to parse and condense our values into a single point estimate.

```julia
using Random
Random.seed!(3)
if tune
    tuning_model = Chain(
        Join(vcat,
            Dense(67,67),
            Chain(
                Conv((3,3), 31 => 16, stride = 1),
                BatchNorm(16,relu),
                MaxPool((2,2)),
                Flux.flatten
            )
            ),
        Dense(643,128,relu),
        Dense(128,1)
    ) 
    tuning_model = tuning_model |> device
    ps = Flux.params(tuning_model);
end
```

Using our train/test sets, we can find the epoch number that minimizes our test RMSE.

```julia
Random.seed!(3)

if tune
    epochs = 200
    train_loss = zeros(epochs)
    valid_loss = zeros(epochs)

    for epoch in 1:epochs
        for (w, x, y) in train_loader
            w, x, y = w |> device, x |> device, y |> device
            gs = Flux.gradient(ps) do
                    ŷ = tuning_model((w,x))
                    Flux.Losses.mse(ŷ, y')
                end
            Flux.Optimise.update!(opt, ps, gs)
        end
        train_loss[epoch] = eval_mse(train_loader, tuning_model, device)
        valid_loss[epoch] = eval_mse(valid_loader, tuning_model, device)
    end

    results = DataFrame(Epoch = 1:epochs, Train = train_loss, Validation = valid_loss);
end
```

```julia
if tune
    @df results plot(
        :Epoch,
        :Train,
        label = "Train MSE",
        title = "Train v. Validation MSE by Epoch",
        xlim = (1,epochs),
        ylim = (200,400)
    )
    @df results plot!(
        :Epoch,
        :Validation,
        label = "Validation MSE"
    )
end
```

```julia
if tune
    epochs = results.Epoch[results.Validation .== minimum(results.Validation)][1]
else
    epochs = 90
end
```

```
90
```

## Final Model

With our tuned parameters, we can train our model on our entire dataset, including `turns_train` and `turns_test`.

```julia
turns_test_xs_a = turns_test_game_data[(turns_test.turn_number .<= 19) .&& (turns_test_game_data.points .>= 0),Not(:points)]
turns_test_xs_a = Array(turns_test_xs_a)'
turns_test_xs_b = Float32.(turns_test_boards_data[:,:,:,(turns_test.turn_number .<= 19) .&& (turns_test_game_data.points .>= 0)])

turns_test_ys = turns_test_game_data.points[(turns_test.turn_number .<= 19) .&& (turns_test_game_data.points .>= 0)]

xs_a = cat(turns_train_xs_a, turns_test_xs_a, dims = 2)
xs_b = cat(turns_train_xs_b, turns_test_xs_b, dims = 4)
ys = vcat(turns_train_ys, turns_test_ys)
ys = Int32.(ys) # making this conversion because turn_test_ys dtype is Union{missing, Int64}

if train
    loader = DataLoader((xs_a, xs_b, ys), batchsize=batch_size, shuffle=true);
end
```

```julia
using BSON: @save
using Dates
Random.seed!(3)

if train
    model_final = Chain(
        Join(vcat,
            Dense(67,67),
            Chain(
                Conv((3,3), 31 => 16, stride = 1),
                BatchNorm(16,relu),
                MaxPool((2,2)),
                Flux.flatten
            )
            ),
        Dense(643,128,relu),
        Dense(128,1)
    ) 
    model_final = model_final |> device

    ps = Flux.params(model_final) 
    opt = ADAM(eta)

    for epoch in 1:epochs
        for (w, x, y) in loader
            w, x, y = w |> device, x |> device, y |> device
            gs = Flux.gradient(ps) do
                    ŷ = model_final((w,x))
                    Flux.Losses.mse(ŷ, y')
                end
            Flux.Optimise.update!(opt, ps, gs)
        end
    end

    model_final = cpu(model_final)
    file_name = "models/$(string(Dates.format(now(), "d_m_yyyy_HH_MM")))_mod_epochs_$(epochs)_eta_$(eta).bson"
    @save file_name model_final
    @save "models/model_final.bson" model_final
end
```

# Constructing Predictions

## Base Predictions

We are now prepared to predict on our test dataset. First, let us load our saved model into our workspace, then predict the number of expected points `xpoints` on `turns_test`.

```julia
using BSON: @load
@load "models/model_final.bson" model_final

turns_test_xs_a = turns_test_game_data[:,Not(:points)]
turns_test_xs_a = Array(turns_test_xs_a)'
turns_test_xs_b = Float32.(turns_test_boards_data)

turns_test.xpoints = vec(model_final((turns_test_xs_a, turns_test_xs_b)));
```

Next, we will adjust some our estimates manually. For example, some of our predictions still end up negative—we manually set any negative predictions to a reasonable low value, 5 points. I have also taken the liberty of checking for potential abandons (looking for multiple passes in a row before turn 20), and these are set to zero as well.

```julia
turns_test.xpoints[turns_test.xpoints .< 0] .= 5
turns_test.xpoints[[154441,144221,138001,129301,96621]] .= 0;
```

Now, we can adjust our predictions by checking for challenges, using the same logic as above.

```julia
turns_test_joined = innerjoin(turns_test, games, on = :game_id)
turns_test_joined.ndx = 1:size(turns_test_joined,1)
turns_test_joined.good_challenge .= false
turns_test_joined.bad_challenge .= false
curr_game_id = turns_test_joined.game_id[1]
for n in 2:size(turns_test_joined,1)
    new_game_id = turns_test_joined.game_id[n]
    dict = lexicons[turns_test_joined.lexicon[n]]
    if ((new_game_id == curr_game_id) & (turns_test_joined.nickname[n] == turns_test_joined.nickname[n-1])) && valid_turn(turns_test_boards[:,:,n-1], turns_test_boards[:,:,n], dict)
        turns_test_joined.bad_challenge[n] = true
    elseif ((new_game_id == curr_game_id) & (turns_test_joined.nickname[n] == turns_test_joined.nickname[n-1])) && !valid_turn(turns_test_boards[:,:,n-1], turns_test_boards[:,:,n], dict)
        turns_test_joined.good_challenge[n] = true
    end
    curr_game_id = new_game_id
end

turns_test_joined.xpoints[turns_test_joined.bad_challenge] .= sum(skipmissing(turns_test_joined.points[turns_test_joined.turn_type .== "Challenge"])) / sum(skipmissing(turns_test_joined.turn_type .== "Challenge"))
turns_test_joined.xpoints[turns_test_joined.good_challenge] = -1 .* skipmissing(turns_test_joined.points[turns_test_joined.ndx[turns_test_joined.good_challenge] .- 1]);
```

With these changes in place, we can save our initial predictions.

```julia
test_preds = DataFrame(game_id = turns_test.game_id[turns_test.turn_number .== 20], points = turns_test_joined.xpoints[turns_test.turn_number .== 20])
file_name = "outputs/$(string(Dates.format(now(), "d_m_yyyy_HH_MM")))_base_pred.csv"
CSV.write(file_name, test_preds)
```

```
"outputs/1_3_2022_17_25_base_pred.csv"
```

## Mixed Model Predictions

There is a part of our data that we are not capturing—we have players whose skill level may not be adequately captured in pre-game ratings, such as players who are new to Woogles but otherwise experienced Scrabble players. Neural networks are poorly equipped to capture individual player effects, but we can run our model through a secondary scaler capable of accounting for random effects to adjust our estimates—a mixed model, treating `nickname` as a random effect. Our mixed model will also ensure our estimates are appropriately scaled such that the mean of our estimates and the means of our outputs are aligned.

```julia
using MixedModels

ŷs = model_final((xs_a, xs_b))
mm_df = DataFrame(nickname = vcat(turns_train.nickname[turns_train.points .>= 0], turns_test.nickname[(turns_test.turn_number .<= 19) .&& (turns_test.points .>= 0)]),
                  points = vcat(turns_train.points[turns_train.points .>= 0], turns_test.points[(turns_test.turn_number .<= 19) .&& (turns_test.points .>= 0)]), 
                  xpoints = vec(ŷs))

m1 = fit(MixedModel, @formula(points ~ 0 + xpoints + (1|nickname)), mm_df)
```

```
┌ Warning: ProgressMeter by default refresh meters with additional information in IJulia via `IJulia.clear_output`, which clears all outputs in the cell. 
│  - To prevent this behaviour, do `ProgressMeter.ijulia_behavior(:append)`. 
│  - To disable this warning message, do `ProgressMeter.ijulia_behavior(:clear)`.
└ @ ProgressMeter C:\Users\edwar\.julia\packages\ProgressMeter\Vf8un\src\ProgressMeter.jl:620
Minimizing 16 	 Time: 0:00:01 (63.06 ms/it)
Linear mixed model fit by maximum likelihood
 points ~ 0 + xpoints + (1 | nickname)
     logLik     -2 logLik        AIC           AICc          BIC      
 -2770806.9315  5541613.8629  5541619.8629  5541619.8629  5541654.0969

Variance components:
            Column    Variance Std.Dev. 
nickname (Intercept)   10.39388  3.22395
Residual              235.59633 15.34915
 Number of obs: 667515; levels of grouping factors: 673

  Fixed-effects parameters:
───────────────────────────────────────────────
            Coef.  Std. Error       z  Pr(>|z|)
───────────────────────────────────────────────
xpoints  0.944415  0.00140901  670.27    <1e-99
───────────────────────────────────────────────
```

```julia
pred_df = DataFrame(nickname = turns_test.nickname,
                    xpoints = vec(model_final((turns_test_xs_a, turns_test_xs_b))))
pred_df.points .= 0
turns_test.xpoints = predict(m1, pred_df)

turns_test.xpoints[turns_test.xpoints .< 0] .= 5
turns_test.xpoints[[154441,144221,138001,129301,96621]] .= 0

turns_test_joined = innerjoin(turns_test, games, on = :game_id)
turns_test_joined.ndx = 1:size(turns_test_joined,1)
turns_test_joined.good_challenge .= false
turns_test_joined.bad_challenge .= false
curr_game_id = turns_test_joined.game_id[1]
for n in 2:size(turns_test_joined,1)
    new_game_id = turns_test_joined.game_id[n]
    dict = lexicons[turns_test_joined.lexicon[n]]
    if ((new_game_id == curr_game_id) & (turns_test_joined.nickname[n] == turns_test_joined.nickname[n-1])) && valid_turn(turns_test_boards[:,:,n-1], turns_test_boards[:,:,n], dict)
        turns_test_joined.bad_challenge[n] = true
    elseif ((new_game_id == curr_game_id) & (turns_test_joined.nickname[n] == turns_test_joined.nickname[n-1])) && !valid_turn(turns_test_boards[:,:,n-1], turns_test_boards[:,:,n], dict)
        turns_test_joined.good_challenge[n] = true
    end
    curr_game_id = new_game_id
end

turns_test_joined.xpoints[turns_test_joined.bad_challenge] .= sum(skipmissing(turns_test_joined.points[turns_test_joined.turn_type .== "Challenge"])) / sum(skipmissing(turns_test_joined.turn_type .== "Challenge"))
turns_test_joined.xpoints[turns_test_joined.good_challenge] = -1 .* skipmissing(turns_test_joined.points[turns_test_joined.ndx[turns_test_joined.good_challenge] .- 1])

test_preds = DataFrame(game_id = turns_test.game_id[turns_test.turn_number .== 20], points = turns_test_joined.xpoints[turns_test.turn_number .== 20])
file_name = "outputs/$(string(Dates.format(now(), "d_m_yyyy_HH_MM")))_mixed_pred.csv"
CSV.write(file_name, test_preds)
```

```
"outputs/1_3_2022_17_29_mixed_pred.csv"
```

We can submit our predicts using the Kaggle API.

```julia
submit = false
if submit
    run(`kaggle competitions submit scrabble-point-value -f $file_name -m "Submission at $(Dates.now())"`)
end
```

# Conclusions

This was a fun exploration of an interesting problem. I found that my usual tools (R / some kind of tabular machine learning approach such as a random forest or XGBoost model) were inadequate to deal with a problem like this, and it forced me to go outside of my comfort zone and learn new techniques. Additionally, I had intended on doing more projects with Julia—this was my first medium scale project with Julia—so I felt like this was a great way to learn by doing. I also learned some of the limitations of Julia—encoding the data the way that I did, I frequently ran into memory issues. Still, I appreciated the ease and fluidity of programming with Julia, along with the speed.

I think I could likely improve my results further if I knew how to better construct deep learning models—I kept the architecture as simple as possible ala Occam's razor and could not justify making it more complex in other ways with my present knowledge base. Learning more about deep learning is a priority for me this year, and I am really looking forward to doing more projects like these in the future.

For me, this project also emphasized the importance of researching a problem in-depth before attempting a solution. Jumping into this problem without any prior research would have made this task much more difficult than it already was, but previous approaches to similar problems provided a successful framework, albeit with some modifications.

# Citations

Agarwal, R. (2019). Evaluation Function Approximation for Scrabble. ArXiv, abs/1901.08728.

Bezanson, J., Edelman, A., Karpinski, S., & Shah, V.B. (2017). Julia: A Fresh Approach to Numerical Computing. ArXiv, abs/1411.1607.

Chellapilla, K., & Fogel, D.B. (1999). Evolving neural networks to play checkers without relying on expert knowledge. IEEE transactions on neural networks, 10 6, 1382-91 .

Oshri, B. (2015). Predicting Moves in Chess using Convolutional Neural Networks.

Ningrum, D.N., Yuan, S., Kung, W., Wu, C., Tzeng, I., Huang, C., Li, J.Y., & Wang, Y. (2021). Deep Learning Classifier with Patient’s Metadata of Dermoscopic Images in Malignant Melanoma Detection. Journal of Multidisciplinary Healthcare, 14, 877 - 885.

Silver, D., Schrittwieser, J., Simonyan, K., Antonoglou, I., Huang, A., Guez, A., Hubert, T., Baker, L., Lai, M., Bolton, A., Chen, Y., Lillicrap, T., Hui, F., Sifre, L., Driessche, G. van den, Graepel, T., & Hassabis, D. (2017). Mastering the game of Go without human knowledge. Nature, 550(7676), 354–359.

Wang, S., Guo, W., Gao, H., & Long, B. (2020). Efficient Neural Query Auto Completion. Proceedings of the 29th ACM International Conference on Information & Knowledge Management.

Yin, W., Kann, K., Yu, M., & Schütze, H. (2017). Comparative Study of CNN and RNN for Natural Language Processing. ArXiv, abs/1702.01923.
