# Welcome to the Swish Insights team project website

### By: Brandon Smeltz, Michael Parker, & Elvin Lee

This is a website to showcase our final project for FIN 377, Advanced Investment: Data Science for Finance, course at Lehigh University.

To see the complete analysis files click [here](https://github.com/Brandon4106/Fin_377_Swish_Insights/tree/main/notebooks).

## Table of contents
1. [Introduction](#intro)
2. [Methodology](#meth)
3. [Analysis](#anal)
    1. [Preprocessing](#prep)
    2. [Custom Scoring](#cscore)
    3. [CV Fold](#fold)
    4. [Machine Learning](#ML)
    5. [Outputting Model Prediction](#output)
4. [Takeaways & Next Steps](#takeaways)

## Introduction  <a name="intro"></a>

The main goal of this project is to train a model that can aid sports bettors with bets for the NBA season. In particular, the model will do an in depth analysis of one team and try to make a profit on the spread for the games in March, the holdout set. The model will accomplish by using the statistics it's learned throughout the season, in the training set. This analysis requires a significant amount of scraping for the one team we have selected the Celtics as well as additional scraping to build predictions for the Celtics opponents.

## Methodology <a name="meth"></a>

Here is some code that we used to develop our analysis:

#### Scraping stats from box scores
 

```python
def save_box_scores(formatted_date_list, simple_games_df, url_base):
    # Create a main directory to hold all data before zipping
    main_folder = 'NBA_Box_Scores'
    os.makedirs(main_folder, exist_ok=True)
    
    for date in formatted_date_list:
        games_df = simple_games_df[simple_games_df['Date'] == date]
        for index, row in games_df.iterrows():
            home_abbr = row['Home']
            away_abbr = row['Away']
            game_folder = f"{date}/{away_abbr}@{home_abbr}"  # Folder name format: YYYYMMDD/Away@Home
            full_folder_path = os.path.join(main_folder, game_folder)
            os.makedirs(full_folder_path, exist_ok=True)

            # Format the URL
            formatted_url = f"{url_base}{date}0{home_abbr}.html"
            # Fetch and save box scores
            try: 
                response = fetch_with_retry_after(formatted_url)
                
            # Save each team's box score in the specific game folder
                away_df,home_df = get_boxscore(response.text)
                away_df.to_csv(f"{full_folder_path}/away_team.csv", index=False)
                home_df.to_csv(f"{full_folder_path}/home_team.csv", index=False)
            except Exception as e:
                print(f"Error fetching data for URL {formatted_url}: {str(e)}")
            
    # Zip the entire directory
    with ZipFile(f"{main_folder}.zip", 'w') as zipf:
        for root, dirs, files in os.walk(main_folder):
            for file in files:
                zipf.write(os.path.join(root, file), os.path.relpath(os.path.join(root, file), os.path.join(main_folder, '..')))

# Example usage
url_base = "https://www.basketball-reference.com/boxscores/"

save_box_scores(formatted_date_list, simple_games_df, url_base)
```

Here is a sample output of part of the box score data received for one of the Lakers games this season:

![](pics/Sample_Box_Score_Output.png)
<br><br>
    
**Note: The full script to obtain these box scores is in runboxscores.ipynb which can be found** [here](https://github.com/Brandon4106/Fin_377_Swish_Insights/tree/main/notebooks)

    
## Analysis <a name="anal"></a>

#### After obtaining the box scores we began to analyze the data by breaking it down into the following pieces:
    1. Preprocessing
    2. Custom Scoring 
    3. CV Fold
    4. Machine Learning
    5. Outputting Model Prediction

### Preprocessing <a name="prep"></a>
Preprocessing was done here in both a numerical pipeline and categorical pipeline. For numerical data, all missing values were imputed using the median and passed through a scaler:
```python
numer_pipe = Pipeline(
    [
        ("imputer",SimpleImputer(strategy='median')),
        ("scaler",StandardScaler()),
        ("feature_creation",'passthrough')
    ])
```
Two categorical values, `In_Season_Tournament` and `Opp_Win_Last_Game`, were encoded:
```python
cat_pipe = Pipeline(
    [
        ("encoder",OneHotEncoder())
    ])
```
### Custom Scoring <a name="cscore"></a>
As we are on a sports betting project, we are obviously trying to maximize **profit**. To do this, however, it is necessary to create a custom scoring metric. The code for this takes into account all of the payouts you get from bets that you guess correctly, and subtracts every single bet that you made.
### CV Fold <a name="fold"></a>
As this data is time dependent, the folds cannot be KFold or at random. The folds were created using `TimeSeriesSplit`.
### Machine Learning <a name="ML"></a>
Now that all the busy work is out of the way, we can start making some models! Well, first we have to do the actual cross validation and gridsearch. Well at least that part is easy... Or so we thought. When you create a complex custom scoring metric like this, it is necessary to create *your own* cross validation and *your own* grid_search. Because these steps are crucial and were some of the hardest to figure out, the code is displayed here:
```python
def perform_cross_validation(model, X, y, cv, line, celtics_payout, opp_payout, bet_size):
    scores=[]
    for train_index, test_index in cv.split(X):
        X_training, X_testing = X.iloc[train_index], X.iloc[test_index]
        y_training, y_testing = y.iloc[train_index], y.iloc[test_index]
        line_test = line.iloc[test_index]
        celtics_payout_test =  celtics_payout.iloc[test_index]
        opp_payout_test =  opp_payout.iloc[test_index]

        model_clone = clone(model)
        ypred = model_clone.fit(X_training, y_training).predict(X_testing)
        score = custom_profit_score(y=y_testing.values, 
                                    y_pred=ypred,
                                    celtics_line=line_test.values,
                                    celtics_payout=celtics_payout_test.values,
                                    opp_payout=opp_payout_test.values,
                                    bet=bet_size)
        scores.append(score)
    return {'scores': scores}

def grid_search_custom_cv(model, param_grid, X, y, line, celtics_payout, opp_payout, cv, bet_size):
    results = []
   
    for params in ParameterGrid(param_grid):
        model_clone = clone(model)
        model_clone.set_params(**params)
       
        cv_results = perform_cross_validation(model_clone, X, y, cv, line, celtics_payout, opp_payout, bet_size)
        results.append({
            **params,
            'scores': cv_results['scores'],
            'mean_score': np.mean(cv_results['scores']),
            'std_score': np.std(cv_results['scores'])
        })
   
    return pd.DataFrame(results)
```
### Outputting Model Prediction <a name="output"></a>
Now, we can finally predict some models. 
#### Model 1
For Model 1, we used a Lasso regressor, fine tuning the alpha. Here were the results:
<br><br>
<img src="pics/model_one.png" alt="model_one" width="800"/>
<br><br>
Profit!!! This model yielded a profit of 309.09. It should be noted that for all of these models, the sample size is truly too small to draw finite conclusions. Nonetheless, it is gratifying to see some positive results

#### Model 2
Now onto some not so positive results. Model 2 also used a Lasso regressor with fine tuning the alpha, except this time polynomial features were created to give the model more variables:
<br><br>
<img src="pics/model_two.png" alt="model_two" width="800"/>
<br><br>
Despite having more variables, this model yielded a net loss of over 800 dollars. Overfitting? Potentially.

#### Model 3
Changing it up, model 3 now includes a feature selection in `SelectKBest`, and switches to a Ridge regressor. It now optimizes the number of features to select and the alpha in ridge. Here are the results:
<br><br>
<img src="pics/model_three.png" alt="model_three" width="800"/>
<br><br>
Profit once again! This model yielded a profit of 500 dollars.

#### Model 4
This model uses a `SequentialFeatureSelector` instead of a `SelectKBest`. Now, it uses a LinearRegression, but still fine tunes the number of features to select. Here are the results:
<br><br>
<img src="pics/model_four.png" alt="model_four" width="800"/>
<br><br>
This model had a net loss of 72 dollars.


## Takeaways & Next Steps <a name="takeaways"></a>

1. We were able to learn a lot during the completion of this project...

2. Our models yield profitable results over the 16 game span that the we used in the holdout set for the Celtics this season. When the model was told to predict the spread of these 16 games and bet $100 on the predicted winning line for each game, it correctly predicted 11/16 of the games for a profit of $500. This is a 69% win       rate and correlates to an ROI of 31.25% which is very high.
    
3. From this analysis, we are curious to expand this project to look at the other 29 teams as well to identify more potentially profitable spreads as well as to collect more data and further verify the validity of our model.

4. Another interesting avenue would be to also look at the total score prop and use a similar process to identify whether or not the over/ under should be selected for a particular game.



## About the team

<img src="pics/Brandon_Photo.jpg" alt="julio" width="300"/>
<br>
Brandon is a junior at Lehigh University in the IBE Honors Program as well as the MFE Program. Brandon is currently studying mechanical engineering and finance with an interest in pursuing a masters in financial engineering. Brandon will be working as a financial analyst at Air Products this summer and is excited to apply his knowledge learned at Lehigh into the real world. In his free time, Brandon likes to watch/ play sports with friends, relax at the pool, and cook.
<br><br><br>

<img src="pics/Elvin_Photo.png" alt="don" width="300"/>
<br>
Elvin is a junior at Lehigh University pursuing a degree in Finance. Elvin is from Los Angeles and enjoys playing poker in his free time.
<br><br><br>

<img src="pics/Michael_Photo.png" alt="julio" width="300"/>
<br>
Michael is a junior at Lehigh University in the IBE Honors Program pursuing degrees in Finance and Mechanical Engineering, with plans to complete a masters in financial engineering. This summer, Michael will be completing Financial Engineering research. In his spare time, Michael likes to play piano and guitar, watch soccer, play tennis, and spend time with friends.

## More 

To view the GitHub repo for this website, click [here](https://github.com/Brandon4106/Fin_377_Swish_Insights).
