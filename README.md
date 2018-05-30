A prediction of FIFA World Cup 2018
================

<img src="pics/cover2.jpg" alt="drawing" style="width: 1000px; height: 400px"/>

The UEFA Champion League final last weekend beteween Real Madrid and Liverpool was the only match I watched properly in over ten years. How dare I supposed to guess Brazil is going to lift the trophy in World Cup 2018? If you find the below dry to read, it is because of my limited natural language on the subject matter. Data science tricks to the rescue.

> This blogpost is largely based on the prediction framework by an eRum 2018 talk from Claus Thorn Ekstrøm. For first hand materials please direct to [slides](http://www.biostatistics.dk/talks/eRum2018/#1), [video](https://www.youtube.com/watch?v=urJ1obHPsV8) and [code](github.com/ekstroem/socceR2018).

The idea is that in each simulation run of a tournament, we find team winner, runners-up, third and fourth etc. N times of simulation runs e.g. 10k returns a list of winners with highest probability to be ranked top.

Apart from the winner question, this blogpost seeks to answer which team will be top scorer and how many goals will they score. After following the Claus's analysis rmarkdown file, I collected new data, put functions in a package and tried another modelling approach. Whilst the model is too simplistic to be correct, it captures the trend and is a fair starting point to add complex layers on top.

Initialization
==============

First we load packages that are required and set some global parameters such as normalgoals (The average number of goals scored in a world cup match) and nsim (number of simulation). They are first defined as YAML header parameters. Functionality that feed into this analysis is wrapped in an R package `worldcup`. Package is a convenient way to share code, seal utility functions and speed up iteration.

Next we load three datasets that have been tidied up before analysis. Plenty of time was spent on gathering data, aligning team names and cleaning up levels (surprise huh).

-   `team_data` contains features associated with team, more columns added to original.

-   `group_match_data` is game schedule that is known to public, no change to original.

-   `wcmatches_train` is a match dataset available on [this Kaggle competetion](https://www.kaggle.com/abecklas/fifa-world-cup/data). It is used as training set to estimate parameter lamda i.e. the average goals scored in a match for a single team. Records from 1994 up to 2014 are kept in the training set.

``` r
library(tidyverse)
library(magrittr)
devtools::load_all("worldcup")

normalgoals <- params$normalgoals 
nsim <- params$nsim

data(team_data) 
data(group_match_data) 
data(wcmatches_train)
```

Play game
=========

Claus proposed three working models to calculate single match outcome. First one is based on two independent poisson distributions of universal goal average, indicating that two teams are equal and so result is random regardless of their actual skills and talent. The second one assumes the scoring events in a match are two possion events, the difference of two poisson events believed to have skellam distribution. The result turns out to be much more reliable as the parameters are estimated from actual bettings. The third one is based on [World Football ELO Ratings](https://www.eloratings.net/about) rules. From current ELO ratings, we calculate expected result of one side in a match. It can be seen as the probability of success in a binomial distribution. It seems that this approach overlooked draw due to nature of binomial distribution i.e. binary.

Model candidate each has its own function, and it is specified by the **play\_fun** parameter and provided to higher level wrapper function `play_game`.

``` r
# Specify team Spain and Portugal
play_game(team_data = team_data, play_fun = "play_fun_simplest", 
          team1 = 7, team2 = 8, 
          musthavewinner=FALSE, normalgoals = normalgoals)
```

    ##      Agoals Bgoals
    ## [1,]      1      3

``` r
play_game(team_data = team_data, play_fun = "play_fun_skellam", 
          team1 = 7, team2 = 8, 
          musthavewinner=FALSE, normalgoals = normalgoals)
```

    ##      Agoals Bgoals
    ## [1,]      1      4

``` r
play_game(team_data = team_data, play_fun = "play_fun_elo", 
          team1 = 7, team2 = 8)
```

    ##      Agoals Bgoals
    ## [1,]      1      0

``` r
play_game(team_data, play_fun = "play_fun_double_poisson", 
          team1 = 7, team2 = 8)
```

    ##      Agoals Bgoals
    ## [1,]      0      0

``` r
#res = replicate(
#  100,
#  play_game(
#    play_fun = "play_fun_skellam",
#    team_data = team_data,team1 = 7, team2 = 8
#  ),
#  simplify = FALSE
#) %>% 
#do.call(rbind, .)
```

The fourth model presented here is my first attempt. To spell out, we assume two independent poisson events, with lambdas predicted from a trained poisson model. Then predicted goal are simulated by `rpois`.

Target variable in the training model is number of goals a team obtains in a match. Predictors include FIFA and ELO ratings at a point before the 2014 tournament started. Both are popular ranking systems, difference being FIFA rating is official and the latter is in the wild, adapted based on origianl chess ranking methodology.

From the model summary, ELO rating is statistically significant whereas FIFA rating is not. More interesting is that the estimate for the FIFA ratings variable is negative, inferring the effect is 0.9997704 relative to average. Overall, FIFA rating appears to be less predictive to the goals one may score than ELO rating. One possible reason is that ratings in 2014 alone are collected, and it may be worth future effort to go into history. Chellenge to FIFA ratings' predictive power is not [new story](https://www.sbnation.com/soccer/2017/11/16/16666012/world-cup-2018-draw-elo-rankings-fifa) after all.

The training set **wcmatches\_train** has a **home** column, representing whether team X in match Y is home team. However, it's hard to say in a third country a team/away position makes much difference comparing to league competetions. Also I didn't find an explicit home/away split for Russian World Cup. We can derive a home variable indicating host nation or continent in future model interation. Home advantage is not considered for now.

``` r
mod <- glm(goals ~ elo.x + fifa_start, family = poisson(link = log), data = wcmatches_train)
summary(mod)
```

    ## 
    ## Call:
    ## glm(formula = goals ~ elo.x + fifa_start, family = poisson(link = log), 
    ##     data = wcmatches_train)
    ## 
    ## Deviance Residuals: 
    ##     Min       1Q   Median       3Q      Max  
    ## -2.0472  -1.3133  -0.1354   0.6026   3.3879  
    ## 
    ## Coefficients:
    ##               Estimate Std. Error z value Pr(>|z|)    
    ## (Intercept) -3.5673415  0.7934373  -4.496 6.92e-06 ***
    ## elo.x        0.0021479  0.0005609   3.829 0.000129 ***
    ## fifa_start  -0.0002296  0.0003288  -0.698 0.485012    
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## (Dispersion parameter for poisson family taken to be 1)
    ## 
    ##     Null deviance: 679.15  on 536  degrees of freedom
    ## Residual deviance: 630.87  on 534  degrees of freedom
    ##   (207 observations deleted due to missingness)
    ## AIC: 1586.3
    ## 
    ## Number of Fisher Scoring iterations: 5

Group and kickout stages
========================

Find the winners at various stages, from group to round of 16, quarter-finals, semi-finals and final. They are internal functions, and run in every tournament simulation run.

``` r
# Set seed to get same result
set.seed(1984)
find_group_winners(team_data = team_data, group_match_data = group_match_data, 
                   play_fun = "play_fun_skellam")
```

    ## # A tibble: 32 x 11
    ##    number name         group  rating   elo fifa_start points goalsFore
    ##     <int> <chr>        <chr>   <dbl> <dbl>      <dbl>  <dbl>     <int>
    ##  1      4 Uruguay      A       34.0   1890        976   9.00         9
    ##  2      2 Russia       A       41.0   1685        493   4.00         5
    ##  3      1 Egypt        A      151     1646        636   4.00         3
    ##  4      3 Saudi Arabia A     1001     1582        462   0            2
    ##  5      8 Spain        B        7.00  2048       1162   9.00         4
    ##  6      7 Portugal     B       26.0   1975       1306   6.00         6
    ##  7      6 Morocco      B      501     1711        681   1.00         2
    ##  8      5 Iran         B      501     1793        727   1.00         1
    ##  9     11 France       C        7.50  1984       1166   7.00         5
    ## 10     10 Denmark      C      101     1843       1054   6.00         5
    ## # ... with 22 more rows, and 3 more variables: goalsAgainst <int>,
    ## #   goalsDifference <int>, groupRank <int>

``` r
find_knockout_winners(team_data = team_data, 
                      match_data = structure(c(3L, 8L, 10L, 13L), .Dim = c(2L, 2L)), 
                      play_fun = "play_fun_skellam")
```

    ## $winners
    ## [1] 10  8
    ## 
    ## $goals
    ##   V1 V2 Agoals Bgoals
    ## 1  3 10      0      4
    ## 2  8 13      3      1

Run the tournament
==================

Here is the most exciting part. We simulate the tournament. The following `resultX` objects are 32 \* `R params$nsim` matrices, each row representing predicted rankings each simulation run. `get_winner()` reports a winner list who has highest probability to be ranked top. We lock the simulation outcome by setting a seed.

``` r
# Run nsim number of times world cup tournament
set.seed(1984)
result <- simulate_tournament(nsim = nsim, play_fun = "play_fun_simplest") 
result2 <- simulate_tournament(nsim = nsim, play_fun = "play_fun_skellam")
result3 <- simulate_tournament(nsim = nsim, play_fun = "play_fun_elo")
result4 <- simulate_tournament(nsim = nsim, play_fun = "play_fun_double_poisson")
```

Get winner list
===============

The simple two possion model and skellam model results are presented. Clearly first one is random and second one is more in line with common sense.

``` r
get_winner(result_data = result)$pic
```

![](dataMod_files/figure-markdown_github/winner-1.png)

``` r
get_winner(result_data = result2)$pic
```

![](dataMod_files/figure-markdown_github/winner-2.png)

``` r
get_winner(result_data = result3)$pic
```

![](dataMod_files/figure-markdown_github/winner-3.png)

``` r
get_winner(result_data = result4)$pic
```

![](dataMod_files/figure-markdown_github/winner-4.png)

Who will be top scoring team?
=============================

The second model seems more reliable, the forth one gives systematically lower scoring frequency than probable actuals. They both favours Brazil though.

``` r
get_top_scorer(nsim = nsim, result_data = result2)$pic
```

![](dataMod_files/figure-markdown_github/top_score_team-1.png)

``` r
get_top_scorer(nsim = nsim, result_data = result4)$pic
```

![](dataMod_files/figure-markdown_github/top_score_team-2.png)

Conclusion
==========

The framework is pretty clear, all you need is to customise the `play_game` function such as `game_fun_simplest`, `game_fun_skellam` and `game_fun_elo`.

Tick-tock... Don't hesitate to send a pull request to [ekstroem/socceR2018](github.com/ekstroem/socceR2018) on GitHub. Who is winning the guessing-who-wins-worldcup2018 game? The R community have got your back.

Notes to readers and future myself
==================================

1.  Data collectin. I didn't get to feed models with most updated betting odds and ELO ratings in the **team\_data** dataset. If you would like to, they are available on the below three sources. FIFA rating is the easiest can be scraped by rvest in the usual way. The ELO ratings and betting odds tables seem to have been rendered by javascript and I haven't found a working solution. For betting information, Betfair, an online betting exchange has an API and R package [`abettor`](https://github.com/phillc73/abettor) helps to pull those odds which are definetly interesting for anyone who are after strategy beyond predction.

-   <https://www.betfair.com/sport/football>
-   <https://www.eloratings.net/2018_World_Cup>
-   <http://www.fifa.com/fifa-world-ranking/ranking-table/men/index.html>

1.  Model enhancement. Previous research have suggested various bivariate poissons for football predictions.

2.  Feature engineering. Economic factors such as national GDP, market information like total player value or insurance value and player injure data may be useful to improve accuracy.

3.  Model evaluation. One way to understand if our model has good prediction capibility or not is to evaluate the predictions against actual outcomes after 15 July 2018. Current odds from bookies can also be referred to. It is not imporssible to run the whole thing on historical data e.g. Year 2014. and perform model selection and tuning.
