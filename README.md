
seer <img src="logo/seer.png" align="right" height="200"/>
==========================================================

[![Project Status: Active ? The project has reached a stable, usable state and is being actively developed.](http://www.repostatus.org/badges/latest/active.svg)](http://www.repostatus.org/#active) [![Build Status](https://travis-ci.org/thiyangt/seer.svg?branch=master)](https://travis-ci.org/thiyangt/seer)

------------------------------------------------------------------------

The `seer` package provides implementations of a novel framework for forecast model selection using time series features. We call this framework **FFORMS** (**F**eature-based **FOR**ecast **M**odel **S**election). For more details see our [paper](https://www.monash.edu/business/econometrics-and-business-statistics/research/publications/ebs/wp06-2018.pdf).

Installation
------------

You can install seer from github with:

``` r
# install.packages("devtools")
devtools::install_github("thiyangt/seer")
library(seer)
```

Usage
-----

The FFORMS framework consists of two main phases: i) offline phase, which includes the development of a classification model and ii) online phase, use the classification model developed in the offline phase to identify "best" forecast-model. This document explains the main functions using a simple dataset based on M3-competition data. To load data,

``` r
library(Mcomp)
data(M3)
yearly_m3 <- subset(M3, "yearly")
m3y <- M3[1:2]
```

#### FFORMS: offline phase

**1. Augmenting the observed sample with simulated time series.**

We augment our reference set of time series by simulating new time series. In order to produce simulated series, we use several standard automatic forecasting algorithms such as ETS or automatic ARIMA models, and then simulate multiple time series from the selected model within each model class. `sim_arimabased` can be used to simulate time series based on (S)ARIMA models.

``` r
library(seer)
simulated_arima <- lapply(m3y, sim_arimabased, Future=TRUE, Nsim=2, extralength=6, Combine=FALSE)
simulated_arima
#> $N0001
#> $N0001[[1]]
#> Time Series:
#> Start = 1989 
#> End = 2008 
#> Frequency = 1 
#>  [1]  5502.981  6039.700  6543.768  7074.822  7617.867  8132.125  8697.460
#>  [8]  9196.514  9578.127  9931.793 10162.880 10541.750 10833.819 11190.664
#> [15] 11430.559 11847.598 12266.847 12623.438 12883.927 13038.141
#> 
#> $N0001[[2]]
#> Time Series:
#> Start = 1989 
#> End = 2008 
#> Frequency = 1 
#>  [1]  5424.126  5969.090  6829.353  7780.855  8556.370  9284.492 10091.853
#>  [8] 10899.032 11697.469 12558.747 13485.736 14439.659 15407.947 16297.434
#> [15] 17110.261 17668.426 18376.127 19124.564 19809.235 20446.112
#> 
#> 
#> $N0002
#> $N0002[[1]]
#> Time Series:
#> Start = 1989 
#> End = 2008 
#> Frequency = 1 
#>  [1] 5552.816 6185.176 5906.697 5781.580 5046.073 4640.911 4315.080
#>  [8] 4943.844 5530.476 4158.374 4626.694 5263.665 5519.519 6246.075
#> [15] 5815.767 4706.952 3952.248 3893.357 3659.237 3884.406
#> 
#> $N0002[[2]]
#> Time Series:
#> Start = 1989 
#> End = 2008 
#> Frequency = 1 
#>  [1] 5394.466 5934.255 5624.784 4368.360 3274.271 3528.251 3809.485
#>  [8] 4793.599 5555.248 5548.560 4598.849 4285.629 4373.166 4409.114
#> [15] 4385.172 5878.530 5266.489 4934.202 5263.296 6121.699
```

Similarly, `sim_etsbased` can be used to simulate time series based on ETS models.

``` r
simulated_ets <- lapply(m3y, sim_etsbased, Future=TRUE, Nsim=2, extralength=6, Combine=FALSE)
simulated_ets
```

**2. Calculate features based on the training period of time series.**

Our proposed framework operates on the features of the time series. `cal_features` function can be used to calculate relevant features for a given list of time series.

``` r
library(tsfeatures)
M3yearly_features <- seer::cal_features(yearly_m3, database="M3", h=6, highfreq = FALSE)
head(M3yearly_features)
#> # A tibble: 6 x 25
#>   entropy lumpiness stability hurst trend   spikiness linearity curvature
#>     <dbl>     <dbl>     <dbl> <dbl> <dbl>       <dbl>     <dbl>     <dbl>
#> 1   0.773         0         0 0.971 0.995 0.000000237     3.58      0.424
#> 2   0.837         0         0 0.947 0.869 0.000179        2.05     -2.08 
#> 3   0.825         0         0 0.949 0.865 0.000193        1.75     -2.26 
#> 4   0.854         0         0 0.949 0.853 0.000368        2.87     -1.25 
#> 5   0.899         0         0 0.855 0.586 0.00127        -0.765    -1.77 
#> 6   0.798         0         0 0.964 0.964 0.0000217       3.56     -0.574
#> # ... with 17 more variables: e_acf1 <dbl>, y_acf1 <dbl>,
#> #   diff1y_acf1 <dbl>, diff2y_acf1 <dbl>, y_pacf5 <dbl>,
#> #   diff1y_pacf5 <dbl>, diff2y_pacf5 <dbl>, nonlinearity <dbl>,
#> #   lmres_acf1 <dbl>, ur_pp <dbl>, ur_kpss <dbl>, N <int>, y_acf5 <dbl>,
#> #   diff1y_acf5 <dbl>, diff2y_acf5 <dbl>, alpha <dbl>, beta <dbl>
```

**Calculate features from the simulated time series in the step 1**

``` r
features_simulated_arima <- lapply(simulated_arima, function(temp){
    lapply(temp, cal_features, h=6, database="other", highfreq=FALSE)})
fea_sim <- lapply(features_simulated_arima, function(temp){do.call(rbind, temp)})
do.call(rbind, fea_sim)
#> # A tibble: 4 x 25
#>   entropy lumpiness stability hurst trend    spikiness linearity curvature
#> *   <dbl>     <dbl>     <dbl> <dbl> <dbl>        <dbl>     <dbl>     <dbl>
#> 1   0.768         0         0 0.973 0.999      5.98e-9     3.57     -0.343
#> 2   0.771         0         0 0.973 1.000      1.45e-9     3.61      0.104
#> 3   0.933         0         0 0.798 0.563      1.03e-3    -0.196     1.98 
#> 4   0.890         0         0 0.860 0.438      2.16e-3    -0.747     0.446
#> # ... with 17 more variables: e_acf1 <dbl>, y_acf1 <dbl>,
#> #   diff1y_acf1 <dbl>, diff2y_acf1 <dbl>, y_pacf5 <dbl>,
#> #   diff1y_pacf5 <dbl>, diff2y_pacf5 <dbl>, nonlinearity <dbl>,
#> #   lmres_acf1 <dbl>, ur_pp <dbl>, ur_kpss <dbl>, N <int>, y_acf5 <dbl>,
#> #   diff1y_acf5 <dbl>, diff2y_acf5 <dbl>, alpha <dbl>, beta <dbl>
```

**3. Calculate forecast accuracy measure(s)**

`fcast_accuracy` function can be used to calculate forecast error measure (in the following example MASE) from each candidate model. This step is the most computationally intensive and time-consuming, as each candidate model has to be estimated on each series. In the following example ARIMA(arima), ETS(ets), random walk(rw), random walk with drift(rwd), standard theta method(theta) and neural network time series forecasts(nn) are used as possible models. In addition to these models following models can also be used in the case of handling seasonal time series,

-   snaive: seasonal naive method
-   stlar: STL decomposition is applied to the time series and then seasonal naive method is used to forecast seasonal component. AR model is used to forecast seasonally adjusted data.
-   mstlets: STL decomposition is applied to the time series and then seasonal naive method is used to forecast seasonal component. ETS model is used to forecast seasonally adjusted data.
-   mstlarima: STL decomposition is applied to the time series and then seasonal naive method is used to forecast seasonal component. ARIMA model is used to forecast seasonally adjusted data.
-   tbats: TBATS models

``` r
tslist <- list(M3[[1]], M3[[2]])
accuracy_info <- fcast_accuracy(tslist=tslist, models= c("arima","ets","rw","rwd", "theta", "nn"), database ="M3", cal_MASE, h=6, length_out = 1)
accuracy_info
#> $accuracy
#>         arima       ets       rw       rwd    theta        nn
#> [1,] 1.566974 1.5636089 7.703518 4.2035176 6.017236 2.5100445
#> [2,] 1.698388 0.9229687 1.698388 0.6123443 1.096000 0.2799302
#> 
#> $ARIMA
#> [1] "ARIMA(0,2,0)" "ARIMA(0,1,0)"
#> 
#> $ETS
#> [1] "ETS(M,A,N)" "ETS(M,A,N)"
```

**4. Construct a dataframe of input:features and output:lables to train a random forest**

`prepare_trainingset` can be used to create a data frame of input:features and output: labels.

``` r
# steps 3 and 4 applied to yearly series of M1 competition
data(M1)
yearly_m1 <- subset(M1, "yearly")
accuracy_m1 <- fcast_accuracy(tslist=yearly_m1, models= c("arima","ets","rw","rwd", "theta", "nn"), database ="M1", cal_MASE, h=6, length_out = 1)
features_m1 <- cal_features(yearly_m1, database="M1", h=6, highfreq = FALSE)

# prepare training set
prep_tset <- prepare_trainingset(accuracy_set = accuracy_m1, feature_set = features_m1)

# provides the training set to build a rf classifier
head(prep_tset$trainingset)
#> # A tibble: 6 x 26
#>   entropy lumpiness stability hurst trend  spikiness linearity curvature
#>     <dbl>     <dbl>     <dbl> <dbl> <dbl>      <dbl>     <dbl>     <dbl>
#> 1   0.683   0.0400      0.977 0.985 0.985 0.00000132      4.46    0.705 
#> 2   0.711   0.0790      0.894 0.988 0.989 0.00000154      4.47    0.613 
#> 3   0.716   0.0160      0.858 0.987 0.989 0.00000113      4.60    0.695 
#> 4   0.761   0.00201     1.32  0.982 0.957 0.00000896      4.48    0.0735
#> 5   0.628   0.00112     0.446 0.993 0.973 0.00000180      5.77    1.21  
#> 6   0.708   0.00774     0.578 0.986 0.975 0.00000331      4.75    0.748 
#> # ... with 18 more variables: e_acf1 <dbl>, y_acf1 <dbl>,
#> #   diff1y_acf1 <dbl>, diff2y_acf1 <dbl>, y_pacf5 <dbl>,
#> #   diff1y_pacf5 <dbl>, diff2y_pacf5 <dbl>, nonlinearity <dbl>,
#> #   lmres_acf1 <dbl>, ur_pp <dbl>, ur_kpss <dbl>, N <int>, y_acf5 <dbl>,
#> #   diff1y_acf5 <dbl>, diff2y_acf5 <dbl>, alpha <dbl>, beta <dbl>,
#> #   classlabels <chr>

# provides additional information about the fitted models
head(prep_tset$modelinfo)
#> # A tibble: 6 x 4
#>   ARIMA_name              ETS_name   min_label model_names            
#>   <chr>                   <chr>      <chr>     <chr>                  
#> 1 ARIMA(0,1,0) with drift ETS(A,A,N) ets       ETS(A,A,N)             
#> 2 ARIMA(0,1,1) with drift ETS(M,A,N) rwd       rwd                    
#> 3 ARIMA(0,1,2) with drift ETS(M,A,N) ets       ETS(M,A,N)             
#> 4 ARIMA(1,1,0) with drift ETS(M,A,N) rwd       rwd                    
#> 5 ARIMA(0,1,1) with drift ETS(M,A,N) arima     ARIMA(0,1,1) with drift
#> 6 ARIMA(1,1,0) with drift ETS(M,A,N) ets       ETS(M,A,N)
```

#### FFORMS: online phase is activated.

**5. Train a random forest and predict class labels for new series (FFORMS: online phase)**

`build_rf` in the `seer` package enables the training of a random forest model and predict class labels ("best" forecast-model) for new time series. In the following example we use only yearly series of the M1 and M3 competitions to illustrate the code. A random forest classifier is build based on the yearly series on M1 data and predicted class labels for yearly series in the M3 competition. Users can further add the features and classlabel information calculated based on the simulated time series.

``` r
rf <- build_rf(training_set = prep_tset$trainingset, testset=M3yearly_features,  rf_type="rcp", ntree=100, seed=1, import=FALSE)

# to get the predicted class labels
predictedlabels_m3 <- rf$predictions
table(predictedlabels_m3)
#> predictedlabels_m3
#>                 ARIMA            ARMA/AR/MA       ETS-dampedtrend 
#>                    72                     0                     0 
#> ETS-notrendnoseasonal             ETS-trend                    nn 
#>                     4                    29                    11 
#>                    rw                   rwd                 theta 
#>                    10                   510                     4 
#>                    wn 
#>                     5

# to obtain the random forest for future use
randomforest <- rf$randomforest
```

**6. Generate point foecasts and 95% prediction intervals**

`rf_forecast` function can be used to generate point forecasts and 95% prediction intervals based on the predicted class labels obtained in step 5.

``` r
forecasts <- rf_forecast(predictions=predictedlabels_m3[1:2], tslist=yearly_m3[1:2], database="M3", function_name="cal_MASE", h=6, accuracy=TRUE)

# to obtain point forecasts
forecasts$mean
#>          [,1]     [,2]     [,3]     [,4]     [,5]     [,6]
#> [1,] 5486.429 6035.865 6585.301 7134.737 7684.173 8233.609
#> [2,] 4402.227 4574.454 4746.681 4918.908 5091.135 5263.362

# to obtain lower boundary of 95% prediction intervals
forecasts$lower
#>          [,1]     [,2]     [,3]     [,4]     [,5]      [,6]
#> [1,] 4984.162 4893.098 4629.135 4199.745 3606.858 2848.8735
#> [2,] 2890.401 2366.671 1959.916 1608.186 1288.666  990.2221

# to obtain upper boundary of 95% prediction intervals
forecasts$upper
#>          [,1]     [,2]     [,3]      [,4]      [,5]      [,6]
#> [1,] 5988.696 7178.632 8541.467 10069.729 11761.488 13618.344
#> [2,] 5914.053 6782.236 7533.445  8229.629  8893.603  9536.501

# to obtain MASE
forecasts$accuracy
#> [1] 1.5636089 0.6123443
```
