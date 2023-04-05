Understanding Uncertainty in Ecological Forecasts: Assignment
================
Mary Lofton, Tadhg Moore, Quinn Thomas, Cayelan Carey
2023-04-04

## Purpose of this R Markdown

This R Markdown contains an assignment asking students to expand on the
basic functionality of “Macrosystems EDDIE Module 6: Understanding
Uncertainty in Ecological Forecasts” outside of R Shiny. The code can be
used by students to better understand what is happening “under the hood”
of the Shiny app, which can be found at the following link:  
<https://macrosystemseddie.shinyapps.io/module6/>.

This assignment is designed to be completed after students have
completed the module6.Rmd, which reproduces the basic functionality of
the Shiny app.

## Overview

In this assignment, you will generate forecasts of lake water
temperature for 1-7 days into the future using a slightly more complex
model than was used in the module6.Rmd. In addition to using today’s
water temperature and tomorrow’s air temperature as predictors of
tomorrow’s water temperature, you will choose one or more additional
weather variables to add as drivers to your model. Then, you will
explore how the forecast uncertainty of forecasts made using the
multiple linear regression model compare to forecasts made using the
simple linear regression model. This will involve the following steps:

1.  Read in and visualize data from Lake Barco, FL, USA.
2.  Read in and visualize a weather forecast for Lake Barco.
3.  Build a multiple linear regression forecast model with multiple
    weather variables as drivers.
4.  Generate a forecast with driver uncertainty.
5.  Generate a forecast with parameter uncertainty.
6.  Generate a forecast with process uncertainty.
7.  Generate a forecast with initial conditions uncertainty.
8.  Generate a forecast incorporating all sources of uncertainty.
9.  Partition uncertainty.  
10. Compare forecast uncertainty across models.

There are a total of 10 questions embedded throughout this assignment,
many of which ask you to expand on module code to complete the
assignment objectives. Please see the module rubric for possible points
per question and confirm with your instructor whether and how the module
will be graded.

## Set-up

We will install and load some packages that are needed to run the code.
If you do not currently have the packages below downloaded for RStudio,
you will need to install them first using the `install.packages()`
function.

``` r
# install.packages("tidyverse")
# install.packages("lubridate")
# install.packages("RColorBrewer")
# install.packages("ggthemes")
library(tidyverse)
library(lubridate)
library(RColorBrewer)
library(ggthemes)

source("./plot_functions.R")
```

## 1. Read in and visualize data from Lake Barco, FL, USA

Lake Barco is one of the lake sites in the U.S. National Ecological
Observatory Network (NEON). Please refer to
<https://www.neonscience.org/field-sites/barc> to learn more about this
site.

Read in and view lake data.

``` r
lake_df <- read_csv("data/BARC_lakedata.csv", show_col_types = FALSE)
```

Build time series plot of lake data.

``` r
ggplot(data = lake_df, aes(x = date, y = value))+
  facet_grid(rows = vars(variable), scales = "free_y")+
  geom_point()+
  theme_bw()
```

    ## Warning: Removed 128 rows containing missing values (`geom_point()`).

