---
layout: page
title: Are Teaching Assistants (TAs) evaluated fairly?
permalink: /projects/ta_eval/
---

Introduction
============

Teaching Assistant (TA) evaluations are an important form of anonymous feedback from students. They allow students to freely speak their mind and rate the teaching assistant on a variety of topics. They are also taken into account when the department allocates TAships in the future and since they are paid, can negatively impact a TA's financial sitaution. The goal of this report is to find if TAs are being fairly evaluated or if there are external factors active.

Material and Methods
====================

My data is taken from: <https://archive.ics.uci.edu/ml/datasets/Teaching+Assistant+Evaluation>

A description from the website follows: The data consist of evaluations of teaching performance over three regular semesters and two summer semesters of 151 teaching assistant (TA) assignments at the Statistics Department of the University of Wisconsin-Madison.

Attribute Information:
Since Course instructor and Course contained 25 and 26 levels respectively, I left them out of my data for ease of analysis and because I was not interested in whether or not certain TAs were evaluated better than others. So, the data I did use consisted of 151 observations with the four attributes remaining: native English speaker, Summer or regular semester, Class size, and Class attribute.

Since Class size was a numerical variable, I categorized it according to the criteria:
To analyze my data I used loglinear models to fit my data followed by 2-dimensional contingency tables for testing independence/associations between pairs of factors.

Results
=======

To start my analysis, I fit a homogeneous model to my data:

The bolded p-values are significant at level *α* = 0.05. This suggests that there is dependence between English and Attribute and maybe between English and Size. The p-value for this homogeneous model is 0.7251979 so it is a good model for our data, but its interpretation is meaningless.

I fit a model with the interaction between English and Attribute, and English and Size.

    ## 
    ## Call:
    ## glm(formula = count ~ English + Type + Size + Attribute + English:Size + 
    ##     English:Attribute, family = poisson, data = tae.data)
    ## 
    ## Deviance Residuals: 
    ##     Min       1Q   Median       3Q      Max  
    ## -2.4582  -0.8557  -0.2796   0.2937   3.6204  
    ## 
    ## Coefficients:
    ##                            Estimate Std. Error z value Pr(>|z|)    
    ## (Intercept)                  1.5528     0.2691   5.771 7.89e-09 ***
    ## EnglishYes                  -0.1156     0.4661  -0.248 0.804215    
    ## TypeSummer                  -1.7165     0.2265  -7.579 3.47e-14 ***
    ## SizeMedium                   1.0116     0.2611   3.874 0.000107 ***
    ## SizeSmall                    0.8544     0.2670   3.200 0.001373 ** 
    ## AttributeLow                 0.2578     0.2283   1.129 0.258836    
    ## AttributeMedium              0.2578     0.2283   1.129 0.258836    
    ## EnglishYes:SizeMedium       -0.6061     0.5258  -1.153 0.249040    
    ## EnglishYes:SizeSmall        -0.7366     0.5544  -1.329 0.183967    
    ## EnglishYes:AttributeLow     -1.5388     0.5547  -2.774 0.005537 ** 
    ## EnglishYes:AttributeMedium  -1.3564     0.5238  -2.590 0.009608 ** 
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## (Dispersion parameter for poisson family taken to be 1)
    ## 
    ##     Null deviance: 237.194  on 35  degrees of freedom
    ## Residual deviance:  64.072  on 25  degrees of freedom
    ## AIC: 166.28
    ## 
    ## Number of Fisher Scoring iterations: 6

If we look at the p-values of coefficients for this model, we find that the interaction between English and Size is not significant anymore. That is, we can remove this interaction effect and maybe get a better model. Also, the p-value of this model is 2.8078075 × 10<sup>−5</sup>.

Now I fit a model with the interaction between English and Attribute, and three-way interactions interpreted as conditional independence of Type and Size given English and Attribute.

If we look at the p-values of the interaction between English and Attribute, we find that Attribute is dependent on English. That is, the evaulation score of a TA is dependent on whether or not they are a native English speaker or not. Also, the p-value of this model is 8.9232104 × 10<sup>−4</sup> which is larger than corresponding p-value of the model above indicating that this is a better fit. Note that while the p-value is less than *α* = 0.05, this is the best model we can fit outside of the homogeneous model.

I use the AIC criterion to select the best model of the three presented. For AIC we have

    ##                   df      AIC
    ## homogeneous.model 32 146.2617
    ## tae.model1        11 166.2765
    ## tae.model4        24 161.4306

Our fitted model above with the interaction between English and Attribute, and three-way interactions interpreted as conditional independence of Type and Size given English and Attribute has the smallest AIC of the practical interpretation models (not homogeneous model). Hence, I choose the fitted model above.

We look at the contingency table of English and Attribute.

    ##        Attribute
    ## English  1  2  3
    ##       1  5  6 18
    ##       2 44 44 34

I perform a test for independence and find the p-value to be 0.0022546 &lt; *α* = 0.05. So, we reject the null hypothesis of independence and conclude that English and Attribute are dependent.

I also examine why the model with both English and Attribute, and English and Size is a worse fit than our selected model. The contingency table of English and Size is given below

    ##        Size
    ## English Small Medium Large
    ##       1     9     12     8
    ##       2    47     55    20

and we see that the p-value = 0.3660786 &gt; *α* = 0.05. So, English and Size are independent of one another hence why the interaction between them did not work well with our model.

