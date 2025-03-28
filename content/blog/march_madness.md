---
date: "2021-06-02"
description: My 2021 March Madness Kaggle Solution
draft: false
keywords:
- college_basketball
- basketball
- machine_learning
math: true
slug: march_madness_2021
pagination: true
tags:
- python
- r
title: 2021 March Madness Kaggle Solution
toc: false
bsky_thread: https://bsky.app/profile/johnbedwards.io/post/3lcjtzoxzyk2l
---

Our approach is ensemble the shit out of everything. We will hold out the 2015-2019 games for validation purposes. We will prepare and optimize two sets of models - one, an ensemble of general team strength features trained on 1985-2014 games, and two - an ensemble of general team strength features + adjusted ratings based on box-score data trained on 2002-2014 games. Let's prepare the first approach.


```python
import sys
!{sys.executable} -m pip install pandas sklearn numpy rpy2 trueskill catboost hyperopt ray

import numpy as np
import pandas as pd
from sklearn.linear_model import LogisticRegression
import math
import trueskill as ts
import warnings
warnings.filterwarnings("ignore") # because pandas indexing gets mad at me

import rpy2.robjects as ro
from rpy2.robjects import pandas2ri
from rpy2.robjects.conversion import localconverter
import rpy2.robjects.packages as rpackages
from rpy2.robjects.vectors import StrVector
```

    Requirement already satisfied: pandas in c:\users\edwar\anaconda3\lib\site-packages (1.1.3)
    Requirement already satisfied: sklearn in c:\users\edwar\anaconda3\lib\site-packages (0.0)
    Requirement already satisfied: numpy in c:\users\edwar\anaconda3\lib\site-packages (1.19.2)
    Requirement already satisfied: rpy2 in c:\users\edwar\anaconda3\lib\site-packages (3.4.2)
    Requirement already satisfied: trueskill in c:\users\edwar\anaconda3\lib\site-packages (0.4.5)
    Requirement already satisfied: catboost in c:\users\edwar\anaconda3\lib\site-packages (0.24.4)
    Requirement already satisfied: hyperopt in c:\users\edwar\anaconda3\lib\site-packages (0.2.5)
    Requirement already satisfied: ray in c:\users\edwar\anaconda3\lib\site-packages (1.2.0)
    Requirement already satisfied: python-dateutil>=2.7.3 in c:\users\edwar\anaconda3\lib\site-packages (from pandas) (2.8.1)
    Requirement already satisfied: pytz>=2017.2 in c:\users\edwar\anaconda3\lib\site-packages (from pandas) (2020.1)
    Requirement already satisfied: scikit-learn in c:\users\edwar\anaconda3\lib\site-packages (from sklearn) (0.23.2)
    Requirement already satisfied: jinja2 in c:\users\edwar\anaconda3\lib\site-packages (from rpy2) (2.11.2)
    Requirement already satisfied: cffi>=1.10.0 in c:\users\edwar\anaconda3\lib\site-packages (from rpy2) (1.14.3)
    Requirement already satisfied: tzlocal in c:\users\edwar\anaconda3\lib\site-packages (from rpy2) (2.1)
    Requirement already satisfied: six in c:\users\edwar\anaconda3\lib\site-packages (from trueskill) (1.15.0)
    Requirement already satisfied: matplotlib in c:\users\edwar\anaconda3\lib\site-packages (from catboost) (3.3.2)
    Requirement already satisfied: plotly in c:\users\edwar\anaconda3\lib\site-packages (from catboost) (4.14.3)
    Requirement already satisfied: scipy in c:\users\edwar\anaconda3\lib\site-packages (from catboost) (1.5.2)
    Requirement already satisfied: graphviz in c:\users\edwar\anaconda3\lib\site-packages (from catboost) (0.16)
    Requirement already satisfied: cloudpickle in c:\users\edwar\anaconda3\lib\site-packages (from hyperopt) (1.6.0)
    Requirement already satisfied: future in c:\users\edwar\anaconda3\lib\site-packages (from hyperopt) (0.18.2)
    Requirement already satisfied: tqdm in c:\users\edwar\anaconda3\lib\site-packages (from hyperopt) (4.50.2)
    Requirement already satisfied: networkx>=2.2 in c:\users\edwar\anaconda3\lib\site-packages (from hyperopt) (2.5)
    Requirement already satisfied: jsonschema in c:\users\edwar\anaconda3\lib\site-packages (from ray) (3.2.0)
    Requirement already satisfied: aiohttp-cors in c:\users\edwar\anaconda3\lib\site-packages (from ray) (0.7.0)
    Requirement already satisfied: grpcio>=1.28.1 in c:\users\edwar\anaconda3\lib\site-packages (from ray) (1.36.1)
    Requirement already satisfied: protobuf>=3.8.0 in c:\users\edwar\anaconda3\lib\site-packages (from ray) (3.14.0)
    Requirement already satisfied: prometheus-client>=0.7.1 in c:\users\edwar\anaconda3\lib\site-packages (from ray) (0.8.0)
    Requirement already satisfied: colorama in c:\users\edwar\anaconda3\lib\site-packages (from ray) (0.4.4)
    Requirement already satisfied: colorful in c:\users\edwar\anaconda3\lib\site-packages (from ray) (0.5.4)
    Requirement already satisfied: opencensus in c:\users\edwar\anaconda3\lib\site-packages (from ray) (0.7.12)
    Requirement already satisfied: py-spy>=0.2.0 in c:\users\edwar\anaconda3\lib\site-packages (from ray) (0.3.4)
    Requirement already satisfied: gpustat in c:\users\edwar\anaconda3\lib\site-packages (from ray) (0.6.0)
    Requirement already satisfied: msgpack<2.0.0,>=1.0.0 in c:\users\edwar\anaconda3\lib\site-packages (from ray) (1.0.0)
    Requirement already satisfied: filelock in c:\users\edwar\anaconda3\lib\site-packages (from ray) (3.0.12)
    Requirement already satisfied: click>=7.0 in c:\users\edwar\anaconda3\lib\site-packages (from ray) (7.1.2)
    Requirement already satisfied: redis>=3.5.0 in c:\users\edwar\anaconda3\lib\site-packages (from ray) (3.5.3)
    Requirement already satisfied: aioredis in c:\users\edwar\anaconda3\lib\site-packages (from ray) (1.3.1)
    Requirement already satisfied: aiohttp in c:\users\edwar\anaconda3\lib\site-packages (from ray) (3.7.4.post0)
    Requirement already satisfied: requests in c:\users\edwar\anaconda3\lib\site-packages (from ray) (2.24.0)
    Requirement already satisfied: pyyaml in c:\users\edwar\anaconda3\lib\site-packages (from ray) (5.3.1)
    Requirement already satisfied: joblib>=0.11 in c:\users\edwar\anaconda3\lib\site-packages (from scikit-learn->sklearn) (0.17.0)
    Requirement already satisfied: threadpoolctl>=2.0.0 in c:\users\edwar\anaconda3\lib\site-packages (from scikit-learn->sklearn) (2.1.0)
    Requirement already satisfied: MarkupSafe>=0.23 in c:\users\edwar\anaconda3\lib\site-packages (from jinja2->rpy2) (1.1.1)
    Requirement already satisfied: pycparser in c:\users\edwar\anaconda3\lib\site-packages (from cffi>=1.10.0->rpy2) (2.20)
    Requirement already satisfied: kiwisolver>=1.0.1 in c:\users\edwar\anaconda3\lib\site-packages (from matplotlib->catboost) (1.3.0)
    Requirement already satisfied: pyparsing!=2.0.4,!=2.1.2,!=2.1.6,>=2.0.3 in c:\users\edwar\anaconda3\lib\site-packages (from matplotlib->catboost) (2.4.7)
    Requirement already satisfied: certifi>=2020.06.20 in c:\users\edwar\anaconda3\lib\site-packages (from matplotlib->catboost) (2020.6.20)
    Requirement already satisfied: cycler>=0.10 in c:\users\edwar\anaconda3\lib\site-packages (from matplotlib->catboost) (0.10.0)
    Requirement already satisfied: pillow>=6.2.0 in c:\users\edwar\anaconda3\lib\site-packages (from matplotlib->catboost) (8.0.1)
    Requirement already satisfied: retrying>=1.3.3 in c:\users\edwar\anaconda3\lib\site-packages (from plotly->catboost) (1.3.3)
    Requirement already satisfied: decorator>=4.3.0 in c:\users\edwar\anaconda3\lib\site-packages (from networkx>=2.2->hyperopt) (4.4.2)
    Requirement already satisfied: setuptools in c:\users\edwar\anaconda3\lib\site-packages (from jsonschema->ray) (50.3.1.post20201107)
    Requirement already satisfied: attrs>=17.4.0 in c:\users\edwar\anaconda3\lib\site-packages (from jsonschema->ray) (20.3.0)
    Requirement already satisfied: pyrsistent>=0.14.0 in c:\users\edwar\anaconda3\lib\site-packages (from jsonschema->ray) (0.17.3)
    Requirement already satisfied: opencensus-context==0.1.2 in c:\users\edwar\anaconda3\lib\site-packages (from opencensus->ray) (0.1.2)
    Requirement already satisfied: google-api-core<2.0.0,>=1.0.0 in c:\users\edwar\anaconda3\lib\site-packages (from opencensus->ray) (1.26.1)
    Requirement already satisfied: psutil in c:\users\edwar\anaconda3\lib\site-packages (from gpustat->ray) (5.7.2)
    Requirement already satisfied: nvidia-ml-py3>=7.352.0 in c:\users\edwar\anaconda3\lib\site-packages (from gpustat->ray) (7.352.0)
    Requirement already satisfied: blessings>=1.6 in c:\users\edwar\anaconda3\lib\site-packages (from gpustat->ray) (1.7)
    Requirement already satisfied: hiredis in c:\users\edwar\anaconda3\lib\site-packages (from aioredis->ray) (1.1.0)
    Requirement already satisfied: async-timeout in c:\users\edwar\anaconda3\lib\site-packages (from aioredis->ray) (3.0.1)
    Requirement already satisfied: multidict<7.0,>=4.5 in c:\users\edwar\anaconda3\lib\site-packages (from aiohttp->ray) (5.1.0)
    Requirement already satisfied: typing-extensions>=3.6.5 in c:\users\edwar\anaconda3\lib\site-packages (from aiohttp->ray) (3.7.4.3)
    Requirement already satisfied: chardet<5.0,>=2.0 in c:\users\edwar\anaconda3\lib\site-packages (from aiohttp->ray) (3.0.4)
    Requirement already satisfied: yarl<2.0,>=1.0 in c:\users\edwar\anaconda3\lib\site-packages (from aiohttp->ray) (1.6.3)
    Requirement already satisfied: urllib3!=1.25.0,!=1.25.1,<1.26,>=1.21.1 in c:\users\edwar\anaconda3\lib\site-packages (from requests->ray) (1.25.11)
    Requirement already satisfied: idna<3,>=2.5 in c:\users\edwar\anaconda3\lib\site-packages (from requests->ray) (2.10)
    Requirement already satisfied: google-auth<2.0dev,>=1.21.1 in c:\users\edwar\anaconda3\lib\site-packages (from google-api-core<2.0.0,>=1.0.0->opencensus->ray) (1.28.0)
    Requirement already satisfied: packaging>=14.3 in c:\users\edwar\anaconda3\lib\site-packages (from google-api-core<2.0.0,>=1.0.0->opencensus->ray) (20.4)
    Requirement already satisfied: googleapis-common-protos<2.0dev,>=1.6.0 in c:\users\edwar\anaconda3\lib\site-packages (from google-api-core<2.0.0,>=1.0.0->opencensus->ray) (1.53.0)
    Requirement already satisfied: rsa<5,>=3.1.4; python_version >= "3.6" in c:\users\edwar\anaconda3\lib\site-packages (from google-auth<2.0dev,>=1.21.1->google-api-core<2.0.0,>=1.0.0->opencensus->ray) (4.7.2)
    Requirement already satisfied: pyasn1-modules>=0.2.1 in c:\users\edwar\anaconda3\lib\site-packages (from google-auth<2.0dev,>=1.21.1->google-api-core<2.0.0,>=1.0.0->opencensus->ray) (0.2.8)
    Requirement already satisfied: cachetools<5.0,>=2.0.0 in c:\users\edwar\anaconda3\lib\site-packages (from google-auth<2.0dev,>=1.21.1->google-api-core<2.0.0,>=1.0.0->opencensus->ray) (4.2.1)
    Requirement already satisfied: pyasn1>=0.1.3 in c:\users\edwar\anaconda3\lib\site-packages (from rsa<5,>=3.1.4; python_version >= "3.6"->google-auth<2.0dev,>=1.21.1->google-api-core<2.0.0,>=1.0.0->opencensus->ray) (0.4.8)
    