![](module6_assignment_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

## 2. Read in and visualize a weather forecast for Lake Barco

We have obtained a weather forecast for Lake Barco from the U.S.
National Oceanic and Atmospheric Administration Global Ensemble Forecast
System (NOAA GEFS).

Here, we read in NOAA forecast data that has been wrangled into a
suitable format for our exercise: a two-dimensional data frame
containing a 7-day-ahead forecast of air temperature, precipitation,
wind speed, and shortwave radiation with 30 ensemble members.

Read in and view the weather forecast. We will be working with a NOAA
forecast generated on 2020-09-25

``` r
weather_forecast <- read_csv("./data/BARC_forecast_NOAA_GEFS.csv", show_col_types = FALSE)
```

Voila! We now have an object called `weather_forecast` which is a
two-dimensional data frame containing a 7-day-ahead NOAA air temperature
forecast. Let’s look at `weather_forecast`.

``` r
head(weather_forecast)
```

    ## # A tibble: 6 × 4
    ##   forecast_date ensemble_member variable             value
    ##   <date>                  <dbl> <chr>                <dbl>
    ## 1 2020-09-25                  1 air_temperature      27.0 
    ## 2 2020-09-25                  1 shortwave_radiation 230.  
    ## 3 2020-09-25                  1 wind_speed            2.36
    ## 4 2020-09-26                  1 air_temperature      27.6 
    ## 5 2020-09-26                  1 shortwave_radiation 219.  
    ## 6 2020-09-26                  1 wind_speed            1.27

The columns are the following:  
- forecast_date: this is the date for which temperature is forecasted.  
- forecast_variable: this is the variable being forecasted.  
- ensemble_member: this is an identifier for each member of the
30-member ensemble.  
- value: this is the value of the forecasted variable.

The units for the variables are:  
- air_temperature: degrees Celsius.  
- precipitation: millimeters per…  
- wind_speed: meters per second.  
- surface_radiation: Watts per meter squared.

Now we will plot the NOAA forecast.

Build plot.

``` r
ggplot(data = weather_forecast, aes(x = forecast_date, y = value, group = ensemble_member))+
  facet_grid(rows = vars(variable), scales = "free_y")+
  geom_line(color = "gray")+
  theme_bw()
```

![](module6_assignment_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

## 3. Build a multiple linear regression forecast model

We will use observed water temperature and air temperature data to build
a multiple linear regression model to predict water temperature.

### Fitting model

**Add in schematic of how met drivers affect lake temperature.**

First, build data frame to fit model. We will use `pivot_wider()` to
create a column for each lake variable and then create a column
`wtemp_yday` which is a column of 1-day lags of water temperature.
Finally, we will `filter` to only include days for which we have values
for all variables using `complete.cases()`.

``` r
model_data <- lake_df %>%
  pivot_wider(names_from = "variable", values_from = "value") %>%
  mutate(wtemp_yday = lag(wtemp)) %>%
  filter(complete.cases(.))
```

Now it’s up to you! Fit a multiple linear regression model using
yesterday’s water temperature and any of the weather covariates (air
temperature, wind speed, short wave radiation) that you would like to
include in your model.

``` r
fit <- lm(model_data$wtemp ~ model_data$wtemp_yday + model_data$airt + model_data$swr + model_data$wnd) #INSERT ADDITIONAL COVARIATES HERE
fit.summ <- summary(fit)
```

View model coefficients and save them for our forecasts later.

``` r
coeffs <- round(fit$coefficients, 2)
coeffs
```

    ##           (Intercept) model_data$wtemp_yday       model_data$airt 
    ##                  2.09                  0.76                  0.21 
    ##        model_data$swr        model_data$wnd 
    ##                  0.00                 -0.22

View standard errors of estimated model coefficients and save them for
our forecasts later.

``` r
params.se <- fit.summ$coefficients[,2]
params.se
```

    ##           (Intercept) model_data$wtemp_yday       model_data$airt 
    ##          0.6177903084          0.0269833599          0.0252572636 
    ##        model_data$swr        model_data$wnd 
    ##          0.0006114442          0.0595286959

Calculate model predictions.

``` r
mod <- predict(fit, model_data)
```

Assess model fit by calculating $R^2$, (`r2`), mean bias (`err`), and
RMSE (`RMSE`).

``` r
r2 <- round(fit.summ$r.squared, 2) 
err <- mean(mod - model_data$wtemp, na.rm = TRUE) 
rmse <- round(sqrt(mean((mod - model_data$wtemp)^2, na.rm = TRUE)), 2) 
```

Prepare data frames for plotting.

``` r
lake_df2 <- model_data %>%
  filter(date > "2019-05-10" & date < "2019-10-10")
  
pred <- tibble(date = model_data$date,
               model = mod) %>%
  filter(date > "2019-05-10" & date < "2019-10-10")
```

Build plot of modeled and observed water temperature. This should match
the plot you see in the Shiny app Activity A, Objective 5 - Improve
Model for Forecasting IF you selected Lake Barco as your site.

``` r
mod_predictions_watertemp(lake_df2, pred)
```

![](module6_assignment_files/figure-gfm/unnamed-chunk-14-1.png)<!-- -->

**Q.1** Describe the structure of the multiple linear regression model
in your own words. How is water temperature being predicted? How does
the structure of this model differ from the simple linear regression
model fitted in the module6.Rmd?

**Answer Q.1**

**Q.2** Examine the values of the fitted model parameters (`coeffs`).
What does this tell you about the relative importance of past water
temperature and current air temperature in driving current water
temperature? Examine the standard errors of the parameters
(`params.se`). What do the standard errors tell you about how confident
we are in the fitted model parameter values?

**Answer Q.2**

**Q.3** Assess the model fit. Examine the values of `r2`, `err`, and
`rmse`, as well as the plot showing the model fit and observations. What
does this tell you about model performance? How does performance of the
multiple linear regression model compare to the linear regression model
fitted in the module6.Rmd?

**Answer Q.3**

## 4. Generate a forecast with driver uncertainty

Set number of ensemble members. We are using all ensemble members of the
NOAA air temperature forecast.

``` r
n_members <- 30 
forecast_start_date <- as.Date("2020-09-25")
```

Set up a date vector of dates we want to forecast. Our maximum forecast
horizon (the farthest into the future we want to forecast) is 7 days.

``` r
forecast_horizon <- 7
forecasted_dates <- seq(from = ymd(forecast_start_date), to = ymd(forecast_start_date)+7, by = "day")
```

Pull the current observed water temperature to be our initial, or
starting, condition.

``` r
curr_wt <- lake_df %>% 
  filter(date == forecast_start_date & variable == "wtemp") %>%
  pull(value)
```

Reformat weather forecast data frame for forecasting.

``` r
driver_data <- weather_forecast %>%
  pivot_wider(names_from = "variable", values_from = "value")
```

Setting up an empty dataframe that we will fill with our water
temperature predictions.

``` r
forecast_driver_unc <- tibble(forecast_date = rep(forecasted_dates, times = n_members),
                            ensemble_member = rep(1:n_members, each = length(forecasted_dates)),
                            forecast_variable = "water_temperature",
                            value = as.double(NA),
                            uc_type = "driver") %>%
  mutate(value = ifelse(forecast_date == forecast_start_date, curr_wt, NA)) 
```

Run forecast. **Notice** that we pull the entire driver ensemble of the
NOAA forecast for each day, and that in addition to pulling this driver
ensemble for the relevant forecast date, we also pull lagged water
temperature values to drive our multiple linear regression model.

``` r
for(d in 2:length(forecasted_dates)) {
  
  #pull prediction dataframe for relevant date
    temp_pred <- forecast_driver_unc %>%
      filter(forecast_date == forecasted_dates[d])
    
  #pull driver ensemble for relevant date; here we are using all 30 NOAA ensemble members
    temp_driv <- driver_data %>%
      filter(forecast_date == forecasted_dates[d])
    
  #pull lagged water temp values
    temp_lag <- forecast_driver_unc %>%
      filter(forecast_date == forecasted_dates[d-1])
    
  #run model
    temp_pred$value <- coeffs[1] + temp_lag$value * coeffs[2] + temp_driv$air_temperature * coeffs[3] + temp_driv$shortwave_radiation * coeffs[4] + temp_driv$wind_speed * coeffs[5] 
    
  #insert values back into forecast dataframe
    forecast_driver_unc <- forecast_driver_unc %>%
      rows_update(temp_pred, by = c("forecast_date","ensemble_member","forecast_variable","uc_type"))
}
```

Build plot - this should resemble the water temperature forecast plot in
the R Shiny app, Activity B Objective 9 (“Both” model)

``` r
lake_obs <- lake_df %>% 
  pivot_wider(names_from = "variable", values_from = "value") %>%
  filter(date >= "2020-09-22" & date <= "2020-10-02") %>%
  mutate(wtemp = ifelse(date > forecast_start_date, NA, wtemp))

plot_forecast(lake_obs, 
              forecast = forecast_driver_unc, 
              forecast_start_date,
              title = "Forecast with Driver Uncertainty")
```

    ## Warning: Removed 7 rows containing missing values (`geom_point()`).

![](module6_assignment_files/figure-gfm/unnamed-chunk-21-1.png)<!-- -->

## 5. Generate a forecast with parameter uncertainty

Our model now has several parameters: $\beta_1$, the intercept of the
linear regression, $\beta_2$, the coefficient on lagged water
temperature, and other parameters which are the coefficients on each of
the weather forecast variables you selected to include in your model.

$$WaterTemp_{t+1} = \beta_1 + \beta_2*WaterTemp_t + ...$$

When we fit our model, we obtained an estimate of the error around the
mean of each of these parameters, which is stored in the `params.se`
object. So instead of thinking of parameters as fixed values, we can
think of them as distributions (here, a normal distribution) with some
mean $(\mu)$ and variance (here represented by standard deviation, or
$\sigma$):

$$\beta_1 \sim {\mathrm Norm}(\mu, \sigma)$$

Now, we will generate parameter distributions based on parameter
estimates for the multiple linear regression model.

**Q.4** Use the `rnorm()` function and the information in the `coeffs`
and `params.se` objects to adapt the code below to generate parameter
distributions for each of the *three* parameters in the multiple
regression model. *Hint:* you will need to add a third column, `beta3`,
to the dataframe and populate the column with draws from a normal
distribution.

**Answer Q.4**

``` r
param.df <- data.frame(beta1 = rnorm(30, coeffs[1], params.se[1]),
                 beta2 = rnorm(30, coeffs[2], params.se[2]),
                 beta3 = rnorm(30, coeffs[3], params.se[3]),
                 beta4 = rnorm(30, coeffs[4], params.se[4]),
                 beta5 = rnorm(30, coeffs[5], params.se[5]))
```

Plot each of the parameter distributions you have created.

``` r
plot_param_dist(param.df)
```

    ## Using  as id variables

![](module6_assignment_files/figure-gfm/unnamed-chunk-23-1.png)<!-- -->

Now, we will adjust the forecasting for-loop to incorporate parameter
uncertainty into your forecasts.

**NOTE** Similar to how we used each of the 30 NOAA GEFS ensemble
members to generate 30 slightly different forecasts that made up an
ensemble with driver uncertainty, you will need to use slightly
different parameter values to generate multiple forecasts that together,
make up an ensemble incorporating parameter uncertainty. So here, each
of your water temperature forecast ensemble members will use the
**same** NOAA forecast but will have **different** parameter values
drawn from your parameter distributions. This allows us to quantify how
much uncertainty is coming from our model parameters while holding all
other sources of uncertainty constant.

Setting up an empty dataframe that we will fill with our water
temperature predictions.

``` r
forecast_parameter_unc <- tibble(forecast_date = rep(forecasted_dates, times = n_members),
                            ensemble_member = rep(1:n_members, each = length(forecasted_dates)),
                            forecast_variable = "water_temperature",
                            value = as.double(NA),
                            uc_type = "parameter") %>%
  mutate(value = ifelse(forecast_date == forecast_start_date, curr_wt, NA)) 
```

Run forecast.

**Notice** that we only pull a single member of the NOAA air temperature
forecast so that we can focus on the contribution of parameter
uncertainty alone to our forecast.

**Q.5** Adjust the forecast code below to pull parameter values from the
parameter distributions you created in the `param.df` object, rather
than using fixed values from the `coeffs` object. *Hint:* you will know
you have succeeded if your forecast plot shows multiple ensemble
members; if your plot has only one line, you have not yet successfully
included parameter uncertainty.

**Answer Q.5**

``` r
for(d in 2:length(forecasted_dates)) {
  
  #pull prediction dataframe for relevant date
    temp_pred <- forecast_parameter_unc %>%
      filter(forecast_date == forecasted_dates[d])
    
  #pull driver data for relevant date; here we select only 1 ensemble member from the NOAA air temperature forecast
    temp_driv <- driver_data %>%
      filter(forecast_date == forecasted_dates[d] & ensemble_member == 1)
    
  #pull lagged water temp values
    temp_lag <- forecast_parameter_unc %>%
      filter(forecast_date == forecasted_dates[d-1])
    
  #run model using parameter distributions instead of fixed values
    temp_pred$value <- param.df$beta1 + temp_lag$value * param.df$beta2 + temp_driv$air_temperature * param.df$beta3 + temp_driv$shortwave_radiation * param.df$beta4 + temp_driv$wind_speed * param.df$beta5 
    
  #insert values back into forecast dataframe
    forecast_parameter_unc <- forecast_parameter_unc %>%
      rows_update(temp_pred, by = c("forecast_date","ensemble_member","forecast_variable","uc_type"))
}
```

Build plot - this should resemble the water temperature forecast plot in
the R Shiny app, Activity B Objective 7 (“Both” model)

``` r
plot_forecast(lake_obs, 
              forecast = forecast_parameter_unc, 
              forecast_start_date,
              title = "Forecast with Parameter Uncertainty")
```

    ## Warning: Removed 7 rows containing missing values (`geom_point()`).

![](module6_assignment_files/figure-gfm/unnamed-chunk-26-1.png)<!-- -->

## 6. Generate a forecast with process uncertainty

We can add in process noise (W) to our model at each time step.

$$WaterTemp_{t+1} = \beta_1 + \beta_2*WaterTemp_t + ... + W$$

where process noise is equal to a random number drawn from a normal
distribution with a mean of zero and a standard deviation ($\sigma$).

$$W \sim {\mathrm Norm}(0, \sigma)$$ To account for process uncertainty,
we can run the model multiple times with random noise added to each
model run. More noise is associated with high process uncertainty, and
vice versa.

Define the standard deviation of the process uncertainty distribution,
`sigma`.

``` r
sigma <- 0.2 # Process Uncertainty Noise Std Dev.; this is your sigma
```

Setting up an empty dataframe that we will fill with our water
temperature predictions.

``` r
forecast_process_unc <- tibble(forecast_date = rep(forecasted_dates, times = n_members),
                            ensemble_member = rep(1:n_members, each = length(forecasted_dates)),
                            forecast_variable = "water_temperature",
                            value = as.double(NA),
                            uc_type = "process") %>%
  mutate(value = ifelse(forecast_date == forecast_start_date, curr_wt, NA)) 
```

Run forecast.

**Q.6** Adjust the forecast code below to add process uncertainty to
your forecast at each timestep. *Hint:* remember that you are *only*
calculating process uncertainty, so your parameter values should be
constant (i.e., use the `coeffs` object, not the `params.df` object).

**Answer Q.6**

``` r
for(d in 2:length(forecasted_dates)) {
  
  #pull prediction dataframe for relevant date
    temp_pred <- forecast_process_unc %>%
      filter(forecast_date == forecasted_dates[d])
    
  #pull driver data for relevant date; here we select only 1 ensemble member from the NOAA air temperature forecast
    temp_driv <- driver_data %>%
      filter(forecast_date == forecasted_dates[d] & ensemble_member == 1)
    
  #pull lagged water temp values
    temp_lag <- forecast_parameter_unc %>%
      filter(forecast_date == forecasted_dates[d-1])
    
  #run model using parameter distributions instead of fixed values
    temp_pred$value <- coeffs[1] + temp_lag$value * coeffs[2] + temp_driv$air_temperature * coeffs[3] + temp_driv$shortwave_radiation * coeffs[4] + temp_driv$wind_speed * coeffs[5] + rnorm(30, mean = 0, sd = sigma)
    
  #insert values back into forecast dataframe
    forecast_process_unc <- forecast_process_unc %>%
      rows_update(temp_pred, by = c("forecast_date","ensemble_member","forecast_variable","uc_type"))
}
```

Build plot - this should resemble the water temperature forecast plot in
the R Shiny app, Activity B Objective 6 (“Both” model)

``` r
plot_forecast(lake_obs, 
              forecast = forecast_process_unc, 
              forecast_start_date,
              title = "Forecast with Process Uncertainty")
```

    ## Warning: Removed 7 rows containing missing values (`geom_point()`).

![](module6_assignment_files/figure-gfm/unnamed-chunk-30-1.png)<!-- -->

## 7. Generate a forecast with initial conditions uncertainty

Generate a distribution of initial conditions for your forecast using
the current water temperature (`curr_wt`) and a standard deviation of
0.1 degrees Celsius (`ic_sd`).

``` r
ic_sd <- 0.1 
ic_uc <- rnorm(n = 30, mean = curr_wt, sd = ic_sd)
```

Plot the distribution around your initial condition. This should
resemble the initial condition distribution plot in the R Shiny app,
Activity B Objective 8 (“Both” model).

``` r
plot_ic_dist(curr_wt, ic_uc)
```

![](module6_assignment_files/figure-gfm/unnamed-chunk-32-1.png)<!-- -->

Create dataframe with distribution of initial conditions.

``` r
ic_df <- tibble(forecast_date = rep(as.Date(forecast_start_date), times = n_members),
                ensemble_member = c(1:n_members),
                forecast_variable = "water_temperature",
                value = ic_uc,
                uc_type = "initial_conditions")
```

Setting up an empty dataframe that we will fill with our water
temperature predictions. **Note** the use of the `rows_update()`
function to populate the starting date of the forecast with values from
our initial conditions distribution.

``` r
forecast_ic_unc <- tibble(forecast_date = rep(forecasted_dates, times = n_members),
                            ensemble_member = rep(1:n_members, each = length(forecasted_dates)),
                            forecast_variable = "water_temperature",
                            value = as.double(NA),
                            uc_type = "initial_conditions") %>%
  rows_update(ic_df, by = c("forecast_date","ensemble_member","forecast_variable",
                            "uc_type")) 
```

Run forecast.

**Q.7** Adjust the forecast code below to incorporate initial conditions
uncertainty into your forecast.

**Answer Q.7**

``` r
for(d in 2:length(forecasted_dates)) {
  
  #pull prediction dataframe for relevant date
    temp_pred <- forecast_ic_unc %>%
      filter(forecast_date == forecasted_dates[d])
    
  #pull driver data for relevant date; here we select only 1 ensemble member from the NOAA air temperature forecast
    temp_driv <- driver_data %>%
      filter(forecast_date == forecasted_dates[d] & ensemble_member == 1)
    
  #pull lagged water temp values
    temp_lag <- forecast_ic_unc %>%
      filter(forecast_date == forecasted_dates[d-1])
    
  #run model using initial conditions distribution instead of a fixed value
    temp_pred$value <- coeffs[1] + temp_lag$value * coeffs[2] + temp_driv$air_temperature * coeffs[3] + temp_driv$shortwave_radiation * coeffs[4] + temp_driv$wind_speed * coeffs[5]
    
  #insert values back into forecast dataframe
    forecast_ic_unc <- forecast_ic_unc %>%
      rows_update(temp_pred, by = c("forecast_date","ensemble_member","forecast_variable","uc_type"))
}
```

Build plot - this should resemble the water temperature forecast plot in
the R Shiny app, Activity B Objective 8 (“Both” model)

``` r
plot_forecast(lake_obs, 
              forecast = forecast_ic_unc, 
              forecast_start_date,
              title = "Forecast with Initial Condition Uncertainty")
```

    ## Warning: Removed 7 rows containing missing values (`geom_point()`).

![](module6_assignment_files/figure-gfm/unnamed-chunk-36-1.png)<!-- -->

## 8. Generate a forecast incorporating all sources of uncertainty

To plot a forecast with all sources of uncertainty incorporated, we need
to generate a forecast that incorporates driver, parameter, process, and
initial conditions uncertainty.

Below, we will adjust the forecasting for-loop to incorporate all four
sources of uncertainty (driver, parameter, process, initial conditions)
into your forecasts. The forecast ensemble will have 30 members.

Setting up an empty dataframe that we will fill with our water
temperature predictions. **Note** the use of the `rows_update()`
function to populate the starting date of the forecast with values from
our initial conditions distribution.

``` r
forecast_total_unc <- tibble(forecast_date = rep(forecasted_dates, times = n_members),
                            ensemble_member = rep(1:n_members, each = length(forecasted_dates)),
                            forecast_variable = "water_temperature",
                            value = as.double(NA),
                            uc_type = "total") %>%
  rows_update(ic_df, by = c("forecast_date","ensemble_member","forecast_variable")) 
```

Run forecast. **Note** that we use all 30 NOAA air temperature forecast
ensemble members, use a distribution of parameter values rather than
fixed values, and have added process uncertainty to each iteration of
our forecast.

``` r
for(d in 2:length(forecasted_dates)) {
  
  #pull prediction dataframe for relevant date
    temp_pred <- forecast_total_unc %>%
      filter(forecast_date == forecasted_dates[d])
    
  #pull driver ensemble for relevant date; here we are using all 30 NOAA ensemble members
    temp_driv <- driver_data %>%
      filter(forecast_date == forecasted_dates[d])
    
  #pull lagged water temp values
    temp_lag <- forecast_total_unc %>%
      filter(forecast_date == forecasted_dates[d-1])
    
  #run model using initial conditions and parameter distributions instead of fixed values, and adding process uncertainty
    temp_pred$value <- param.df$beta1 + temp_lag$value * param.df$beta2 + temp_driv$air_temperature * param.df$beta3 + temp_driv$shortwave_radiation * param.df$beta4 + temp_driv$wind_speed * param.df$beta5 + rnorm(n = 30, mean = 0, sd = sigma) 
    
  #insert values back into forecast dataframe
    forecast_total_unc <- forecast_total_unc %>%
      rows_update(temp_pred, by = c("forecast_date","ensemble_member","forecast_variable","uc_type"))
}
```

Build plot - this should resemble the water temperature forecast plot in
the R Shiny app, Activity C Objective 10 (“Both” model)

``` r
plot_forecast(lake_obs, 
              forecast = forecast_total_unc, 
              forecast_start_date,
              title = "Forecast with Total Uncertainty")
```

    ## Warning: Removed 7 rows containing missing values (`geom_point()`).

![](module6_assignment_files/figure-gfm/unnamed-chunk-39-1.png)<!-- -->

## 9. Partition uncertainty

Now, we will partition the relative contributions of each source of
uncertainty to total forecast uncertainty.

Combine the forecasts with a single source of uncertainty into one
dataframe.

``` r
all_forecast_df <- bind_rows(forecast_driver_unc, forecast_parameter_unc) %>%
  bind_rows(., forecast_process_unc) %>%
  bind_rows(., forecast_ic_unc)
```

Group the dataframe by date and the type of uncertainty included in the
forecast, then calculate the standard deviation across all 30 ensemble
members for each date and uncertainty type.

``` r
sd_df <- all_forecast_df %>%
  group_by(forecast_date, uc_type) %>%
  summarize(sd = sd(value, na.rm = TRUE)) %>%
  ungroup()
```

    ## `summarise()` has grouped output by 'forecast_date'. You can override using the
    ## `.groups` argument.

Plot the contribution of each source of uncertainty to total forecast
uncertainty. This should resemble the uncertainty quantification plot in
the R Shiny app, Activity C, Objective 10 (“Both” model).

``` r
plot_partitioned_uc(sd_df)
```

![](module6_assignment_files/figure-gfm/unnamed-chunk-42-1.png)<!-- -->
\## 10. Compare across models

**Q.8** Which source of uncertainty contributes the most to total
forecast uncertainty for your model?

**Answer Q.8**

**Q.9** Compare how total forecast uncertainty over time differed
between forecasts made with your model vs. the multiple linear
regression model fitted in the module6.Rmd. Be sure to address how the
contribution of each of the four sources of uncertainty (process,
parameter, initial conditions & driver) is different between the two
models, and why this might be.

**Answer Q.9** Compare amount of total forecast uncertainty over time:

Compare contribution of driver uncertainty:

Compare contribution of parameter uncertainty:

Compare contribution of process uncertainty:

Compare contribution of initial conditions uncertainty:

**Q.10** You have been given \$5,000 to improve your water temperature
forecasts. Now that you have seen how increasing model complexity
affects forecast uncertainty, what would you spend this money on to
reduce your forecast uncertainty?

**Answer Q.10**

Congratulations! You have quantified all the uncertainty. Now, have a
nap. :-)

## Knitting and committing

Remember to Knit your document as a `github_document` and comment+push
to GitHub your code, knitted document, and any files in the `figure-gfm`
subdirectory that was created when you knitted the document.