I also looked at my model's residuals and found that they are in the interval \[-2, 2\] indicating that the model fits.

    ##             1             2             3             4             5 
    ##  1.500000e+00 -1.000000e+00 -1.000000e+00 -1.332268e-15 -1.000000e+00 
    ##             6             7             8             9            10 
    ## -1.000000e+00  6.666667e-01  2.960595e-16 -1.000000e+00 -1.000000e+00 
    ##            11            12            13            14            15 
    ##  6.666667e-01  2.220446e-16  2.000000e+00  5.529412e-01 -4.000000e-01 
    ##            16            17            18            19            20 
    ## -7.088989e-02 -1.000000e+00 -2.222222e-02  2.000000e-01  2.849003e-03 
    ##            21            22            23            24            25 
    ## -1.000000e+00 -1.000000e+00  2.000000e-01  1.282051e-01  1.400000e+00 
    ##            26            27            28            29            30 
    ##  8.888889e-01 -7.000000e-01 -3.200000e-01 -5.714286e-01 -1.000000e+00 
    ##            31            32            33            34            35 
    ##  2.857143e-01  3.600000e-01 -5.000000e-01 -1.000000e+00  2.500000e-01 
    ##            36 
    ##  3.600000e-01

Conclusion and Discussion
=========================

In summary, my chosen model for this data contains the main effects, interaction between English and Attribute, and three-way interactions interpreted as conditional independence of Type and Size given English and Attribute.
*l**o**g*(*μ*<sub>*i**j**k**l*</sub>)=*λ* + *λ*<sub>*i*</sub><sup>*E**n**g**l**i**s**h*</sup> + *λ*<sub>*j*</sub><sup>*T**y**p**e*</sup> + *λ*<sub>*k*</sub><sup>*S**i**z**e*</sup> + *λ*<sub>*l*</sub><sup>*A**t**t**r**i**b**u**t**e*</sup> + *λ*<sub>*i**l*</sub><sup>*E**n**g**l**i**s**h*, *A**t**t**r**i**b**u**t**e*</sup> + *λ*<sub>*i**j**l*</sub><sup>*E**n**g**l**i**s**h*, *T**y**p**e*, *A**t**t**r**i**b**u**t**e*</sup> + *λ*<sub>*i**k**l*</sub><sup>*E**n**g**l**i**s**h*, *S**i**z**e*, *A**t**t**r**i**b**u**t**e*</sup>

That is, the evaulation score of a TA is dependent on whether or not they are a native English speaker or not. In the case of TAs in the Statistics Department of the University of Wisconsin-Madison, we can conclude that native English speaking TAs are going to be rated higher than non-native English speaking TAs.

Of course, this project does not give a definitive answer of the general question of whether ALL TAs, regardless of school, department, etc., are evaluated fairly because of its shortcomings (small sample size, few factors, single sample from one university, small p-value of fitted model, counts of 0, etc.).

Code Appendix
=============

``` r
setwd("C:/Users/Christopher/Desktop/STA 138/Project 1")

# https://archive.ics.uci.edu/ml/datasets/Teaching+Assistant+Evaluation
tae <- read.csv("C:/Users/Christopher/Desktop/STA 138/Project 1/tae.data", header=FALSE)

colnames(tae) <- c("English", "Instructor", "Course", "Type", "Size", "Attribute")

# We will only deal with English, Type, Size (categorized), and Attribute
tae <- tae[, -c(2, 3)]

# Small = 0-20, Medium = 21-40, Large = 41+
tae$Size <- cut(tae$Size, c(0, 20, 40, max(tae$Size)), labels = c("Small", "Medium", "Large"))

# English: 1 = native English speaker
# Type: 1 = Summer session
# Size: categorized into small, medium, large
# Attribute: 1 = Low, 2 = Medium, 3 = High

tae.data <- data.frame(expand.grid(English = factor(c("Yes", "No")), 
                                   Type = factor(c("Summer", "Semester")), 
                                   Size = factor(c("Small", "Medium", "Large")), 
                                   Attribute = factor(c("Low", "Medium", "High"))), 
                       count = c(2, 0, 0, 12,
                                 0, 0, 2, 24,
                                 0, 0, 1, 8,
                                 1, 3, 1, 14,
                                 0, 2, 3, 16,
                                 0, 0, 1, 9, 
                                 4, 9, 1, 9, 
                                 1, 0, 6, 13,
                                 1, 0, 5, 3))

homogeneous.model <- glm(count~English+Type+Size+Attribute+
                      English:Type+English:Size+English:Attribute+
                        Type:Size+Type:Attribute+Size:Attribute+
                      English:Type:Size+English:Type:Attribute+
                        Type:Size:Attribute+English:Size:Attribute, 
                      family = poisson, 
                      data = tae.data)

tae.model1 <- glm(count~English+Type+Size+Attribute+
                    English:Size+English:Attribute, 
                  family = poisson,
                  data = tae.data)
summary(tae.model1)

tae.model4 <- glm(count~English+Type+Size+Attribute+
                    English:Attribute+English:Type:Attribute+English:Size:Attribute,
                  family = poisson,
                  data = tae.data)

AIC(homogeneous.model, tae.model1, tae.model4)

fit.table <- xtabs(~English+Attribute, data = tae)
fit.table

fit.table2 <- xtabs(~English+Size, data = tae)
fit.table2

tae.model4$residuals

### Extra code for more models
tae.model2 <- glm(count~English+Type+Size+Attribute+English:Attribute,
                  family = poisson, 
                  data = tae.data)

tae.model3 <- glm(count~English+Type+Size+Attribute+
                    English:Size+English:Attribute+Size:Attribute+English:Size:Attribute,
                  family = poisson, 
                  data = tae.data)

tae.model5 <- glm(count~English+Type+Size+Attribute+
                    English:Type:Size+Type:Size:Attribute, 
                  family = poisson,
                  data = tae.data)
```
