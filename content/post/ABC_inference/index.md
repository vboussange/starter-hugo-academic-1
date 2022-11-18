---
featured: false
subtitle: ""
summary: "In this tutorial, you'll learn the basics of approximate Bayesian computation (ABC). ABC is an inference method with very little requirements in terms of the model structure - yet it can be very powerful. It is very simple to apply to any model, and to understand. We'll play around with Julia, and we will visualize graphically the inference results, so that you can build an intuition of the inference method."
date: "2022-11-18"
header-includes:
  - "\\newcommand{\\M}{\\mathcal{M}}"
draft: false
title: "A practical introduction to Approximate Bayesian Computation"
tags: []
categories: []
authors: []
lastmod: "2022-11-18"
---


## Introduction
In this tutorial, you'll learn the basics of approximate Bayesian computation (ABC). <!-- and Bayesian inference, and compare the two approaches. -->
ABC is an inference method with very little requirements in terms of the model structure - yet it can be very powerful. It is very simple to apply to any model, and to understand. 
We'll play around with Julia, and we will visualize graphically the inference results, so that you can build an intuition of the inference method.

## Package loading
We first need to load a few packages.
```julia
cd(@__DIR__)
using Pkg; Pkg.activate(".")
import Optimisers:destructure # required to simplify parameter indexing
using ApproxBayes
using Distributions # to define the priors
using OrdinaryDiffEq
using UnPack
using ParametricModels # package not registered
using Random
using LinearAlgebra
```



<!--
using ComponentArrays # a nice package to index vectors in a similar style as NamedTuples, but behaving as Vectors
-->

It is always important to make your work reproducible, so we'll set manually a seed to the random number generator.
```julia
# Set a seed for reproducibility.
Random.seed!(11);
```




## Model definition and data generation
Let's consider a predator-prey ecological model, known as [Lotka-Volterra equations](https://en.wikipedia.org/wiki/Lotka–Volterra_equations). This model has a total of four different parameters $\alpha, \beta, \gamma, \delta$, that describe the interactions between the two species. For the sake of simplicity, we'll assume that we know perfectly the parameters $\gamma, \delta$, and seek to infer $\alpha$ and $\beta$. Assuming to know $\gamma, \delta$ is of course a very unrealistic assumption...

In the following, we use ParametricModels.jl to define our ODE problem. ParametricModels.jl is a simple wrapper around DifferentialEquations.jl, but makes it easier for multiple model simulation. For readability and maintenance, we'll encapsulate the parameters in a NamedTuple

```julia
# Declaration of the model
ParametricModels.@model LotkaVolterra
# Definition of the model
function (lv::LotkaVolterra)(du, u, p, t)
    # Model parameters
    @unpack α, β, = p
    # Current state
    x, y = u
    # fixed parameters
    γ = 3.0
    δ = 1.0

    # Evaluate differential equations
    du[1] = (α[] - β[] * y) * x # prey
    du[2] = (δ * x - γ) * y # predator

    return nothing
end
# Define initial-value problem.
u0 = [1.0, 1.0]
p = (α=[1.5], β=[1.0])
tspan = (0.0, 10.0)

model = LotkaVolterra(ModelParams(;u0, p, tspan, alg=Tsit5(), saveat=0.1))
sol = simulate(model);
```



And here we go, we have our model defined! ParametricModels.jl make the work easy, but as an exercise, you can try to write the same model with OrdinaryDiffEq.jl.

Let's now plot the model output.
We first need to import some plotting utilities.
For plotting, I tend to use PyPlot.jl, which is a wrapper around the Python library matplotlib. I find it much more convenient than the Julia package `Plots.jl`, in the sense that it is much more complete.

```julia
using PyPlot # to plot 3d landscape
using PyCall
```




Now we can simply plot the model output without noise, 
```julia
fig = PyPlot.figure()
PyPlot.plot(sol')
display(fig)
```

{{< figure src="figures/ABC_inference_5_1.png"  >}}