```python
compact_results = pd.read_csv('data/MRegularSeasonCompactResults.csv')
compact_results['WTeamID'] = compact_results['WTeamID'].astype('str')
compact_results['LTeamID'] = compact_results['LTeamID'].astype('str')
compact_results.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Season</th>
      <th>DayNum</th>
      <th>WTeamID</th>
      <th>WScore</th>
      <th>LTeamID</th>
      <th>LScore</th>
      <th>WLoc</th>
      <th>NumOT</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1985</td>
      <td>20</td>
      <td>1228</td>
      <td>81</td>
      <td>1328</td>
      <td>64</td>
      <td>N</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1985</td>
      <td>25</td>
      <td>1106</td>
      <td>77</td>
      <td>1354</td>
      <td>70</td>
      <td>H</td>
      <td>0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1985</td>
      <td>25</td>
      <td>1112</td>
      <td>63</td>
      <td>1223</td>
      <td>56</td>
      <td>H</td>
      <td>0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1985</td>
      <td>25</td>
      <td>1165</td>
      <td>70</td>
      <td>1432</td>
      <td>54</td>
      <td>H</td>
      <td>0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1985</td>
      <td>25</td>
      <td>1192</td>
      <td>86</td>
      <td>1447</td>
      <td>74</td>
      <td>H</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
</div>



I am not a big fan of this format, so I will quickly write a function to re-oriented our dataframe around home/away teams rather than winning/losing teams.


```python
def refactor_games_compact(dataframe):
    df = dataframe
    neutral_winners = df.loc[df.WLoc == 'N']
    home_winners = df.loc[df.WLoc == 'H']
    away_winners = df.loc[df.WLoc == 'A']
    
    neutral_winners['NeutralFlg'] = True
    neutral_winners.columns = ['Season','DayNum','HTeamID','HScore','ATeamID','AScore','WLoc','NumOT','NeutralFlg']
    
    home_winners['NeutralFlg'] = False
    home_winners.columns = ['Season','DayNum','HTeamID','HScore','ATeamID','AScore','WLoc','NumOT','NeutralFlg']
    
    away_winners['NeutralFlg'] = False
    away_winners.columns = ['Season','DayNum','ATeamID','AScore','HTeamID','HScore','WLoc','NumOT','NeutralFlg']
    
    df = neutral_winners.append(home_winners.append(away_winners))
    df = df.drop(columns=['WLoc'])
    df = df.sort_values(by=['Season','DayNum'])
    return df
    
compact_results = refactor_games_compact(compact_results)
compact_results.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Season</th>
      <th>DayNum</th>
      <th>HTeamID</th>
      <th>HScore</th>
      <th>ATeamID</th>
      <th>AScore</th>
      <th>NumOT</th>
      <th>NeutralFlg</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1985</td>
      <td>20</td>
      <td>1228</td>
      <td>81</td>
      <td>1328</td>
      <td>64</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>6</th>
      <td>1985</td>
      <td>25</td>
      <td>1228</td>
      <td>64</td>
      <td>1226</td>
      <td>44</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>7</th>
      <td>1985</td>
      <td>25</td>
      <td>1242</td>
      <td>58</td>
      <td>1268</td>
      <td>56</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>11</th>
      <td>1985</td>
      <td>25</td>
      <td>1344</td>
      <td>75</td>
      <td>1438</td>
      <td>71</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>13</th>
      <td>1985</td>
      <td>25</td>
      <td>1412</td>
      <td>70</td>
      <td>1397</td>
      <td>65</td>
      <td>0</td>
      <td>True</td>
    </tr>
  </tbody>
</table>
</div>



