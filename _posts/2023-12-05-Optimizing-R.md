# Optimizing R Code

Recently, I've been trying to plan out how to move [libaffy](https://gene.moffitt.org/libaffy/) to R. Most of the package is not needed anymore thanks to the excellent [affy](https://bioconductor.org/packages/release/bioc/html/affy.html) R/Bioconductor package. But IRON (iterative rank-order normalization) is popular at my institution so a R port is on the table. Part of this planning is optimization - the goal of libaffy was to create optimized code for processing Affymetrix GeneChip data. To convert this into very slow R seems contrary then.

## Options
I decided to perform some experiments to determine what the "right" approach is to optimize my code in R. Meaning the code should work over large datasets, taking into account (possibly) memory and compute. As I understand it, there are several approaches to this problem:

- **All R**: This approach means using the optimized numerical routines of R, with as much vectorized operations (apply, sweep, etc) as possible to keep it performant. The benefit is that there are fewer dependencies and needs to compile.
- **Optimized R**: The second approach is to consider packages like `matrixStats` to perform optimized matrix-level operations. Since most of libaffy is based on fairly basic numerical routines, this seems a reasonable tradeoff between writing my own low-level code and using R directly.
- **Rcpp**: An excellent C++ interface for R. It does a lot of the interfacing and provides some very nice helpers to get C routines implemented. Given that the routines are C, this should be much faster, but with a nice wrapper for R.
- **R/C Interface**: There is a lower level interface to R/C which involves using a number of C macros, protecting memory allocations and other steps. This is as good as it gets for executing code (minimal additional overhead), it is also much harder to write/maintain.

Right away this indicates that any hope I have of maintaining one code base (libaffy C and libaffy R) is not ideal. While it is possible to unbox various data structures then call straight C calls (reusing the same library), there is code and performance overhead for doing this. Since R has become one of the de facto tools for bioinformatics, it seems that a move (vs. code sharing) is in order.

## Example
I chose mean, median and Tukey's BiWeight as the examples to implement in various ways to explore the benefits. Mean and median are easy, although with partial ordering medians it can be optimized. Tukey's Biweight is fairly self-contained but does require two iterations of medians which can be expensive. Plus, the `affy` package already implements what turns out to be a fairly efficient version (`affy::tukey.biweight()`). This is also a good example since we can simulate large data sets with many different "probesets", or row indices.

The dataset is a simple simulated dataset of 100 "arrays" (columns) by 1000000 "probes" (rows). A series of "probesets" are defined as row indices. The code to generate these is:

```{r}
generate_dataset <- function(n=100, p = 1000000, np = 50000) {
  d <- matrix(data = 1000 * rexp(n*p), ncol=n, nrow=p)
  probes <- sapply(1:np,\(.x) {as.integer(runif(n=15,min=1,max=p))},simplify=FALSE)
  probes <- setNames(
    probes,
    nm = sprintf("%05d_at", as.integer(runif(n=np, min=10000,max =50000)))
  )
  list(data = d, probes = probes)
}
```

## Initial Experiment
To get an early read on results and to test the process, I started with a smaller dataset:
```r
> d<-generate_dataset(n=10,p=1000,np=100)
```

A quick summary of the results: Optimized R is the most efficient, followed by the Rcpp version. The R-code based approaches (apply) were the slowest. Of note, `purrr::map` is generally a tad bit slower than sapply (but that was not the purpose of the experiment).

### Means

|expr                                                                                                   |      min|        lq|      mean|    median|        uq|       max| neval|
|:------------------------------------------------------------------------------------------------------|--------:|---------:|---------:|---------:|---------:|---------:|-----:|
|sapply(d$probes, function(.x) {     colMeans(d$data[.x, ]) }, simplify = FALSE)                        |  214.840|  223.5525|  230.6885|  228.4725|  237.2875|   272.814|   100|
|sapply(d$probes, function(.x) {     apply(d$data[.x, ], 2, mean) }, simplify = FALSE)                  | 2343.191| 2463.3620| 2795.6420| 2497.3305| 2565.4110| 12769.860|   100|
|sapply(d$probes, function(.x) {     apply(d$data[.x, ], 2, libaffyperftest::mean) }, simplify = FALSE) | 2334.294| 2487.9825| 2712.8142| 2529.8230| 2560.6345| 12119.846|   100|
|sapply(d$probes, function(.x) {     libaffyperftest::col_means(d$data[.x, ]) }, simplify = FALSE)      |  479.946|  499.4005|  513.5799|  511.0035|  526.1940|   570.392|   100|
|sapply(d$probes, function(.x) {     matrixStats::colMeans2(d$data[.x, ]) }, simplify = FALSE)          |  200.039|  207.7470|  217.0446|  214.8810|  222.4455|   258.177|   100|
|purrr::map(d$probes, function(.x) {     colMeans(d$data[.x, ]) })                                      |  227.960|  239.2555|  350.5229|  247.5990|  258.1975| 10228.803|   100|

