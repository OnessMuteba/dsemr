---
title: "Week 6"
author: "Renewable Energy Forecasting"
date: ""
output:
  #pdf_document: default
  html_document:
    keep_md: yes
    number_sections: true 
---



# Introduction to Renewable Energy Forecasting

Overview of Week 6, which focuses on renewable energy forecasting.

## Introduction Video

![Video: Intorduction to Week 6](loading.png)

## Weather and Renewables

Wind and solar power provide clean energy from abundant natural resources, but are highly variable as the amount of energy they produce at any given moment depends on the weather. Therefore, in order to make sure there is sufficient energy supply to meet demand, the energy sector relies on renewable energy production forecasts on a range of timescales.  

Weather forecasting has been of interest to humans for millennia, but only in the last 50 years or so have we developed the capability to accurately forecast local weather conditions using computer models of the Earth's atmosphere in a process called Numerical Weather Prediction (NWP). This week we will concentrate on how NWP are converted into accurate renewable energy production forecasts from wind and solar farms using some of the statistical methods seen earlier in the course.  

### Renewable Energy Forecasting Model Chain

![Renewable Energy Forecasting Model Chain](ModelChain.png)

Let's have a look at the chain of data flows and models that are used in energy forecasting. The model chain begins with observations that are used to initialise NWP models. Before the future state of the atmosphere can be predicted an accurate estimate of the atmosphere's current state is produced by assimilating observations of atmospheric variables such as pressure, temperature and wind speeds. The starting point for the NWP model is an estimate of a wide range of atmospheric variables on a regular grid spanning either the entire globe or a region of interest. This initial estimate is then propagated forwards by applying physical laws to produce an estimate of future values of atmospheric variables on the same regular grid.  

The next step it to produce energy production forecasts from gridded NWP. This process first involves selecting relevant atmospheric variables, such as wind speed and direction for wind power, or irradiance and cloud cover for solar power, before converting these into energy forecasts. This conversion process may be based on a physical representation of the weather-to-energy process, or a statistical model trained on historic NWP and generation data. We will look at both on this course week.  

The final step is forecast use and evaluation. As we will see, forecasts may be presented in different ways for different use cases, and should to be evaluated appropriately. Users should also have an awareness of what level of accuracy, or skill more generally, the can expect from the forecasts they base their decisions on. Robust forecast evaluation is also important when discriminating between different, competing methods or providers of forecasts.  

