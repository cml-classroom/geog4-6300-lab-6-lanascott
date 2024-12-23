Geog6300: Lab 6
================

## Regression

``` r
library(sf)
library(tidyverse)
library(tmap)
library(car)
library(lmtest)
```

**Overview:** This lab focuses on regression techniques. You’ll be
analyzing the association of various physical and climatological
characteristics in Australia with observations of several animals
recorded on the citizen science app iNaturalist.

### Data and research questions

Let’s import the dataset.

``` r
lab6_data<-st_read("data/aus_climate_inat.gpkg")
```

    ## Reading layer `aus_climate_inat' from data source 
    ##   `C:\Users\lanas\Documents\geog4-6300-lab-6-lanascott\data\aus_climate_inat.gpkg' 
    ##   using driver `GPKG'
    ## Simple feature collection with 716 features and 22 fields
    ## Geometry type: POLYGON
    ## Dimension:     XY
    ## Bounding box:  xmin: 113.875 ymin: -43.38632 xmax: 153.375 ymax: -11.92074
    ## Geodetic CRS:  WGS 84 (CRS84)

The dataset for this lab is a 1 decimal degree hexagon grid that has
aggregate statistics for a number of variables:

- ndvi: NDVI/vegetation index values from Landsat data (via Google Earth
  Engine). These values range from -1 to 1, with higher values
  indicating more vegetation.
- maxtemp_00/20_med: Median maximum temperature (C) in 2000 or 2020
  (data from SILO/Queensland government)
- mintemp_00/20_med: Median minimum temperature (C) in 2020 or 2020
  (data from SILO/Queensland government)
- rain_00/20_sum: Total rainfall (mm) in 2000 or 2020 (data from
  SILO/Queensland government)
- pop_00/20: Total population in 2000 or 2020 (data from NASA’s Gridded
  Population of the World)
- water_00/20_pct: Percentage of land covered by water at some point
  during the year in 2000 or 2020
- elev_med: Median elevation (meters) (data from the Shuttle Radar
  Topography Mission/NASA)

There are also observation counts from iNaturalist for several
distinctively Australian animal species: the central bearded dragon, the
common emu, the red kangaroo, the agile wallaby, the laughing
kookaburra, the wombat, the koala, and the platypus.

Our primary research question is how the climatological/physical
variables in our dataset are predictive of the NDVI value. We will build
models for 2020 as well as the change from 2000 to 2020. The second is
referred to as a “first difference” model and can sometimes be more
useful for identifying causal mechanisms.

### Part 1: Analysis of 2020 data

We will start by looking at data for 2020.

**Question 1** *Create histograms for NDVI, max temp., min temp., rain,
and population, and water in 2020 as well as elevation. Based on these
graphs, assess the normality of these variables.*

``` r
hist(lab6_data$ndvi_20_med) #skewed right
```

