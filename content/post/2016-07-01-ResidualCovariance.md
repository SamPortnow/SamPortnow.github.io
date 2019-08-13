---
layout: post
title:  "Residual Covariance"
author: "Sam Portnow"
date:   2016-07-01
excerpt: "A note on residual covariance"
tags:
- SEM
- CFA
- lavaan
categories: 
- R
comments: true
---

# A Way to Look At Why Your SEM Model Has Poor Fit

If you've done any Structural Equation Modeling, you've run into a bad fitting model. Embarassingly, aside from estimating covariances among correlated variables, I didn't know much about how to inspect poor model fit until very recently. Enter the residual covariance matrix. It's a super simple way to examine poor model fit. 

Here's an example. You have 6 observed variables, and you want to create a latent factor. I'm going to use the lavaan and the simsem package in R to generate data.

	{% raw %}

	library(simsem)
	library(lavaan)

	loading <- matrix(0, 6, 2)
	
	loading[1:3, 1] <- NA
	
	loading[4:6, 2] <- NA
	
	LY <- bind(loading, 0.7)
	
	latent.cor <- matrix(NA, 2, 2)
	
	diag(latent.cor) <- 1
	
	RPS <- binds(latent.cor, 0.5)
	
	RTE <- binds(diag(6))
	
	VY <- bind(rep(NA,6),2)
	
	CFA.Model <- model(LY = LY, RPS = RPS, RTE = RTE, modelType = "CFA")
	
	dat <- generate(CFA.Model,200)

	mod <-
	
	'
	
	L =~ y1 + y2 + y3 + y4 + y5 + y6
	
	'
	
	fit <- cfa(mod, data = dat)

	{% endraw %}

If we examine the residual correlation matrix, with ` resid(fit, type='cor')$cor `, we see moderate correlations between y1, y2, and y3. 

	{% raw %}

	   y1     y2     y3     y4     y5     y6    
	y1  0.000                                   
	y2  0.303  0.000                            
	y3  0.253  0.208  0.000                     
	y4 -0.153 -0.097 -0.067  0.000              
	y5 -0.035 -0.089 -0.069  0.075  0.000       
	y6 -0.041 -0.069 -0.071  0.071  0.008  0.000

	{% endraw %}

We also have really poor fit, RMSEA = .20. So, that means that we're not accounting for the covariance between these indicators in our model. Or, a one factor solution doesn't take into accout how y1, y2, and y3 vary toghether. Let's see what happens if we make two factors.

	{% raw %}

	mod.2 <-
	  '
	L1 =~ y1 + y2 + y3 
	L2 =~ y4 + y5 + y6

	'
	fit.2 <- cfa(mod.2, data = dat)

	{% endraw %}

If we examine the residual correlation matrix, with {% raw %} resid(fit.2, type='cor')$cor {% endraw %}, we see those moderate correlations have disappeared.

	{% raw %}

	   y1     y2     y3     y4     y5     y6    
	y1  0.000                                   
	y2  0.012  0.000                            
	y3 -0.009 -0.005  0.000                     
	y4 -0.085 -0.049  0.032  0.000              
	y5  0.049 -0.028  0.043  0.008  0.000       
	y6  0.041 -0.009  0.039  0.011 -0.023  0.000


	{% endraw %}


We also now have a much improved fit, RMSEA = .024.

I find this method super helpful for anything in SEM -- path models, CFA, growth curves, etc.

