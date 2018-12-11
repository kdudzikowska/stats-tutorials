---
title: "Linear Mixed Effects"
author: "LT"
output:
  html_document:
    toc: TRUE
    toc_float: TRUE
    df_print: "paged"
    keep_md: TRUE
---


```r
library(ggplot2)
```

# Hierarchical structure

We will often have data that are organized in a hierarchical fashion. For example, we may have a group of people, and then from each person we have multiple measurements of their behavior. Or we may have several groups, such as companies or school classes, and have data on multiple people within each group.

Let's take a look at some data of this kind. These data come from the [National Longitudinal Study of Adolescent to Adult Health (Add Health)](https://www.icpsr.umich.edu/icpsrweb/content/DSDR/add-health-data-guide.html), conducted in the USA from 1994 to 2008. Over this time period, students in schools answered various questions about their health.


```r
ah = read.csv('add_health.csv')

nrow(ah)
```

```
## [1] 4344
```

```r
head(ah)
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":[""],"name":["_rn_"],"type":[""],"align":["left"]},{"label":["Depression"],"name":[1],"type":["int"],"align":["right"]},{"label":["Anxiety"],"name":[2],"type":["int"],"align":["right"]},{"label":["Grade"],"name":[3],"type":["fctr"],"align":["left"]}],"data":[{"1":"1","2":"1","3":"11th","_rn_":"1"},{"1":"1","2":"1","3":"10th","_rn_":"2"},{"1":"1","2":"1","3":"12th","_rn_":"3"},{"1":"1","2":"2","3":"7th","_rn_":"4"},{"1":"1","2":"2","3":"8th","_rn_":"5"},{"1":"1","2":"1","3":"8th","_rn_":"6"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

The full data set is massive. Here I have stripped it down to just three variables:

* Grade: Which grade the student was in
* Anxiety: Self-reported anxiety on a scale from 1 to 5
* Depression: Self-reported depression on a scale from 1 to 5

Since the students are grouped in grades, we have a hierarchical structure.


```r
table(ah$Grade)
```

```
## 
## 10th 11th 12th  7th  8th  9th 
##  817  790  673  622  664  778
```

Let's imagine we are interested in estimating the linear relationship between anxiety and depression in the population of students in the USA, in different grades, using these data.

We should plot the data first. Since there are only a few possible values on the anxiety and depression scales (1, 2, 3, 4, and 5), we should jitter the points in order to avoid multiple points being plotted exactly on top of one another and appearing as only one point. Using a filled circle rather than a solid dot can also help distinguish individual data points when we have many that overlap.


```r
fig1 = ggplot(ah, aes(y=Depression, x=Anxiety)) +
  geom_point(shape='circle filled', fill='grey', position=position_jitter())

fig2 = fig1 + facet_wrap(~Grade, labeller=label_both)

print(fig2)
```

![](LME_files/figure-html/unnamed-chunk-4-1.png)<!-- -->

## Ignoring hierarchical structure

If we want to estimate a linear model using such data, we could just ignore their hierarchical structure. There are broadly two different ways of doing this.

### Lumping

We could throw all the students together as if they were all independent observations of the same phenomenon, and then estimate one model using all the data together.


```r
print(fig1 + geom_smooth(method=lm, se=FALSE))
```

![](LME_files/figure-html/unnamed-chunk-5-1.png)<!-- -->

```r
lump_model = lm(Depression ~ Anxiety, ah)
print(lump_model)
```

```
## 
## Call:
## lm(formula = Depression ~ Anxiety, data = ah)
## 
## Coefficients:
## (Intercept)      Anxiety  
##      1.0483       0.5883
```

However, doing this violates one of the central assumptions of many inferential procedures: Independent observations. Students who are in the same grade are probably slightly more similar to each other than to students in other grades, since they are exposed to some of the same influences. This means that each student within a grade might not be giving us a completely new piece of information about the anxiety-depression relationship that the other students in that grade did not already give us.

In the best case, if this simplifying assumption happens to be true, and the students within a given grade are no more similar to each other than students in general, then lumping is fine. But in the worst case, if students within each grade are in fact just providing us with the same piece of information over and over again, then our true sample size is close to the number of different grades, and not the total number of students.

If the students within each grade were giving us completely redundant information, then the relevant sample size for the hypothesis test would be the number of grades, not the number of students. Inferences that take sample size into account will make it appear that we have more certainty about the true relationship than we really do. For example, in a *t*-test of the regression coefficient the *t*-value and its degrees of freedom will be larger than they should be.


```r
summary(lump_model)
```

```
## 
## Call:
## lm(formula = Depression ~ Anxiety, data = ah)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -2.9898 -0.6366 -0.2249  0.5985  3.3634 
## 
## Coefficients:
##             Estimate Std. Error t value Pr(>|t|)    
## (Intercept)  1.04834    0.03082   34.01   <2e-16 ***
## Anxiety      0.58829    0.01376   42.77   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 1 on 4342 degrees of freedom
## Multiple R-squared:  0.2964,	Adjusted R-squared:  0.2962 
## F-statistic:  1829 on 1 and 4342 DF,  p-value: < 2.2e-16
```

But lumping can sometimes result in a more serious problem for inference. Consider the data below.


```r
d = data.frame(x=1:8, y=c(5:8,1:4), Group=rep(c('A','B'), each=4))