For more about weather forecasting and the interaction between the weather and energy, see this [open-access text book](https://doi.org/10.1007/978-3-319-68418-5).

## Data Exploration for Energy Forecasting

We will be using numerical weather predictions (NWP) as the input data for our wind and solar power forecasts. In all of the examples we'll see, NWP data for a single location close to a wind or solar farm has been extracted and combined with the corresponding energy production. Lets take a closer look at the wind power data.


```r
load("WindData.Rda")

# Have a look at the data...
head(WindData)
```

```
##              ISSUEdtm           TARGETdtm      Power      U10        V10
## 1 2011-12-31 12:00:00 2012-01-01 01:00:00 0.00000000 2.124600 -2.6819664
## 2 2011-12-31 12:00:00 2012-01-01 02:00:00 0.05487912 2.521695 -1.7969601
## 3 2011-12-31 12:00:00 2012-01-01 03:00:00 0.11023400 2.672210 -0.8225162
## 4 2011-12-31 12:00:00 2012-01-01 04:00:00 0.16511606 2.457504 -0.1436423
## 5 2011-12-31 12:00:00 2012-01-01 05:00:00 0.15694013 2.245898  0.3895761
## 6 2011-12-31 12:00:00 2012-01-01 06:00:00 0.16878113 1.986038  0.7963042
##       U100       V100
## 1 2.864280 -3.6660758
## 2 3.344859 -2.4647615
## 3 3.508448 -1.2140929
## 4 3.215233 -0.3555464
## 5 2.957678  0.3327009
## 6 2.655406  0.8826480
```

First, notice that we have two time stamps. `ISSUEdtm` is the time at which the NWP was produced and "issued", and `TARGETdtm` is the time in the future that we have a prediction for. Here, we have forecasts issues at 12-noon for each hour of the next day. `Power` is the amount of energy generation at the `TARGETdtm`, and the other columns are the NWP variables. Here we have a simplified example with only one forecast for each target time, in reality a forecast user may have to deal with overlapping forecasts. For example, forecast for 0-48 hours ahead may be issued every 6 hours leading to significant overlap!

The four NWP variables are zonal (West to East) winds at 10m and 100m, `U10` and `U100`, and meridional (South to North) winds also at 10m and 100m, `V10` and `V100` respectively. Using Pythagoras' Theorem we can calculate wind speed and wind direction, and then plot the data to get a better picture of what it is like.


```r
WindData[["WindSpd"]] <- sqrt(WindData$U100^2+WindData$V100^2)
WindData[["WindDir"]] <- atan(WindData$V100/WindData$U100)

par(mar=c(5.1,4.1,2.1,4.1))
plot(Power~TARGETdtm,data=WindData[1:240,],
     type="l",ylim=c(0,1),xlab="Time",las=1)
lines(WindSpd/20~TARGETdtm,data=WindData[1:240,],col="blue")
axis(4,seq(0,1,length.out = 11),seq(0,20,length.out = 11),las=1)
mtext("Wind Speed [m/s]", side = 4, line = 3)
legend("top",c("Wind Power","Wind Speed Forecast"),col=c(1,"blue"),lty=1,bty = "n")
```

![](Week6_StandAlone_files/figure-html/unnamed-chunk-2-1.png)<!-- -->

```r
plot(Power~WindSpd,data=WindData,pch=".",xlab="Wind Speed [m/s]")
```

![](Week6_StandAlone_files/figure-html/unnamed-chunk-2-2.png)<!-- -->

It looks like there is a strong relationship between the predicted wind speed and wind power, but clearly we have some work to do in order to turn the NWP variables into accurate power forecast.

We can do the same for the solar power data...

```r
load("SolarData.Rda")
head(SolarData)
```

```
##              ISSUEdtm           TARGETdtm      Power CloudCover      U10
## 1 2012-03-31 12:00:00 2012-04-01 01:00:00 0.75410256  0.2446011 1.039334
## 2 2012-03-31 12:00:00 2012-04-01 02:00:00 0.55500000  0.4571376 2.482865
## 3 2012-03-31 12:00:00 2012-04-01 03:00:00 0.43839744  0.7714294 3.339867
## 4 2012-03-31 12:00:00 2012-04-01 04:00:00 0.14544872  0.9658662 3.106102
## 5 2012-03-31 12:00:00 2012-04-01 05:00:00 0.11198718  0.9446687 2.601146
## 6 2012-03-31 12:00:00 2012-04-01 06:00:00 0.05724359  0.6413534 1.333368
##         V10       T2 SurfaceRad
## 1 -2.503039 294.4485   830.7864
## 2 -2.993330 295.6514   771.7397
## 3 -1.982535 294.4546   712.6931
## 4 -1.446051 293.2615   538.5367
## 5 -1.904493 292.7329   356.2714
## 6 -1.728431 292.0771   186.8528
```

```r
par(mar=c(5.1,4.1,2.1,4.1))
plot(Power~TARGETdtm,data=SolarData[1:240,],
     type="l",ylim=c(0,1),xlab="Time",las=1)
lines(SurfaceRad/1000~TARGETdtm,data=SolarData[1:240,],col="blue")
axis(4,seq(0,1,length.out = 11),seq(0,1000,length.out = 11),las=1)
mtext("Surface Radiation [W/m^2]", side = 4, line = 3)
legend("top",c("Solar Power","Solar Radiation Forecast"),col=c(1,"blue"),lty=1,bty = "n")
```

![](Week6_StandAlone_files/figure-html/unnamed-chunk-3-1.png)<!-- -->

```r
plot(Power~SurfaceRad,data=SolarData,pch=".",xlab="Surface Radiation [W/m^2]")
```

![](Week6_StandAlone_files/figure-html/unnamed-chunk-3-2.png)<!-- -->


## Hands-on Data Exploration


Now it's your turn. Download the Wind and Solar data sets below and load them in to `R`. Explore the data using visualisations and the tools from previous weeks. Then post your results below and discuss them with your course mates.

Data can be downloaded from the [course GitHub page](https://github.com/levvers/dsemr). Wind power and wind  NWP data are in a single `data.frame` in `WindData.Rda`. Solar power and various NWP are in a single `data.frame` in `SolarData.Rda`.

Notes:

Notice that there are have two time stamps. `ISSUEdtm` is the time at which the NWP was produced and "issued", and `TARGETdtm` is the time in the future that we have a prediction for. Here, we have forecasts issues at 12-noon for each hour of the next day. `Power` is the amount of energy generation at the `TARGETdtm`, and the other columns are the NWP variables. Here we have a simplified example with only one forecast for each target time, in reality a forecast user may have to deal with overlapping forecasts. For example, forecast for 0-48 hours ahead may be issued every 6 hours leading to significant overlap!

In the wind dataset there are four NWP variables: zonal (West to East) winds at 10m and 100m, `U10` and `U100`, and meridional (South to North) winds also at 10m and 100m, `V10` and `V100` respectively. Wind speeds are in units of meters-per-second. Using Pythagoras' Theorem we can calculate wind speed and wind direction, and then plot the data to get a better picture of what it is like.

In the solar dataset the NWP variables are as follows: `T2` is the temperature 2m about ground in degrees Kelvin. `CloudCover` is cloud cover where 0 is no cloud and 1 is complete coverage. `U10` and `V10` are wind speed in the West-East and South-North direction, respectively, in units of meters-per-second. `SurfaceRad` is solar radiation at the surface in units of watts-per-square meter.

Acknowledgement: This data was used for the 2014 Global Energy Forecasting Competition and was published as supplementary data with the paper detailed below and on [Tao Hong's personal website](http://blog.drhongtao.com/2017/03/gefcom2014-load-forecasting-data.html).

Tao Hong, Pierre Pinson, Shu Fan, Hamidreza Zareipour, Alberto Troccoli, Rob J. Hyndman, "Probabilistic energy forecasting: Global Energy Forecasting Competition 2014 and beyond", International Journal of Forecasting, 32(3), 896-913, 2016, doi: 10.1016/j.ijforecast.2016.02.001.







# Deterministic Forecasting and Power Curves

Methods for producing forecasts of wind and solar power using power curves and statistical methods.

## Forecasting using the Wind Turbine Power Curve

Wind turbines produce energy by converting kinetic energy in the wind into electrical energy. This conversion is given by the "power equation"

$$P=\frac{1}{2} c_p \rho Av^3$$
where $P$ is power, $c_p$ is the power coefficient, which is defined as the proportion of the total power in the wind extracted by the turbine, $\rho$ is air density, $A$ is the area of the wind turbine rotor, and $v$ is the wind speed.

In reality, wind turbines only operate when the wind speed is above a lower limit, called "cut-in" and their power is limited to their rated output in high wind speeds. In extreme wind speeds they will even shut-down to protect themselves from damage.

$$
P =
\begin{cases}
0 & \text{if  $v<v_\text{cut-in}$ }\\
\frac{1}{2} c_p \rho Av^3 & \text{if  $v_\text{cut-in} \le v \le v_\text{rated}$ }\\
P_\text{rated} & \text{if  $v_\text{rated} \le v < v_\text{cut-out}$ }\\
0 & \text{if  $v \ge v_\text{cut-out}$ }\\
\end{cases}  
$$

We can use this physical relationship to to convert the NWP wind speed forecasts into energy forecasts! In the following examples we'll use historic data to fit a statistical model to produce forecasts, but if no historic data is available, as is the case with a new wind farm, for example, a physical approach is the only option.


<!-- ## Activity: Power Curver Forecast -->

<!-- Here, the power data has been normalised so that it is in the range 0 to 1, and we can assume the following values: -->
<!-- $$ -->
<!-- \begin{aligned} -->
<!-- v_\text{cut-in} &= 3\text{ms}^{-1} \\ -->
<!-- v_\text{rated} &= 10\text{ms}^{-1} \\ -->
<!-- v_\text{cut-out} &= 25\text{ms}^{-1} \\ -->
<!-- P_\text{rated} &= 1 \\ -->
<!-- \frac{1}{2}c_p\rho A &= \frac{1}{v_\text{rated}^3} = 10^{-3}\text{m}^{-3}\text{s}^3 \\ -->
<!-- \end{aligned} -->
<!-- $$ -->


<!-- Using the information on this page, write a function in `R` to convert the wind speed forecasts into power forecasts. Compare your forecasts to the actual values and discuss what you find. Think about the qualities you would like your forecast to have. -->



## Evaluating Deterministic Forecasts

Congratulations! Now you have produced your first wind power forecasts we can think about forecast evaluation. Once we accept that forecasts are never going to be perfect, it is important to consider what kind of imperfections are acceptable, and what we want to avoid. But first, what makes a *good* forecast? Is it:

* A prediction which is as likely to be too high as too low?
* Mostly small errors all of the time?
* Mostly very small errors and only a few large errors?
* Mostly small errors and warnings when errors might be large?
* Intervals showing the range of possible outcomes?
* Multiple intervals showing the range of possible outcome and their probability of occurring?

As you can see, things can get complicated pretty quickly and we might find ourselves asking a lot of our forecast!

**Importantly, a forecast's value is only realised when it leads to better decision making.** We should always keep this in mind, but for now lets work on the basis that forecast users  prefer smaller errors than large ones.

Some notation:
$$
\begin{aligned}
y_t &= f(x_t) + \epsilon_t \\
\hat{y}_t &= f(x_t)
\end{aligned}  
$$
For the remainder of this week our target variable, the quantity we are trying to predict at time $t$, will be denoted $y_t$, and predictions of it will be labelled with a hat, $\hat{y}_t$. Predictions are produced by some function $f(\cdot)$, and example of which you have already produced. The forecast error at time $t$ is $\epsilon_t=y_t-\hat{y}_t$.

Some metrics we will consider for evaluating forecasts are as follows:

Mean Absolute Error
$$
\text{MAE} = \frac{1}{T}\sum_{t=1}^T |\epsilon_t|
$$
Root Mean Squared Error
$$
\text{RMSE} = \sqrt{\frac{1}{T}\sum_{t=1}^T \epsilon_t^2}
$$
Bias
$$
\text{Bias} = \frac{1}{T}\sum_{t=1}^T \epsilon_t
$$

Have a go a calculating these for your power curve based wind power forecasts and compare to my results below.


```r
plot(Power~WindSpd,data=WindData,pch=".",xlim=c(0,27),xlab="Wind Speed Forecast")
lines(seq(0,30,by=0.1),WindPC(seq(0,30,by=0.1)),col=2)
legend("right",c("Data","Power Curve"),pch=c(".",NA),lty=c(NA,1),col=1:2)
```

![](Week6_StandAlone_files/figure-html/unnamed-chunk-5-1.png)<!-- -->

```r
plot(Power~TARGETdtm,data=WindData[1:240,],type="l",xlab="Date/Time")
lines(PC~TARGETdtm,data=WindData[1:240,],col=2)
legend("top",c("Actual Power","Forecast"),lty=1,col=1:2)
```

![](Week6_StandAlone_files/figure-html/unnamed-chunk-5-2.png)<!-- -->

```r
# MAE
mean(abs(WindData$Power - WindData$PC),na.rm = T)
```

```
## [1] 0.1465964
```

```r
# RMSE
sqrt(mean((WindData$Power - WindData$PC)^2,na.rm = T))
```

```
## [1] 0.2108798
```

```r
# Bias
mean(WindData$Power - WindData$PC,na.rm = T)
```

```
## [1] -0.02890889
```

All of these metrics are averages. While this give an indication of performance in the long run, it is not clear where a particular method's strengths and weaknesses lie. For instance, you may only interested in forecasts of particular events, such as large changes. Forecast verification is the process of evaluating this kind of thing. The details are beyond this course, but you can find out more [here](http://www.wmo.int/pages/prog/arep/wwrp/new/Forecast_Verification.html).



## Activity: Wind Power Forecasting using the Power Curve

Now its time to make some forecasts! Using the information below, write a power curve function in `R` to convert wind speed forecast into wind power. Then evaluate your forecasts using the metrics introduced in *Evaluating Deterministic Forecasts*.

If you haven't already, download `WindPower.Rda` from the [course GitHub page](https://github.com/levvers/dsemr). The wind power observations have been normalised so that it is in the range 0 to 1, and we can assume the following values based on a typical wind turbine:

$$
\begin{aligned}
v_\text{cut-in} &= 3\text{ms}^{-1} \\
v_\text{rated} &= 10\text{ms}^{-1} \\
v_\text{cut-out} &= 25\text{ms}^{-1} \\
P_\text{rated} &= 1 \\
\frac{1}{2}c_p\rho A &= \frac{1}{v_\text{rated}^3} = 10^{-3}\text{m}^{-3}\text{s}^3 \\
\end{aligned}
$$

1. Write an `R` function to convert wind speed to wind power using the information above.
2. Use your function to produce wind power forecasts from the NWP data.
3. Evaluate your forecasts by comparing them to the actual Power data. Calculate the Bias, Mean Absolute Error and Root Mean Squared Error for your forecasts.
4. Can you improve your error metrics by modifying the power curve model? The actual power curve for this wind farm will likely differ from the *typical* single wind turbine.
5. Share and discuss your results.


## Statistical Learning for Energy Forecasting

While the theoretical power curve provides a reasonable forecast it does not reflect the true complexity of the relationship between weather forecasts and power produced by our wind or solar farm. Effects such as the layout and local geography have a an impact, as does the physical condition of the wind or solar farm. All of this would be extremely difficult to model physically, so where sufficient data are available, we can try to *learn* the relationship.

In this article we will attempt to learn the relationship between the NWP and observed power so that we might make better quality forecasts. First, we must split out data in to "training"" and "validation" data sets so that we can evaluate the actual predictive power of our models and avoid over-fitting. Then we'll try out some of the models seen earlier in the course.

Let's look at some data. Below solar power and relevant NWP variable are plotted together. `T2` is the temperature 2m about ground. `hour` is the hour of the day. `CloudCover` is cloud cover where 0 is no cloud and 1 is complete coverage. `SurfaceRad` is solar radiation at the surface in units of watts-per-square meter.


<!-- For this example we'll look at the Solar Data. Let's use $\frac{2}{3}$ of the data to train a model and then test it on the remaining $\frac{1}{3}$. -->
![](Week6_StandAlone_files/figure-html/unnamed-chunk-6-1.png)<!-- -->

There is a strong relationship between the surface radiation forecast `SurfaceRad` and power
production so let's begin with a simple linear regression model using this variable. We should remove the intercept term from our model as we know that when radiation is zero we would expect power production to be zero too. We can then plot the residuals against the other NWP data we have to see which other variables could help us improve our forecasts.


```r
SolarForeacst1 <- lm(Power~SurfaceRad-1,data=SolarData[SolarData$Training==T,])

SolarData$Pred <- predict(SolarForeacst1,newdata = SolarData)
SolarData$Error <- SolarData$Power - SolarData$Pred

plot(Pred~Power,data=SolarData,pch=".",ylab="Forecast",xlab="Actual")
```

![](Week6_StandAlone_files/figure-html/unnamed-chunk-7-1.png)<!-- -->

```r
print(paste0("Bias: ",mean(SolarData$Error[!SolarData$Training]),
             ", MAE: ",mean(abs(SolarData$Error[!SolarData$Training])),
             ", RMSE: ",sqrt(mean(SolarData$Error[!SolarData$Training]^2))))
```

```
## [1] "Bias: -0.0205396699114192, MAE: 0.0632691400072093, RMSE: 0.11644330856614"
```

```r
plot(Power~TARGETdtm,data=SolarData[which(SolarData$Training==F)[1:240],],
     type="l",xlab="Date/Time",
     ylim=c(0,1))
lines(Pred~TARGETdtm,data=SolarData[which(SolarData$Training==F)[1:240],],col="blue")
legend("top",c("Actual Power","Forecast Power"),lty=1,col=c(1,"blue"))
```

![](Week6_StandAlone_files/figure-html/unnamed-chunk-7-2.png)<!-- -->

```r
plot(data.frame(Residuals = SolarForeacst1$residuals,
                SurfaceRad = SolarData$SurfaceRad[SolarData$Training],
                T2 = SolarData$T2[SolarData$Training],
                CC = SolarData$CloudCover[SolarData$Training],
                Hour = SolarData$hour[SolarData$Training],
                WS = sqrt(SolarData$U10[SolarData$Training]^2+SolarData$U10[SolarData$Training]^2)),pch=".")
```

![](Week6_StandAlone_files/figure-html/unnamed-chunk-7-3.png)<!-- -->

It appears that some of the other NWP variables can help explain some of the forecast errors from this simple model, but there is also a complication: the time of day. The radiation forecasts follows the same pattern, increasing during the day, zero during the night, but other variables, such as temperature, cloud cover and so on, do not.


## Produce Your Own Solar Power Forecasts

The article *Statistical Models for Solar Power Forecasting* introduced a linear model for converting solar radiation forecasts in to power forecasts. In this activity try to include some of the other NWP to improve forecast performance.

Instructions:  Download `SolarPower.Rda` from the [course GitHub page](https://github.com/levvers/dsemr). The solar power observations have been normalised so that they are in the range 0 to 1. Split the solar data in to training and test data. Use the first $$\frac{2}{3}$$ rows of the data to train a model and then test it on the remaining $$\frac{1}{3}$$.

1. Build a linear model for solar power production with NWP variables as the input. You will need to take care to ensure that your model is sensible. For example, ensure it does not predict solar power production during hours of darkness.
2. Calculate the error metrics for your forecasts on the test data. Compare them to those from the *Statistical Models for Solar Power Forecasting* article.
4. Try to improve your model (reduce error scores) by including more NWP variables, or using them in a different way.
3. Share and discuss your methodology with your course mates.


# Forecasting and Uncertainty

A look at how we can quantify and communicate forecast uncertainty.

## Introduction to Probabilistic Energy Forecasting

Probabilistic forecasting aims to quantify uncertainty. Energy forecasts are never going to be perfect due to the complexity of weather and energy conversion processes. But, we can try to describe *how wrong* they might be in order to make sound decisions.

So far, we have considered forecasts that provide a single estimate of power production at some point in the future. We've also evaluated the accuracy of these predictions using error metrics. I some sense this gives an indication of uncertainty. A large MAE might imply large uncertainty, for example. However, this doesn't give us information about specific predictions. In some circumstances we might be very confident in a forecast. In others we may be less confident. Making this distinction can be very valuable when interpreting a particular forecast. 

Probabilistic forecasting aims to predict the range and associated probability of specific outcomes. For example, an *interval forecast* predicts the range that the outcome has some probability of being within. For example, the 90%  interval forecast for wind power is illustrated in the plot below. Notice how the width of the interval varies across time, and how the deterministic forecast (the median or *p50*, the 50th percentile) is sometimes closer the the upper or lower boundary. This provides a much more detailed picture of uncertainty than and average deterministic error score. 


![](Week6_StandAlone_files/figure-html/unnamed-chunk-8-1.png)<!-- -->


Interval forecasts may be produced using quantile regression, seen in Week 3 of the course. In the example above, the 5th and 95th quantiles have been estimated in order to produce the upper and lower boundaries of the interval.

For more detailed uncertainty information, we might like to consider multiple intervals, or even a predictive probability density function for the future power production.




## Evaluating Probabilistic Forecasts

We need to be able to evaluate probabilistic forecasts to measure performance and compare forecasting systems. These forecasts are a bit more complicated than deterministic forecast as they have two desirable properties:

1. Forecasts must be *calibrated*. This means that outcomes that are predicted to occur with probability $$x\%$$ should be observed in $$x\%$$ of cases. This property is also called *reliability*.
2. Forecast should be *sharp*. It is desirable to have narrow prediction intervals. This is the same as having confidence that the observation will fall in a narrow range. Sharpness should not be compromised for calibration, otherwise the forecast probabilities loose their meaning.


These qualities are quite different. Calibration is a necessary property for a forecast user. There is no point predicting probabilities if they do not match reality! Whereas sharpness is merely desirable, like small errors in deterministic forecasting. When discriminating between probabilistic forecasts one must apply the principle of *sharpness subject to clibration*.


**Evaluating Calibration/Reliability**

We can write down the definition of calibration mathematically as follows:

$$
\frac{1}{T}\sum_{t=1}^T \mathbf{1}(x_t \le q_{\alpha,t}) \approx \alpha
$$

where $$\mathbf{1}(\cdot)$$ is the indicator function and is equal to 1 if its argument is true and 0 otherwise. $$q_{\alpha,t}$$ is the $$\alpha$$ quantile forecast of $$x_t$$.

We can evaluate the reliability of a forecast by calculating the quantity on the left for a range of $$\alpha$$ values and comparing these to the ideal value, i.e. $$\alpha$$. Plotting the empirical proportion $$\frac{1}{T}\sum_{t=1}^T \mathbf{1}(x_t \le q_{\alpha,t})$$ against $$\alpha$$. This is called a *reliability diagram*. An example of a reliability diagram is shown below. A reliable forecast should produce a line along the diagonal, i.e. empirical proportion = nominal proportion. Forecasts that are too sharp, too broad (the opposite of sharp) and with some bias are also shown.

![](Week6_StandAlone_files/figure-html/unnamed-chunk-9-1.png)<!-- -->

**Evaluating Sharpness**

Sharpness is used to discriminate between reliable forecasts. A narrow interval is desirable as this indicates confidence that an observation will fall within a small range. Interval width can be calculated for a given nominal coverage. We can compare the average width of a particular interval of interest. The 90% interval is commonly used in practice for situational awareness. Mathematically, the average interval width is

$$
\text{Interval Width}_\alpha=\frac{1}{T}\sum_{t=1}^T \left( q_{0.5+\frac{\alpha}{2},t} - q_{0.5-\frac{\alpha}{2},t} \right)
$$

For a more detailed view, we can plot the interval width for a range of intervals to compare forecasts. An example of a sharpness diagram is given below for the same example forecasts as the reliability diagram.

![](Week6_StandAlone_files/figure-html/unnamed-chunk-10-1.png)<!-- -->

Notice how the forecast with a positive bias has the same sharpness as the true process. And that while the "too sharp" forecast may have the most confident forecasts, they are not reliable, as we have seen.

**Summary**

Probabilistic forecasts have two desirable properties: we would like them to be both calibrated and sharp. Care is required when evaluating probabilistic forecasts. While a confident or *sharp* forecast may be appealing, it must also be *reliable*/*calibrated* to have meaning. In the next section we'll see how probabilistic forecasts are used in decision-making.


**Extension**

A range of scores are available to quantify probabilistic forecast performance. Interested learners may like to search for the following:

* The Quantile Loss (sometimes called Pinball Loss)
* Continuous Rank Probability Score
* Log Score

In some cases, probabilistic forecasts of multiple quantities are required. For example: wind power at several wind farms at the same time instance, or forecasts of power at the same wind farm at multiple time instances, or both. These forecasts, must exhibit and an additional property: that the dependency structure between quantities is correct. This may be evaluated using the *multivariate energy score* or *variogram score*.



## Activity: Identify the best probabilistic forecast

Now you have learnt the basics of probabilistic forecast evaluation it's time to try it out for yourself! I have provided 4 different probabilistic forecasts for the wind power dataset. Your task is to identify which is the best.

You can download the four forecasts from the [course GitHub page](https://github.com/levvers/dsemr). The files `WindProbForecast_1.Rda`,...,`WindProbForecast_4.Rda` each contain quantile forecasts from 0.05 to 0.95 in steps of 0.05 for every row of the Wind Power Data (`WindData.Rda`) used in previous examples.

1. Download 4 different sets of probabilistic forecasts for the wind power dataset.
2. Write an `R` script to produce reliability and sharpness diagrams.
3. Produce reliability and sharpness diagrams for each of the forecasts.
4. Decide which is the best!
5. Discuss your choice and reasoning with your course mates.



# Forecast End Use

A look at how forecasts are used in the energy industry, and how to make decisions under uncertainty.

## Wind and Solar Forecast Use Cases

Forecasts of renewable generation are used across the energy sector. Generators must sell their power and maintain their assets. Suppliers must purchase enough energy for their customers. And network companies must make sure that power can get from generators to consumers. All of these activities rely on forecasts to ensure reliable and economic operation of energy systems. Let's look at a few examples.

### Energy Trading

Electricity is a special commodity because supply must meet demand in real time. Any *imbalance* will cause the frequency of the AC grid to deviate from its nominal value (50Hz in Europe, China, India, and 60Hz in North and Central America, Brazil). Therefore, energy is bought and sold for specific periods of time. 15-minute, 30-minute and 1-hour periods are typical. As this buying and selling happens in advance of delivery, forecasts are required to inform trading decisions.

For example, a company generating wind and solar power must forecast their generation for each period of the next day and sell this power. But what happens when the forecast is wrong? As we have seen, forecasting is never going to be perfect, so errors are inevitable. To make sure that supply and demand match, the power system operator buys back-up or *reserve* energy in a secondary market. Reserves are then used to fill the gaps caused by forecast errors. To cover the cost of reserves and incentives accurate forecasting, generators must pay for the difference between what they sold and what they generated. This is often call the *imbalance cost*. When evaluating a forecast, traders may consider the imbalance cost associated with a forecast provider as well as metric like MAE and RMSE.


### Maintenance Scheduling

Wind turbines and solar panels require maintenance just like any other piece of technology. Regular check-ups and replacing worn components is part of life. Performing maintenance usually requires the wind turbine or solar panel to be switched off for obvious safety reasons. It is desirable to minimise the amount of potential energy generation that is missed due to maintenance. Therefore, maintenance should be performed at times when either the wind is low or the sun is not shining. This means planning in advance using forecasts! Furthermore, some maintenance require working at heights and using large cranes. This is only safe when the wind speed is sufficiently low.

Large maintenance operations can last many hours or even days. The forecast user will be looking for *weather windows* where production is low and conditions are save for an extended period of time. This is different to the trading example above which was concerned with relatively short time periods.


### Selecting a Forecast Provider

Many organisations use forecasts provided by others. For example, an energy company by buy forecasts from a specialist provider. This has the advantage of enabling access to forecasting expertise and NWP. But there are many companies offering forecasts, how do we choose the best supplier for us? Often trials are run to help decide. But evaluating forecast quality, and then valuing differences in quality, can be very difficult. Here are some things to consider...


1. Do I need to run a trial?
    * Trials are time consuming
    * Care must be taken to ensure the trial is fair
    * Evaluating different providers can be difficult. Average error metrics do not tell the whole story. Things like customer service, performance in critical weather situations, quality of uncertainty information all play a role too.
2. How valuable is an extra 0.1% improvement in accuracy?
    * Is it worth spending more on a service which only offers a marginal improvement?
    * Is the observed improvement significant?
3. Do I need multiple forecast providers?
    * Combining different forecasts often produces a better final result.
    * Are potential suppliers using the same or different NWP inputs?
  
For more detailed information see guidance available from the [International Energy Agentcy Wind Power Forecasting Task](http://www.ieawindforecasting.dk/publications).


## Inverview with Forecast User

![Video: Inverview with Forecast User](loading.png)


## Decision-making Under Uncertainty

<!-- Perhap's the physasist Niels Bohr put it best when he remarked: -->

<!-- > It's difficult to make predicictions, especially about the future. -->

We all routinely make decisions based on predictions. And usually there is some uncertainty involved too. Imagine you have a ticket for the 17:20h train. You must predict how long it will take you to get to the station in order to board the train before it departs. Let's say 15-20 minutes, depending on how busy the streets are. What time do you set off for the train station?

The thought of missing the train is so terrible (long wait until the next one, cost of a new ticket...) that you may leave more than 20 minutes to get to the station. More than the upper limit of your prediction. Perhaps much more if you don't like taking risks. After all, what if the traffic if very bad and it takes 30 minutes to get to the station!

This is an example of an asymmetric penalty for a prediction error. If you *over predict* the journey time to the station you arrive early and wonder how you got your forecast so wrong before boarding the train. If you *under predict* the journey time you miss the train entirely and suffer the consequences!

We can formalise this type of problem to help us make optimal decisions in the long run. This is especially useful when the costs associated with decisions are easily quantified. Lets suppose that you normally leave work at 17:00h. If the streets are quiet then you get to the station in less than 20 minutes and catch the train. Hurray! But if the streets are busy, it takes longer than 20 minutes.  If you miss the train you'll have to buy a new ticket for $$Loss=�5$$. Boo! However, you can choose to leave early at a cost of $$Cost=�1$$ to guarantee that you catch the 17:20h train.

We can summarise this situation in a table:

|                 | Busy Streets | Quiet Streets   |
|-----------------|:------------:|:----------------:|
| Leave Early     | $$C=�1$$     | $$C=�1$$         |
| Leave at 17:00h | $$L=�5$$     | $$L=�0$$         |

Looking out of your office window, you see the street below and estimate the probability that it will take longer than 20 minutes to reach the station - $$p$$. What should you do? Incur a cost of $$�1$$ to leave early, or risk loosing $$�5$$ if you miss the train?

The optimal decision would be to calculate the *expected* loss.  If you don't leave early there is still a chance $$1-p$$ that you will catch the train, of course. The expected loss is $$p \times L$$. This is like saying that in the long run, the average loss if you left at 17:00h would work out to be $$p \times L$$. Now we can make a decision. If the expected loss is greater than the cost of leaving early, it would work out better in the long run to pay $$�1$$ to leave early. Mathematically, you should take action if $$pL>C$$. Or alternatively if $$p>\frac{C}{L}$$, which in this case is $$p>0.2$$.

So when you look out of your office window, if you predict that there is more than a 20% chance that you'll miss the train if you leave at 17:00h, you should leave early. Notice that a deterministic forecast would not help here. The probability is central to making the optimal decision.


OK. Perhaps this is a silly example. But many real-world decisions are very similar. The general table for this kind of decision is repeated below.


|                 | Event Occurs | Event Does Not Occur | Expected Cost |
|-----------------|:------------:|:--------------------:|:------:|
| Action Taken    | $$C$$        | $$C$$                | $$C$$  |
| No Action Taken | $$L$$        | $$0$$                | $$pL$$ |


The result is the same for a situation where you have to decide whether to incur a cost in order to achieve a possible gain. For example, you need to spend $$C=�1$$ to buy a lottery ticket, which comes with the chance of winning BIG! For consistency, I'll call the *win* $$L$$. In this case, the table becomes:

|                 | Event Occurs | Event Does Not Occur | Expected Cost |
|-----------------|:------------:|:--------------------:|:------:|
| Action Taken    | $$C-L$$        | $$C$$                | $$p(C-L) + (1-p)C$$  |
| No Action Taken | $$0$$        | $$0$$                | $$0$$ |

Note: a *negative cost* is like *income*! The result is the same, if $$p>\frac{C}{L}$$, in the long run, taking the action pays off.


### Use in the Electricity Market

Electricity supply must meet demand in real time. In order to incentives buying and selling only what is required, market participants are penalised for generating or consuming more than they have bought or sold. This penalty is often asymmetric to reflect the cost to the market of finding more generation when it is needed compared to reducing generation. It is typically much more expensive to find extra energy than to produce a bit less. This *price signal* results in there being excess energy more than 50% of the time. This is because it is more economic to incur a small penalty frequency and a large penalty rarely.

To make *optimal* decisions, probabilistic forecast of generation, demand and prices are required.


### Maintenance Scheduling

The cost of performing maintenance is often much less than the value of getting a piece of equipment up and running again. Therefore, it is usually worth planning maintenance, even when there is less than 50% chance of success, or a risk of inuring additional costs from over-running.

### Risk Neutrality

We've only looked at expected cost in these examples. In some situations it may be more important to consider *risk*. For example, when safety is a factor, or one-off big decisions. This is beyond this course, but the ideas we've seen here can be extended to account for risk aversion. Or even risk-seeking!



