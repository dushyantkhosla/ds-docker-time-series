# Build Time Series Forecasting Ensembles

This CentOS-based container running on Python3 has the tools necessary to build an ensemble of time-series forecacasting models. Inside the `/notebooks` folder, there are sample data and notebooks with examples on building the following models

- `ETS, TBATS` and `AUTO.ARIMA` (Using `R` through `rpy2`)
- `ARIMA, ARIMAX, SARIMAX` (Using `statsmodels`)
- `Prophet` (using Facebook's Python Library)
- `tsfresh` from Blue Yonder for automated feature extraction from time-series data.



# Forecasting 101

### Just a Regression Problem?

At a high level, forecasting should seem like a regression problem - we are, after all, trying to predict a *quantity*. In typical regression problems, we try to predict $y$ using some linear combination of features in $X$. Linear models make certain assumptions about the underlying data. 

These include

| Assumption              | Explanation                                         |
| ----------------------- | --------------------------------------------------- |
| Linearity               | Each $X$ must be correlated with $y$                |
| Multicollinearity       | $X$s must not be correlated with each other         |
| Homoskedasticity        | The variance must not vary                          |
| Residuals are $N(0, 1)$ | Error terms follow the Standard Normal Distribution |

### future = $f$(past)  

In typical forecasting problems, the most important $X$s are just **lagged values** of $y$.  This should be fairly intuitive; all we're saying is that *past values predict future values.*

So we will differentiate forecasting into two paths

- _Time Series only_
  - we have historical values and we would like to predict future values
- _Time Series and Exogenous Variables_
  - in addition to historical data, we have external variables that might affect $y$
  - for example, store sales might be affected not only by past values, but also weather, disposable income of zipcode, holiday effects etc.



## Key Definitions

As it turns out, running a regression on $y$ (our time-series) using its lagged values suffers from more than a few problems, notably

- **autocorrelation,** 
- **non-stationarity** and 
- **seasonality**.

## 1. Autocorrelation

The residuals ($actual - predicted$) of a regression analysis are supposed to be white noise, or in statistical terms, they should follow the Standard Normal Distribution (bell-curve, $\mu=0$ and $ \sigma=1$). With time-series data, we observe that residuals exhibit clear patterns; the $e_i$ are correlated.

### ACF Plots

- **ACF** (*autocorrelation function*) plots display correlation between a series and its lags. 
- Help to choose the ($q$) parameter for **MA** model. 
- ACF plots show the value of the correlation coefficient $\rho$ at each lag of the time-series. 
  - Shaded regions mark the 95% confidence interval. 
  - Any value of $\rho$ outside this region is significant. 
  - White noise will show no correlation with its lags.
- If the series is correlated with its lags then, generally, there are some trend or seasonal components and therefore its statistical properties are not constant over time.

### PACF Plots

- Partial autocorrelation plots (PACF), display correlation between a variable and its lags that is not explained by previous lags. 
- PACF plots are **useful when determining the order ($p$) of the AR model.**

## 2. Stationarity

> *Time-series models cannot be applied on data that is not stationary.* 

A time-series is said to be stationary if its **statistical properties** such as *mean* and *variance* remain **constant over time**. Formally, for stationarity, the data must obey the following assumptions

- $\mu \neq f(t)$, the mean shouldn't change over time
- $\sigma \neq f(t)$, the variance shouldn't change over time
- $cov(i, i+m) \neq f(t)$, the covariance should be constant over time

Intuitively, if $y$ exhibits stable behaviour over time then it is probable that it will follow the same in the future. _This is critical because time-series models like ARIMA do not operate on non-stationary data._ **Trend** and **Seasonality** are the **main causes of non-stationarity**. The central idea behind time series models is therefore to

-  to <u>estimate the trend and seasonality</u> in the series
-  <u>remove these from the series</u> to get a stationary series.
-  implement statistical <u>forecasting</u> techniques on this series.
-  convert the forecasted values into the original scale by <u>applying trend and seasonality constraints back</u>.

**a. Stationarity Detection**

Non-stationarity can be detected by

- Visual Inspection
  - A simple plot of the time-series using will help us spot **overall** increasing/decreasing **trends**, and some **seasonal variation**.
  - A plot of _rolling means and variances_ using `pandas.Series.rolling().mean()` or `pandas.Series.rolling().std()` will exhibit a trend.
- **Augmented Dickey-Fuller Test**
  - implemented as `adfuller()` under `statsmodels`
  - $H_0$ : $y$ is non-stationary
  - If the test-statistic > critical-value, and p-value > 0.05 we fail to reject $H_0$

```python
def stationarity(ts, w):
    """
    Produce plot of rolling mean, stddev
    Results of the Dickey-Fuller Test
    """
	from statsmodels.tsa.stattools import adfuller

    # Plot rolling means and variances
    pd.DataFrame({'Actual': ts,
                  'Means': ts.rolling(window=w).mean(),
                  'Stddevs': ts.rolling(window=w).std()
                 }, index=ts.index).plot(figsize=(16, 8));

    # Run the Augmented Dickey
    test_df = adfuller(ts, autolag='AIC')
    result = pd.concat([pd.Series(test_df[:4],
                                  index=['stat', 'pval', 'lags', 'numobs']),
                        pd.Series(test_df[4])])
    return result
```

**b. Treatment of Stationarity**

The typical strategy to remove non-stationarity is to *difference the variable until it is stationary.* This can be done using `pd.Series.diff()`

```python
# First order lag/differencing
lag_01 = ts.diff(1)
```

Other strategies include mathematical transformations like taking the log.



## 3. Seasonality

If you can observe a regular cycle in the residuals, your data has a seasonality.

**a. Seasonality Detection**

- Use the `seasonal_decompose()` plot in `statsmodels`. 
- Group the data over time units and make *boxplots*. For example, if data has a monthly frequency, make a boxplot for each month across the years - observe the means and variances.

**b. Seasonality Treatment**

- Though there are a few remedies available, we generally let the algorithm handle this. 
  - `SARIMAX` is a good example of one such algorithm.



## Transforming & Differencing the time-series

* **Log Transformations** help reducing variance in the data.
* **Differencing** helps reduce both trend and seasonality. First and second orders are useful.
* **Decomposition** is an automated method that returns a dataframe with the original time-series broken out into its constituent elements - trend, seasonality, and residuals. These residuals are supposed to be stationary.
    * Note that by default, `seasonal_decompose()` assumes that the model is additive. 



- 









# ARIMA Modeling: Choosing $p, d, q$

The aim is to determine the **order** of the model (a tuple of the form $(p, d, q)$) that fits our data.

### a. Differencing to find $d$

 The first (and most important) step in fitting an ARIMA model is the determination of the *order of differencing needed to stationarize the series.* Differencing tends to introduce negative correlation: if the series initially shows strong positive autocorrelation, then a non-seasonal difference will reduce the autocorrelation and perhaps even drive the lag-1 autocorrelation to a negative value. If you apply a second nonseasonal difference (which is occasionally necessary), the lag-1 autocorrelation will be driven even further in the negative direction.

If the lag-1 autocorrelation is zero or even negative, then the series does *not* need further differencing. Normally, the correct $d$ is the lowest order of differencing that yields a time series which

- which fluctuates around a well-defined mean value, and
- whose autocorrelation function (ACF) plot decays fairly rapidly to zero

| **Rules**                                                    |
| ------------------------------------------------------------ |
| If the series has positive autocorrelations out to a high number of lags, then it probably needs a higher order of differencing. |
| If the lag-1 autocorrelation is zero or negative, or the autocorrelations are all small and patternless, then the series does not need a higher order of  differencing. If the lag-1 autocorrelation is -0.5 or more negative, the series may be overdifferenced. |
| The optimal order of differencing is often the order of differencing at which the standard deviation is lowest. |
| A model with no orders of differencing assumes that the original series is stationary (mean-reverting). A model with one order of differencing assumes that the original series has a constant average trend (e.g. a random walk or SES-type model, with or without growth). A model with two orders of total differencing assumes that the original series has a time-varying trend (e.g. a random trend or LES-type model). |

### b. Exploratory analysis of **ACF** and **PACF** plots to find $p, q$

- **Autocorrelation Function (ACF)** is a measure of the correlation between $y$ and lagged versions of itself
- **Partial Autocorrelation Function (PACF)**  measures the correlation between $y$ with a lagged version of itself after controlling for the variations already explained by earlier lags. A partial *auto*correlation is the amount of correlation between a variable and a lag of itself that is not explained by correlations at all *lower-order*-lags.

| Rules                                                        |
| ------------------------------------------------------------ |
| If the PACF plot has a significant spike only at Lag $l$, it means that all the higher-order autocorrelations are effectively explained by the Lag $l-1$ autocorrelation. |
| If the partial autocorrelation is significant at lag *k* and not significant at any higher order lags-(i.e., if the PACF "cuts off" at lag *k*) then this suggests that you should try fitting an autoregressive model of order *k*. |
| If the ACF decays more slowly (i.e., has significant spikes at higher lags), the time-series may be under-differenced. |
| In general, the "partial" correlation between two variables is the amount of correlation between them which is not explained by their mutual correlations with a specified set of other variables.<br /><br />For example, if we are regressing a variable $Y$ on variables $X1, X2$, and $X3$, the partial correlation between Y and X3 is the square root of the reduction in variance that is achieved by adding X3 to the regression of Y on X1 and X2 |

**Code**

```python
from pandas.tools.plotting import autocorrelation_plot
from statsmodels.graphics.tsaplots import plot_acf, plot_pacf

autocorrelation_plot(y)

# Here, y_tf refers to the de-trended, stationary (transformed) y
fig, ax = plt.subplots(nrows=2, ncols=1, figsize=(16, 8))
plot_acf(y_tf, lags=40, ax=ax[0])
plot_pacf(y_tf, lags=40, ax=ax[1])
```

Use these plots, and the following rules-of-thumb to choose $p, q$

- **p** is the lag value where the **PACF** crosses the upper confidence interval for the first time.
- **q** is the lag value where the **ACF** crosses the upper confidence interval for the first time

| Rules                                                        |
| ------------------------------------------------------------ |
| If the **PACF** of the differenced series **displays a sharp cutoff** and/or the lag-1 autocorrelation is positive--i.e., if the series appears slightly "underdifferenced"--then consider adding an AR term to the model. The lag at which the PACF cuts off is the indicated number of AR terms. |
| If the **ACF** of the differenced series **displays a sharp cutoff** and/or the lag-1 autocorrelation is negative--i.e., if the series appears slightly "overdifferenced"--then consider adding an MA term to the model. The lag at which the ACF cuts off is the indicated number of MA terms. |

### c. Automatically finding $p, d, q$

The selection of orders for model components is a search problem. We have two options

- The `forecast` package in `R` has an **`auto.arima()`** function which may be leveraged through the `rpy2` library.
- We may set up a **grid-search** and automate the selection of model parameters based on a performance metric like `mean-squared-error` or `AIC`.





# Modeling Univariate Time Series with `ARIMA`

- **ARIMA** stands for **A**uto**R**egressive **I**ntegrated **M**oving **A**verage. 
- It is made up of 3 components, and is generally written as $ARIMA(p,d,q)$. 
- The aim is to determine the **order** of the model, which is a tuple of the form $(p, d, q)$

Let's discuss these individually.

### a. AutoRegressive, $AR(p)$

> *Quantify the influence of the lags on the y*     

-  $p$, is the *number of auto-regressive terms*
  - found from the **PACF plot**. 
- We obtain lagged variables of $y$ and 
- Identify those that are significantly correlated $y$

$$y_t = c + \sum\phi_iy_{t-i} + e_t$$

### b. Integrated, $I(d)$

> *Difference y with its lags to remove trends*

- $d$ is the *order of differencing*
  - calculated by trying out various orders of differencing and checking for stationarity
- Generally, $d = 1, 2$, ie, the residual of differenced $y$ usually become trend-less after the first/second order differencing.

### c. Moving Average, $MA(q)$

> *Extract the influence of the previous period’s <u>error terms</u> on the current period’s error*

- $q$ is the *number of moving-average terms* 
  - found from the **ACF plot** as the lag where the plot clears the confidence interval for the first time.

$$y_t = c + \sum\theta_ie_{t-i} + e_t$$

### Code

```python
import statsmodels.tsa.api as smt

# Fitting a ARIMA(1,1,1) model.
arima_111 = smt.SARIMAX(y, trend='c', order=(1, 1, 1)).fit()
arima_111.summary()
arima_111.resid
```



# Accounting for seasonality with `SARIMA`

We act like we have **two processes**, 

- one for **non-seasonal** component and 
- one for **seasonal** components, 

and we multiply them together.

A seasonal ARIMA model is therefore written as $ARIMA(p,d,q)×(P,D,Q)_s$

- Lowercase letters are for the non-seasonal component, just like before.
- Upper-case letters are a similar specification for the seasonal component, where $s$ is the periodicity (4 for quarterly, 12 for monthly, etc.)

As an example, we can fit a $SeasonalARIMA(2,0,1)×(0,1,2)_{12}$ as

The **non-seasonal** component

- $p=2$: period autoregressive: use $y_{t−1}$ and $y_{t−2}$
- $d=0$: no first-differencing of the data
- $q=1$: use the previous non-seasonal residual, $e_{t−1}$, to forecast

And the **seasonal** component is

- $P=0$: Don't use any previous seasonal values
- $D=1$: Difference the series 12 periods back: `y.diff(12)`
- $Q=2$: Use the two previous seasonal residuals

### Code

```python
sarima_201_012_12 = smt.SARIMAX(y, trend='c', order=(2, 0, 1),
                                seasonal_order=(0, 1, 2)).fit(disp=0)
sarima_201_012_12.summary()
sarima_201_012_12.resid
```



## One Step Ahead vs. Dynamic Forecasts

The **One Step Ahead** strategy makes rolling forecasts. At each point in time, we take the history up to that point and make a forecast for the next point. For each next forecast, we use actual values.

On the other hand, **Dynamic Forecasts** use information available at some point in time to make forecasts. Here, instead of plugging in the *actual* values beyond $t$, we plug in the *forecast* values.
