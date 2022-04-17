Parallel Processing in R
================
Sam Csik
April 18, 2022

-   [Things to remember before we get
    started:](#things-to-remember-before-we-get-started)
-   [0. Install & load packages](#0-install--load-packages)
-   [I. Create a sloowwww function (for
    testing)](#i-create-a-sloowwww-function-for-testing)
-   [II. Using the `{parallel}` package](#ii-using-the-parallel-package)
-   [III.](#iii)

*The following code and content was borrowed/adapted from Danielle
Ferraro’s R-Ladies SB workshop, [A Gentle Introduction to Parallel
Processing in
R](https://danielleferraro.github.io/rladies-sb-parallel/#1)…who
borrowed code/exercises from Grant McDermott’s lecture notes, [Parallel
programming](https://raw.githack.com/uo-ec607/lectures/master/12-parallel/12-parallel.html).
Thanks to both Grant and Danielle for putting together some awesome
teaching resources!*

### Things to remember before we get started:

-   **Use parallel processing when** (a) your task is embarrassingly
    parallel i.e. if your analysis could easily be separated into many
    identical but separate tasks that do not rely on one another,
    and (b) when your tasks are computationally intensive and take time
    to complete.

-   **Do not use parallel processing if** (a) it’s not the right tool
    (e.g. maybe you’re memory limited), or (b) it might not be efficient
    (there is computational overhead for setting up, maintaining, and
    terminating multiple processors — for small processes, the overhead
    cost may be greater than the speedup)

-   Always **profile** your code first (i.e. find the bottleneck and try
    to eliminate it)

-   Parallel code can be implemented in two ways: **forking** or
    **sockets** (we’ll explore both below)

    -   **forking** = faster but doesn’t work on Windows machines and
        may cause problems in IDEs

    -   **sockets** = a bit slower but works across operating systems
        and in IDEs

### 0. Install & load packages

#### Install packages (if needed)

``` r
# install.packages("foreach")
# install.packages("doParallel")
# install.packages("furrr") 
# install.packages("future.apply")
# install.packages("tictoc")
```

#### Load packages

``` r
library(tidyverse) # for some data wrangling
library(tictoc) # for timing how long our code takes to run
library(parallel) # a base R package for parallel processing
```

### I. Create a sloowwww function (for testing)

This `slow_square()` function calculates the square of a number, saves
the value and the corresponding squared value to a data frame, then put
R to sleep for 2 seconds.

``` r
slow_square <- function(x) {
  x_squared <- x^2 
  d <- tibble(value = x, 
              value_squared = x_squared)
  Sys.sleep(2)
  return(d)
}
```

### II. Using the `{parallel}` package

**The `{parallel}` package provides parallel versions of base R
`apply()` functions**

We’ll be using the following functions from the `{parallel}` package:

-   `detectCores()`

-   `mclapply()` (forking method)

-   `parLapply()`, `makePSockcluster()`, and `clusterEvalQ()` (socket
    method)

#### Detecting cores

First, use `parallel::detectCores()` to determine how many cores your
computer has (mine has 8)

``` r
detectCores()
```

**You decide how many cores to use** when parallel processing, though
many suggest **reserving one core** for running other background
processes. For extremely computationally-intensive tasks, you may choose
to shut down all other applications on your computer and use all
available cores. Remember, there is **computational overhead** for
setting up, maintaining, and terminating multiple processors, so *take
care when using computers/servers with many cores available.*

#### `lapply()` vs. `mclapply()`

You may already be familiar with the base R function, `lapply()`, which
applies a function over the elements of a list or vector. `mcapply()` is
the parallel equivalent! Let’s test both using our `slow_square()`
function.

First up, `lapply()` which takes two arguments, a vector `X` and a
function, `FUN`, to be applied over each element of `X`. We’ll bind the
results into a tibble using `bind_rows()`.

``` r
tic("using lapply()")
lapply(X = 1:5, FUN = slow_square) %>% 
  bind_rows()
toc()
```

`lapply()`, which runs serially using 1 core, took (me) **10.686 sec**

Next up, `mclapply()` which takes the same first two arguments as
`lapply()`, `X` and `FUN`. We’ll also specify the number of cores to use
with `mc.cores`. **REMINDER: `mclapply()` will only work for Mac users.
Windows users, see the next section on `parLapply()`**

``` r
#.................your total number of cores - 1.................
n_cores <- detectCores() - 1 

#................parallelize using forking method................
tic("using mclapply()")
mclapply(X = 1:5, FUN = slow_square, mc.cores = n_cores) %>% 
  bind_rows()
toc()
```

`mclapply()`, which ran our code in parallel using the forking method
and n-1 cores (for me, that’s 7 cores) took **2.819 sec**

#### `parLapply()`

Windows users will need to use the `parLapply()` function

``` r
#......................some necessary setup......................
cluster <- makePSOCKcluster(n_cores) # make a local cluster (i.e. a collection of cores on our computer)
clusterEvalQ(cluster, library("tibble")) # export necessary pkgs into our cluster 

#................parallelize using socket method.................
tic("parLapply")
parLapply(cluster, 1:5, slow_square) %>% 
  bind_rows()
toc()
```

`parLapply()`, which ran our code in parallel using the socket method
and n-1 cores (for me, that’s 7 cores) was only slightly slower than
`mclapply()` at **2.886 sec**

### III.
