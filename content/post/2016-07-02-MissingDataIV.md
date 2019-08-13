---
layout: post
title:  "Missing data in IV"
author: "Sam Portnow"
date:   2016-07-02
excerpt: "Missing data in instrumental variables"
tags:
- instrumental variables
- missing data
- fiml
categories: 
- R
comments: true
---

# What should you do with missing data in an instrumental variables analysis?

For my dissertation, I'm using instrumental variables to examine the impact of family income on children's cognitive skills. Having been trained as a psychologlist, there is an emphasis on handling missing data appropritately -- Full Information Maximum Likelihood (FIML) and Multiple Imputation (MI) are popular techniques. However, in the causal inference courses I've taken, there seems to be less of a focus on how to handle missing data. In this post, I'll conduct a simulation to see if FIML improves estimates in an instrumental variables analysis.

## Instrumental Variables in a Nutshell

In an instrumental variable analysis, exogenous variation in a predictor of a treatment is used to predict the treatment variable. These predicted scores of the treatment represent exogenous variation in the treatment, and are use to predict the outcome. For example, imagine a scenario in which the president, unbeknownst to anyone in the country, randomly assigns different levels of tax rebates to different states. In this scenario, existed, for my disseration, I could predict family income from state, and the predicted value of family income will be an unbiased estimator of child cognitive skills because the rebates were randomly assigned.

## Missing Data

I wouldnt want to predict family income while controlling for demographic variables, such as race/ethnicity and education. But what if not all the families in my dataset provided that information? A lot of statistical programs would just remove those families from the analysis, which, from a psychological analysis point of view, is something that you definitely do not want to do. Let's see if that really is a bad idea.

## The Simulation

We're going to simulate y, x, i, and c. i is our instrument and is exogenously correlated with our predictor, x; c is our covariate. I'm specificy a moderate relation between x, and c and i, and a moderate relation between y, and c and x. To look to see which method is superior, I'll examine the distance between the true estimate of the predicted values of x from y versus the estimate from a model in which the data has been listwise deleted, and a model which use FIML to keep everything.

	{% raw %}

	library(ggplot2)
	library(lavaan)
	library(plyr)

	true_est <- NULL
	fiml_est <- NULL
	miss_est <- NULL


	set.seed(35)


	#generate the instrument

	i <- rnorm(2000, sd = 1)

	# generate the covariate 

	c <- rnorm(2000, sd = 1)

	# generate x. it has a moderate relation with c and i

	x <- 0 + .3 * c + .3 * i + rnorm(2000, sd = .5) 

	# generate y. it has a moderate relation with c and x.

	y <- 0 + .3 * c + .3 * x + rnorm(2000, sd = .5)

	# create the data frame

	dat <- data.frame(i, x, c, y)

	# now get the predicted values of x, controlling for c

	pred <- predict(lm(x ~ i + c, data = dat))

	dat$pred <- pred

	# and predict y from the predicted values of x, controlling for c

	mod.iv <- lm(y ~ pred + c, data = dat)

	# store the true estimate

	true_est <- c(true_est, coef(mod.iv)['pred'])

	# now remove values

	dat <- dat[,names(dat) != 'pred']

	for (j in 5:1995)
	{
	    dat_miss <- dat
	    
	    dat_miss[sample(nrow(dat_miss), j),][,c('c')] <- NA
	      
	    # predict x from i, controlling for c
	    
	    pred <- predict(lm(x ~ i + c, dat_missa = dat_miss))
	    
	    # get ready to merge the predicted values into the dat_missaset
	    
	    pred <- data.frame(names(pred), pred)
	    
	    names(pred) <- c('ID', 'pred')
	    
	    dat_miss$ID <- rownames(dat_miss)
	    
	    # merge the predicted values
	    
	    dat_miss <- join(dat_miss, pred, by='ID')
	    
	    # predict y from the predicted values, controlling for c. this will listwise delete cases with missing values
	    
	    mod.miss <- lm(y ~ pred + c, data = dat_miss)
	    
	    # store the estimate
	    
	    miss_est <- c(miss_est, coef(mod.miss)['pred'])
	  
	    # predict y from the predicted values, controlling for c. this will use FIML to store the missing values
	    
	    mod.lav <-
	      
	    '
	  
	    y ~ pred + c
	  
	    '
	    
	    fit <- sem(mod.lav, missing='FIML', data = dat_miss)
	    
	    # store the estimate
	    
	    fiml_est <- c(fiml_est, parameterEstimates(fit)$est[1])
	    
	}

	frame <- data.frame(miss_est, fiml_est)

	frame$true_est <- true_est

	frame$dist_miss <- frame$true_est - frame$miss_est

	frame$dist_fiml <- frame$true_est - frame$fiml_est

	frame$x <- 5:1995


	{% endraw %}

Now we can plot the distance from the true estimate for FIML and listwise deletion. Listwise deletion is in red, and FIML is in blue. The y axis is the distance from the true estimate, and the x axis is the number of missing data points.

<figure>
	<img src="/post/img/IVSim.png"></a>
</figure>

To my surprise, listwise deletion is the clear winner. Listwise deletion doesn't even look that bad.

A BIG caveat here is that the predicted values we obtained were obtained with listwise deleting -- lavaan can't get predictions with missing values, so that wasn't an option. However, multiple imputation could solve that problem. I'll check that out next time.


