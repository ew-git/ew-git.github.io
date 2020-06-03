---
layout: post
title:  "Julia PoissonFE.jl Package"
date:   2020-06-02 12:00:00 -0500
categories: coding julia
---

I've released a Julia package, [PoissonFE.jl](https://github.com/ew-git/PoissonFE.jl), to estimate Poisson regression with fixed effects. 
Similar to my R package, [poisFErobust](https://cran.r-project.org/package=poisFErobust), it computes robust standard errors from [Wooldridge (1999)](https://doi.org/10.1016/S0304-4076%2898%2900033-5).

The Julia package is quite a bit faster than R. 
Aside from the raw speed improvement of Julia, it implements the likelihood function without the fixed effects parameters, so it should be faster than [GLM.jl](https://github.com/JuliaStats/GLM.jl) for models with many fixed effect levels (thousands). 
The estimates should match Stata's command, `xtpoisson y x, fe vce(r)`.

I continue to be impressed by Julia's ecosystem. 
Estimating the coefficients only required implementing the likelihood function. 
Automatic differentiation via [ForwardDiff.jl](https://github.com/JuliaDiff/ForwardDiff.jl) enables a simple function call to [Optim.jl](https://github.com/JuliaNLSolvers/Optim.jl) which implements efficient line search algorithms. 
There was no need to implement the gradient or Hessian by hand. 

### References
Wooldridge, Jeffrey M. (1999): "Distribution-free estimation of some nonlinear panel data models," *Journal of Econometrics*, 90, 77-97. [https://doi.org/10.1016/S0304-4076%2898%2900033-5](https://doi.org/10.1016/S0304-4076%2898%2900033-5)
