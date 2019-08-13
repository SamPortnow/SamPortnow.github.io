---
layout: post
title:  "Looking Through the Idiographic Filter"
author: "Sam Portnow"
date:   2016-07-25
excerpt: "Looking Through the Idiographic Filter"
tags:
- measurement invariance
- idiographic filter
categories: 
- R
comments: true
---

For my dissertation, I'm going to apply causal inference techniques to three separate datasets to examine the impact of family income on school readiness skills, while examining supportive parenting and cognitive stimulation as mediators. For one dataset, the Secondary Education Childcare and Youth Development (SECCYD) dataset, I plan to use a longitundal fixed effects approach. Unfortunately, this approach makes examining supportive parenting as a mediator difficult because the items that assess supportive parenting change at different assessment points. As such, the change that I am examining in the longitudinal fixed effects approach may be true change, or may be due to a change in measurement. But, an idiographic filter may solve this problem. A BIG thank you to [Geneva Dodson](http://people.virginia.edu/~gtd9fe/) for introducing this method to me and sharing her awesome dissertation with me.

With an idiographic filter, we can test whether supportive parenting is the same over time; we fix autoregressive parameter of supportive parenting to by the same across ages (assessment points). If model fit does not worsen, then we have evidence that the construct is the same at each assessment, even when its indicators change. 

This test can be represented as an equation. In the dataset for my dissertation, I have approximately 1,000 persons (n) measured five times. Supportive parenting is measured with five indicators (p), and I am attempting to estimate one common factor (k). 

$$ Y = \wedge F + E $$ 								(1)

where Y is an n (1,000) × p (five) matrix of observations, $$ \wedge $$ is an n × k (1) matrix of factor loadings, F is a k × p matrix of factor scores, and E is an n × p matrix of errors. In the terms of my diss, Y is the observed indicators of supportive parenting, $$ \wedge $$ is each assessment points' estimate of supportive parenting, and F is the factor scores.

The factors scores themselves can also be predicted:

$$ F = \wedge * F * + E * $$ 							(2)

in which F is a k × p matrix of first-order factor scores, $$ \wedge * $$ is a k* × k matrix of (second order or regression estimates), F* is a k* × p matrix of (second order or regression) factor scores, and E* is a k × p matrix of (second order or regression) errors. In terms of my diss, F is the observed factor scores of supportive parenting, $$ \wedge * $$ is the autoregressive parameter of supportive parenting, and F* is the autoregressive parameter of cognitive stimulation for each timepoint.

Putting it altogether, we have the following:

$$ Y = \wedge(\wedge * F * + E *) + E $$ 				(3)

In terms of my diss (SP = Supportive Parenting):

SP = SPFactorLoadings(AutoRegressiveParameterSample*AutoRegressiveParameterTimePoint + Error) + Error

With the idiographic filter, invariance is imposed on the terms within the paranetheses (AutoRegressiveParameterTimePoint + Error). The loadings themselves can than vary across assessment points and items, but the autoregressive parameter and error of the factor scores must be constant because we are filtering out idiographic (person-centered; or in my case, group-specific) information). Hence, the idiographic filter.

Nesselroade, Gerstoff, Hardy, and Ram (2007) present an intuitive application of this procedure. Consider the calculation of the area for a triangle (1/2 *bh*), circle ( $$ \pi $$ *R^2*), and square (*lw*). As they currently stand, the areas are not comparable. BUT, the volume calculation is the same (*V = A x S*) across triangles, circles, and squares, and so A and V have the same meaning from one group to another, but the calculation for area is idiographic.


Using the idiographic filter is comprised of three steps:

1. Fit the latent variables.
2. Add the regression parameter.
3. Impose equality restraints across families in the following order:
	1. Regression parameter
	2. The variance of predicted supportive parenting at each timepoint (the
	 error of the regression parameter).

## An Example

# Step 1

