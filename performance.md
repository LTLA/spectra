---
layout: page
title: Performance
---

# Performance

This page shows some benchmark results of Spectra compared with other similar
libraries, based on the following environment setting:

<h4><span class="label label-success">Hardware</span></h4>

- CPU: Intel Core i7-4700HQ 2.40GHz x 8
- Memory: 8GB

<h4><span class="label label-success">Software</span></h4>

- OS: Fedora 24 64-bit
- Compiler: GCC 6.3.1 with flag `g++ -Wall -O2 -mtune=native -mavx`
- Spectra: version 0.5.0
- BLAS: OpenBLAS 0.2.19, single threaded
- ARPACK: ARPACK-NG 3.3.0
- R: version 3.3.2
- R packages
  - RSpectra 0.12-0
  - svd 0.4
  - irlba 2.1.2
  - microbenchmark 1.4.2.1

### Comparison with ARPACK

Spectra is designed to be comparable to or even faster than ARPACK.
The [benchmark](https://github.com/yixuan/spectra/tree/master/benchmark)
directory in Spectra's source code contains a program to benchmark both
ARPACK and Spectra on some simulated matrices.

Below shows the comparison results:

<img src="{{ '/img/benchmark-sym.png' | prepend: site.baseurl }}" class="img-responsive" />

<img src="{{ '/img/benchmark-gen.png' | prepend: site.baseurl }}" class="img-responsive" />

The symmetric eigen solver by Spectra is uniformly faster than ARPACK in all the
tested cases, and general eigen solver is almost identical to ARPACK.

### Comparison with SVD solvers

The R interface of Spectra,
[RSpectra](https://CRAN.R-project.org/package=RSpectra), is compared with
other SVD solvers in R.

<img src="{{ '/img/benchmark-svd-small.png' | prepend: site.baseurl }}" class="img-responsive" />

<img src="{{ '/img/benchmark-svd-large.png' | prepend: site.baseurl }}" class="img-responsive" />

The figures are generated using the following R code:

<pre><code class="r">library(Matrix)
library(RSpectra)
library(svd)
library(irlba)
library(microbenchmark)
library(dplyr)
library(ggplot2)

n = 1000
p = 500
nu = nv = k = 20

set.seed(123)
x = matrix(rnorm(n * p), n)
x[sample(n * p, floor(n * p / 2))] = 0
xsp = as(x, "dgCMatrix")

## For svd package
f = function(v) as.numeric(xsp %*% v)
tf = function(v) as.numeric(crossprod(xsp, v))
extx = extmat(f, tf, nrow(xsp), ncol(xsp))

res = microbenchmark(
    "svd"             = svd(x, nu, nv),
    "svds"            = svds(x, k, nu, nv, opts = list(tol = 1e-8)),
    "propack"         = propack.svd(x, k, opts = list(tol = 1e-8)),
    "trlan"           = trlan.svd(x, k, opts = list(tol = 1e-8)),
    "irlba"           = irlba(x, nu, nv, tol = 1e-8),
    "svds[sparse]"    = svds(xsp, k, nu, nv, opts = list(tol = 1e-8)),
    "propack[sparse]" = propack.svd(extx, k, opts = list(tol = 1e-8)),
    "trlan[sparse]"   = trlan.svd(extx, k, opts = list(tol = 1e-8)),
    "irlba[sparse]"   = irlba(xsp, nu, nv, tol = 1e-8),
    times = 10
)

dat = as.data.frame(res)
dat$type = ifelse(grepl("sparse", dat$expr), "Matrix type: Sparse",
                                             "Matrix type: Dense")
dat$expr = factor(gsub("\\[sparse\\]", "", dat$expr),
                  levels = c("svd", "irlba", "propack", "trlan", "svds"))
dat = dat %>% group_by(expr, type) %>% summarize(medtime = median(time) / 1e6)
dat$Package = ifelse(grepl("svds", dat$expr), "Spectra", "Other")

ggplot(dat, aes(x = expr, y = medtime)) +
    facet_wrap(~ type, scales = "free", ncol = 2) +
    geom_bar(aes(fill = Package), stat = "identity", width = 0.7) +
    scale_y_continuous("Median Elapsed time (ms)") +
    scale_x_discrete("Functions") +
    ggtitle(sprintf("Top-%d SVD on %dx%d Matrix", k, n, p)) +
    theme_bw(base_size = 20) +
    theme(plot.title = element_text(hjust = 0.5))
</code></pre>