![](Lab-6_regression_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

``` r
hist(lab6_data$maxtemp_20_med) #skewed left
```

![](Lab-6_regression_files/figure-gfm/unnamed-chunk-3-2.png)<!-- -->

``` r
hist(lab6_data$mintemp_20_med) #skewed left
```

![](Lab-6_regression_files/figure-gfm/unnamed-chunk-3-3.png)<!-- -->

``` r
hist(lab6_data$rain_20_sum) #skewed right
```

![](Lab-6_regression_files/figure-gfm/unnamed-chunk-3-4.png)<!-- -->

``` r
hist(lab6_data$pop_20) #skewed right
```

![](Lab-6_regression_files/figure-gfm/unnamed-chunk-3-5.png)<!-- -->

``` r
hist(lab6_data$water_20_pct) #skewed right
```

![](Lab-6_regression_files/figure-gfm/unnamed-chunk-3-6.png)<!-- -->

``` r
hist(lab6_data$elev_med) # skewed right
```

![](Lab-6_regression_files/figure-gfm/unnamed-chunk-3-7.png)<!-- -->

Non of these data appear to be normally distributed. The min and max
temperature variables are skewed to the left, and the remaining
variables are skewed to the right.

**Question 2** *Use tmap to map these same variables using Jenks natural
breaks as the classification method. For an extra challenge, use
`tmap_arrange` to plot all maps in a single figure.*

``` r
tmap_mode("plot")
```

    ## tmap mode set to plotting

``` r
ndvi20 <- tm_shape(lab6_data)+
  tm_polygons("ndvi_20_med",style="jenks")+
  tm_layout(title="2020 Median NDVI",title.position=c("left", "bottom"), legend.show=FALSE)

maxt20 <- tm_shape(lab6_data)+
  tm_polygons("maxtemp_20_med",style="jenks")+
  tm_layout(title="2020 Median Max Temp",title.position=c("left", "bottom"), legend.show=FALSE)

mint20 <- tm_shape(lab6_data)+
  tm_polygons("mintemp_20_med",style="jenks")+
  tm_layout(title="2020 Median Min Temp",title.position=c("left", "bottom"), legend.show=FALSE)

rain20 <- tm_shape(lab6_data)+
  tm_polygons("rain_20_sum",style="jenks")+
  tm_layout(title="2020 Total Rainfall (mm)",title.position=c("left", "bottom"), legend.show=FALSE)

pop20 <- tm_shape(lab6_data)+
  tm_polygons("pop_20",style="jenks")+
  tm_layout(title="2020 Population",title.position=c("left", "bottom"), legend.show=FALSE)

water20 <- tm_shape(lab6_data)+
  tm_polygons("water_20_pct",style="jenks")+
  tm_layout(title="2020 % Water Coverage", title.position=c("left", "bottom"), legend.show=FALSE)

elev <- tm_shape(lab6_data)+
  tm_polygons("elev_med",style="jenks")+
  tm_layout(title="Median Elevation",title.position=c("left", "bottom"), legend.show=FALSE)

tmap_arrange(ndvi20, maxt20, mint20, rain20, pop20, water20, elev)
```

    ## Variable(s) "elev_med" contains positive and negative values, so midpoint is set to 0. Set midpoint = NA to show the full spectrum of the color palette.

    ## Variable(s) "elev_med" contains positive and negative values, so midpoint is set to 0. Set midpoint = NA to show the full spectrum of the color palette.

![](Lab-6_regression_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

I decided to remove the legends here due to formatting and legibility
reasons.

**Question 3** *Based on the maps from question 2, summarise major
patterns you see in the spatial distribution of these data from any of
your variables of interest. How do they appear to be associated with the
NDVI variable?*

NDVI tends to be higher where rainfall is higher AND temperatures are
lower. In general, areas with higher population also have higher
relative NDVI, but not all high NDVI areas have a high population. I
cannot identify a pattern between % water cover and NDVI. Higher
elevation areas on the east side of the country have higher NDVI values
than those in central and western Australia.

**Question 4** *Create univariate models for each of the variables
listed in question 1, with NDVI in 2020 as the dependent variable. Print
a summary of each model. Write a summary of those results that indicates
the direction, magnitude, and significance for each model coefficient.*

``` r
modmax20 <- lm(ndvi_20_med ~ maxtemp_20_med,data=lab6_data)
summary(modmax20)
```

    ## 
    ## Call:
    ## lm(formula = ndvi_20_med ~ maxtemp_20_med, data = lab6_data)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -0.41874 -0.07657 -0.01927  0.06833  0.36382 
    ## 
    ## Coefficients:
    ##                  Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)     0.6612389  0.0294372   22.46   <2e-16 ***
    ## maxtemp_20_med -0.0130902  0.0009601  -13.63   <2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1251 on 714 degrees of freedom
    ## Multiple R-squared:  0.2066, Adjusted R-squared:  0.2055 
    ## F-statistic: 185.9 on 1 and 714 DF,  p-value: < 2.2e-16

``` r
modmin20 <- lm(ndvi_20_med ~ mintemp_20_med,data=lab6_data)
summary(modmin20)
```

    ## 
    ## Call:
    ## lm(formula = ndvi_20_med ~ mintemp_20_med, data = lab6_data)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -0.36375 -0.08418 -0.03047  0.06972  0.40383 
    ## 
    ## Coefficients:
    ##                 Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)     0.464461   0.018997   24.45   <2e-16 ***
    ## mintemp_20_med -0.012282   0.001131  -10.86   <2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1301 on 714 degrees of freedom
    ## Multiple R-squared:  0.1418, Adjusted R-squared:  0.1406 
    ## F-statistic:   118 on 1 and 714 DF,  p-value: < 2.2e-16

``` r
modrain20 <- lm(ndvi_20_med ~ rain_20_sum,data=lab6_data)
summary(modrain20)
```

    ## 
    ## Call:
    ## lm(formula = ndvi_20_med ~ rain_20_sum, data = lab6_data)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -0.56681 -0.04753 -0.01210  0.04599  0.30930 
    ## 
    ## Coefficients:
    ##              Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept) 1.303e-01  7.060e-03   18.45   <2e-16 ***
    ## rain_20_sum 9.124e-07  3.953e-08   23.08   <2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1063 on 714 degrees of freedom
    ## Multiple R-squared:  0.4273, Adjusted R-squared:  0.4265 
    ## F-statistic: 532.6 on 1 and 714 DF,  p-value: < 2.2e-16

``` r
modpop20 <- lm(ndvi_20_med ~ pop_20,data=lab6_data)
summary(modpop20)
```

    ## 
    ## Call:
    ## lm(formula = ndvi_20_med ~ pop_20, data = lab6_data)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -0.47003 -0.07883 -0.03949  0.06384  0.48974 
    ## 
    ## Coefficients:
    ##              Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept) 2.552e-01  5.013e-03  50.902   <2e-16 ***
    ## pop_20      1.500e-06  1.500e-07   9.998   <2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1316 on 714 degrees of freedom
    ## Multiple R-squared:  0.1228, Adjusted R-squared:  0.1216 
    ## F-statistic: 99.97 on 1 and 714 DF,  p-value: < 2.2e-16

``` r
modwater20 <- lm(ndvi_20_med ~ water_20_pct,data=lab6_data)
summary(modwater20)
```

    ## 
    ## Call:
    ## lm(formula = ndvi_20_med ~ water_20_pct, data = lab6_data)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -0.26898 -0.08838 -0.04838  0.06871  0.50911 
    ## 
    ## Coefficients:
    ##               Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)   0.268988   0.006287  42.781   <2e-16 ***
    ## water_20_pct -0.178263   0.154480  -1.154    0.249    
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1403 on 714 degrees of freedom
    ## Multiple R-squared:  0.001862,   Adjusted R-squared:  0.0004636 
    ## F-statistic: 1.332 on 1 and 714 DF,  p-value: 0.2489

``` r
modelev20 <- lm(ndvi_20_med ~ elev_med,data=lab6_data)
summary(modelev20)
```

    ## 
    ## Call:
    ## lm(formula = ndvi_20_med ~ elev_med, data = lab6_data)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -0.27082 -0.09585 -0.04270  0.07954  0.44272 
    ## 
    ## Coefficients:
    ##              Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept) 2.138e-01  9.741e-03  21.952  < 2e-16 ***
    ## elev_med    1.787e-04  2.895e-05   6.171 1.14e-09 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1369 on 714 degrees of freedom
    ## Multiple R-squared:  0.05064,    Adjusted R-squared:  0.04931 
    ## F-statistic: 38.08 on 1 and 714 DF,  p-value: 1.136e-09

As median max temperature increases by 1 degree Celsius, NDVI decreases
by -0.013. This is a moderate and significant change (p=0.00). Likewise,
NDVI decreases by 0.012 for every 1 degree C increase in median min
temperature (p=0.00).

Total yearly rainfall has a small yet significant effect on NDVI
(p=0.00). For every 1000 cm (10000 mm) increase in rainfall, NDVI
increases by an estimated 0.00912.  
NOTE: I looked at the data to get a sense of the range of rainfall
values, and noticed they were huge. I think something is wonky there,
but for the purposes of this interpretation I’m going with it.

Population is a significant predictor of NDVI (p=0.00), but the slope is
relatively small. As population increases by 1,000, NDVI increases by
0.0015.

The univariate water model estimates that for every 1% increase in
percent water coverage, NVDI decreases by 0.178. This would be a large
change in NDVI, but this is insignificant; the observed trend could be
due to chance (p=0.249).

Lastly, elevation is also a significant predictor of NDVI value
(p=1.14e-09). For every 100 m increase in elevation, the NDVI increases
by 0.018.

**Question 5** *Create a multivariate regression model with the
variables of interest, choosing EITHER max or min temperature (but not
both) You may also choose to leave out any variables that were
insignificant in Q4. Use the univariate models as your guide. Call the
results.*

``` r
multimod <- lm(ndvi_20_med ~ rain_20_sum + mintemp_20_med + elev_med + pop_20, data=lab6_data)
summary(multimod)
```

    ## 
    ## Call:
    ## lm(formula = ndvi_20_med ~ rain_20_sum + mintemp_20_med + elev_med + 
    ##     pop_20, data = lab6_data)
    ## 
    ## Residuals:
    ##      Min       1Q   Median       3Q      Max 
    ## -0.49488 -0.02773  0.00412  0.03928  0.25115 
    ## 
    ## Coefficients:
    ##                  Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)     3.208e-01  1.413e-02  22.707  < 2e-16 ***
    ## rain_20_sum     9.420e-07  3.276e-08  28.756  < 2e-16 ***
    ## mintemp_20_med -1.391e-02  7.675e-04 -18.121  < 2e-16 ***
    ## elev_med        1.028e-04  1.774e-05   5.797 1.02e-08 ***
    ## pop_20          2.424e-07  1.032e-07   2.349   0.0191 *  
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.0832 on 711 degrees of freedom
    ## Multiple R-squared:  0.6507, Adjusted R-squared:  0.6487 
    ## F-statistic: 331.1 on 4 and 711 DF,  p-value: < 2.2e-16

**Question 6** *Summarize the results of the multivariate model. What
are the direction, magnitude, and significance of each coefficient? How
did it change from the univariate models you created in Q4 (if at all)?
What do the R2 and F-statistic values tell you about overall model fit?*

Yearly rainfall, median minimum temperature, median elevation, and
population are all significant predictors of NDVI value (p\<0.05). This
model estimates that for every 10,000 mm increase in rainfall, NDVI
increases by 0.0094. For every 1 degree C increase in median min
temperature, NDVI decreases by 0.014. For every 100 m increase in
elevation, NDVI increases by 0.01. Lastly, for every 1000 person
increase in population, NDVI increases by just 0.0002.

For rainfall, median min temperature, and elevation, there was only a
slight change in the coefficients compared to their respective
univariate models. For population, however, the slope became much
smaller (0.0002/1000 people compared to 0.0015/1000 people before).

For this multivariate model, the R squared is 0.6507. This suggests this
model is a good fit as it accounts for over 65% of the variation seen in
the dataset. The F-statistic is effectively 0, so we reject the null
hypothesis that this model performs the same as a null model.

**Question 7** *Use a histogram and a map to assess the normality of
residuals and any spatial autocorrelation. Summarise any notable
patterns that you see.*

``` r
lab6_data$residuals<-residuals(multimod) #Pull the residuals from the model and add them to the dataset
hist(lab6_data$residuals)
```

![](Lab-6_regression_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->

``` r
tmap_mode("plot")
```

    ## tmap mode set to plotting

``` r
tm_shape(lab6_data) +
  tm_polygons("residuals")
```

    ## Variable(s) "residuals" contains positive and negative values, so midpoint is set to 0. Set midpoint = NA to show the full spectrum of the color palette.

![](Lab-6_regression_files/figure-gfm/unnamed-chunk-7-2.png)<!-- -->

``` r
residuals <- tm_shape(lab6_data) +
  tm_polygons("residuals")+
  tm_layout(title="Residual Error",title.position=c("left", "bottom"), legend.show=FALSE)

# plotting residuals alongside other variables to look for patterns
tmap_arrange(ndvi20, maxt20, mint20, rain20, pop20, water20, elev, residuals)
```

    ## Variable(s) "elev_med" contains positive and negative values, so midpoint is set to 0. Set midpoint = NA to show the full spectrum of the color palette.

    ## Variable(s) "residuals" contains positive and negative values, so midpoint is set to 0. Set midpoint = NA to show the full spectrum of the color palette.

    ## Variable(s) "elev_med" contains positive and negative values, so midpoint is set to 0. Set midpoint = NA to show the full spectrum of the color palette.

    ## Variable(s) "residuals" contains positive and negative values, so midpoint is set to 0. Set midpoint = NA to show the full spectrum of the color palette.

![](Lab-6_regression_files/figure-gfm/unnamed-chunk-7-3.png)<!-- -->

The histogram of the residuals shows a cluster of large negative
residual values, causing the distribution to be skewed to the left.

Interestingly, all of the cells with the largest residuals (greater than
0.2 or less than -0.2) are located immediately along the coast or just a
few cells inland.

**Question 8** *Assess any issues with multicollinearity or
heteroskedastity in this model using the techniques shown in class. Run
the appropriate tests and explain what their results show you.*

``` r
# multicollinearity
vif(multimod)
```

    ##    rain_20_sum mintemp_20_med       elev_med         pop_20 
    ##       1.121045       1.127167       1.015657       1.183240

``` r
# heteroskedasticity
bptest(multimod)
```

    ## 
    ##  studentized Breusch-Pagan test
    ## 
    ## data:  multimod
    ## BP = 120.07, df = 4, p-value < 2.2e-16

``` r
# plotting residuals vs fitted values to visualize heteroskedasticity
ggplot(multimod, aes(x= .fitted, y= .resid)) +
  geom_point(aes(color=ndvi_20_med)) +
  geom_hline(yintercept= 0)
```

![](Lab-6_regression_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

The `vif()` function returned values \<5 for each variable, which
suggests there are not any issues with multicollinearity.

The `bptest()` function reurned a p value of 0.00, so we reject the null
hypothesis that the data are not heteroskedastic. Based on the plot (see
my disclosure of assistance for why I included this), variance in the
residuals is greater at lower predicted values.

**Question 9** *How would you summarise the results of this model in a
sentence or two? In addition, looking at the full model and your
diagnostics, do you feel this is a model that provides meaningful
results? Explain your answer.*

Overall, I think the model does a good job of explaining the variation
in NDVI factors across Australia. Based in the location of the large
residuals and the heteroskedasticity plot, there is likely some factor
related to being near the coast that effects NDVI that is not accounted
for in this model.

**Disclosure of assistance:** *Besides class materials, what other
sources of assistance did you use while completing this lab? These can
include input from classmates, relevant material identified through web
searches (e.g., Stack Overflow), or assistance from ChatGPT or other AI
tools. How did these sources support your own learning in completing
this lab?*

I referenced documentation for the `tmap` package to format the map
panels. I spoke with Dr. Shannon in class, which helped me better
interpret model results and understand the NDVI values.

In another course, FANR 6750, we used the residual vs. fitted values
plots to visualize heteroskedasticity. I find it helpful to see where
the variance differs in the model.

**Lab reflection:** *How do you feel about the work you did on this lab?
Was it easy, moderate, or hard? What were the biggest things you learned
by completing it?*

I think this was the most involved lab we have had so far, but I enjoyed
it. I struggled to format using `tmap_arrange()`; in earnest I probably
spent more time trying to get that to look nice than I did on any other
question. Summarizing model results, especially the results of multiple
models, can be tedious and repetitive. I think this lab helped me to be
more succinct and find ways to explain the results in a way that has
better context (i.e. explaining the change for 1000 people vs 1 person).

**Challenge question**

# Option 1

Create a first difference model. To do that, subtract the values in 2000
from the values in 2020 for each variable for which that is appropriate.
Then create a new model similar to the one you created in question 5,
but using these new variables showing the *change in values* over time.
Call the results of the model, and interpret the results in the same
ways you did above. Also chart and map the residuals to assess model
error. Finally, write a short section that summarises what, if anything,
this model tells you.

# Option 2

The animal data included in this dataset is an example of count data,
and usually we would use a Poisson or similar model for that purpose.
Let’s try it with regular OLS regression though. Create two regression
models to assess how the counts of two different animals (say, koalas
and emus) are associated with at least three of the
environmental/climatological variables given above. Be sure to use the
same independent variables in each model. Interpret the results of each
model and then explain the importance of any differences in the model
coefficients between them, focusing on direction, magnitude, and
significance.