First, we fit the latent variables at each timepoint, but we don't estimate the autoregressive parameter.

	{% raw %}

	library(simsem)

	Model <- '

	SP_1 =~ 1*y1 + 0.6*y2 + 0.7*y3
	SP_2 =~ 1*y4 + 1.1*y5 + 0.9*y6
	SP_3 =~ 1*y7 + 1.2*y8 + 1.1*y9

	SP_1 ~~ 0.5*SP_1
	SP_2 ~~ 0.5*SP_2
	SP_3 ~~ 0.5*SP_3

	SP_2 ~ 0.5*SP_1
	SP_3 ~ 0.5*SP_2

	y1 ~~ 0.5*y1
	y2 ~~ 1.1*y2
	y3 ~~ 0.8*y3
	y4 ~~ 0.4*y4
	y5 ~~ 0.4*y5
	y6 ~~ 0.8*y6
	y7 ~~ 0.8*y7
	y8 ~~ 0.5*y8
	y9 ~~ 0.6*y9

	'


	dat <- generate(Model, 1000)
	dat$ID <- rownames(dat)


	# first step - estimate supportive parenting at each time point

	mod.1 <-
	'

	SP_1 =~ y1 + y2 + y3
	SP_2 =~ y4 + y5 + y6
	SP_3 =~ y7 + y8 + y9

	SP_2 ~ 0*SP_1
	SP_3 ~ 0*SP_2
	SP_3 ~ 0*SP_1
	'

	fit.1 <- sem(mod.1, missing='FIML', estimator='MLR', data = dat)
	summary(fit.1, fit=T, rsquare=T)

	{% endraw %}

Unsurprisingly, this model does not have good fit (RMSEA = .09).

# Step 2

Next, we add the autoregressive parameteter.

	{% raw %}

	# second step - add the autoregressive parameters 

	mod.2 <- 
	'

	SP_1 =~ y1 + y2 + y3
	SP_2 =~ y4 + y5 + y6
	SP_3 =~ y7 + y8 + y9

	SP_2 ~ SP_1
	SP_3 ~ SP_2

	'
	fit.2 <- sem(mod.2, missing='FIML', estimator='MLR', data = dat)
	summary(fit.2, fit=T, rsquare=T)

	{% endraw %}

Model fit is much improved here (RMSEA = .02). 

# Step 3a.

We then fix the autoregressive paramaters to be equal at each timepoint.

	{% raw %}
	# step 3.a -- fix the autoregressive parameters
	mod.3a <- 
	'

	SP_1 =~ y1 + y2 + y3
	SP_2 =~ y4 + y5 + y6
	SP_3 =~ y7 + y8 + y9

	SP_2 ~ a*SP_1
	SP_3 ~ a*SP_2

	'
	fit.3a <- sem(mod.3a, missing='FIML', estimator='MLR', data = dat)
	summary(fit.3a, fit=T, rsquare=T)
	{% endraw %}

Model fit stays virtually the same (RMSEA = .02).

# Step 3b. 

Finally, we fix the residual variances to be equal at each timepoint.

	{% raw %}
	# Step 3b -- fix the variance of the predicted supportive parenting -- the error 

	mod.3b <- 
	  '

	SP_1 =~ y1 + y2 + y3
	SP_2 =~ y4 + y5 + y6
	SP_3 =~ y7 + y8 + y9

	SP_2 ~ a*SP_1
	SP_3 ~ a*SP_2

	SP_2 ~~ b*SP_2
	SP_3 ~~ b*SP_3

	'
	fit.3b <- sem(mod.3b, missing='FIML', estimator='MLR', data = dat)
	summary(fit.3b, fit=T, rsquare=T)


	{% endraw %}

And model fit stays the same (RMSEA = .02).


So, there is evidence that we are measuring the same construct at each time point. This result is unsurprising, I generated data that reflected the same construct three separate times. This simulation shows what you would expect in an ideal situation.