Our first step will be to write a function to rescore games based on home court advantage. To estimate home court advantage, we  will use the [LRMC methodology](https://link.springer.com/chapter/10.1007/978-981-13-9364-8_20).


```python
def calc_hfa(dataframe):
    df = dataframe
    home_games = df.loc[~df['NeutralFlg']]
    
    X_df = home_games
    Y_df = home_games
    
    X_df['HDiff'] = X_df['HScore'] - X_df['AScore']
    X_df.loc[X_df['NumOT'] > 0,'HDiff'] = 0 # any overtime games are treated as ties
    X_df = X_df[['HTeamID','ATeamID','HDiff']]
    
    Y_df['HWin'] = (Y_df['AScore'] > Y_df['HScore']) * 1
    Y_df = Y_df[['HTeamID','ATeamID','HWin']]
    
    log_df = pd.merge(X_df, Y_df, left_on=['HTeamID','ATeamID'],right_on=['ATeamID','HTeamID'])
    X = log_df['HDiff']
    Y = log_df['HWin']
    
    X = np.array(X).reshape(-1,1)
    
    logreg = LogisticRegression(random_state=0).fit(X, Y)
    
    a = float(logreg.coef_[0])
    b = float(logreg.intercept_)
    h = (-b / a) / 2
    
    return {'a':a,'b':b,'h':h}

def rescore_hfa(dataframe):
    df = dataframe
    hfa_mod = calc_hfa(df)
    
    h_2 = hfa_mod['h'] / 2
    
    df['HScore'] = df['HScore'] - h_2 * df['NeutralFlg']
    df['AScore'] = df['AScore'] + h_2 * df['NeutralFlg']
    
    return df
    
testing_df = compact_results.loc[compact_results['Season'] == 2014]

rescore_hfa(testing_df).head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Season</th>
      <th>DayNum</th>
      <th>HTeamID</th>
      <th>HScore</th>
      <th>ATeamID</th>
      <th>AScore</th>
      <th>NumOT</th>
      <th>NeutralFlg</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>129204</th>
      <td>2014</td>
      <td>4</td>
      <td>1102</td>
      <td>75.734604</td>
      <td>1119</td>
      <td>71.265396</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>129211</th>
      <td>2014</td>
      <td>4</td>
      <td>1124</td>
      <td>68.734604</td>
      <td>1160</td>
      <td>63.265396</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>129220</th>
      <td>2014</td>
      <td>4</td>
      <td>1163</td>
      <td>74.734604</td>
      <td>1268</td>
      <td>80.265396</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>129223</th>
      <td>2014</td>
      <td>4</td>
      <td>1184</td>
      <td>79.734604</td>
      <td>1198</td>
      <td>64.265396</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>129240</th>
      <td>2014</td>
      <td>4</td>
      <td>1258</td>
      <td>74.734604</td>
      <td>1213</td>
      <td>78.265396</td>
      <td>0</td>
      <td>True</td>
    </tr>
  </tbody>
</table>
</div>



We can call on this function when working with team strength methodologies that are not adjusted to account for home field advantage. Additionally, this should aid us in scoring 2021 teams, when home field advantage is not as concrete as it has been in previous years.

We will also write another function to adjust the scoring of overtime games. To be points per 40 minutes. This will come in handy when estimating team strength from points-based metrics.


```python
def rescore_ot(dataframe):
    df = dataframe
    df.HScore = df.HScore * (40 + df.NumOT * 5) / 40
    df.AScore = df.AScore * (40 + df.NumOT * 5) / 40
    
    return df

rescore_ot(testing_df.loc[testing_df['NumOT'] > 0]).head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Season</th>
      <th>DayNum</th>
      <th>HTeamID</th>
      <th>HScore</th>
      <th>ATeamID</th>
      <th>AScore</th>
      <th>NumOT</th>
      <th>NeutralFlg</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>129265</th>
      <td>2014</td>
      <td>4</td>
      <td>1344</td>
      <td>92.25000</td>
      <td>1130</td>
      <td>87.75000</td>
      <td>1</td>
      <td>False</td>
    </tr>
    <tr>
      <th>129227</th>
      <td>2014</td>
      <td>4</td>
      <td>1414</td>
      <td>109.12500</td>
      <td>1201</td>
      <td>110.25000</td>
      <td>1</td>
      <td>False</td>
    </tr>
    <tr>
      <th>129251</th>
      <td>2014</td>
      <td>4</td>
      <td>1330</td>
      <td>75.37500</td>
      <td>1283</td>
      <td>88.87500</td>
      <td>1</td>
      <td>False</td>
    </tr>
    <tr>
      <th>129276</th>
      <td>2014</td>
      <td>4</td>
      <td>1274</td>
      <td>69.75000</td>
      <td>1383</td>
      <td>74.25000</td>
      <td>1</td>
      <td>False</td>
    </tr>
    <tr>
      <th>129344</th>
      <td>2014</td>
      <td>5</td>
      <td>1464</td>
      <td>79.57643</td>
      <td>1198</td>
      <td>84.67357</td>
      <td>1</td>
      <td>True</td>
    </tr>
  </tbody>
</table>
</div>



We will compute the following metrics to estimate team strength:

  * [Elo](https://www.kaggle.com/kplauritzen/elo-ratings-in-python)
  * [LRMC](https://blog.collegefootballdata.com/talking-tech-applying-lrmc-rankings-to-college-football-part-two/)
  * [SRS](https://blog.collegefootballdata.com/talking-tech-bu/)
  * [RPI](https://kenpom.com/blog/rpi-help/)
  * [TrueSkill](https://trueskill.org/)
  * Mixed Model Ratings with [LME4](https://www.rdocumentation.org/packages/lme4/versions/1.1-26/topics/lme4-package)
  * [Colley Matrix method](https://www.colleyrankings.com/matrate.pdf)
  * Win Percentage
  * Average Margin of Victory
  
First is Elo. Normally we would tune Elo paramters, but that process can be somewhat intensive, especially with large datasets such as this -- hence, I will be using a premade series of parameters to estimate team strength. Note that with Elo, you can normally adjust for home field advantage with a constant factor, and that availability still exists with this function, but passing in arguments for rescoring overrides this factor.


```python
def calc_elo(dataframe, avg_elo = 1500, elo_width = 400, mov_factor = 8, hfa = 135, hfa_rescore = False, ot_rescore = False):
    df = dataframe
    
    if ot_rescore:
        df = rescore_ot(df)
    
    if hfa_rescore:
        df = rescore_hfa(df)
        
    df = df.sort_values(by=['Season','DayNum'])
    
    teams = list(set(df['HTeamID'].append(df['ATeamID'])))
    elos = {team:avg_elo for team in teams}
    
    for index, row in df.iterrows():
        home_elo = elos[row['HTeamID']]
        away_elo = elos[row['ATeamID']]
        
        home_elo_adj = home_elo
        if not row['NeutralFlg'] and not hfa_rescore and not ot_rescore:
            home_elo_adj += hfa
            
        # calculate expected score
        home_prob = 1.0/(1+10**((away_elo - home_elo_adj)/elo_width))
        away_prob = 1 - home_prob
        
        # calculate k factor as a function of margin of the log of margin of victory
        home_diff = row['HScore'] - row['AScore']
        k_factor = math.log(abs(home_diff)+1) * mov_factor
        
        home_win = (row['HScore'] > row['AScore']) * 1
        away_win = (row['AScore'] > row['HScore']) * 1
        
        # update elos
        elos[row['HTeamID']] = home_elo + k_factor * (home_win - home_prob)
        elos[row['ATeamID']] = away_elo + k_factor * (away_win - away_prob)
    
    elo_df = pd.DataFrame(elos,index=[0]).transpose()
    elo_df.columns = ['Elo']
    
    return elo_df

calc_elo(testing_df,hfa_rescore=True,ot_rescore=True).head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Elo</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1106</th>
      <td>1500.823216</td>
    </tr>
    <tr>
      <th>1219</th>
      <td>1505.567477</td>
    </tr>
    <tr>
      <th>1285</th>
      <td>1517.728113</td>
    </tr>
    <tr>
      <th>1350</th>
      <td>1526.039480</td>
    </tr>
    <tr>
      <th>1249</th>
      <td>1336.231722</td>
    </tr>
  </tbody>
</table>
</div>



Next up is LRMC. We're already halfway done with our code adjusting for HFA, we will simply take our logistic model and apply it to all of the games in our dataframe (because adjusting for HFA is integral to LRMC, it is not an optional argument).


```python
def calc_lrmc(dataframe, ot_rescore=True):
    df = dataframe
    
    if ot_rescore:
        df = rescore_ot(df)
        
    hfa_mod = calc_hfa(df)
    h = hfa_mod['h']
    a = hfa_mod['a']
    b = hfa_mod['b']
    
    teams = list(set(df['HTeamID'].append(df['ATeamID'])))
    
    n_teams = len(teams)
    p = np.zeros((n_teams,n_teams))
    n_games = np.zeros(n_teams)
    
    for index, row in df.iterrows():
        home_team_ndx = teams.index(row['HTeamID'])
        away_team_ndx = teams.index(row['ATeamID'])
        
        # calculate r_x
        spread = row['HScore'] - row['AScore'] + h * (row['NeutralFlg'])
        r_x = math.exp(a * spread + b) / (1 + math.exp(a * spread + b))
        
        # update respective matrices
        n_games[home_team_ndx] += 1
        n_games[away_team_ndx] += 1
        
        p[home_team_ndx, away_team_ndx] += 1 - r_x
        p[away_team_ndx, home_team_ndx] += r_x
        p[home_team_ndx, home_team_ndx] += r_x
        p[away_team_ndx, away_team_ndx] += 1 - r_x
        
    # solve matrix
    p = p / n_games[:,None]
    prior = n_teams - np.array(list(range(n_teams)))
    steady_state = np.linalg.matrix_power(p, 1000)
    rating = prior.dot(steady_state)
    
    return pd.DataFrame({'LRMC':rating}, index = teams)

calc_lrmc(testing_df).head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>LRMC</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1106</th>
      <td>132.928218</td>
    </tr>
    <tr>
      <th>1219</th>
      <td>137.471394</td>
    </tr>
    <tr>
      <th>1285</th>
      <td>157.845504</td>
    </tr>
    <tr>
      <th>1350</th>
      <td>203.865228</td>
    </tr>
    <tr>
      <th>1249</th>
      <td>99.113128</td>
    </tr>
  </tbody>
</table>
</div>



SRS on deck.


```python
def calc_srs(dataframe, hfa_rescore=True, ot_rescore=True):
    df = dataframe
    
    if hfa_rescore:
        df = rescore_hfa(df)
    if ot_rescore:
        df = rescore_ot(df)
        
    df['HSpread'] = df['HScore'] - df['AScore']
    df['ASpread'] = -1 * df['HSpread']
    
    teams = pd.concat([
        df[['HTeamID','HSpread','ATeamID']].rename(columns={'HTeamID':'TeamID','HSpread':'Spread','ATeamID':'OppID'}),
        df[['ATeamID','ASpread','HTeamID']].rename(columns={'ATeamID':'TeamID','ASpread':'Spread','HTeamID':'OppID'})
    ])
    
    spreads = teams.groupby('TeamID').Spread.mean()
    
    terms = []
    solutions = []

    for team in spreads.keys():
        row = []
        opps = list(teams[teams['TeamID'] == team]['OppID'])

        for opp in spreads.keys():
            if opp == team:
                row.append(1)
            elif opp in opps:
                row.append(-1.0 / len(opps))
            else:
                row.append(0)

        terms.append(row)
        solutions.append(spreads[team])
        
    solutions = np.linalg.solve(np.array(terms), np.array(solutions))
    return pd.DataFrame({'SRS':solutions},index = list(spreads.keys()))
    
calc_srs(testing_df).head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>SRS</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1101</th>
      <td>-18.642546</td>
    </tr>
    <tr>
      <th>1102</th>
      <td>-6.101457</td>
    </tr>
    <tr>
      <th>1103</th>
      <td>2.412745</td>
    </tr>
    <tr>
      <th>1104</th>
      <td>6.819273</td>
    </tr>
    <tr>
      <th>1105</th>
      <td>-5.714953</td>
    </tr>
  </tbody>
</table>
</div>



RPI's usefulness is highly debatable, there's a very good reason why the NCAA does not use it in tournament rankings and instead relies on NET ratings now. Still, we're not too picky, and we can always throw it out when we reach the feature selection stage of the process. We'll adjust for OT but not for HFA when it comes to calculating RPI (RPI likes to think it already incorporates HFA, even though it really doesn't, but we'll humor it for the time being).


```python
def calc_rpi(dataframe, ot_rescore = True):
    df = dataframe
    
    if ot_rescore:
        df = rescore_ot(df)
        
    df['HWin'] = df['HScore'].ge(df['AScore']) * 1
    df['AWin'] = 1 - df['HWin']
    df['HLoss'] = 1 - df['HWin']
    df['ALoss'] = df['HWin']
    
    teams = pd.concat([
        df[['HTeamID','ATeamID','HWin','HLoss']].rename(columns={'HTeamID':'TeamID','ATeamID':'OppID','HWin':'HWin',
                                                                 'HLoss':'HLoss'}),
        df[['ATeamID','HTeamID','AWin','ALoss']].rename(columns={'ATeamID':'TeamID','HTeamID':'OppID','AWin':'AWin',
                                                                 'ALoss':'ALoss'})
    ])
    
    teams = teams.fillna(0)
    
    rpi = teams.groupby('TeamID')[['HWin','HLoss','AWin','ALoss']].sum()
    rpi['W'] = rpi['HWin'] + rpi['AWin']
    rpi['L'] = rpi['HLoss'] + rpi['ALoss']
    rpi['WP'] = rpi['W'] / (rpi['W'] + rpi['L'])
    rpi['AdjWP'] = (rpi['HWin'] * 0.6 + rpi['AWin'] * 1.4) / (rpi['HWin'] * 0.6 + rpi['AWin'] * 1.4 + rpi['HLoss'] * 1.4 + rpi['ALoss'] * 0.6)
    
    for index, row in rpi.iterrows():
        temp_teams = teams.loc[teams['TeamID'] == index]
        rpi.loc[index,'OWP'] = float(pd.merge(temp_teams, rpi, left_on = 'OppID', right_on = 'TeamID')['WP'].mean())
        
    for index, row in rpi.iterrows():
        temp_teams = teams.loc[teams['TeamID'] == index]
        rpi.loc[index,'OOWP'] = float(pd.merge(temp_teams, rpi, left_on = 'OppID', right_on = 'TeamID')['OWP'].mean())
        
    rpi['RPI'] = 0.25 * rpi['AdjWP'] + 0.5 * rpi['OWP'] + 0.25 * rpi['OOWP']
    
    return pd.DataFrame({'RPI':rpi['RPI']},index = list(rpi.index))

calc_rpi(testing_df).head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>RPI</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1101</th>
      <td>0.380277</td>
    </tr>
    <tr>
      <th>1102</th>
      <td>0.465483</td>
    </tr>
    <tr>
      <th>1103</th>
      <td>0.538925</td>
    </tr>
    <tr>
      <th>1104</th>
      <td>0.561932</td>
    </tr>
    <tr>
      <th>1105</th>
      <td>0.443962</td>
    </tr>
  </tbody>
</table>
</div>



Next is TrueSkill, which might actually be more valuable than Elo given the different numbers of games teams played. We'll just record the Mu values for each Teams' TS rating.


```python
def calc_ts(dataframe, hfa_rescore = True, ot_rescore = True):
    df = dataframe
    
    if hfa_rescore:
        df = rescore_hfa(df)
    if ot_rescore:
        df = rescore_ot(df)
        
    teams = list(set(df['HTeamID'].append(df['ATeamID'])))
    ts_dict = {}
    
    for team in teams:
        ts_dict[team] = {'TS':ts.Rating()}
        
    for index, row in df.iterrows():
        home_rating = ts_dict[row['HTeamID']]['TS']
        away_rating = ts_dict[row['ATeamID']]['TS']
        if row['HScore'] > row['AScore']:
            home_rating, away_rating = ts.rate_1vs1(home_rating, away_rating)
        elif row['AScore'] > row['HScore']:
            away_rating, home_rating = ts.rate_1vs1(away_rating, home_rating)
        else:
            home_rating, away_rating = ts.rate_1vs1(home_rating, away_rating, drawn = True)

        ts_dict[row['HTeamID']]['TS'] = home_rating
        ts_dict[row['ATeamID']]['TS'] = away_rating

    ts_list = [ts_dict[team]['TS'].mu for team in teams]
    
    return pd.DataFrame({'TS':ts_list},index = teams)

calc_ts(testing_df).head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>TS</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1106</th>
      <td>21.073280</td>
    </tr>
    <tr>
      <th>1219</th>
      <td>21.556950</td>
    </tr>
    <tr>
      <th>1285</th>
      <td>24.166877</td>
    </tr>
    <tr>
      <th>1350</th>
      <td>28.791182</td>
    </tr>
    <tr>
      <th>1249</th>
      <td>12.829917</td>
    </tr>
  </tbody>
</table>
</div>



Next up, we're generating some mixed model ratings for each team. To do this, we'll quickly, temporarily borrow from R's `lme4` package (because Python really doesn't have any good mixed model packages).


```python
utils = rpackages.importr('utils')
base = rpackages.importr('base')

utils.chooseCRANmirror(ind=1) # select the first mirror in the list

packnames = ('lme4', 'Matrix')

from rpy2.robjects.vectors import StrVector

names_to_install = [x for x in packnames if not rpackages.isinstalled(x)]
    
if len(names_to_install) > 0:
    utils.install_packages(StrVector(names_to_install))

ro.r('''
    gen_ratings = function(df){
        model <- lme4::lmer(Score ~ 0 + Loc + (1|Offense) + (1|Defense),
                            data = df)

        offense_ratings = lme4::ranef(model)$Offense
        offense_ratings$Team = row.names(lme4::ranef(model)$Offense)
        colnames(offense_ratings) = c('Off','TeamID')
        defense_ratings = lme4::ranef(model)$Defense
        defense_ratings$Team = row.names(lme4::ranef(model)$Defense)
        colnames(defense_ratings) = c('Def','TeamID')
        
        rtg_df = merge(offense_ratings, defense_ratings,on='TeamID')
        return(rtg_df)
    }
''')

gen_ratings = ro.globalenv['gen_ratings']

def calc_lme(dataframe, hfa_rescore = True, ot_rescore = True):
    df = dataframe
    
    if hfa_rescore:
        df = rescore_hfa(df)
    if ot_rescore:
        df = rescore_ot(df)
        
    home_scoring = df[['HTeamID','ATeamID','HScore','NeutralFlg']].rename(columns={'HTeamID':'Offense','ATeamID':'Defense',
                                                                                   'HScore':'Score','NeutralFlg':'NeutralFlg'})
    home_scoring['Loc'] = ''
    home_scoring.loc[home_scoring['NeutralFlg'],'Loc'] = 'N'
    home_scoring.loc[~home_scoring['NeutralFlg'],'Loc'] = 'H'
    
    away_scoring = df[['ATeamID','HTeamID','AScore','NeutralFlg']].rename(columns={'ATeamID':'Offense','HTeamID':'Defense',
                                                                                   'AScore':'Score','NeutralFlg':'NeutralFlg'})
    away_scoring['Loc'] = ''
    away_scoring.loc[away_scoring['NeutralFlg'],'Loc'] = 'N'
    away_scoring.loc[~away_scoring['NeutralFlg'],'Loc'] = 'A'    
    
    scoring = pd.concat([home_scoring,away_scoring])
    
    with localconverter(ro.default_converter + pandas2ri.converter):
        df_r = ro.conversion.py2rpy(scoring)
    
    df_pd = gen_ratings(df_r)
    
    with localconverter(ro.default_converter + pandas2ri.converter):
          rtg_df = ro.conversion.rpy2py(df_pd)
            
    return pd.DataFrame({'Off':list(rtg_df['Off']),'Def':list(rtg_df['Def'])},index=list(rtg_df['TeamID']))
    

calc_lme(testing_df).head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Off</th>
      <th>Def</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1101</th>
      <td>0.037220</td>
      <td>3.001805</td>
    </tr>
    <tr>
      <th>1102</th>
      <td>-2.803390</td>
      <td>-0.991113</td>
    </tr>
    <tr>
      <th>1103</th>
      <td>-0.359363</td>
      <td>-1.211838</td>
    </tr>
    <tr>
      <th>1104</th>
      <td>1.990535</td>
      <td>0.245586</td>
    </tr>
    <tr>
      <th>1105</th>
      <td>-2.146115</td>
      <td>-1.168408</td>
    </tr>
  </tbody>
</table>
</div>



Home stretch: Colley Matrix Method!


```python
def calc_colley(dataframe, hfa_rescore = True, ot_rescore = True):
    df = dataframe
    
    if hfa_rescore:
        df = rescore_hfa(df)
    if ot_rescore:
        df = rescore_ot(df)
        
    teams = list(set(df['HTeamID'].append(df['ATeamID'])))
    n_teams = len(teams)
    c_mat = np.identity(n_teams) * 2
    b_mat = np.zeros(n_teams)
    
    for index, row in df.iterrows():
        home_team_ndx = teams.index(row['HTeamID'])
        away_team_ndx = teams.index(row['ATeamID'])
        spread = row['HScore'] - row['AScore']

        c_mat[home_team_ndx, home_team_ndx] += 1
        c_mat[away_team_ndx, away_team_ndx] += 1
        c_mat[home_team_ndx, away_team_ndx] -= 1
        c_mat[away_team_ndx, home_team_ndx] -= 1

        b_mat[home_team_ndx] += (row['HScore'] > row['AScore']) * 1 + (row['AScore'] > row['HScore']) * -1
        b_mat[away_team_ndx] += (row['AScore'] > row['HScore']) * 1 + (row['HScore'] > row['AScore']) * -1

    b_mat = b_mat / 2 + 1
    ranks = np.linalg.solve(c_mat,b_mat).tolist()
    return pd.DataFrame({'Colley':ranks},index = teams)

calc_colley(testing_df).head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Colley</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1106</th>
      <td>0.429206</td>
    </tr>
    <tr>
      <th>1219</th>
      <td>0.427629</td>
    </tr>
    <tr>
      <th>1285</th>
      <td>0.508115</td>
    </tr>
    <tr>
      <th>1350</th>
      <td>0.670517</td>
    </tr>
    <tr>
      <th>1249</th>
      <td>0.091091</td>
    </tr>
  </tbody>
</table>
</div>



Finally, we'll lift a couple prior pieces of code to calculate win percent and MOV.


```python
def calc_mov(dataframe, hfa_rescore=True, ot_rescore=True):
    df = dataframe
    
    if hfa_rescore:
        df = rescore_hfa(df)
    if ot_rescore:
        df = rescore_ot(df)
        
    df['HSpread'] = df['HScore'] - df['AScore']
    df['ASpread'] = -1 * df['HSpread']
    
    teams = pd.concat([
        df[['HTeamID','HSpread','ATeamID']].rename(columns={'HTeamID':'TeamID','HSpread':'Spread','ATeamID':'OppID'}),
        df[['ATeamID','ASpread','HTeamID']].rename(columns={'ATeamID':'TeamID','ASpread':'Spread','HTeamID':'OppID'})
    ])
    
    spreads = teams.groupby('TeamID').Spread.mean()
    return pd.DataFrame({'MOV':spreads},index = list(spreads.index))

calc_mov(testing_df).head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>MOV</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1101</th>
      <td>-17.664136</td>
    </tr>
    <tr>
      <th>1102</th>
      <td>-6.344923</td>
    </tr>
    <tr>
      <th>1103</th>
      <td>4.572178</td>
    </tr>
    <tr>
      <th>1104</th>
      <td>11.177569</td>
    </tr>
    <tr>
      <th>1105</th>
      <td>-3.956765</td>
    </tr>
  </tbody>
</table>
</div>




```python
def calc_wp(dataframe, hfa_rescore=True, ot_rescore=True):
    df = dataframe
    
    if hfa_rescore:
        df = rescore_hfa(df)
    if ot_rescore:
        df = rescore_ot(df)
        
    df['HWin'] = df['HScore'].ge(df['AScore']) * 1
    df['AWin'] = 1 - df['HWin']
    df['HLoss'] = 1 - df['HWin']
    df['ALoss'] = df['HWin']
    
    teams = pd.concat([
        df[['HTeamID','ATeamID','HWin','HLoss']].rename(columns={'HTeamID':'TeamID','ATeamID':'OppID','HWin':'HWin',
                                                                 'HLoss':'HLoss'}),
        df[['ATeamID','HTeamID','AWin','ALoss']].rename(columns={'ATeamID':'TeamID','HTeamID':'OppID','AWin':'AWin',
                                                                 'ALoss':'ALoss'})
    ])
    
    teams = teams.fillna(0)
    
    wp = teams.groupby('TeamID')[['HWin','HLoss','AWin','ALoss']].sum()
    wp['W'] = wp['HWin'] + wp['AWin']
    wp['L'] = wp['HLoss'] + wp['ALoss']
    wp['WP'] = wp['W'] / (wp['W'] + wp['L'])
    
    return pd.DataFrame({'WP':wp['WP']},index = list(wp.index))

calc_wp(testing_df).head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>WP</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1101</th>
      <td>0.095238</td>
    </tr>
    <tr>
      <th>1102</th>
      <td>0.357143</td>
    </tr>
    <tr>
      <th>1103</th>
      <td>0.666667</td>
    </tr>
    <tr>
      <th>1104</th>
      <td>0.516129</td>
    </tr>
    <tr>
      <th>1105</th>
      <td>0.428571</td>
    </tr>
  </tbody>
</table>
</div>



Alright, that's the bulk of the heavy lifting done for generating team ratings from only schedule data. Next, we're going to estimate team factors from advanced box score data, which is only available from 2002-on. We'll use `lme4` for this as well.

First, we'll read in our data.


```python
detailed_results = pd.read_csv('data/MRegularSeasonDetailedResults.csv')
detailed_results['WTeamID'] = detailed_results['WTeamID'].astype('str')
detailed_results['LTeamID'] = detailed_results['LTeamID'].astype('str')
detailed_results.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Season</th>
      <th>DayNum</th>
      <th>WTeamID</th>
      <th>WScore</th>
      <th>LTeamID</th>
      <th>LScore</th>
      <th>WLoc</th>
      <th>NumOT</th>
      <th>WFGM</th>
      <th>WFGA</th>
      <th>...</th>
      <th>LFGA3</th>
      <th>LFTM</th>
      <th>LFTA</th>
      <th>LOR</th>
      <th>LDR</th>
      <th>LAst</th>
      <th>LTO</th>
      <th>LStl</th>
      <th>LBlk</th>
      <th>LPF</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2003</td>
      <td>10</td>
      <td>1104</td>
      <td>68</td>
      <td>1328</td>
      <td>62</td>
      <td>N</td>
      <td>0</td>
      <td>27</td>
      <td>58</td>
      <td>...</td>
      <td>10</td>
      <td>16</td>
      <td>22</td>
      <td>10</td>
      <td>22</td>
      <td>8</td>
      <td>18</td>
      <td>9</td>
      <td>2</td>
      <td>20</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2003</td>
      <td>10</td>
      <td>1272</td>
      <td>70</td>
      <td>1393</td>
      <td>63</td>
      <td>N</td>
      <td>0</td>
      <td>26</td>
      <td>62</td>
      <td>...</td>
      <td>24</td>
      <td>9</td>
      <td>20</td>
      <td>20</td>
      <td>25</td>
      <td>7</td>
      <td>12</td>
      <td>8</td>
      <td>6</td>
      <td>16</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2003</td>
      <td>11</td>
      <td>1266</td>
      <td>73</td>
      <td>1437</td>
      <td>61</td>
      <td>N</td>
      <td>0</td>
      <td>24</td>
      <td>58</td>
      <td>...</td>
      <td>26</td>
      <td>14</td>
      <td>23</td>
      <td>31</td>
      <td>22</td>
      <td>9</td>
      <td>12</td>
      <td>2</td>
      <td>5</td>
      <td>23</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2003</td>
      <td>11</td>
      <td>1296</td>
      <td>56</td>
      <td>1457</td>
      <td>50</td>
      <td>N</td>
      <td>0</td>
      <td>18</td>
      <td>38</td>
      <td>...</td>
      <td>22</td>
      <td>8</td>
      <td>15</td>
      <td>17</td>
      <td>20</td>
      <td>9</td>
      <td>19</td>
      <td>4</td>
      <td>3</td>
      <td>23</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2003</td>
      <td>11</td>
      <td>1400</td>
      <td>77</td>
      <td>1208</td>
      <td>71</td>
      <td>N</td>
      <td>0</td>
      <td>30</td>
      <td>61</td>
      <td>...</td>
      <td>16</td>
      <td>17</td>
      <td>27</td>
      <td>21</td>
      <td>15</td>
      <td>12</td>
      <td>10</td>
      <td>7</td>
      <td>1</td>
      <td>14</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 34 columns</p>
</div>



We'll first write a quick function converting these results to our preferred format.


```python
def refactor_games_detailed(dataframe):
    df = dataframe
    neutral_winners = df.loc[df.WLoc == 'N']
    home_winners = df.loc[df.WLoc == 'H']
    away_winners = df.loc[df.WLoc == 'A']
    
    neutral_winners['NeutralFlg'] = True
    neutral_winners.columns = ['Season','DayNum','HTeamID','HScore','ATeamID','AScore','WLoc','NumOT',
                               'HFGM','HFGA','HFGM3','HFGA3','HFTM','HFTA','HOR','HDR','HAST','HTO','HSTL','HBLK','HPF',
                               'AFGM','AFGA','AFGM3','AFGA3','AFTM','AFTA','AOR','ADR','AAST','ATO','ASTL','ABLK','APF',
                               'NeutralFlg']
    
    home_winners['NeutralFlg'] = False
    home_winners.columns = ['Season','DayNum','HTeamID','HScore','ATeamID','AScore','WLoc','NumOT',
                            'HFGM','HFGA','HFGM3','HFGA3','HFTM','HFTA','HOR','HDR','HAST','HTO','HSTL','HBLK','HPF',
                            'AFGM','AFGA','AFGM3','AFGA3','AFTM','AFTA','AOR','ADR','AAST','ATO','ASTL','ABLK','APF',
                            'NeutralFlg']
    
    away_winners['NeutralFlg'] = False
    away_winners.columns = ['Season','DayNum','HTeamID','HScore','ATeamID','AScore','WLoc','NumOT',
                            'AFGM','AFGA','AFGM3','AFGA3','AFTM','AFTA','AOR','ADR','AAST','ATO','ASTL','ABLK','APF',
                            'HFGM','HFGA','HFGM3','HFGA3','HFTM','HFTA','HOR','HDR','HAST','HTO','HSTL','HBLK','HPF',
                            'NeutralFlg']
    
    df = neutral_winners.append(home_winners.append(away_winners))
    df = df.drop(columns=['WLoc'])
    df = df.sort_values(by=['Season','DayNum'])
    return df
    
detailed_results = refactor_games_detailed(detailed_results)
detailed_results.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Season</th>
      <th>DayNum</th>
      <th>HTeamID</th>
      <th>HScore</th>
      <th>ATeamID</th>
      <th>AScore</th>
      <th>NumOT</th>
      <th>HFGM</th>
      <th>HFGA</th>
      <th>HFGM3</th>
      <th>...</th>
      <th>AFTM</th>
      <th>AFTA</th>
      <th>AOR</th>
      <th>ADR</th>
      <th>AAST</th>
      <th>ATO</th>
      <th>ASTL</th>
      <th>ABLK</th>
      <th>APF</th>
      <th>NeutralFlg</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2003</td>
      <td>10</td>
      <td>1104</td>
      <td>68</td>
      <td>1328</td>
      <td>62</td>
      <td>0</td>
      <td>27</td>
      <td>58</td>
      <td>3</td>
      <td>...</td>
      <td>16</td>
      <td>22</td>
      <td>10</td>
      <td>22</td>
      <td>8</td>
      <td>18</td>
      <td>9</td>
      <td>2</td>
      <td>20</td>
      <td>True</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2003</td>
      <td>10</td>
      <td>1272</td>
      <td>70</td>
      <td>1393</td>
      <td>63</td>
      <td>0</td>
      <td>26</td>
      <td>62</td>
      <td>8</td>
      <td>...</td>
      <td>9</td>
      <td>20</td>
      <td>20</td>
      <td>25</td>
      <td>7</td>
      <td>12</td>
      <td>8</td>
      <td>6</td>
      <td>16</td>
      <td>True</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2003</td>
      <td>11</td>
      <td>1266</td>
      <td>73</td>
      <td>1437</td>
      <td>61</td>
      <td>0</td>
      <td>24</td>
      <td>58</td>
      <td>8</td>
      <td>...</td>
      <td>14</td>
      <td>23</td>
      <td>31</td>
      <td>22</td>
      <td>9</td>
      <td>12</td>
      <td>2</td>
      <td>5</td>
      <td>23</td>
      <td>True</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2003</td>
      <td>11</td>
      <td>1296</td>
      <td>56</td>
      <td>1457</td>
      <td>50</td>
      <td>0</td>
      <td>18</td>
      <td>38</td>
      <td>3</td>
      <td>...</td>
      <td>8</td>
      <td>15</td>
      <td>17</td>
      <td>20</td>
      <td>9</td>
      <td>19</td>
      <td>4</td>
      <td>3</td>
      <td>23</td>
      <td>True</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2003</td>
      <td>11</td>
      <td>1400</td>
      <td>77</td>
      <td>1208</td>
      <td>71</td>
      <td>0</td>
      <td>30</td>
      <td>61</td>
      <td>6</td>
      <td>...</td>
      <td>17</td>
      <td>27</td>
      <td>21</td>
      <td>15</td>
      <td>12</td>
      <td>10</td>
      <td>7</td>
      <td>1</td>
      <td>14</td>
      <td>True</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 34 columns</p>
</div>



Our next function will generate a dataframe with several rating columns of interest, adjusted for opponent and home field advantage, which should give us an estimate of each team's overall strength in each box-score category.


```python
ro.r('''
    gen_custom_rating = function(df, term){
        formula = as.formula(paste0(term,' ~ 0 + Loc + (1|TeamID) + (1|OppID)'))
        model <- lme4::lmer(formula,
                            data = df)

        made_ratings = lme4::ranef(model)$TeamID
        made_ratings$Team = row.names(lme4::ranef(model)$TeamID)
        colnames(made_ratings) = c(paste0(term,'Made'),'TeamID')
        allowed_ratings = lme4::ranef(model)$OppID
        allowed_ratings$Team = row.names(lme4::ranef(model)$OppID)
        colnames(allowed_ratings) = c(paste0(term,'Allowed'),'TeamID')
        
        rtg_df = merge(made_ratings, allowed_ratings ,on='TeamID')
        return(rtg_df)
    }
''')

gen_custom_rating = ro.globalenv['gen_custom_rating']

def calc_box_score_rtgs(dataframe, raw = False):
    df = dataframe
    
    # calculate possessions as our denominator
    df['HPOS'] = (df['HFGA'] - df['HOR']) + df['HTO'] + (0.44 * df['HFTA'])
    df['APOS'] = (df['AFGA'] - df['AOR']) + df['ATO'] + (0.44 * df['AFTA'])
    
    # calculate stats per possession
    df['HFGM_POS'] = df['HFGM'] / df['HPOS']
    df['HFGA_POS'] = df['HFGA'] / df['HPOS']
    df['HFGM3_POS'] = df['HFGM3'] / df['HPOS']
    df['HFGA3_POS'] = df['HFGA3'] / df['HPOS']
    df['HFTM_POS'] = df['HFTM'] / df['HPOS']
    df['HFTA_POS'] = df['HFTA'] / df['HPOS']
    df['HOR_POS'] = df['HOR'] / df['HPOS']
    df['HDR_POS'] = df['HDR'] / df['APOS'] # denominator is opponent possesions because these are a defensive stat
    df['HAST_POS'] = df['HAST'] / df['HPOS']
    df['HTO_POS'] = df['HTO'] / df['HPOS']
    df['HSTL_POS'] = df['HSTL'] / df['APOS'] # likewise
    df['HBLK_POS'] = df['HBLK'] / df['APOS'] 
    df['HPF_POS'] = df['HPF'] / (df['HPOS'] + df['APOS']) # denom is all possessions
    df['HPTS_POS'] = df['HScore'] / df['HPOS']
    
    df['AFGM_POS'] = df['AFGM'] / df['APOS']
    df['AFGA_POS'] = df['AFGA'] / df['APOS']
    df['AFGM3_POS'] = df['AFGM3'] / df['APOS']
    df['AFGA3_POS'] = df['AFGA3'] / df['APOS']
    df['AFTM_POS'] = df['AFTM'] / df['APOS']
    df['AFTA_POS'] = df['AFTA'] / df['APOS']
    df['AOR_POS'] = df['AOR'] / df['APOS']
    df['ADR_POS'] = df['ADR'] / df['HPOS']
    df['AAST_POS'] = df['AAST'] / df['APOS']
    df['ATO_POS'] = df['ATO'] / df['APOS']
    df['ASTL_POS'] = df['ASTL'] / df['HPOS']
    df['ABLK_POS'] = df['ABLK'] / df['HPOS']
    df['APF_POS'] = df['APF'] / (df['HPOS'] + df['APOS'])
    df['APTS_POS'] = df['AScore'] / df['APOS']
    
    # calculate "tempo", an estimate of possession per 40 minutes
    
    df['HTEMPO'] = 40 * df['HPOS'] / (40 + df['NumOT'] * 5)
    df['ATEMPO'] = 40 * df['APOS'] / (40 + df['NumOT'] * 5)
    
    home_df = df[['HTeamID','ATeamID','HFGM_POS','HFGA_POS','HFGM3_POS','HFGA3_POS','HFTM_POS','HFTA_POS','HOR_POS',
                  'HDR_POS','HAST_POS','HTO_POS','HSTL_POS','HBLK_POS','HPF_POS','HPTS_POS','HTEMPO','NeutralFlg']]
    home_df.columns = ['TeamID','OppID','FGM_POS','FGA_POS','FGM3_POS','FGA3_POS','FTM_POS','FTA_POS','OR_POS',
                       'DR_POS','AST_POS','TO_POS','STL_POS','BLK_POS','PF_POS','PTS_POS','TEMPO','NeutralFlg']
    home_df['Loc'] = ''
    home_df.loc[home_df['NeutralFlg'],'Loc'] = 'N'
    home_df.loc[~home_df['NeutralFlg'],'Loc'] = 'H'
    
    away_df = df[['ATeamID','HTeamID','AFGM_POS','AFGA_POS','AFGM3_POS','AFGA3_POS','AFTM_POS','AFTA_POS','AOR_POS',
                  'ADR_POS','AAST_POS','ATO_POS','ASTL_POS','ABLK_POS','APF_POS','APTS_POS','ATEMPO','NeutralFlg']]
    away_df.columns = ['TeamID','OppID','FGM_POS','FGA_POS','FGM3_POS','FGA3_POS','FTM_POS','FTA_POS','OR_POS',
                       'DR_POS','AST_POS','TO_POS','STL_POS','BLK_POS','PF_POS','PTS_POS','TEMPO','NeutralFlg']
    away_df['Loc'] = ''
    away_df.loc[home_df['NeutralFlg'],'Loc'] = 'N'
    away_df.loc[~home_df['NeutralFlg'],'Loc'] = 'A'
    
    main_df = pd.concat([home_df,away_df])
    
    if raw:
        main_df = main_df.sort_values(by=['TeamID'])
        made = main_df.groupby('TeamID')[['FGM_POS','FGA_POS','FGM3_POS','FGA3_POS','FTM_POS','FTA_POS','OR_POS','DR_POS',
                                          'AST_POS','TO_POS','STL_POS','BLK_POS','PF_POS','PTS_POS','TEMPO']].mean()
        main_df = main_df.sort_values(by=['OppID'])
        allowed = main_df.groupby('OppID')[['FGM_POS','FGA_POS','FGM3_POS','FGA3_POS','FTM_POS','FTA_POS','OR_POS',
                                            'DR_POS','AST_POS','TO_POS','STL_POS','BLK_POS','PF_POS','PTS_POS','TEMPO']].mean()
        
        made.columns = ['FGM_POSMadeRAW','FGA_POSMadeRAW','FGM3_POSMadeRAW','FGA3_POSMadeRAW','FTM_POSMadeRAW','FTA_POSMadeRAW',
                        'OR_POSMadeRAW','DR_POSMadeRAW','AST_POSMadeRAW','TO_POSMadeRAW','STL_POSMadeRAW','BLK_POSMadeRAW',
                        'PF_POSMadeRAW','PTS_POSMadeRAW','TEMPOMadeRAW']
        
        allowed.columns = ['FGM_POSAllowedRAW','FGA_POSAllowedRAW','FGM3_POSAllowedRAW','FGA3_POSAllowedRAW',
                           'FTM_POSAllowedRAW','FTA_POSAllowedRAW','OR_POSAllowedRAW','DR_POSAllowedRAW','AST_POSAllowedRAW',
                           'TO_POSAllowedRAW','STL_POSAllowedRAW','BLK_POSAllowedRAW','PF_POSAllowedRAW','PTS_POSAllowedRAW',
                           'TEMPOAllowedRAW']
        
        
        
        made = pd.DataFrame(made)
        allowed = pd.DataFrame(allowed)
        
        main_df = pd.concat([made, allowed], axis = 1)
        return main_df
        
        
    with localconverter(ro.default_converter + pandas2ri.converter):
        df_r = ro.conversion.py2rpy(main_df)
    
    # next, we run a lmer on each column to adjust for HFA and opponent to estimate how frequently a team does something 
    # per possession
    
    terms = ['FGM_POS','FGA_POS','FGM3_POS','FGA3_POS','FTM_POS','FTA_POS','OR_POS','DR_POS','AST_POS','TO_POS','STL_POS',
             'BLK_POS','PF_POS','PTS_POS','TEMPO']
    df_list = []
    for term in terms:
        # print('Calculating ratings for ' + term + '...')
        df_pd = gen_custom_rating(df_r, term)
        
        with localconverter(ro.default_converter + pandas2ri.converter):
            rtg_df = ro.conversion.rpy2py(df_pd)
        
        df_list.append(rtg_df)
        
    indexes = df_list[0].TeamID
    df_list = [df.drop(columns = ['TeamID']) for df in df_list]
    main_df = pd.concat(df_list,axis = 1)
    main_df.index = list(indexes)
    return(main_df)
```

Alright, almost there. We'll make two quick functions for feature engineering. One of them will simply return the Massey ratings for each team immediately prior to the tournament for a given dataframe, the other will return the tournament seed for each team (if applicable).


```python
def grab_massey(dataframe, massey):
    ms = massey
    df = dataframe
    
    year = str(df.iloc[0].Season)
    
    ms = ms.loc[ms['Season'].astype('str') == year]
    ms = ms.loc[ms['RankingDayNum'] == 133]
    ms = ms[['SystemName','TeamID','OrdinalRank']]
    
    ms = ms.pivot(index='TeamID',columns='SystemName')
    ms.columns=[y for y in ms.columns.get_level_values(1)]
    ms['AVG'] = ms.mean(axis=1)
    
    for col in ms.columns: # impute missing values as the mean of massey rankings
        ms[col].fillna(ms.AVG, inplace=True)
        
    del ms['AVG']
    ms.index.name = None
    
    return (ms)

mass = pd.read_csv('data/MMasseyOrdinals.csv')  
mass['TeamID'] = mass['TeamID'].astype('str')
grab_massey(testing_df,mass).head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>7OT</th>
      <th>ADE</th>
      <th>AP</th>
      <th>BBT</th>
      <th>BIH</th>
      <th>BLS</th>
      <th>BOB</th>
      <th>BUR</th>
      <th>CJB</th>
      <th>CNG</th>
      <th>...</th>
      <th>TPR</th>
      <th>TRP</th>
      <th>TW</th>
      <th>UPS</th>
      <th>USA</th>
      <th>WIL</th>
      <th>WLK</th>
      <th>WMR</th>
      <th>WOB</th>
      <th>WOL</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1101</th>
      <td>343.0</td>
      <td>297.0</td>
      <td>338.163636</td>
      <td>347.0</td>
      <td>346.0</td>
      <td>346.0</td>
      <td>340.0</td>
      <td>342.0</td>
      <td>332.0</td>
      <td>346.0</td>
      <td>...</td>
      <td>348.0</td>
      <td>339.0</td>
      <td>329.0</td>
      <td>338.163636</td>
      <td>338.163636</td>
      <td>342.0</td>
      <td>330.0</td>
      <td>346.0</td>
      <td>323.0</td>
      <td>345.0</td>
    </tr>
    <tr>
      <th>1102</th>
      <td>290.0</td>
      <td>241.0</td>
      <td>229.196721</td>
      <td>266.0</td>
      <td>227.0</td>
      <td>231.0</td>
      <td>220.0</td>
      <td>207.0</td>
      <td>232.0</td>
      <td>225.0</td>
      <td>...</td>
      <td>234.0</td>
      <td>240.0</td>
      <td>129.0</td>
      <td>236.000000</td>
      <td>229.196721</td>
      <td>193.0</td>
      <td>228.0</td>
      <td>209.0</td>
      <td>206.0</td>
      <td>233.0</td>
    </tr>
    <tr>
      <th>1103</th>
      <td>103.0</td>
      <td>89.0</td>
      <td>113.885246</td>
      <td>114.0</td>
      <td>104.0</td>
      <td>142.0</td>
      <td>123.0</td>
      <td>128.0</td>
      <td>113.0</td>
      <td>138.0</td>
      <td>...</td>
      <td>136.0</td>
      <td>129.0</td>
      <td>124.0</td>
      <td>93.000000</td>
      <td>113.885246</td>
      <td>99.0</td>
      <td>126.0</td>
      <td>132.0</td>
      <td>94.0</td>
      <td>93.0</td>
    </tr>
    <tr>
      <th>1104</th>
      <td>82.0</td>
      <td>129.0</td>
      <td>106.967213</td>
      <td>111.0</td>
      <td>123.0</td>
      <td>75.0</td>
      <td>92.0</td>
      <td>82.0</td>
      <td>110.0</td>
      <td>75.0</td>
      <td>...</td>
      <td>79.0</td>
      <td>72.0</td>
      <td>178.0</td>
      <td>124.000000</td>
      <td>106.967213</td>
      <td>104.0</td>
      <td>84.0</td>
      <td>74.0</td>
      <td>116.0</td>
      <td>129.0</td>
    </tr>
    <tr>
      <th>1105</th>
      <td>288.0</td>
      <td>262.0</td>
      <td>292.344262</td>
      <td>286.0</td>
      <td>288.0</td>
      <td>314.0</td>
      <td>305.0</td>
      <td>306.0</td>
      <td>300.0</td>
      <td>309.0</td>
      <td>...</td>
      <td>314.0</td>
      <td>325.0</td>
      <td>285.0</td>
      <td>259.000000</td>
      <td>292.344262</td>
      <td>285.0</td>
      <td>308.0</td>
      <td>319.0</td>
      <td>279.0</td>
      <td>275.0</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 65 columns</p>
</div>




```python
def grab_seeding(dataframe, seeding):
    sd = seeding
    df = dataframe
    
    year = str(df.iloc[0].Season)
    sd = sd.loc[sd['Season'].astype('str') == year]
    
    teams = list(set(df['HTeamID'].append(df['ATeamID'])))
    teams = pd.DataFrame({'TeamID':teams})
    
    main_df = pd.merge(teams,sd,how='outer')
    main_df['SeedNum'] = main_df['Seed'].str[1:3]
    main_df['Seed'].fillna('NS',inplace=True)
    main_df['SeedNum'].fillna('NS',inplace=True)
    main_df.index = list(main_df['TeamID'])
    del main_df['TeamID']
    del main_df['Season']
    return main_df
    
seed = pd.read_csv('data/MNCAATourneySeeds.csv')  
seed['TeamID'] = seed['TeamID'].astype('str')
grab_seeding(testing_df,seed).head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Seed</th>
      <th>SeedNum</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1106</th>
      <td>NS</td>
      <td>NS</td>
    </tr>
    <tr>
      <th>1219</th>
      <td>NS</td>
      <td>NS</td>
    </tr>
    <tr>
      <th>1285</th>
      <td>NS</td>
      <td>NS</td>
    </tr>
    <tr>
      <th>1350</th>
      <td>NS</td>
      <td>NS</td>
    </tr>
    <tr>
      <th>1249</th>
      <td>NS</td>
      <td>NS</td>
    </tr>
  </tbody>
</table>
</div>



Alright, we're done with all of our feature engineering. Kinda exhausting? Next step is to wrap all this into an enourmous function to generate a big-ass dataframe for generating ratings for every team, first for teams from 1985-2014, second from 2003-2014. 


```python
# score data

def gen_score_data(dataframe, seeding):
    df = dataframe
    
    elo = calc_elo(df,hfa_rescore=True,ot_rescore=True)
    lrmc = calc_lrmc(df)
    srs = calc_srs(df)
    rpi = calc_rpi(df)
    ts = calc_ts(df)
    lme = calc_lme(df)
    col = calc_colley(df)
    wp = calc_wp(df)
    mov = calc_mov(df)
    sd = grab_seeding(df, seeding)
    
    return pd.concat([elo, lrmc, srs, rpi, ts, lme, col, wp, mov, sd],axis=1)

gen_score_data(compact_results.loc[compact_results['Season'] == 1990], seeding = seed).head() # why 1990? no reason...
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Elo</th>
      <th>LRMC</th>
      <th>SRS</th>
      <th>RPI</th>
      <th>TS</th>
      <th>Off</th>
      <th>Def</th>
      <th>Colley</th>
      <th>WP</th>
      <th>MOV</th>
      <th>Seed</th>
      <th>SeedNum</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1106</th>
      <td>1457.690655</td>
      <td>114.959698</td>
      <td>-6.468152</td>
      <td>0.477352</td>
      <td>23.586893</td>
      <td>13.981185</td>
      <td>16.435633</td>
      <td>0.429118</td>
      <td>0.520000</td>
      <td>-3.809030</td>
      <td>NS</td>
      <td>NS</td>
    </tr>
    <tr>
      <th>1108</th>
      <td>1389.483739</td>
      <td>93.172849</td>
      <td>-11.313684</td>
      <td>0.457892</td>
      <td>20.018400</td>
      <td>1.030306</td>
      <td>7.373383</td>
      <td>0.339644</td>
      <td>0.360000</td>
      <td>-8.870126</td>
      <td>NS</td>
      <td>NS</td>
    </tr>
    <tr>
      <th>1425</th>
      <td>1491.137635</td>
      <td>131.578084</td>
      <td>-0.651833</td>
      <td>0.471407</td>
      <td>26.394144</td>
      <td>1.260576</td>
      <td>1.200736</td>
      <td>0.498420</td>
      <td>0.428571</td>
      <td>-0.500000</td>
      <td>NS</td>
      <td>NS</td>
    </tr>
    <tr>
      <th>1238</th>
      <td>1413.525492</td>
      <td>114.288471</td>
      <td>-6.379905</td>
      <td>0.440363</td>
      <td>19.715176</td>
      <td>-1.448090</td>
      <td>1.866536</td>
      <td>0.246491</td>
      <td>0.280000</td>
      <td>-3.879578</td>
      <td>NS</td>
      <td>NS</td>
    </tr>
    <tr>
      <th>1162</th>
      <td>1353.674199</td>
      <td>72.754592</td>
      <td>-14.088478</td>
      <td>0.386421</td>
      <td>14.703133</td>
      <td>-7.259959</td>
      <td>4.629205</td>
      <td>0.060706</td>
      <td>0.076923</td>
      <td>-15.385427</td>
      <td>NS</td>
      <td>NS</td>
    </tr>
  </tbody>
</table>
</div>




```python
def gen_adv_data(dataframe, det_dataframe, seeding, massey):
    df = dataframe
    det_df = det_dataframe
    
    elo = calc_elo(df,hfa_rescore=True,ot_rescore=True)
    lrmc = calc_lrmc(df)
    srs = calc_srs(df)
    rpi = calc_rpi(df)
    ts = calc_ts(df)
    lme = calc_lme(df)
    col = calc_colley(df)
    wp = calc_wp(df)
    mov = calc_mov(df)
    sd = grab_seeding(df, seeding)
    
    ms = grab_massey(df, massey)
    ppos = calc_box_score_rtgs(det_df)
    ppos_raw = calc_box_score_rtgs(det_df,raw = True)
    
    return pd.concat([elo, lrmc, srs, rpi, ts, lme, col, wp, mov, sd, ms, ppos, ppos_raw],axis=1)
```

Next step is to generate these ratings for every season and store them in an accessible manner.


```python
basic_ratings_dict = {}
for year in compact_results['Season'].unique():
    print('Generating basic ratings for ' + str(year) + '...')
    temp_df = compact_results.loc[compact_results['Season'] == int(year)]
    basic_ratings_dict[str(year)] = gen_score_data(temp_df, seed)

# basic_ratings_dict['2014'].head()
basic_list = []
for key in basic_ratings_dict.keys():
    tmp_df = basic_ratings_dict[key]
    tmp_df['Season'] = int(key)
    basic_list.append(tmp_df)
    
basic_ratings_df = pd.concat(basic_list)
basic_ratings_df.head()
```

    Generating basic ratings for 1985...
    Generating basic ratings for 1986...
    Generating basic ratings for 1987...
    Generating basic ratings for 1988...
    Generating basic ratings for 1989...
    Generating basic ratings for 1990...
    Generating basic ratings for 1991...
    Generating basic ratings for 1992...
    Generating basic ratings for 1993...
    Generating basic ratings for 1994...
    Generating basic ratings for 1995...
    Generating basic ratings for 1996...
    Generating basic ratings for 1997...
    Generating basic ratings for 1998...
    Generating basic ratings for 1999...
    Generating basic ratings for 2000...
    Generating basic ratings for 2001...
    Generating basic ratings for 2002...
    Generating basic ratings for 2003...
    Generating basic ratings for 2004...
    Generating basic ratings for 2005...
    Generating basic ratings for 2006...
    Generating basic ratings for 2007...
    Generating basic ratings for 2008...
    Generating basic ratings for 2009...
    Generating basic ratings for 2010...
    Generating basic ratings for 2011...
    Generating basic ratings for 2012...
    Generating basic ratings for 2013...
    Generating basic ratings for 2014...
    Generating basic ratings for 2015...
    Generating basic ratings for 2016...
    Generating basic ratings for 2017...
    Generating basic ratings for 2018...
    Generating basic ratings for 2019...
    Generating basic ratings for 2020...
    Generating basic ratings for 2021...
    




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Elo</th>
      <th>LRMC</th>
      <th>SRS</th>
      <th>RPI</th>
      <th>TS</th>
      <th>Off</th>
      <th>Def</th>
      <th>Colley</th>
      <th>WP</th>
      <th>MOV</th>
      <th>Seed</th>
      <th>SeedNum</th>
      <th>Season</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1106</th>
      <td>1461.053111</td>
      <td>107.258502</td>
      <td>-2.698899</td>
      <td>0.459318</td>
      <td>19.541427</td>
      <td>0.952114</td>
      <td>2.104203</td>
      <td>0.348392</td>
      <td>0.500000</td>
      <td>0.264010</td>
      <td>NS</td>
      <td>NS</td>
      <td>1985</td>
    </tr>
    <tr>
      <th>1108</th>
      <td>1573.024506</td>
      <td>199.131764</td>
      <td>6.038746</td>
      <td>0.533009</td>
      <td>25.704259</td>
      <td>7.593771</td>
      <td>3.356364</td>
      <td>0.564220</td>
      <td>0.680000</td>
      <td>4.066550</td>
      <td>NS</td>
      <td>NS</td>
      <td>1985</td>
    </tr>
    <tr>
      <th>1425</th>
      <td>1566.036711</td>
      <td>172.630828</td>
      <td>4.867537</td>
      <td>0.541256</td>
      <td>30.437145</td>
      <td>-0.582718</td>
      <td>-3.477101</td>
      <td>0.728202</td>
      <td>0.642857</td>
      <td>2.047567</td>
      <td>Y08</td>
      <td>08</td>
      <td>1985</td>
    </tr>
    <tr>
      <th>1238</th>
      <td>1429.471832</td>
      <td>103.695027</td>
      <td>-6.388395</td>
      <td>0.437554</td>
      <td>18.680670</td>
      <td>-2.368737</td>
      <td>1.600165</td>
      <td>0.256282</td>
      <td>0.333333</td>
      <td>-6.041667</td>
      <td>NS</td>
      <td>NS</td>
      <td>1985</td>
    </tr>
    <tr>
      <th>1162</th>
      <td>1494.682611</td>
      <td>104.933737</td>
      <td>-2.299174</td>
      <td>0.455248</td>
      <td>23.805288</td>
      <td>-6.828934</td>
      <td>-5.287778</td>
      <td>0.448240</td>
      <td>0.520000</td>
      <td>0.146725</td>
      <td>NS</td>
      <td>NS</td>
      <td>1985</td>
    </tr>
  </tbody>
</table>
</div>




```python
adv_ratings_list = []
for year in detailed_results['Season'].unique():
    print('Generating advanced ratings for ' + str(year) + '...')
    temp_df = compact_results.loc[compact_results['Season'] == int(year)]
    temp_det_df = detailed_results.loc[detailed_results['Season'] == int(year)]
    temp_adv_data = gen_adv_data(temp_df, temp_det_df, seed, mass)
    temp_adv_data['Season'] = year
    adv_ratings_list.append(temp_adv_data)
```

    Generating advanced ratings for 2003...
    Generating advanced ratings for 2004...
    Generating advanced ratings for 2005...
    Generating advanced ratings for 2006...
    Generating advanced ratings for 2007...
    Generating advanced ratings for 2008...
    Generating advanced ratings for 2009...
    Generating advanced ratings for 2010...
    Generating advanced ratings for 2011...
    Generating advanced ratings for 2012...
    Generating advanced ratings for 2013...
    Generating advanced ratings for 2014...
    Generating advanced ratings for 2015...
    Generating advanced ratings for 2016...
    Generating advanced ratings for 2017...
    Generating advanced ratings for 2018...
    Generating advanced ratings for 2019...
    Generating advanced ratings for 2020...
    Generating advanced ratings for 2021...
    


```python
tmp_list = []
for d in adv_ratings_list:
    d = d.loc[:,~d.columns.duplicated()]
    tmp_list.append(d)
adv_ratings_df = pd.concat(tmp_list)
```

We're all the way wrapped on feature engineering, we can down assemble our training and testing dataframes.


```python
tournament_games = pd.read_csv('data/MNCAATourneyCompactResults.csv')  
tournament_games['WTeamID'] = tournament_games['WTeamID'].astype('str')
tournament_games['LTeamID'] = tournament_games['LTeamID'].astype('str')
tournament_games['WLoc'] = 'N'
tournament_games = refactor_games_compact(tournament_games)

tournament_games.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Season</th>
      <th>DayNum</th>
      <th>HTeamID</th>
      <th>HScore</th>
      <th>ATeamID</th>
      <th>AScore</th>
      <th>NumOT</th>
      <th>NeutralFlg</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1985</td>
      <td>136</td>
      <td>1116</td>
      <td>63</td>
      <td>1234</td>
      <td>54</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1985</td>
      <td>136</td>
      <td>1120</td>
      <td>59</td>
      <td>1345</td>
      <td>58</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1985</td>
      <td>136</td>
      <td>1207</td>
      <td>68</td>
      <td>1250</td>
      <td>43</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1985</td>
      <td>136</td>
      <td>1229</td>
      <td>58</td>
      <td>1425</td>
      <td>55</td>
      <td>0</td>
      <td>True</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1985</td>
      <td>136</td>
      <td>1242</td>
      <td>49</td>
      <td>1325</td>
      <td>38</td>
      <td>0</td>
      <td>True</td>
    </tr>
  </tbody>
</table>
</div>



One trick I like to do is flip the data and double it, so the model doesn't get biased towards predicting winners or losers.


```python
tournament_games_flipped = tournament_games.copy()

tournament_games_flipped['HTeamID'] = tournament_games['ATeamID']
tournament_games_flipped['ATeamID'] = tournament_games['HTeamID']
tournament_games_flipped['HScore'] = tournament_games['AScore']
tournament_games_flipped['AScore'] = tournament_games['HScore']

tournament_games = pd.concat([tournament_games, tournament_games_flipped])

tournament_games['HDiff'] = tournament_games['HScore'] - tournament_games['AScore']
tournament_games.tail()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Season</th>
      <th>DayNum</th>
      <th>HTeamID</th>
      <th>HScore</th>
      <th>ATeamID</th>
      <th>AScore</th>
      <th>NumOT</th>
      <th>NeutralFlg</th>
      <th>HDiff</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2246</th>
      <td>2019</td>
      <td>146</td>
      <td>1246</td>
      <td>71</td>
      <td>1120</td>
      <td>77</td>
      <td>1</td>
      <td>True</td>
      <td>-6</td>
    </tr>
    <tr>
      <th>2247</th>
      <td>2019</td>
      <td>146</td>
      <td>1181</td>
      <td>67</td>
      <td>1277</td>
      <td>68</td>
      <td>0</td>
      <td>True</td>
      <td>-1</td>
    </tr>
    <tr>
      <th>2248</th>
      <td>2019</td>
      <td>152</td>
      <td>1277</td>
      <td>51</td>
      <td>1403</td>
      <td>61</td>
      <td>0</td>
      <td>True</td>
      <td>-10</td>
    </tr>
    <tr>
      <th>2249</th>
      <td>2019</td>
      <td>152</td>
      <td>1120</td>
      <td>62</td>
      <td>1438</td>
      <td>63</td>
      <td>0</td>
      <td>True</td>
      <td>-1</td>
    </tr>
    <tr>
      <th>2250</th>
      <td>2019</td>
      <td>154</td>
      <td>1403</td>
      <td>77</td>
      <td>1438</td>
      <td>85</td>
      <td>1</td>
      <td>True</td>
      <td>-8</td>
    </tr>
  </tbody>
</table>
</div>




```python
h_adv_ratings = adv_ratings_df.copy()
a_adv_ratings = adv_ratings_df.copy()

h_adv_ratings['HTeamID'] = h_adv_ratings.index
a_adv_ratings['ATeamID'] = a_adv_ratings.index

tournament_merged = pd.merge(tournament_games, h_adv_ratings)
tournament_merged = pd.merge(tournament_merged, a_adv_ratings, on = ['ATeamID','Season'], suffixes = ['_H','_A'])
tournament_merged.head()

```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Season</th>
      <th>DayNum</th>
      <th>HTeamID</th>
      <th>HScore</th>
      <th>ATeamID</th>
      <th>AScore</th>
      <th>NumOT</th>
      <th>NeutralFlg</th>
      <th>HDiff</th>
      <th>Elo_H</th>
      <th>...</th>
      <th>MGS_A</th>
      <th>NET_A</th>
      <th>PIR_A</th>
      <th>BNZ_A</th>
      <th>CBR_A</th>
      <th>CWL_A</th>
      <th>EMK_A</th>
      <th>FDM_A</th>
      <th>KCX_A</th>
      <th>RWP_A</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2003</td>
      <td>134</td>
      <td>1421</td>
      <td>92</td>
      <td>1411</td>
      <td>84</td>
      <td>1</td>
      <td>True</td>
      <td>8</td>
      <td>1443.840515</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2003</td>
      <td>137</td>
      <td>1421</td>
      <td>61</td>
      <td>1400</td>
      <td>82</td>
      <td>0</td>
      <td>True</td>
      <td>-21</td>
      <td>1443.840515</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2003</td>
      <td>144</td>
      <td>1163</td>
      <td>78</td>
      <td>1400</td>
      <td>82</td>
      <td>0</td>
      <td>True</td>
      <td>-4</td>
      <td>1615.681074</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2003</td>
      <td>146</td>
      <td>1277</td>
      <td>76</td>
      <td>1400</td>
      <td>85</td>
      <td>0</td>
      <td>True</td>
      <td>-9</td>
      <td>1621.855151</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2003</td>
      <td>139</td>
      <td>1345</td>
      <td>67</td>
      <td>1400</td>
      <td>77</td>
      <td>0</td>
      <td>True</td>
      <td>-10</td>
      <td>1575.549719</td>
      <td>...</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 493 columns</p>
</div>



All of our data prep work is done, I'm gonna swap over to R for some tidymodels work.


```python
tournament_merged.to_csv('data/adv_data.csv')

h_basic_ratings = basic_ratings_df.copy()
a_basic_ratings = basic_ratings_df.copy()

h_basic_ratings['HTeamID'] = h_basic_ratings.index
a_basic_ratings['ATeamID'] = a_basic_ratings.index

tournament_merged = pd.merge(tournament_games, h_basic_ratings)
tournament_merged = pd.merge(tournament_merged, a_basic_ratings, on = ['ATeamID','Season'], suffixes = ['_H','_A'])
tournament_merged.head()

tournament_merged.to_csv('data/basic_data.csv')
```

Actually, before we go, I'm going to prepare our stage 1 and stage 2 datasets.


```python
stage1 = pd.read_csv('MSampleSubmissionStage1.csv')
stage2 = pd.read_csv('MSampleSubmissionStage2.csv')

stage1['Season'] = stage1['ID'].str.slice(start=0,stop=4)
stage1['HTeamID'] = stage1['ID'].str.slice(start=5,stop=9)
stage1['ATeamID'] = stage1['ID'].str.slice(start=10,stop=14)
stage1['Season'] = stage1['Season'].astype('int')

stage1_tmp = stage1.copy()
stage1_tmp['HTeamID'] = stage1['ATeamID']
stage1_tmp['ATeamID'] = stage1['HTeamID']
stage1_merged = pd.merge(stage1, h_adv_ratings)
stage1_merged = pd.merge(stage1_merged, a_adv_ratings, on = ['ATeamID','Season'], suffixes = ['_H','_A'])
stage1_merged.to_csv('data/stage1_2021.csv')

stage2['Season'] = stage2['ID'].str.slice(start=0,stop=4)
stage2['HTeamID'] = stage2['ID'].str.slice(start=5,stop=9)
stage2['ATeamID'] = stage2['ID'].str.slice(start=10,stop=14)
stage2['Season'] = stage2['Season'].astype('int')

stage2_tmp = stage1.copy()
stage2_tmp['HTeamID'] = stage2['ATeamID']
stage2_tmp['ATeamID'] = stage2['HTeamID']

stage2_merged = pd.merge(stage2, h_adv_ratings)
stage2_merged = pd.merge(stage2_merged, a_adv_ratings, on = ['ATeamID','Season'], suffixes = ['_H','_A'])
stage2_merged.to_csv('data/stage2_2021.csv')
```


```python
%load_ext rpy2.ipython
```

    C:\Users\edwar\anaconda3\lib\site-packages\rpy2\robjects\packages.py:366: UserWarning: The symbol 'quartz' is not in this R namespace/package.
      warnings.warn(
    


```r
# initializations
library(tidyverse)
library(tidymodels)
library(recipeselectors)
library(MLmetrics)
adv_data <- read.csv("C:/Users/edwar/Documents/GitHub/kaggle-mens-march-madness-2021/data/adv_data.csv", header=TRUE) %>%
  mutate(HWin = HDiff > 0,
         SeedDiff = as.numeric(SeedNum_H) - as.numeric(SeedNum_A)) %>%
  select(-c(X, Seed_H, Seed_A, SeedNum_A, SeedNum_H))
```

    R[write to console]: 
    Attaching package: 'MLmetrics'
    
    
    R[write to console]: The following object is masked from 'package:base':
    
        Recall
    
    
    

What follows is summarizing the result of some experiemnting in RStudio. I'm just keeping things together in the same notebook, this wasn't my exact thought process but is a more refined means of working through things.

First step is to grab train/test data. We'll hold out the 2019 tournament for evaluation. 


```r
pre_2015 = adv_data %>%
  filter(Season <= 2018) %>%
  select(-c(Season, DayNum, HTeamID, HScore, ATeamID, AScore, NumOT, NeutralFlg))

post_2015 = adv_data %>%
  filter(Season == 2019) %>%
  select(-c(Season, DayNum, HTeamID, HScore, ATeamID, AScore, NumOT, NeutralFlg))
```

Next step, we'll see which columns have a large number of NAs, and avoid imputing these in our preprocessing.


```r
colSums(is.na(pre_2015))
```

                   HDiff                Elo_H               LRMC_H 
                       0                    0                    0 
                   SRS_H                RPI_H                 TS_H 
                       0                    0                    0 
                   Off_H                Def_H             Colley_H 
                       0                    0                    0 
                    WP_H                MOV_H                 AP_H 
                       0                    0                    0 
                   ARG_H                BIH_H                BOB_H 
                    1310                  128                  268 
                   BRZ_H                COL_H                DOL_H 
                    1968                    0                    0 
                   DUN_H                DWH_H                ECK_H 
                     524                 1456                 1456 
                   ENT_H                ERD_H                GRN_H 
                    1712                 1968                 1072 
                   GRS_H                HER_H                HOL_H 
                    1706                 1712                 1968 
                   IMS_H                MAS_H                MKV_H 
                    1840                  512                 1712 
                   MOR_H                POM_H                RTH_H 
                       0                    0                    0 
                   SAG_H                SAU_H                 SE_H 
                       0                 1712                  658 
                   SEL_H                STR_H                TSR_H 
                     256                 1712                 1456 
                   USA_H                WLK_H                WOB_H 
                       0                    0                  128 
                   WOL_H                WTE_H        FGM_POSMade_H 
                       0                 1968                    0 
        FGM_POSAllowed_H        FGA_POSMade_H     FGA_POSAllowed_H 
                       0                    0                    0 
          FGM3_POSMade_H    FGM3_POSAllowed_H       FGA3_POSMade_H 
                       0                    0                    0 
       FGA3_POSAllowed_H        FTM_POSMade_H     FTM_POSAllowed_H 
                       0                    0                    0 
           FTA_POSMade_H     FTA_POSAllowed_H         OR_POSMade_H 
                       0                    0                    0 
         OR_POSAllowed_H         DR_POSMade_H      DR_POSAllowed_H 
                       0                    0                    0 
           AST_POSMade_H     AST_POSAllowed_H         TO_POSMade_H 
                       0                    0                    0 
         TO_POSAllowed_H        STL_POSMade_H     STL_POSAllowed_H 
                       0                    0                    0 
           BLK_POSMade_H     BLK_POSAllowed_H         PF_POSMade_H 
                       0                    0                    0 
         PF_POSAllowed_H        PTS_POSMade_H     PTS_POSAllowed_H 
                       0                    0                    0 
             TEMPOMade_H       TEMPOAllowed_H     FGM_POSMadeRAW_H 
                       0                    0                    0 
        FGA_POSMadeRAW_H    FGM3_POSMadeRAW_H    FGA3_POSMadeRAW_H 
                       0                    0                    0 
        FTM_POSMadeRAW_H     FTA_POSMadeRAW_H      OR_POSMadeRAW_H 
                       0                    0                    0 
         DR_POSMadeRAW_H     AST_POSMadeRAW_H      TO_POSMadeRAW_H 
                       0                    0                    0 
        STL_POSMadeRAW_H     BLK_POSMadeRAW_H      PF_POSMadeRAW_H 
                       0                    0                    0 
        PTS_POSMadeRAW_H       TEMPOMadeRAW_H  FGM_POSAllowedRAW_H 
                       0                    0                    0 
     FGA_POSAllowedRAW_H FGM3_POSAllowedRAW_H FGA3_POSAllowedRAW_H 
                       0                    0                    0 
     FTM_POSAllowedRAW_H  FTA_POSAllowedRAW_H   OR_POSAllowedRAW_H 
                       0                    0                    0 
      DR_POSAllowedRAW_H  AST_POSAllowedRAW_H   TO_POSAllowedRAW_H 
                       0                    0                    0 
     STL_POSAllowedRAW_H  BLK_POSAllowedRAW_H   PF_POSAllowedRAW_H 
                       0                    0                    0 
     PTS_POSAllowedRAW_H    TEMPOAllowedRAW_H                 BD_H 
                       0                    0                 1840 
                   CNG_H                DES_H                JON_H 
                     128                  384                 1968 
                   LYN_H                MGY_H                NOR_H 
                     932                 1968                 1968 
                   REI_H                 RM_H                SIM_H 
                    1968                 1968                 1328 
                   ACU_H                BCM_H                CMV_H 
                    1444                 1968                 1968 
                    DC_H                KLK_H                REN_H 
                     390                 1456                 1712 
                   RIS_H                ROH_H                SAP_H 
                    1968                 1712                 1712 
                   SCR_H                WIL_H                DOK_H 
                    1584                  384                  384 
                   JCI_H                KPK_H                 MB_H 
                    1840                  768                 1054 
                    PH_H                PIG_H                PKL_H 
                    1968                  384                 1840 
                   TRX_H                CPR_H                ISR_H 
                    1712                  780                 1584 
                   KRA_H                LYD_H                RTR_H 
                     640                 1840                  780 
                   UCS_H                BKM_H                CPA_H 
                    1968                 1968                  908 
                   JEN_H                PGH_H                REW_H 
                    1444                  640                  640 
                   RSE_H                SPW_H                STH_H 
                    1968                  640                  640 
                   BPI_H                DC2_H                DCI_H 
                    1438                 1438                  768 
                   HKB_H                LMC_H                NOL_H 
                    1304                  768                  902 
                   OMY_H                RTB_H                KEL_H 
                    1566                 1304                 1968 
                   KMV_H                 RT_H                 TW_H 
                    1834                 1030                 1432 
                   AUS_H                KOS_H                PEQ_H 
                    1962                 1694                 1962 
                   PTS_H                ROG_H                RTP_H 
                    1694                 1828                 1024 
                   TMR_H               X7OT_H                ADE_H 
                    1426                 1158                 1426 
                   BBT_H                BNM_H                BUR_H 
                    1158                 1962                 1292 
                   CJB_H                CRO_H                EBP_H 
                    1694                 1292                 1158 
                   HAT_H                MSX_H                SFX_H 
                    1962                 1426                 1158 
                   TBD_H                BLS_H                D1A_H 
                    1962                 1426                 1560 
                   DII_H                KBM_H                TPR_H 
                    1292                 1694                 1292 
                   MvG_H                PPR_H                 SP_H 
                    1828                 1962                 1426 
                   SPR_H                STF_H                STS_H 
                    1426                 1828                 1828 
                   TRP_H                UPS_H                WMR_H 
                    1426                 1962                 1962 
                   BWE_H                LOG_H                TRK_H 
                    1560                 1694                 1560 
                   DAV_H                FAS_H                FSH_H 
                    1694                 1694                 1694 
                   HAS_H                HRN_H                KPI_H 
                    1694                 1962                 1694 
                   MCL_H                CRW_H                DDB_H 
                    1828                 1962                 1828 
                   ESR_H                FMG_H                HKS_H 
                    1828                 1962                 1962 
                   INP_H                JNG_H                JRT_H 
                    1962                 1828                 1962 
                   MUZ_H                OCT_H                PMC_H 
                    1962                 1962                 1962 
                   PRR_H                RSL_H                SGR_H 
                    1828                 1962                 1828 
                   SMN_H                SMS_H                YAG_H 
                    1828                 1828                 1828 
                   ZAM_H                BNT_H                COX_H 
                    1828                 1962                 1962 
                   JJK_H                LAB_H                MMG_H 
                    1962                 1962                 1962 
                   STM_H                WMV_H                AWS_H 
                    1962                 1962                 2096 
                   INC_H                LAW_H                LEF_H 
                    2096                 2096                 2096 
                   MGS_H                NET_H                PIR_H 
                    2096                 2096                 2096 
                   BNZ_H                CBR_H                CWL_H 
                    2096                 2096                 2096 
                   EMK_H                FDM_H                KCX_H 
                    2096                 2096                 2096 
                   RWP_H                Elo_A               LRMC_A 
                    2096                    0                    0 
                   SRS_A                RPI_A                 TS_A 
                       0                    0                    0 
                   Off_A                Def_A             Colley_A 
                       0                    0                    0 
                    WP_A                MOV_A                 AP_A 
                       0                    0                    0 
                   ARG_A                BIH_A                BOB_A 
                    1310                  128                  268 
                   BRZ_A                COL_A                DOL_A 
                    1968                    0                    0 
                   DUN_A                DWH_A                ECK_A 
                     524                 1456                 1456 
                   ENT_A                ERD_A                GRN_A 
                    1712                 1968                 1072 
                   GRS_A                HER_A                HOL_A 
                    1706                 1712                 1968 
                   IMS_A                MAS_A                MKV_A 
                    1840                  512                 1712 
                   MOR_A                POM_A                RTH_A 
                       0                    0                    0 
                   SAG_A                SAU_A                 SE_A 
                       0                 1712                  658 
                   SEL_A                STR_A                TSR_A 
                     256                 1712                 1456 
                   USA_A                WLK_A                WOB_A 
                       0                    0                  128 
                   WOL_A                WTE_A        FGM_POSMade_A 
                       0                 1968                    0 
        FGM_POSAllowed_A        FGA_POSMade_A     FGA_POSAllowed_A 
                       0                    0                    0 
          FGM3_POSMade_A    FGM3_POSAllowed_A       FGA3_POSMade_A 
                       0                    0                    0 
       FGA3_POSAllowed_A        FTM_POSMade_A     FTM_POSAllowed_A 
                       0                    0                    0 
           FTA_POSMade_A     FTA_POSAllowed_A         OR_POSMade_A 
                       0                    0                    0 
         OR_POSAllowed_A         DR_POSMade_A      DR_POSAllowed_A 
                       0                    0                    0 
           AST_POSMade_A     AST_POSAllowed_A         TO_POSMade_A 
                       0                    0                    0 
         TO_POSAllowed_A        STL_POSMade_A     STL_POSAllowed_A 
                       0                    0                    0 
           BLK_POSMade_A     BLK_POSAllowed_A         PF_POSMade_A 
                       0                    0                    0 
         PF_POSAllowed_A        PTS_POSMade_A     PTS_POSAllowed_A 
                       0                    0                    0 
             TEMPOMade_A       TEMPOAllowed_A     FGM_POSMadeRAW_A 
                       0                    0                    0 
        FGA_POSMadeRAW_A    FGM3_POSMadeRAW_A    FGA3_POSMadeRAW_A 
                       0                    0                    0 
        FTM_POSMadeRAW_A     FTA_POSMadeRAW_A      OR_POSMadeRAW_A 
                       0                    0                    0 
         DR_POSMadeRAW_A     AST_POSMadeRAW_A      TO_POSMadeRAW_A 
                       0                    0                    0 
        STL_POSMadeRAW_A     BLK_POSMadeRAW_A      PF_POSMadeRAW_A 
                       0                    0                    0 
        PTS_POSMadeRAW_A       TEMPOMadeRAW_A  FGM_POSAllowedRAW_A 
                       0                    0                    0 
     FGA_POSAllowedRAW_A FGM3_POSAllowedRAW_A FGA3_POSAllowedRAW_A 
                       0                    0                    0 
     FTM_POSAllowedRAW_A  FTA_POSAllowedRAW_A   OR_POSAllowedRAW_A 
                       0                    0                    0 
      DR_POSAllowedRAW_A  AST_POSAllowedRAW_A   TO_POSAllowedRAW_A 
                       0                    0                    0 
     STL_POSAllowedRAW_A  BLK_POSAllowedRAW_A   PF_POSAllowedRAW_A 
                       0                    0                    0 
     PTS_POSAllowedRAW_A    TEMPOAllowedRAW_A                 BD_A 
                       0                    0                 1840 
                   CNG_A                DES_A                JON_A 
                     128                  384                 1968 
                   LYN_A                MGY_A                NOR_A 
                     932                 1968                 1968 
                   REI_A                 RM_A                SIM_A 
                    1968                 1968                 1328 
                   ACU_A                BCM_A                CMV_A 
                    1444                 1968                 1968 
                    DC_A                KLK_A                REN_A 
                     390                 1456                 1712 
                   RIS_A                ROH_A                SAP_A 
                    1968                 1712                 1712 
                   SCR_A                WIL_A                DOK_A 
                    1584                  384                  384 
                   JCI_A                KPK_A                 MB_A 
                    1840                  768                 1054 
                    PH_A                PIG_A                PKL_A 
                    1968                  384                 1840 
                   TRX_A                CPR_A                ISR_A 
                    1712                  780                 1584 
                   KRA_A                LYD_A                RTR_A 
                     640                 1840                  780 
                   UCS_A                BKM_A                CPA_A 
                    1968                 1968                  908 
                   JEN_A                PGH_A                REW_A 
                    1444                  640                  640 
                   RSE_A                SPW_A                STH_A 
                    1968                  640                  640 
                   BPI_A                DC2_A                DCI_A 
                    1438                 1438                  768 
                   HKB_A                LMC_A                NOL_A 
                    1304                  768                  902 
                   OMY_A                RTB_A                KEL_A 
                    1566                 1304                 1968 
                   KMV_A                 RT_A                 TW_A 
                    1834                 1030                 1432 
                   AUS_A                KOS_A                PEQ_A 
                    1962                 1694                 1962 
                   PTS_A                ROG_A                RTP_A 
                    1694                 1828                 1024 
                   TMR_A               X7OT_A                ADE_A 
                    1426                 1158                 1426 
                   BBT_A                BNM_A                BUR_A 
                    1158                 1962                 1292 
                   CJB_A                CRO_A                EBP_A 
                    1694                 1292                 1158 
                   HAT_A                MSX_A                SFX_A 
                    1962                 1426                 1158 
                   TBD_A                BLS_A                D1A_A 
                    1962                 1426                 1560 
                   DII_A                KBM_A                TPR_A 
                    1292                 1694                 1292 
                   MvG_A                PPR_A                 SP_A 
                    1828                 1962                 1426 
                   SPR_A                STF_A                STS_A 
                    1426                 1828                 1828 
                   TRP_A                UPS_A                WMR_A 
                    1426                 1962                 1962 
                   BWE_A                LOG_A                TRK_A 
                    1560                 1694                 1560 
                   DAV_A                FAS_A                FSH_A 
                    1694                 1694                 1694 
                   HAS_A                HRN_A                KPI_A 
                    1694                 1962                 1694 
                   MCL_A                CRW_A                DDB_A 
                    1828                 1962                 1828 
                   ESR_A                FMG_A                HKS_A 
                    1828                 1962                 1962 
                   INP_A                JNG_A                JRT_A 
                    1962                 1828                 1962 
                   MUZ_A                OCT_A                PMC_A 
                    1962                 1962                 1962 
                   PRR_A                RSL_A                SGR_A 
                    1828                 1962                 1828 
                   SMN_A                SMS_A                YAG_A 
                    1828                 1828                 1828 
                   ZAM_A                BNT_A                COX_A 
                    1828                 1962                 1962 
                   JJK_A                LAB_A                MMG_A 
                    1962                 1962                 1962 
                   STM_A                WMV_A                AWS_A 
                    1962                 1962                 2096 
                   INC_A                LAW_A                LEF_A 
                    2096                 2096                 2096 
                   MGS_A                NET_A                PIR_A 
                    2096                 2096                 2096 
                   BNZ_A                CBR_A                CWL_A 
                    2096                 2096                 2096 
                   EMK_A                FDM_A                KCX_A 
                    2096                 2096                 2096 
                   RWP_A                 HWin             SeedDiff 
                    2096                    0                    0 
    

After filting columns with large numbers of missing values, We'll quickly implement a preprocessing recipe, imputing missing values using knn and then running a boruta feature selection algorithm.


```r
res = c('HDiff','HWin','SeedDiff')

metrics = c('Elo','LRMC','SRS','RPI','TS','Off','Def','Colley','WP','MOV',
            'AP','BOB','BIH','COL','DOL','DUN','MOR','POM','RTH','SAG','SE','SEL',
            'USA','WLK','WOB','WOL','FGM_POSMade','FGM_POSAllowed','FGA_POSMade',
            'FGA_POSAllowed','FGM3_POSMade','FGM3_POSAllowed','FGA3_POSMade',
            'FGA3_POSAllowed','FTM_POSMade','FTM_POSAllowed','FTA_POSMade',
            'FTA_POSAllowed','OR_POSMade','OR_POSAllowed','DR_POSMade',
            'DR_POSAllowed','AST_POSMade','AST_POSAllowed','TO_POSMade','TO_POSAllowed',
            'STL_POSMade','STL_POSAllowed','BLK_POSMade','BLK_POSAllowed',
            'PF_POSMade','PF_POSAllowed','PTS_POSMade','PTS_POSAllowed',
            'TEMPOMade','TEMPOAllowed','FGM_POSMadeRAW','FGA_POSMadeRAW',
            'FGM3_POSMadeRAW','FGA3_POSMadeRAW','FTM_POSMadeRAW','FTA_POSMadeRAW',
            'OR_POSMadeRAW','DR_POSMadeRAW','AST_POSMadeRAW','TO_POSMadeRAW',
            'STL_POSMadeRAW','BLK_POSMadeRAW','PF_POSMadeRAW','PTS_POSMadeRAW',
            'TEMPOMadeRAW','FGM_POSAllowedRAW','FGA_POSAllowedRAW',
            'FGM3_POSAllowedRAW','DR_POSAllowedRAW','AST_POSAllowedRAW',
            'TO_POSAllowedRAW','STL_POSAllowedRAW','BLK_POSAllowedRAW',
            'PF_POSAllowedRAW','PTS_POSAllowedRAW','TEMPOAllowedRAW',
            'CNG','DC')

cols = c(res, paste0(metrics,'_A'), paste0(metrics,'_H'))

boruta_df = pre_2015[colnames(pre_2015) %in% cols]

outright_boruta = recipe(HWin ~ ., data = boruta_df %>% select(-c(HDiff))) %>%
  step_knnimpute(all_predictors(), neighbors = 3) %>%
  step_select_boruta(all_predictors(), outcome = "HWin")

prepped = prep(outright_boruta)

prepped[['steps']][[2]][['res']][1]$finalDecision[which(prepped[['steps']][[2]][['res']][1]$finalDecision == 'Confirmed')]
```

                  Elo_H              LRMC_H               SRS_H               RPI_H 
              Confirmed           Confirmed           Confirmed           Confirmed 
                   TS_H            Colley_H                AP_H               BIH_H 
              Confirmed           Confirmed           Confirmed           Confirmed 
                  BOB_H               COL_H               DOL_H               DUN_H 
              Confirmed           Confirmed           Confirmed           Confirmed 
                  MOR_H               POM_H               RTH_H               SAG_H 
              Confirmed           Confirmed           Confirmed           Confirmed 
                   SE_H               SEL_H               USA_H               WLK_H 
              Confirmed           Confirmed           Confirmed           Confirmed 
                  WOB_H               WOL_H       FGM_POSMade_H       BLK_POSMade_H 
              Confirmed           Confirmed           Confirmed           Confirmed 
          PTS_POSMade_H    PTS_POSAllowed_H    BLK_POSMadeRAW_H    PTS_POSMadeRAW_H 
              Confirmed           Confirmed           Confirmed           Confirmed 
                  CNG_H                DC_H               Elo_A              LRMC_A 
              Confirmed           Confirmed           Confirmed           Confirmed 
                  SRS_A               RPI_A                TS_A            Colley_A 
              Confirmed           Confirmed           Confirmed           Confirmed 
                   AP_A               BIH_A               BOB_A               COL_A 
              Confirmed           Confirmed           Confirmed           Confirmed 
                  DOL_A               DUN_A               MOR_A               POM_A 
              Confirmed           Confirmed           Confirmed           Confirmed 
                  RTH_A               SAG_A                SE_A               SEL_A 
              Confirmed           Confirmed           Confirmed           Confirmed 
                  USA_A               WLK_A               WOB_A               WOL_A 
              Confirmed           Confirmed           Confirmed           Confirmed 
          FGM_POSMade_A       BLK_POSMade_A       PTS_POSMade_A    PTS_POSAllowed_A 
              Confirmed           Confirmed           Confirmed           Confirmed 
       BLK_POSMadeRAW_A    PTS_POSMadeRAW_A PTS_POSAllowedRAW_A               CNG_A 
              Confirmed           Confirmed           Confirmed           Confirmed 
                   DC_A            SeedDiff 
              Confirmed           Confirmed 
    Levels: Tentative Confirmed Rejected
    

With our features selected, we can filter our unused columns and grab a cross-validated dataset to train on.


```r
res = c('HDiff','HWin','SeedDiff')

metrics = c('Elo','LRMC','SRS','RPI','TS','Colley',
            'AP','BIH','COL','DOL','MOR','POM','SAG','SEL',
            'USA','WLK','WOB','PTS_POSMade','PTS_POSAllowed','PTS_POSMadeRAW')

cols = c(res, paste0(metrics,'_A'), paste0(metrics,'_H'))

data = pre_2015[colnames(pre_2015) %in% cols]

clf_impute = recipe(HWin ~ ., data = boruta_df %>% select(-c(HDiff))) %>%
  step_knnimpute(all_predictors(), neighbors = 3)

prepped = prep(clf_impute)

data = juice(prepped)
data = data[colnames(data) %in% cols]
data$HDiff = pre_2015$HDiff

set.seed(1986)

df_train  = data
df_test  = post_2015

cv_folds = vfold_cv(df_train %>% mutate(HWin = as.factor(HWin)) %>% select(-c(HDiff)), strata = HWin)

```

We'll tune our xgboost algorithm here. In the interest of not repeating myself, I've copied the code over and it should be able to be run, but it will take a long time to run, and thus I've commented it out.


```r
xgb = boost_tree(
  tree_depth = tune(),
  min_n = tune(),
  trees = tune(),
  loss_reduction = tune(),
  sample_size = tune(),
  mtry = tune(),
  learn_rate = tune()
) %>%
  set_engine("xgboost") %>%
  set_mode("classification")

xgb_params = parameters(
  tree_depth(),
  min_n(),
  trees(range = c(10, 2000)),
  loss_reduction(),
  sample_size = sample_prop(),
  finalize(mtry(), df_train),
  learn_rate()
)

xgb_wrkflw <- workflow() %>%
  add_formula(HWin ~ .) %>%
  add_model(xgb)

doParallel::registerDoParallel()

xgb_tune <- tune_bayes(
  xgb_wrkflw ,
  resamples = cv_folds,
  param_info = xgb_params,
  iter = 1000,
  metrics = metric_set(
    mn_log_loss
  ),
  initial = 15,
  control = control_bayes(
    no_improve = 100,
    uncertain = 10,
    save_pred = F,
    save_workflow = F,
    time_limit = 600,
    verbose = T
  )
)
 
xgb_best <- select_best(xgb_tune, "mn_log_loss")
print(xgb_best)
```

    NULL
    

We have our tuned xgboost model. We'll train it on our train dataset, test it on our test dataset, and then apply predictions on the 2019 tournament.


```r
xgb = boost_tree(mtry=2,
                  trees=401,
                  min_n=9,
                  tree_depth=2,
                  learn_rate=0.012,
                  loss_reduction=0.42,
                  sample_size = 0.443) %>% 
  set_engine("xgboost") %>% 
  set_mode("classification") %>% 
  fit(HWin ~ ., data = df_train %>% mutate(HWin = as.factor(HWin)) %>% select(-c(HDiff)))

saveRDS(xgb,'xgb_outright.RDS')
```

    [20:10:51] WARNING: amalgamation/../src/learner.cc:1061: Starting in XGBoost 1.3.0, the default evaluation metric used with the objective 'binary:logistic' was changed from 'error' to 'logloss'. Explicitly set eval_metric if you'd like to restore the old behavior.
    


```r
xgb = readRDS('xgb_outright.RDS')
xprb = predict(xgb, df_test, type = "prob")
print(LogLoss(xprb$.pred_TRUE,df_test$HWin))
```

    [1] 0.5008113
    

We get a pretty solid log loss for the 2019 tournament -- this would have us at about middle of the pack. There are a few tricks we can do to boost our performance, which I have not implemented here but are arbitrary to do so:

* Cap predictions at 0.95/0.05 -- in the event of (inevitable) upsets where a team with a high probability of winning loses, we are better insulated from those upsets than teams that do not cap their predictions, but the gain from correctly predicting a .99 win versus a .95 win is not that substantial.
* Generate two sets of predictions and randomly flip close games around while artificially boosting confidence -- Kaggle allows you to submit two sets of predictions for stage 2, not just one. We can exploit this by taking games we consider to be coinflips (games where win probabilites fall between 45%-55%) and randomly pick a winner, and assign an artifically high confidence (such as 70%/30% or, if you want to be really, extreme, 100%/0%). If we want to do well enough to win this competition, we can boost our score with this random confidence. If we fail however, yikes! (Spoiler alert, it worked)
* Adjust predictions for injuries -- One thing many Kaggle teams will not do but something that is extremely valuable to do is to monitor the injury status of key players on each team. Missing a star player can result in as much as a 3 point spread, something that affects a team's win probability significantly. Using vegas spreads, I converted my win probability estimates to point spreads, used approximate value to estimate the value of missing players on the spread, adjusted those spreads, and then converted my spreads back to win probability estimates.
