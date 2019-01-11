---
layout: post
title:  "Robust standard errors for Poisson regression with fixed effects"
date:   2019-01-10 19:00:00 -0500
categories: jekyll update
---

Last year I released an R package, [poisFErobust](https://cran.r-project.org/package=poisFErobust), which provides a function to compute standard errors for Poisson regression with fixed effects.
The standard errors are derived in [Wooldridge (1999)](https://doi.org/10.1016/S0304-4076%2898%2900033-5) and are robust to conditional serial correlation of errors within groups.
The function also returns the p-value of the hypothesis test of the conditional mean assumption (3.1) as described in the paper, Section 3.3.

The package is on CRAN, so it may be installed with
{% highlight R %}
install.packages("poisFErobust")
{% endhighlight %}

The examples below show output when the assumption (3.1) is satisfied and when it is not satisfied.
{% highlight R %}
require(poisFErobust)
# ex.dt.good satisfies the conditional mean assumption
data("ex.dt.good")
pois.fe.robust(outcome = "y", xvars = c("x1", "x2"), group.name = "id",
index.name = "day", data = ex.dt.good)
{% endhighlight %}

```
$coefficients
       x1        x2
0.9899730 0.9917526

$se.robust
        x1         x2
0.03112512 0.02481941

$p.value
[1] 0.6996001
```
{% highlight R %}
# ex.dt.bad violates the conditional mean assumption
data("ex.dt.bad")
pois.fe.robust(outcome = "y", xvars = c("x1", "x2"), group.name = "id",
index.name = "day", data = ex.dt.bad)
{% endhighlight %}
```
$coefficients
       x1        x2
0.4800735 2.9866911

$se.robust
       x1        x2
0.2864666 1.2743953

$p.value
[1] 0.02213269
```

The results should match those of Stat's `xtpoisson y x, fe vce(r)`.

The source code is available at [https://bitbucket.org/ew-btb/poisson-fe-robust](https://bitbucket.org/ew-btb/poisson-fe-robust).
Pull requests are welcome.

### References
Wooldridge, Jeffrey M. (1999): "Distribution-free estimation of some nonlinear panel data models," *Journal of Econometrics*, 90, 77-97.