### Medians
|expr                                                                                                     |      min|       lq|      mean|    median|        uq|       max| neval|
|:--------------------------------------------------------------------------------------------------------|--------:|--------:|---------:|---------:|---------:|---------:|-----:|
|sapply(d$probes, function(.x) {     apply(d$data[.x, ], 2, median) }, simplify = FALSE)                  | 2902.964| 3039.146| 3211.5386| 3099.5795| 3155.3805| 12629.230|   100|
|sapply(d$probes, function(.x) {     apply(d$data[.x, ], 2, libaffyperftest::median) }, simplify = FALSE) | 2937.199| 3039.883| 3622.1544| 3116.3690| 3214.1540| 22920.025|   100|
|sapply(d$probes, function(.x) {     apply(d$data[.x, ], 2, naive_median1) }, simplify = FALSE)           | 3008.416| 3128.854| 3296.6329| 3190.9685| 3253.4935| 12945.996|   100|
|sapply(d$probes, function(.x) {     apply(d$data[.x, ], 2, naive_median2) }, simplify = FALSE)           | 3177.418| 3275.224| 3365.5322| 3345.0055| 3404.0865|  3786.965|   100|
|sapply(d$probes, function(.x) {     apply(d$data[.x, ], 2, naive_median3) }, simplify = FALSE)           | 2352.949| 2454.854| 3079.5481| 2507.1705| 2559.9785| 12334.071|   100|
|sapply(d$probes, function(.x) {     libaffyperftest::col_medians(d$data[.x, ]) }, simplify = FALSE)      | 1129.140| 1164.626| 1189.3936| 1179.7955| 1211.1810|  1312.697|   100|
|sapply(d$probes, function(.x) {     matrixStats::colMedians(d$data[.x, ]) }, simplify = FALSE)           |  306.926|  318.529|  425.9756|  325.8475|  341.3455|  9807.938|   100|
|purrr::map(d$probes, function(.x) {     matrixStats::colMedians(d$data[.x, ]) })                         |  319.800|  340.013|  355.8661|  353.9735|  365.6585|   440.381|   100|

### Tukey
|expr                                                                                                             |       min|        lq|      mean|    median|        uq|        max| neval|
|:----------------------------------------------------------------------------------------------------------------|---------:|---------:|---------:|---------:|---------:|----------:|-----:|
|sapply(d$probes, function(.x) {     apply(d$data[.x, ], 2, affy::tukey.biweight) }, simplify = FALSE)            | 18.694852| 19.580452| 21.442210| 19.804455| 22.386656| 100.769595|   100|
|sapply(d$probes, function(.x) {     apply(d$data[.x, ], 2, libaffyperftest::tukey_biweight) }, simplify = FALSE) |  4.581873|  4.745196|  4.926748|  4.794192|  4.856020|   8.676707|   100|
|sapply(d$probes, function(.x) {     apply(d$data[.x, ], 2, matrixStats_tukeybw_vector) }, simplify = FALSE)      |  5.801090|  6.018472|  6.363052|  6.097090|  6.166708|   9.681248|   100|
|sapply(d$probes, function(.x) {     affy::tukeybiweight(d$data[.x, ]) }, simplify = FALSE)                       | 19.101244| 19.806075| 21.080633| 20.040083| 22.658957|  42.038407|   100|
|sapply(d$probes, function(.x) {     matrixStats_tukeybw_matrix(d$data[.x, ]) }, simplify = FALSE)                |  1.520444|  1.599185|  1.700660|  1.629586|  1.672554|   5.077030|   100|
|sapply(d$probes, function(.x) {     libaffyperftest::col_tukey_biweights(d$data[.x, ]) }, simplify = FALSE)      |  2.538720|  2.630724|  2.749495|  2.671253|  2.709936|   6.819407|   100|
|purrr::map(d$probes, function(.x) {     matrixStats_tukeybw_matrix(d$data[.x, ]) })                              |  1.569152|  1.645720|  1.682940|  1.679504|  1.711750|   1.860047|   100|

## Conclusions

### Exclusions
After my experiments, there are two options to rule out: Optimized R and R/C interface. 

Optimized R does not make sense for my purposes. While it is incredibly powerful and useful, it took a fair amount of time to implement and it partitions up the problem differently than one might have originally thought of the algorithm, in terms of understandability. It was faster than straight R, but not by as much as I'd hoped. 

The R/C Interface does not make sense. There is significant code changes to manage the R interface within the C code. This also limits readability. Based on my experiences with Tukeys BiWeight, I was able to convert libaffy C code to Rcpp code in 30 minutes. That has a significant value, and I believe it also maintains a fair level of understandability.

### Design Patterns
One of the things that emerged from the testing (and that presumably matrixStats learned long ago) is that iterative R function calling is not as efficient as a full matrix application of a function. Meaning, applying the median across columns in R is slower than C code to apply median across columns. This means that any apply-type activity would be best served as C code directly. However, it does appear that we can develop entry points for vectors (e.g., rows or columns) within Rcpp and then implement a matrix-wise apply in Rcpp. This allows us to avoid the repeated R interpretation (all in C) while still allowing the vectorized version.

For instance, when considering Tukey's Biweight, it is convenient to have a vectorized version. Then, if you didn't have a massive dataset, you could just `apply(m,2,tukey)` or if performance was an issue, `col_tukeys(m)`. This could be used effectively to provide both options, with relatively low coding overhead for the col option.

### Tradeoffs
Rcpp offered good performance but not the best. Optimized R showed the best results since it uses C underneath (and is, obviously, heavily optimized). In terms of readability and coding, it seems that Rcpp provides the benefits of C-based speedup without the complexity of straight C implementation. 