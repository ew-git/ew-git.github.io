---
layout: post
title:  "A key benefit of Julia"
date:   2019-01-15 12:00:00 -0500
categories: coding julia
---
<link rel="stylesheet" href="/assets/css/hljs-default.css">
<script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/latest.js?config=TeX-MML-AM_CHTML">
</script>
<script type="text/javascript" src="/assets/js/highlight.pack.js"></script>
<script>hljs.initHighlightingOnLoad();</script>
Highlight.js test again:
<pre><code class="language-julia">function f(x,y)
  return x + y
end</code></pre>

*Notice: this post relies on Unicode characters which may not be included in the font(s) on your current platform, especially mobile. Therefore, you may see empty boxes or question marks for certain characters.*

A key benefit of [Julia](https://julialang.org/) is the ability to write code which looks similar to the mathematical representation of an algorithm, with simple and expressive syntax, while still being performant.
Additionally, Julia accepts UTF-8 encoded Unicode characters as variable names, so if a paper defines some quantity, \\( \boldsymbol{\mu}_i \\), then essentially the same Unicode representation, 𝛍ᵢ, could be used in the code.[^1]
Although it may seem minor, there's a certain pedagogical advantage to having such a close match.
When reading through the code of a package for computing a statistical estimator, it's easier to parse  
`𝚲ᵢ' * 𝐖ᵢ * 𝐮ᵢ * 𝐮ᵢ' * 𝐖ᵢ * 𝚲ᵢ` than  
`t(Lambda_i) %*% W_i %*% u_i %*% t(u_i) %*% W_i %*% Lambda_i`  
when the equivalent equation in the accompanying paper is  
$$ \boldsymbol{\Lambda}_i' \boldsymbol{W}_i \boldsymbol{u}_i \boldsymbol{u}_i' \boldsymbol{W}_i \boldsymbol{\Lambda}_i $$.

Now, I don't recommend replacing all your descriptive variable and function names with fancy Unicode characters, but sometimes it does make sense to use the same mathematical symbols.
The Julia code below demonstrates this point by defining a function to compute standard errors and a hypothesis test from [Wooldridge (1999)](https://doi.org/10.1016/S0304-4076%2898%2900033-5).
The comments provide a mapping to definitions in the paper which share notation with the code.

```julia
import StatsFuns.chisqcdf;
using LinearAlgebra;
poisfe_test = function(y, id, X, qcmle_coefs)
    # No "hats" on variables are used for brevity.
    # We are already dealing with estimated quantities.
    β = qcmle_coefs # Estimated parameters, p.79
    K = length(β) # Number of parameters, p.79
    all_ids = unique(id)
    N = length(all_ids) # Number of groups, p.83

    # Initialize arrays described later
    𝐊 = zeros(Float64, K, K)
    𝐀 = zeros(Float64, K, K)
    𝐁 = zeros(Float64, K, K)
    𝚲 = Array{Array{Float64}}(undef, N)
    𝐖 = Array{Array{Float64}}(undef, N)
    𝐮 = Array{Array{Float64}}(undef, N)

    # Particular functional form (Poisson), p.79
    μ = function(𝐱ᵢₜ, β)
        return exp(𝐱ᵢₜ ⋅ β)
    end

    for i=1:N
        this_index = searchsorted(id, all_ids[i])
        T = length(this_index)
        yᵢ = y[this_index] # Tx1 vector of outcomes, p.79
        𝐱ᵢ = X[this_index, :] # TxK matrix of indep vars, p.79
        nᵢ = sum(yᵢ) # Scalar sum of outcomes, p.79

        𝛍ᵢ = similar(yᵢ) # Tx1 vector of conditional means, p.82
        for t in 1:T
            𝛍ᵢ[t] = μ(𝐱ᵢ[t, :], β)
        end
        Σ𝛍ᵢ = sum(𝛍ᵢ) # Scalar sum of conditional means, p.82

        𝐩 = 𝛍ᵢ / Σ𝛍ᵢ # Tx1 vector of "choice probabilities," p.79 (2.5)
        # 𝚲ᵢ (TxK) is the derivative of 𝐩 wrt β; p.86
        𝚲ᵢ = similar(𝐱ᵢ)
        for k in 1:K
            last_term = 0
            for s in 1:T
                last_term += 𝐱ᵢ[s, k] * 𝐩[s]
            end
            for t in 1:T
                𝚲ᵢ[t, k] = 𝐩[t] * (𝐱ᵢ[t, k] - last_term)
            end
        end

        𝐖ᵢ = Diagonal((1 ./ 𝐩))  # TxT, p.82
        𝐮ᵢ = yᵢ - nᵢ * 𝐩 # Tx1, p.82

        # Each group's 𝚲ᵢ, 𝐖ᵢ, and 𝐮ᵢ are stored in an array of arrays for
        # later computation of a hypothesis test, p.86
        𝚲[i] = 𝚲ᵢ
        𝐖[i] = 𝐖ᵢ
        𝐮[i] = 𝐮ᵢ

        𝐊 += 1/N * 𝚲ᵢ' * (nᵢ * 𝚲ᵢ) # p.86 (3.16)
        𝐀 += 1/N * nᵢ * (𝚲ᵢ' * 𝐖ᵢ * 𝚲ᵢ) # p.83 (3.9)
        𝐁 += 1/N * 𝚲ᵢ' * 𝐖ᵢ * 𝐮ᵢ * 𝐮ᵢ' * 𝐖ᵢ * 𝚲ᵢ # p.83 (3.10)
    end

    𝐀⁻¹ = inv(𝐀)
    se_rob = broadcast(sqrt, diag((𝐀⁻¹ * 𝐁 * 𝐀⁻¹) / N )) # p.83, (3.11)

    # Test of the conditional mean specification (3.1) with the null
    # hypothesis (3.14), p.85
    𝐫 = Array{Float64, 2}(undef, N, K) # p.86, (3.17)
    for i=1:N
        𝐫[i,:] = 𝐮[i]' * (𝚲[i] - 𝐖[i] * 𝚲[i] * 𝐀⁻¹ * 𝐊')
    end
    ssr = sum((ones(N) - 𝐫 * (𝐫 \ ones(N))) .^ 2) # p.86 (3.18)
    p_value = 1 - chisqcdf(K, N - ssr) # p.86
    return se_rob, p_value
end
```

The function may be tested with some synthetic data like the example below.
```julia
y = [1.0, 0.0, 0.0, 0.0, 7.0, 1.0, 0.0, 1.0, 0.0, 0.0, 6.0,
     2.0, 3.0, 0.0, 1.0, 6.0, 0.0, 7.0, 0.0, 21.0]
id = [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2]
X = [0.398106 0.496961; -0.612026 -0.224875; 0.34112 -1.11714;
     -1.12936 -0.394995; 1.43302 1.54983; 1.9804 -0.743514; -0.367221 -2.33171;
     -1.04413 0.812245; 0.56972 -0.501311; -0.135055 -0.510887;
     2.40162 -1.21536; -0.03924 -0.0225586; 0.689739 0.701239;
     0.0280022 -0.587482; -0.743273 -0.606728; 0.188792 1.09664;
     -1.80496 -0.24751; 1.46555 -0.159902; 0.153253 -0.625778;
     2.17261 0.900435]
qcmle_coefs = [0.924499; 0.869371]
show(poisfe_test(y, id, X, qcmle_coefs))
```
```
([0.013508, 0.12758], 0.36787944117144233)
```

Despite many optimizations left on the table, the code above is still going to be blazing fast.
An equivalent in a low-level language is likely to be more verbose and difficult for non-experts to parse---especially those from the social sciences, the audience of the paper.
A high level language like R can be written in a more understandable way (see [here](https://bitbucket.org/ew-btb/poisson-fe-robust)), but achieving high performance requires either vectorization (sometimes difficult, sometimes impossible) or dropping back down to a low-level language.
Julia seems to hit the sweet spot.

### References
Wooldridge, Jeffrey M. (1999): "Distribution-free estimation of some nonlinear panel data models," *Journal of Econometrics*, 90, 77-97. [https://doi.org/10.1016/S0304-4076%2898%2900033-5](https://doi.org/10.1016/S0304-4076%2898%2900033-5)

[^1]: Many other languages will technically allow Unicode characters in variable names, but based on personal experience there's typically some type of extra configuration or risk of errors across platforms. Julia is more accommodating, and most Julia development environments---including the REPL---support tab completion of common Unicode characters, e.g. `\alpha<tab>` becomes α.
