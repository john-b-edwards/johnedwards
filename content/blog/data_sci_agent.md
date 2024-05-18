---
date: "2024-05-18"
description: Google's new LLM-powered data science agent doesn't do anything except arm Kaggle spammers with another bullshit generator.
draft: false
keywords:
- machine_learning
slug: google_data_sci_slop
pagination: true
twitterImg: https://upload.wikimedia.org/wikipedia/commons/thumb/e/e0/Google_I-O_2019.jpg/800px-Google_I-O_2019.jpg
tags:
- python
title: Google's Data Science Agent is just a Kaggle slop generator
toc: true
---

![Google I/O 2019](https://upload.wikimedia.org/wikipedia/commons/thumb/e/e0/Google_I-O_2019.jpg/800px-Google_I-O_2019.jpg)

*Google I/O 2019; Alexander Shcherbakov, [CC BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0) via Wikimedia Commons*

Google I/O was this past week, and predictably, the focus was LLM-powered tools. Among the products Google rolled out was something that caught my eye--an LLM powered "Data Science Agent" that would take in data and with prompting, break down a dataset, develop a plan of attack for approaching a data science problem, and even generate Colab notebooks.

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Today we are launching Data Science Agent, a Google Labs experiment.<br><br>◆ Helps with complex tasks like planning, and error correction.<br>◆ Helps with data science tasks like predictive modeling.<br>◆ Outputs an AI-generated Colab notebook based on your prompt.<br><br>Try it at… <a href="https://t.co/ApDB5lNMcq">pic.twitter.com/ApDB5lNMcq</a></p>&mdash; Kath Korevec (@simpsoka) <a href="https://twitter.com/simpsoka/status/1790495455329321423?ref_src=twsrc%5Etfw">May 14, 2024</a></blockquote>

<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Gotta try this out, right? Even though this is merely an "experiment", surely nothing about the fundamental nature of LLMs undermines the utility of this product, right? *Right?*

# Methodology

To take this tool to task, we will provide it with a dataset and run through a simple prompt. I'll borrow [this Formula 1 lap-time dataset from one of Nick Wan's community Kaggle competitions](https://www.kaggle.com/competitions/formula-1-lap-time-prediction-nwvs-s00e02), and ask the LLM to try to visualize the dataset. Pretty straightforward, right?

# Results

I first fed the tool the `train.csv` file from the Kaggle competition, and gave it the following prompt:

> I would like you to visualize which factors contribute the most to the lap-time column in this dataset.

The tool came up with this as its plan:

> Plan
>
> 1.  **data_loading**: Load the data from "/tmp/train.csv" into a pandas DataFrame.
> 2.  **data_exploration**: Check for missing values, outliers, and the distribution of the 'lap_time' column.
> 3.  **feature_engineering**: If necessary, create new features or transform existing ones to improve model performance.
> 4.  **model_training**: Train a regression model (e.g., Linear Regression, Random Forest) to predict 'lap_time' based on other features.
> 5.  **model_evaluation**: Evaluate the model's performance using appropriate metrics (e.g., R-squared, RMSE).
> 6.  **data_visualization**: Create visualizations (e.g., feature importance plot, partial dependence plots) to show the contribution of each feature to the 'lap_time' prediction.
> 7.  **finish_task**: Summarize the insights gained from the analysis, highlighting the most important factors influencing lap time.

Woah woah woah! I'm just looking to visualize the dataset. I didn't think that I needed to make a model first in order to visualize which features correlated most strongly with lap time. Still, the Agent thinks I should make a model, so I'll listen to it I guess... let's click the "Generate Colab" notebook button and see what we get.

------------------------------------------------------------------------

The Agent is able to read in the data set just fine, but things get a little weird once we get to step #2.

> To understand the data and identify potential issues, we need to check for missing values and examine the distribution of the 'lap_time' column.

``` py
# Check for missing values
print(df.isnull().sum())

# Examine the distribution of 'lap_time'
print(df['lapTime_ms'].describe())
df['lapTime_ms'].hist()
df.boxplot(column='lapTime_ms')
```

![A confusing box plot. The y-axis numbers 0 to 2.0 and the x-axis has no labels. There is only a small blue bar in the bottom left corner.](https://github.com/john-b-edwards/johnedwards/raw/main/public/images/data_sci_agent/data_sci_agent_1.png)

```         
STDOUT
id                         0
race                       0
date                       0
time                    7035
circuit                    0
latitude                   0
longitude                  0
altitude                   0
driver                     0
carNumber                  0
constructor                0
avgDriverFinish            0
avgConstructorFinish       0
lapNumber                  0
lapPosition                0
pitStop                    0
pitCount                   0
pitTime_ms                 0
lapTime_ms                 0
dtype: int64
count    3.000000e+04
mean     9.445979e+04
std      6.375514e+04
min      6.620000e+04
25%      8.127850e+04
50%      9.049250e+04
75%      1.004710e+05
max      2.253574e+06
Name: lapTime_ms, dtype: float64
<AxesSubplot: >
```

This kind of data visualization and exploration is essentially useless, and provides a reader with no actual understanding of the data and its features. The histogram it generated is essentially useless.

Despite this, the agent then proceeds into the next steps. Using the 1.5xIQR method, it removes outlier target variables (which I think is defensible given how extreme the outliers are, even if we can predict the outliers with the data available). Then, inexplicably, it decides to create a new variable:

> The data is now cleaned and ready for feature engineering. I will create a new feature by combining 'avgDriverFinish' and 'avgConstructorFinish' to capture the overall performance of the driver and constructor.

``` py
# Create a new feature 'avgOverallFinish'
df_cleaned['avgOverallFinish'] = (df_cleaned['avgDriverFinish'] + df_cleaned['avgConstructorFinish']) / 2
```

This kind of feature engineering is quite perplexing--there is little reason to believe that a model wouldn't be able to isolate the impact of these features together. What's more, this feature is now correlated with both `avgDriverFinish` and `avgConstructorFinish`, so we have introduced correlated features to our model for reasons completely unbeknownst to me.

With this strange, arbitrary feature engineering behind us, the model then decides that it is time to make a random forest model. Again, all I asked was that the agent visualize what contributed to the model!

The agent produces the following code:

``` py
# Import necessary libraries
from sklearn.ensemble import RandomForestRegressor

# Prepare features and target
X = df_cleaned.drop('lapTime_ms', axis=1)
y = df_cleaned['lapTime_ms']

# Instantiate the model
model = RandomForestRegressor(n_estimators=100, random_state=42)

# Train the model
model.fit(X, y)
```

> The error message "ValueError: could not convert string to float: 'British Grand Prix'" indicates that the Random Forest model is trying to process string data, which it cannot handle. This suggests that some of the features in the `X` DataFrame are categorical and need to be converted to numerical values before training the model.

The agent decided to kitchen sink the model with some completely arbitrary parameters--but its failure to check if any of the features were categorical in the data exploration phase resulted in this error. At least it recognized the error, so it can fix it, right? *RIGHT?*

> To train a Random Forest model, we need to handle categorical features in the DataFrame `df_cleaned` by using one-hot encoding.

``` py
# Handle categorical features using one-hot encoding
df_encoded = pd.get_dummies(df_cleaned, columns=['race', 'circuit', 'driver', 'constructor'])

# Separate features and target variable
X = df_encoded.drop('lap_time', axis=1)
y = df_encoded['lap_time']

# Train a Random Forest regression model
model = RandomForestRegressor()
model.fit(X, y)
```

> The error message indicates that the 'lap_time' column is not found in the DataFrame `df_encoded`. This suggests that the column might have been renamed or dropped during the one-hot encoding process. To fix this, we need to check the column names in `df_encoded` and use the correct column name for 'lap_time' when separating features and target variable.

The agent dummy encoded the categorical variables (okay, fine)--but then treated it as though the dataframe with just the dummy features (`df_encoded`) had all of the features. There's no `lap_time` variable to grab from that table. So having recognized the mistake, surely it's going to fix it. *RIGHT??????*

``` py
# Handle categorical features using one-hot encoding
df_encoded = pd.get_dummies(df_cleaned, columns=['race', 'circuit', 'driver', 'constructor'])

# Check the column names in df_encoded
print(df_encoded.columns)

# Separate features and target variable (assuming 'lap_time' is still the correct column name)
X = df_encoded.drop('lap_time', axis=1)
y = df_encoded['lap_time']

# Train a Random Forest regression model
model = RandomForestRegressor()
model.fit(X, y)
```

> The error persists, and the output shows that the 'lap_time' column is not present in `df_encoded`. This means the column was likely dropped or renamed during a previous step. We need to investigate the previous steps to determine what happened to the 'lap_time' column.

Okay, it made the mistake again. But it's just a humble LLM. Surely we can expect it to make some mistakes every now and again? But surely it *recogizes* the mistakes, so surely it won't make the same mistake again, RIGHTDLFLKAJSDL;FKJA;LSKDFJA;LSDKFJLA

``` py
# Perform one-hot encoding on categorical features
df_encoded = pd.get_dummies(df, columns=['race', 'circuit', 'driver', 'constructor'])

# Verify that 'lap_time' column is present
if 'lap_time' in df_encoded.columns:
    print("'lap_time' column is present in the encoded DataFrame.")
else:
    print("'lap_time' column is missing from the encoded DataFrame. Please check previous steps.")
```

> The 'lap_time' column is missing after one-hot encoding, likely due to being dropped during outlier handling in a previous step. We need to revisit the outlier handling process and ensure 'lap_time' is preserved.

The LLM suggests that the column was dropped during outlier handling--which is not what happened! And then, despite saying, "This is what happened, let's revisit this step", the LLM proceeds to completely ignore what it just said, and proceeds to immediately try retraining!

> I will write a code block to train a Random Forest regression model on the `df_encoded` DataFrame.

``` py
# Verify 'lap_time' column
if 'lap_time' not in df_encoded.columns:
    print("Error: 'lap_time' column is missing from df_encoded. Please revisit previous steps to restore it.")
else:
    # Separate features and target
    X = df_encoded.drop('lap_time', axis=1)
    y = df_encoded['lap_time']

    # Instantiate Random Forest model
    from sklearn.ensemble import RandomForestRegressor
    model = RandomForestRegressor()

    # Fit the model
    model.fit(X, y)
```

```         
Error: 'lap_time' column is missing from df_encoded. Please revisit previous steps to restore it.
```

> The 'lap_time' column is missing from the `df_encoded` DataFrame, indicating an error in a previous step. I need to go back and identify where the column was dropped or modified.

> I will check the previous feature engineering steps to see where the 'lap_time' column was dropped or renamed.

At this point, the AI agent simply stopped issuing responses.

# Takeaways

So to recap, all I did was ask the following:

> I would like you to visualize which factors contribute the most to the lap-time column in this dataset.

What the agent actually did was:

1.  Provide a single barebones, unreadable box-plot of the target variable distribution
2.  Clean outliers
3.  Construct an unhelpful, completely arbitrary feature
4.  Attempt to build a random forest model, fail
5.  Attempt to encode categorical variables, fail
6.  Repeat steps 4 and 5 until shutting off without explanation.

Surely I'm just cherry-picking a bad example for the LLM? I've run through similar prompts of varying complexity with this tool a good bit over the past week with this dataset, and the same issues come up over and over again:

-   The agent fails to understand the structure and nature of the dataset. Any kind of EDA is extremely minimal, and feature engineering is arbitrary.
-   The agent makes a mistake in model building.
-   The agent then either fails to identify what the mistake is, or identifies the mistake and then simply does not address it, repeating the same code that produced the mistake.

# The Rant

What's the issue? Even if this is an alpha or beta tool, the lack of utility with this product that Google has stamped their name on is astonishing. It's a complete and utter waste of time trying to get this produce any kind of insights, let alone a model--I was only once able to get it to produce a functioning model, and the model itself performed quite poorly thanks to the agent failing to understand both the dataset, proper data science practices, or how to even tune a model.

The key word here is *understand*. An LLM does not understand anything. An LLM does not think. It merely parrots what it has seen before. LLMs lack the ability to debug themselves, because they parrot one thing, and if they're wrong, they parrot another thing regardless of whether or not it actually solves the problem. They're all parrots. For these reasons, they are deeply unsuited for a wide variety of tasks, among them data science problems.

I can truly only anticipate one kind of use for this Data Science Agent. Kaggle has a significant problem with notebook spam, where users copy and spam low-quality, uninformative notebooks with the same boilerplate visualization code over and over again. [Two of Kaggle's top 10 notebook grandmasters](https://www.kaggle.com/rankings?group=notebooks) have reached this status by publishing literal thousands of notebooks without providing anything of actual insight. If you wanted to churn out more "Looks like it's insightful but actually reveals nothing" spammy notebooks, this kind of tool is perfect, because LLMs are perfectly suited to generate content to occupy internet space without providing anything of value. Slop.

That's all that these tools are good for--Google is embracing the steady waterfall of venture capital money associated with slapping "GenAI" to a product, regardless of its utility or quality, and are glad to ignore any associated downstream effects to their own products. [They're just here to kill the golden goose and get out.](https://www.theatlantic.com/technology/archive/2024/05/google-generative-ai-search-toilet-theory/678411/?utm_campaign=the-atlantic&utm_content=true-anthem&utm_medium=social&utm_source=twitter)