We now use `sol` to generate synthetic data containing some noise (in mathematical terms, we call this a Gaussian white noise, which follows $\mathcal{N}(0,\sigma)$ where $\sigma = 0.8$. We'll pretend that this synthetic data is our empirical data.
```julia
σ = 0.8
odedata = Array(sol) + σ * randn(size(Array(sol)))
fig = figure()
PyPlot.plot(odedata')
display(fig)
```

{{< figure src="figures/ABC_inference_6_1.png"  >}}


The model and the synthetic data are ready, let's get started with ABC!

## Approximate Bayesian computation

For ABC, one needs to define a distance $\rho$ that measures the distance between the model output $\hat\mu$ (in the picture below, $\mu_i$) and the empirical data $\mu$.

![source: Wikipedia](https://upload.wikimedia.org/wikipedia/commons/b/b9/Approximate_Bayesian_computation_conceptual_overview.svg)

In our case, the empirical data available (`odedata`) allows us to explicitly define the likelihood of the model given the data. For additive Gaussian noise, the likelihood of the model is given by
$$
\begin{split}
    p(\by_{1:K} | \theta, \M) &= \prod_{i=1}^K p(y_{i} | \theta, \M)\\
                        &= \prod_{k=1}^K \frac{1}{\sqrt{(2\pi)^d|\Sigma_y|}} \exp \left(-\frac{1}{2} \epsilon_k^{T} \Sigma_y^{-1} \epsilon_k \right)
\end{split}
$$
where $\epsilon_k \equiv \epsilon(t_k) = y(t_k) - h\left(\M(t_k, \theta)\right)$ and $\Sigma_y = \sigma^2 I$ in our case.

As a distance, we'll simply use the negative log of the likelihood!

So let's translate all this in Julia code. 

`ApproxBayes` requires as input a function `simfunc(params, constants, data)`. It should return the distance value, as well as a second value that may be used for logging. To make things simple, we'll return `nothing` for this second value. 

Because `ApproxBayes` requires that `params` is an array (`params <: AbstractArray`), we further use a special trick relying on the function `Optimiser.destructure`. This utility function allows to generate a function `re` that can recover a `NamedTuple` from a vector. Very useful in combination with the utility macro `@unpack`!

```julia
_, re = destructure(p)
function log_likelihood(p; odedata=odedata)
    p_tuple = re(p)
    pred = simulate(model, p=p_tuple)
    return sum([loglikelihood(MvNormal(zeros(2), LinearAlgebra.I * σ^2), odedata[:,i] - pred[:,i]) for i in 1:size(odedata,2)])
end
function simfunc(params, constants, data)
    ρ = - log_likelihood(params, odedata=data)
    return ρ, nothing
end

# testing function
simfunc([1.0, 1.2], nothing, odedata)
```

```
(1403.0830673958446, nothing)
```





Now that the distance function is implemented, we define the priors on the parameters $\alpha, \beta$
```julia
priors = [truncated(Normal(1.1, 1.), 0.5, 2.5), # for α and β
            truncated(Normal(1.1, 1.), 0., 2.)]
num_samples = 500
max_iterations = 1e5
ϵ = 2.0
abcsetup = ABCRejection(simfunc,
                        length(priors),
                        ϵ,
                        ApproxBayes.Prior(priors);
                        nparticles=num_samples,
                        maxiterations=max_iterations)

abcresult = runabc(abcsetup, odedata, progress=true, parallel=true)
```

```
Preparing to run in parallel on 5 processors
Number of simulations: 1.00e+05
Acceptance ratio: 5.00e-03

Median (95% intervals):
Parameter 1: 1.51 (1.48,1.53)
Parameter 2: 1.00 (0.86,1.16)
```





And here we go: `abcresult` informs us that the ABC inference could indeed recover the parameters that generated the data!

## Visualizing the inference process
To better understand what has happened, let's plot the parameters that have not been accepted by the sampler, together with the real likelihood landscape.

As a first step, let's calculate the real likelihoood landscape.
```julia

αs = range(1.45, 1.55, length=100)
βs = range(0.8, 1.3, length=100)
likelihoods = Float64[]
for α in αs
    for β in βs
        p = [α, β]
        push!(likelihoods, log_likelihood(p))
    end
end

likelihoods = exp.(likelihoods); # exponentiating, to make it visually nicer
```




We now define an `Axes3D`, as you would do it in Python:
```julia
# Plotting 3d landscape
fig = plt.figure()
ax = Axes3D(fig, computed_zorder=false);
```




We'll import numpy into Julia, to easily construct the meshgrid required by the matplotlib function `plot_surface`.
```julia
np = pyimport("numpy") # used for plotting
X, Y = np.meshgrid(αs, βs)


# fig.savefig("perturbed_p.png", dpi = 300, bbox_inches="tight")
ax.plot_surface(X .|> Float64,
    Y .|> Float64,
    reshape(likelihoods, (length(αs), length(βs))) .|> Float64,
    edgecolor="0.7",
    linewidth=0.2,
    cmap="Blues", zorder=-1)
ax.set_xlabel(L"p_1")
ax.set_ylabel(L"p_2")
ax.set_zlabel("Likelihood", x=2)
ax.set_xticks([])
ax.set_yticks([])
ax.set_zticks([])

# make the panes transparent
ax.xaxis.set_pane_color((1.0, 1.0, 1.0, 0.0))
ax.yaxis.set_pane_color((1.0, 1.0, 1.0, 0.0))
ax.zaxis.set_pane_color((1.0, 1.0, 1.0, 0.0))
# make the grid lines transparent
ax.xaxis._axinfo["grid"]["color"] = (1.0, 1.0, 1.0, 0.0)
ax.yaxis._axinfo["grid"]["color"] = (1, 1, 1, 0)
ax.zaxis._axinfo["grid"]["color"] = (1, 1, 1, 0)

fig.tight_layout()
fig.set_facecolor("None")
ax.set_facecolor("None")

for p in eachrow(abcresult.parameters)
    ax.scatter(p[1], p[2], exp.(log_likelihood(p)), c="tab:red", zorder=100, s = 0.5)
end
p = abcresult.parameters[1,:]
ax.scatter(p[1], p[2], exp(log_likelihood(p)), c="tab:red", zorder=100, s = 2., label="Accepted simulation")
ax.legend()
display(fig)
```

{{< figure src="figures/ABC_inference_11_1.png"  >}}

Cool, right?