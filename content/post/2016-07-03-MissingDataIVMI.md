---
layout: post
title:  "Missing data in IV-cont'd"
author: "Sam Portnow"
date:   2016-07-03
excerpt: "Missing data in instrumental variables-cont'd"
tags:
- instrumental variables
- missing data
- multiple imputation
- fiml
categories: 
- R
comments: true
---

In a [recent post](http://samportnow.github.io//MissingDataIV/) I pitted listedwise deletion against Full Information Maximum Likelihood (FIML) to see which outperformed which in an instrumental variables analysis (listwise deletion won). However, a big caveat of that analysis was that I didn't use FIML to generate predicted values of x because lavaan can't produce predicted values with incomplete data. So, this post is the same analysis, but with multiple imputation instead of FIML so that we can generated predicted values of x with the incomplete data. Analysis below:

	{% raw %}

	library(ggplot2)
	library(lavaan)
	library(plyr)
	library(mice)

	true_est <- NULL
	mi_est <- NULL
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
	  
	  dat_miss[sample(nrow(dat), j),][,c('c')] <- NA
	  
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
	  
	  # create an imputed dataset
	  
	  dat_miss.mi <- dat_miss[, ! names(dat_miss) %in% c('pred', 'ID')]

	  # impute the dataset

	  dat_miss.mi <- mice(dat_miss.mi)
	  
	  # make a completed dataset 
	  
	  dat_miss.mi <- complete(dat_miss.mi)

	  # get predicted x values
	  
	  pred <- predict(lm(x ~ i + c, data = dat_miss.mi))
	  
	  dat$pred <- pred
	  
	  # and predict y from the predicted values of x, controlling for c
	  
	  mod.iv <- lm(y ~ pred + c, data = dat)
	  
	  # store the mi estimate
	  
	  mi_est <- c(mi_est, coef(mod.iv)['pred'])  

	  
	}


	frame <- data.frame(miss_est, mi_est)

	frame$true_est <- true_est

	frame$dist_miss <- frame$true_est - frame$miss_est

	frame$dist_mi <- frame$true_est - frame$mi_est

	frame$x <- 5:1995

	{% endraw %}

Now we can plot the distance from the true estimate for multiple imputation and listwise deletion. Listwise deletion is in red, and multiple imputation is in blue. The y axis is the distance from the true estimate, and the x axis is the number of missing data points.

<figure>
	<img src="/post/img/IVmi.png"></a>
</figure>

With this one, we see that multiple imputation outperforms listwise deletion. So, the takeaway is that a procedure to handle missing data outperforms listwise deletion, but only if you use that procedure to get your predicted x values, in addition to predicting y.