fig = ggplot(d, aes(x=x, y=y)) +
  geom_smooth(aes(color=Group), method=lm, se=FALSE) +
  geom_smooth(method=lm, se=FALSE) +
  geom_point()

print(fig)
```

![](LME_files/figure-html/unnamed-chunk-7-1.png)<!-- -->

If we have groups that share a common trend but who differ from one another a lot on average, then a lumped analysis may show a completely opposite trend to that which is present in each group. This is because the large average differences between groups influence the overall outcome more than does the trend within each group.

This phenomenon, and similar reversals that occur when lumping data together, is often termed [Simpson's paradox](https://en.wikipedia.org/wiki/Simpson%27s_paradox).

If we are interested in a trend that the groups may have in common, then lumping might not be appropriate. (On the other hand, if we are interested in the trend that governs group averages, then lumping would be giving us the answer we are looking for. Although we might want to first calculate the group averages and work with those, so as to avoid the problem of inflating sample size noted above.)

### Parameter averaging

At the other end of the spectrum, we could ignore hierarchical structure in a different way. We could estimate our model completely separately for each group. Our estimates for each group would be telling us about the trend just for that group, without generalization to the others, and if we also wanted an estimate of the average trend, we could just look at the average parameter values among the separate group models.

In the example for Simpson's paradox above, this approach would be considering the two separate lines for the two groups. For the Add Health data, it would be considering the linear relationship of depression to anxiety separately within each grade.


```r
print(fig2 + geom_smooth(method=lm, se=FALSE))
```

![](LME_files/figure-html/unnamed-chunk-8-1.png)<!-- -->

In the statistical analysis, we would calculate our linear model separately for each grade. There are various ways of doing this in R. Since it is usually convenient to organize data in a data frame, here we create a data frame that can hold the estimated slopes and intercepts for each grade, and then fill it up in a loop.


```r
average_model = data.frame(Grade=levels(ah$Grade), Intercept=0, Anxiety=0)

for(grade in levels(ah$Grade)){
  grade_model = lm(Depression ~ Anxiety, subset(ah, Grade==grade))
  average_model[average_model$Grade==grade, c('Intercept','Anxiety')] = coefficients(grade_model)
}

