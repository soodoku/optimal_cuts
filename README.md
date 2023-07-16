## Optimal Binning

Quantization is an optimization problem \~ Minimize MSE/MAE given the number of bins by finding the optimal bin edges. Options:

1.  Solve the optimization problem

2.  Solve it via CART (which is \~ solving the said problem)

3.  Use percentile binning (\~ probability density)

4.  Use clustering methods like K-Means

5.  If the maximum number of bins was not fixed, we could use popular heuristic solutions for inferring k, e.g., Freedman-Diaconis and Sturges, and then we could use an optimization algorithm to find the optimal bin edges. Or we could use DBScan, etc.

In a couple of notebooks, I walk through the options. 
For #1, #3, #4, and #5, see [R nb](https://github.com/soodoku/optimal_cuts/blob/main/optimal_cuts.md). For #5, see [the python nb](https://github.com/soodoku/optimal_cuts/blob/main/tree_split.ipynb) (R flakes).
 