average_model
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["Grade"],"name":[1],"type":["fctr"],"align":["left"]},{"label":["Intercept"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Anxiety"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"10th","2":"1.0130294","3":"0.6429005"},{"1":"11th","2":"1.1067391","3":"0.5952580"},{"1":"12th","2":"1.2301565","3":"0.5287773"},{"1":"7th","2":"1.0482899","3":"0.4756324"},{"1":"8th","2":"0.9459732","3":"0.6259380"},{"1":"9th","2":"1.0536091","3":"0.5795961"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

And to ask about the overall average model, we could just perform analyses on the group-level parameters. For example, to get a confidence interval for the average slope.


```r
t.test(average_model$Anxiety)
```

```
## 
## 	One Sample t-test
## 
## data:  average_model$Anxiety
## t = 22.457, df = 5, p-value = 3.254e-06
## alternative hypothesis: true mean is not equal to 0
## 95 percent confidence interval:
##  0.5089007 0.6404667
## sample estimates:
## mean of x 
## 0.5746837
```

One advantage of parameter averaging is that we won't confuse the trend that governs group averages with the trend within each group. Another is that we are less likely to overestimate the amount of information our data give us by assuming that we have more independent observations than we really do.

One disadvantage is greater uncertainty. By ignoring the fact that we have multiple observations within each group as well as just their averages we are ignoring some of the information we have, and will be more uncertain about our estimates as a result.

# Mixed effects

Comparing parameter averaging to lumping, we can see that we make opposite assumptions in each case. Lumping assumes that the modelled phenomenon is exactly the same across all groups, so we might as well ignore the groupings. Parameter averaging assumes that the phenomenon is completely unrelated across groups, so we have to estimate them completely separately, and only average our estimates at the end.

In reality, hierarchical data are likely to have been generated by an intermediate process. In the Add Health data, there are probably some unique influences on the students in each grade, for example the materials they study, the fact that they are all of the same age, and so on. But there are probably also some things that influence students in the same way across all grades. To acknowledge both of these types of influence, we need a model that accounts for the hierarchical structure of the data by estimating an overall trend, and some structured variation on that trend among the groups.

Such models are often termed 'mixed effects' models, because of the two different kinds of effect they estimate. The **fixed effects** are the overall trends that we are assuming all groups have in common. The **random effects** are the trends within each group. The key feature of a mixed effects model is that it does not allow the estimated random effects the freedom to take on any values (as parameter averaging does). Instead, the random effects are assumed to be drawn from the same underlying distribution, almost always a normal distribution.

An alternative way of thinking about this is in terms of the information that the model uses. A mixed effects model does not estimate the trend within a group using only that group's data. Instead, the estimates within one group are at least slightly informed by the data from the other groups. This can make our estimates for each group more accurate by 'filling in' some of the uncertainty in the group's estimates with the common trend shared by all groups.

## Estimation

A mixed effects model has several components that need to be estimated from the data:

* The fixed effects
* The random effects for each group
* The variances of the underlying distributions from which the random effects are drawn
* Additionally (though optionally), the covariance between random effects (i.e. the degree to which groups with a higher value for one random effect tend to have a higher or lower value for another random effect)

Estimating all of these things subject to the constraints of the model's hierarchical form is not always easy. There is no general analytic solution, so we have to apply an iterative estimation procedure. We will consider below a few of the details of the fitting procedure, but a detailed description is beyond the scope of this class.

The 'lme4' package can estimate mixed effects models. The name stands for *l*inear *m*ixed *e*ffects.


```r
library(lme4)
```

The main function that lme4 provides is called `lmer()` (*l*inear *m*ixed *e*ffects *r*egression). This function works similarly to the `lm()` function and other model functions in that it takes a formula as its main input. The formula specifies the model. The first part of the formula is the same as for `lm()`. We put the outcome variable, then `~`, then one or more predictors. These predictors specify the fixed effects, i.e. the model structure that we assume all groups have in common. The random effects are added after the fixed effects, in the form `(predictors|group)`. This specifies the predictors that we want also to consider as random effects. Often, though not always, these will be the same as the fixed effects (i.e. we will want to estimate a given model for each group, and then also get an estimate of the overall underlying model).

Let's apply this to the Add Health data and estimate a hierarchical model of Depression with a fixed effect of Anxiety and random effects of Anxiety in each Grade.


```r
model = lmer(Depression ~ Anxiety + (Anxiety|Grade), ah)
```

```
## singular fit
```

We are told here that we get a 'singular fit'. In this context, a 'singularity' is a situation in which one of the estimated parameters has taken on a value that is at the extreme of its range. For example, the variance of one of the random effects has been estimated as zero, or the correlation between two random effects has been estimated as either -1 or 1. This situation is difficult to deal with in statistical analysis, and is in any case unlikely to represent a realistic estimate for real-world data.

We can check a summary of the model to see which parameter is responsible for the singularity. In this case, we see that it is the variance in the random intercepts that has been estimated as zero. And of course if the variance in the random intercepts is zero, then the covariance of the intercept with the slope is undefined, which is why we also see `NaN` (*N*ot *a* *N*umber) for the correlation of random intercept with random slope.


```r
summary(model)
```

```
## Linear mixed model fit by REML ['lmerMod']
## Formula: Depression ~ Anxiety + (Anxiety | Grade)
##    Data: ah
## 
## REML criterion at convergence: 12312
## 
## Scaled residuals: 
##     Min      1Q  Median      3Q     Max 
## -3.1217 -0.6538 -0.2429  0.6282  3.4711 
## 
## Random effects:
##  Groups   Name        Variance Std.Dev. Corr
##  Grade    (Intercept) 0.000000 0.00000      
##           Anxiety     0.002607 0.05106   NaN
##  Residual             0.991124 0.99555      
## Number of obs: 4344, groups:  Grade, 6
## 
## Fixed effects:
##             Estimate Std. Error t value
## (Intercept)  1.06004    0.03079   34.43
## Anxiety      0.57685    0.02504   23.04
## 
## Correlation of Fixed Effects:
##         (Intr)
## Anxiety -0.482
## convergence code: 0
## singular fit
```

There are different reasons why singularity (or other technical problems with estimating the model) may occur. Two common problems are:

* The structure of the random effects is too complex. Remember that we have to estimate the variance of each random effect, and by default will also estimate the covariance between each pair of random effects. This can quickly become too much to estimate from the amount of data we have.
* We have too few groups. It is difficult to estimate the variance in random effects across groups if we only have a few of them.

However, neither of these seems likely to be the case for the Add Health data. We have only two random effects (an intercept and a slope), so there is not a huge number of parameters to be estimated for the random effects structure. And we have six grades, which isn't so many but should be enough for estimating this simple model.

Some difficulties in fitting a mixed effects model can arise instead for purely technical reasons. In searching for a solution, the model fitting algorithm needs to calculate changes in how well the model fits when the parameter values change (a more detailed exploration of this process is the topic of another class). These calculations are influenced by the scale of our predictor variables. If any very large or very small numbers occur during the fitting process, or if our predictors are on very different scales, then the model fit may be inaccurate. This can occur because the improvements in the model fit at each step become very small even before the best fitting vaues are reached, causing the model fitting procedure to 'think' it has already found the best fitting values when it has not, or it can simply result in our computer running out of the precision necessary to represent very large or very small numbers, resulting in inaccurate calculations.

(We can verify that computers have problems with very large or very small numbers by trying out a few extreme calculations.)


```r
10 ^ 1000
```

```
## [1] Inf
```

```r
10 ^ -1000
```

```
## [1] 0
```

The solution to this sort of problem is to balance out the scale of our predictor variables. If we center predictor variables on zero and give them a standard deviation of 1, then we will tend to avoid getting extremely large or extremely small numbers during the calculations involved in the fitting procedure.

The `scale()` function in R performs this kind of scaling.


```r
some_numbers = 1:5
scale(some_numbers)
```

```
##            [,1]
## [1,] -1.2649111
## [2,] -0.6324555
## [3,]  0.0000000
## [4,]  0.6324555
## [5,]  1.2649111
## attr(,"scaled:center")
## [1] 3
## attr(,"scaled:scale")
## [1] 1.581139
```

We can apply `scale()` within a model formula. This avoids the inconvenience of having to scale the variables in our original data set.


```r
model = lmer(Depression ~ scale(Anxiety) + (scale(Anxiety)|Grade), ah)
summary(model)
```

```
## Linear mixed model fit by REML ['lmerMod']
## Formula: Depression ~ scale(Anxiety) + (scale(Anxiety) | Grade)
##    Data: ah
## 
## REML criterion at convergence: 12311.4
## 
## Scaled residuals: 
##     Min      1Q  Median      3Q     Max 
## -3.1074 -0.6793 -0.2566  0.6127  3.4837 
## 
## Random effects:
##  Groups   Name           Variance Std.Dev. Corr
##  Grade    (Intercept)    0.010582 0.10287      
##           scale(Anxiety) 0.003007 0.05484  0.88
##  Residual                0.990667 0.99532      
## Number of obs: 4344, groups:  Grade, 6
## 
## Fixed effects:
##                Estimate Std. Error t value
## (Intercept)     2.18606    0.04468   48.93
## scale(Anxiety)  0.63632    0.02712   23.46
## 
## Correlation of Fixed Effects:
##             (Intr)
## scal(Anxty) 0.688
```

Now we no longer get a warning about singularity, and none of the estimated parameters has an extreme value. We also get an estimate for the correlation between random slopes and intercepts.

We pay a small price for scaling. Since the scaling changes the values of the predictor, any estimates from the model must be interpreted according to this transformed scale. In the example above, since we fixed the Anxiety values to be centred on 0, the fixed effect intercept represents the expected value of Depression at mean Anxiety. And since we fixed the Standard Deviation of Anxiety to be 1, the fixed effect slope for Anxiety represents the expected change in Depression when Anxiety increases by 1 Standard Deviation. This is not a big problem and will not change our qualitative conclusions, it is just something we have to be aware of when interpreting the estimated values.

## Uncorrelated random effects

We saw above that by default `lmer()` assumes that the random effects may be correlated with one another, for example that groups with higher values for one of the random effects may also tend to have higher or lower values for the other random effects.

In the case of the Add Health data, the model produced by `lmer()` estimates a strong positive correlation between the random intercepts and random slopes. So in grades where all students tend to have higher depression there will also tend to be a steeper increase in depression with increasing anxiety. But correlations of random effects are difficult to estimate, and are usually very uncertain. So we should interpret them with caution.

If we want to simplify our model in order to make it feasible to estimate, or if we perhaps have a good theoretical reason to think that the random effects should be uncorrelated with one another, we can force this assumption on the model.

To estimate uncorrelated random effects for a grouping variable, we put `||` instead of `|` in the random effects part of the formula.


```r
model_uncorrelated = lmer(Depression ~ scale(Anxiety) + (scale(Anxiety)||Grade), ah)
summary(model_uncorrelated)
```

```
## Linear mixed model fit by REML ['lmerMod']
## Formula: 
## Depression ~ scale(Anxiety) + ((1 | Grade) + (0 + scale(Anxiety) |  
##     Grade))
##    Data: ah
## 
## REML criterion at convergence: 12314.4
## 
## Scaled residuals: 
##     Min      1Q  Median      3Q     Max 
## -3.0765 -0.6693 -0.2786  0.6335  3.4968 
## 
## Random effects:
##  Groups   Name           Variance Std.Dev.
##  Grade    (Intercept)    0.010349 0.10173 
##  Grade.1  scale(Anxiety) 0.002853 0.05342 
##  Residual                0.990668 0.99532 
## Number of obs: 4344, groups:  Grade, 6
## 
## Fixed effects:
##                Estimate Std. Error t value
## (Intercept)     2.18866    0.04424   49.47
## scale(Anxiety)  0.63728    0.02665   23.91
## 
## Correlation of Fixed Effects:
##             (Intr)
## scal(Anxty) 0.003
```

We see in this case that constraining the random effects to be uncorrelated does not make much difference to the fit of the model or to the estimates of the fixed effects.

For most natural phenomena, it is probably not a very realistic assumption that random effects are completely uncorrelated, but it may sometimes be a justifiable simplification.

## Visualization

We should always check our model visually. To do this, we need some way of plotting the model along with the data.

Often the easiest way of achieving this is to make use of the model's predicted values of the outcome variable. We can then add these predicted values to the plot, for example joining them up with a line if we have a continuous predictor.

The first step is to add the predicted values to the original data frame. Then we should update our plot with this new version of the data frame.


```r
ah$Predicted = predict(model)

fig2 = fig2 %+% ah
```

We can then use this new column of the data frame for plotting. We plot the predicted values on the same plot dimension as the outcome variable.


```r
fig3 = fig2 +
  geom_line(aes(y=Predicted))

print(fig3)
```

![](LME_files/figure-html/unnamed-chunk-19-1.png)<!-- -->

If we are exploring some different random effects structures, for example uncorrelated random effects, then we can also visualize the predictions of these alternative models to check whether they make a large difference to our conclusions.

Here we see for example that the predictions of the uncorrelated random effects model barely differ from those of the model that estimates a correlation between the two random effects.


```r
ah$Predicted_uncorrelated = predict(model_uncorrelated)

print(fig3 %+% ah + geom_line(aes(y=Predicted_uncorrelated), lty='dashed'))
```

![](LME_files/figure-html/unnamed-chunk-20-1.png)<!-- -->

A model that includes only a random intercept (intercepts are given as `1` in R formulas) differs a bit more. This model constrains the slope to be the same for every group.


```r
model_intercept_only = lmer(Depression ~ scale(Anxiety) + (1|Grade), ah)

ah$Predicted_intercept_only = predict(model_intercept_only)

print(fig3 %+% ah + geom_line(aes(y=Predicted_intercept_only), lty='dashed'))
```

![](LME_files/figure-html/unnamed-chunk-21-1.png)<!-- -->

## Shrinkage

When we visualized the form of the estimated mixed effects model, we got a slightly different line for each group within the data. This seems very similar to what we get when we just estimate the model completely separately for each group. So what is the difference between the mixed effects model and separate models for each group? And more importantly, what advantage is there to be gained by using a mixed effects model?

To get an idea of the crucial difference, let's first get the predicted values of depression from a model that assumes no hierarchical structure. A standard linear regression model that includes an interaction of all predictors with the grouping variable will achieve this, since it will use the interactions to account completely for differences in the form of the model between groups.


```r
model_separate = lm(Depression ~ Anxiety*Grade, ah)
```

Now let's compare the predictions of the non-hierarchical model with the predictions based on the fixed and random effects from the mixed effecs model. We show the fixed effects in red, the random effects in black, and the non-hierarchical model in blue.

We can get the predicted values from a mixed effects model using `predict()`, just as for any other model object. By default this gives the predicted values based on all parts of the model, both fixed and random effects. If we want the values based only on the fixed effects (so that the values will be the same for all groups) we can give `predict()` the random effects formula `~0`, which means 'use nothing'.


```r
ah$Predicted_fixed = predict(model, re.form=~0)
ah$Predicted_separate = predict(model_separate)

fig4 = fig2 %+% ah +
  geom_line(aes(y=Predicted)) +
  geom_line(aes(y=Predicted_separate), color='blue', lty='dashed') +
  geom_line(aes(y=Predicted_fixed), color='red', lty='dashed')

print(fig4)
```

![](LME_files/figure-html/unnamed-chunk-23-1.png)<!-- -->

The difference between the mixed effects model and the non-hierarchical model is clearest for the 7th-grade students. Here the predictions of the mixed effects model for this grade (in black) are pulled slightly away from the separate model for that grade only (in blue) and towards the estimated overall model for all grades (in red).

In a hierarchical model, the estimates of the model within each group will tend to 'shrink' back towards the shared model. This is the result of the hierarchical model using some of the information from all groups to inform its estimates in each group.

The amount of shrinkage will depend in part on how much information is available to inform the estimates within each group. We can illustrate this by re-fitting the mixed effects and non-hierarchical models using a much smaller subset of the data.


```r
n_small = 100
ah_small = ah[1:n_small,]

model_small = update(model, data=ah_small)
model_separate_small = update(model_separate, data=ah_small)

ah_small$Predicted = predict(model_small)
ah_small$Predicted_fixed = predict(model_small, re.form=~0)
ah_small$Predicted_separate = predict(model_separate_small)

print(fig4 %+% ah_small)
```

![](LME_files/figure-html/unnamed-chunk-24-1.png)<!-- -->

Now, because there is much less information on which to base separate estimates of the model in each group, the mixed effects model fills in this missing information using the overall estimates (i.e. the fixed effects). So the random effects estimates within each group 'shrink' more towards the fixed effects.

Why might this be a good property for a model?

For many real-world phenomena, the behavior of an individual person is only partly systematic. If we observe behavior at one moment in time, we can think of that behavior as being the sum of three components:

* The peculiar but systematic traits of that person
* Traits the person has in common with other people
* Random fluctuation

The next time we observe that same person, the random fluctuation part will be different, and only the two other components will be the same. So ideally we would like to leave the random fluctuations out of our predictions. The problem is that if we look only at that person now, we have no easy way of telling apart the random fluctuations from the systematic idiosyncracies. The behavior of other people can help us. Although it too is contaminated by random fluctuations, they are not the same random fluctuations as those that contaminate the observed behavior of one individual. So the average behavior of other people gives us some extra information about which part of any one person's behavior is the systematic part.

An oft-cited example of this principle at work is in predicting the future performance of athletes. If we base our predictions of an athlete's performance only on their own past performance, we will do slightly worse than if we also adjust our predictions slightly towards the average performance of other athletes.

We can demonstrate this boost in prediction for the Add Health data. Imagine we only had access to the smaller subset of data that we just used above, and we now want to use the models that we fitted to this data set to predict the depression scores of new students.

We can use the remaining rows of the data to assess the predictions of the mixed effects and non-hierarchical models. (This assessment is not particularly systematic, but just for the purposes of demonstration.)


```r
ah_test = ah[(n_small+1):nrow(ah),]

errors = predict(model_small, newdata=ah_test) - ah_test$Depression
errors_separate = predict(model_separate_small, newdata=ah_test) - ah_test$Depression

ah_cv = data.frame(Error=c(errors, errors_separate),
                   Model=rep(c('Hierarchical','Unstructured'), each=nrow(ah_test)))

fig5 = ggplot(ah_cv, aes(y=Error, x=Model)) +
  geom_boxplot()

print(fig5)
```

![](LME_files/figure-html/unnamed-chunk-25-1.png)<!-- -->

```r
rms = function(x){
  return(sqrt(mean(x^2)))
}

aggregate(Error ~ Model, ah_cv, FUN=rms)
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["Model"],"name":[1],"type":["fctr"],"align":["left"]},{"label":["Error"],"name":[2],"type":["dbl"],"align":["right"]}],"data":[{"1":"Hierarchical","2":"1.033712"},{"1":"Unstructured","2":"1.092615"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

The model without hierarchical structure tends to make larger errors in predicting new data.

## Restricted and full maximum likelihood

As we have seen, in a mixed effects model the random effects can be thought of as the result of some structured random variation around the fixed effects. This means that in order to estimate correctly the variance of the random effects, we require already to know what the fixed effects are. And of course we don't know this with certainty. Likewise, our estimates of the random effects inform our best guess as to what the fixed effects are. This leads to a trade-off between estimation of fixed and random effects. There are two common approaches:

**Maximum Likelihood** (ML). Focus on getting the best estimates of the fixed effects (the values that make the observed data maximally probable), then make the simplifying assumption that these estimates are exactly right, so we can estimate the variance of the random effects around them. This results in unbiased estimates of the fixed effects, but tends to underestimate the variance in the random effects (i.e. groups or individuals will tend in reality to be more heterogeneous than our model estimates).

**Restricted Maximum Likelihood** (REML). Attempt to correct for the uncertainty in the estimates of the fixed effects when estimating the variance in the random effects. Very roughly speaking, this is achieved by transforming the data so that they become independent of the fixed effects. This allows us to estimate the variance in random effects without bias. However, the transformation necessary to achieve this is somewhat arbitrary and is special to the fixed effects of one particular model. This means that statistical assessments of a REML-fitted model are not comparable to those of another model with different fixed effects. If we attempt to compare two different models fitted by REML, we won't get meaningful answers.

Which to use? There are differences of opinion, but we should think about what we want our model for.

If we are estimating just one model, and we want to use that model to say something about the differences among groups, then we may prefer REML, because this gives us unbiased estimates of the variance among groups, and better predictions of group-specific behavior. This was the case in the example above, where we simulated trying to predict the depression scores of students in a particular grade.

And we used REML in the above example, without asking for it explicitly. REML is the default for `lmer()`. We can see this mentioned at the top of the output from `summary()`.

However, it is more common in our line of work that we are interested in generalizations independent of groups or individuals, and that we want to compare two or more models that represent theories about those generalizations. In this case we will need to compare models with different fixed effects, and so we must use ML.

Let's try comparing two mixed effects models using the general model comparison function `anova()`. We compare our mixed effects model of depression and anxiety to a 'null' model that posits no overall relationship between the two. A null model typically includes just an intercept, which is indicated as a `1` in the model formula.

To ensure that the comparison is a fair one, we leave the random effects the same for the two models, so that we are comparing just the inclusion of the fixed effect of Anxiety. (Of course this results in a null model that is itself not very realistic on theoretical grounds: It posits varying consistent effects of Anxiety within grades, but asserts that these effects just happen to cancel out at zero overall.)


```r
model_null = lmer(Depression ~ 1 + (scale(Anxiety)|Grade), ah)

anova(model_null, model)
```

```
## refitting model(s) with ML (instead of REML)
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":[""],"name":["_rn_"],"type":[""],"align":["left"]},{"label":["Df"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["AIC"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["BIC"],"name":[3],"type":["dbl"],"align":["right"]},{"label":["logLik"],"name":[4],"type":["dbl"],"align":["right"]},{"label":["deviance"],"name":[5],"type":["dbl"],"align":["right"]},{"label":["Chisq"],"name":[6],"type":["dbl"],"align":["right"]},{"label":["Chi Df"],"name":[7],"type":["dbl"],"align":["right"]},{"label":["Pr(>Chisq)"],"name":[8],"type":["dbl"],"align":["right"]}],"data":[{"1":"5","2":"12338.90","3":"12370.78","4":"-6164.450","5":"12328.90","6":"NA","7":"NA","8":"NA","_rn_":"model_null"},{"1":"6","2":"12312.82","3":"12351.08","4":"-6150.409","5":"12300.82","6":"28.08162","7":"1","8":"1.163051e-07","_rn_":"model"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

We see that we are warned about the fact that the two models were fitted by REML. Fortunately, if we try to compare two REML-fitted LME models using `anova()`, `lme4` issues this warning and then re-fits the models using ML.

We can also ask for ML explicitly when fitting the model.


```r
model_ML = lmer(Depression ~ scale(Anxiety) + (scale(Anxiety)|Grade), ah, REML=FALSE)

summary(model_ML)
```

```
## Linear mixed model fit by maximum likelihood  ['lmerMod']
## Formula: Depression ~ scale(Anxiety) + (scale(Anxiety) | Grade)
##    Data: ah
## 
##      AIC      BIC   logLik deviance df.resid 
##  12312.8  12351.1  -6150.4  12300.8     4338 
## 
## Scaled residuals: 
##     Min      1Q  Median      3Q     Max 
## -3.1108 -0.6670 -0.2497  0.6255  3.4816 
## 
## Random effects:
##  Groups   Name           Variance Std.Dev. Corr
##  Grade    (Intercept)    0.008463 0.09199      
##           scale(Anxiety) 0.002208 0.04699  0.95
##  Residual                0.990721 0.99535      
## Number of obs: 4344, groups:  Grade, 6
## 
## Fixed effects:
##                Estimate Std. Error t value
## (Intercept)     2.18581    0.04053   53.94
## scale(Anxiety)  0.63680    0.02454   25.95
## 
## Correlation of Fixed Effects:
##             (Intr)
## scal(Anxty) 0.694
```

Comparing the ML model to the one fit by REML we can see that ML gives lower estimates of the variance in random effects:


```r
summary(model)
```

```
## Linear mixed model fit by REML ['lmerMod']
## Formula: Depression ~ scale(Anxiety) + (scale(Anxiety) | Grade)
##    Data: ah
## 
## REML criterion at convergence: 12311.4
## 
## Scaled residuals: 
##     Min      1Q  Median      3Q     Max 
## -3.1074 -0.6793 -0.2566  0.6127  3.4837 
## 
## Random effects:
##  Groups   Name           Variance Std.Dev. Corr
##  Grade    (Intercept)    0.010582 0.10287      
##           scale(Anxiety) 0.003007 0.05484  0.88
##  Residual                0.990667 0.99532      
## Number of obs: 4344, groups:  Grade, 6
## 
## Fixed effects:
##                Estimate Std. Error t value
## (Intercept)     2.18606    0.04468   48.93
## scale(Anxiety)  0.63632    0.02712   23.46
## 
## Correlation of Fixed Effects:
##             (Intr)
## scal(Anxty) 0.688
```

The more data we have, the smaller the differences in estimates between ML and REML. But we will still always need to use ML when comparing models with different fixed effects.

## Troubleshooting

Fitting mixed effects models is computationally quite complex, and there are yet more subtleties to consider. When working with your own data you may find that `lmer()` reports an error, saying that the model failed to converge or that the fit was singular. This isn't always fatal. To diagnose problems, check your data carefully.

The most important golden rule (for all of statistics, not just mixed effects models), is to *always* visualize your data, and ideally also visualize the model you want to describe them with. This may already alert you to problems, such as not enough observations in a particular category, some very extreme observations, or unreasonable predictions by the model.

Other than this, scaling predictors and simplifying unnecessary random effects will often help. Ben Bolker, one of the developers of lme4, offers some more detailed troubleshooting information at his [FAQ](https://bbolker.github.io/mixedmodels-misc/glmmFAQ.html).