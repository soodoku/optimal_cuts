Quantization is an optimization problem ~ Minimize MSE/MAE given the
number of bins by finding the optimal bin edges. Options:

1.  Solve the optimization problem

2.  Solve it via CART (which is ~ solving the said problem)

3.  Use percentile binning (~ probability density)

4.  Use clustering methods like K-Means

5.  If the maximum number of bins was not fixed, we could use popular
    heuristic solutions for inferring k, e.g., Freedman-Diaconis and
    Sturges, and then we could use an optimization algorithm to find the
    optimal bin edges. Or we could use DBScan, etc.

Let’s generate some normally distributed data.

    # Set the seed for reproducibility
    set.seed(123)

    # Simulate data from a unit normal distribution
    data <- rnorm(10000)

### 1. Solve the optimization problem.

    ## Objective Function
    # Define the objective function to minimize the MSE
    mse_objective <- function(bin_edges) {
      bins <- cut(data, breaks = bin_edges, include.lowest = TRUE)
      binned_means <- tapply(data, bins, mean)
      mse <- mean((data - binned_means[bins])^2)
      return(mse)
    }

    # Set the number of bins
    k <- 15

    # Set the lower and upper bounds for the bin edges
    lower_bound <- min(data)
    upper_bound <- max(data)

    # Add a small amount of noise to ensure unique bin edges
    epsilon <- 1e-10
    bin_edges <- seq(lower_bound, upper_bound, length.out = k+1) + runif(k+1, -epsilon, epsilon)

    # Define the optimization problem
    optim_result <- optim(par = bin_edges[-1], fn = mse_objective, method = "SANN")

    # Get the optimal bin edges
    optimal_bin_edges <- c(lower_bound, sort(optim_result$par), upper_bound)

    # Calculate the binned data based on the mean of each bin
    bins <- cut(data, breaks = optimal_bin_edges, include.lowest = TRUE)

    # Calculate the bin means --- it can have NAs as some bins may have 0 entries
    bin_means <- tapply(data, bins, mean)

    # Map data points to the corresponding bin means
    binned_data <- bin_means[as.numeric(bins)]

    # Calculate the mean squared error (MSE)
    mse <- mean((data - binned_data)^2)
    cat("Mean Squared Error (MSE) for Optimal with K of 15:", mse, "\n")

    ## Mean Squared Error (MSE) for Optimal with K of 15: 0.04129698

### 2. CART solution [here](tree_split.ipynb) as R flakes. Bug in rpart.

### 3. Percentile binning

    # Set the number of percentiles (bins)
    k <- 15

    # Split the data into k percentiles
    percentile_bins <- cut(data, breaks = quantile(data, probs = seq(0, 1, length.out = k + 1)), include.lowest = TRUE, labels = FALSE)

    # Calculate the bin values as the mean within each percentile
    bin_values <- tapply(data, percentile_bins, mean)

    # Assign each data point to the corresponding bin value
    binned_data <- bin_values[percentile_bins]

    # Calculate the mean squared error (MSE)
    mse <- mean((data - binned_data)^2)

    # Print the MSE
    cat("Mean Squared Error (MSE) for percentile =", mse, "\n")

    ## Mean Squared Error (MSE) for percentile = 0.0242557

### 4. K-Means

    # Apply k-means clustering
    k <- 15  # Number of clusters
    kmeans_result <- kmeans(data, centers = k)

    # Get the cluster means
    cluster_means <- kmeans_result$centers

    # Assign each data point to the nearest cluster mean
    nearest_cluster <- kmeans_result$cluster

    # Assign to bin mean
    binned_data <- cluster_means[nearest_cluster]

    # Calculate the Mean Squared Error (MSE)
    mse <- mean((data - binned_data)^2)

    # Print the MSE
    cat("Mean Squared Error (MSE) for kmeans for k = 15 =", mse)

    ## Mean Squared Error (MSE) for kmeans for k = 15 = 0.0106888

### 5. Using Freedman Diaconis Rule but with linear binning (which can result in bins w/ 0 entries) yield excellent results (though note the number of bins).

    library(MASS)

    # Calculate the number of bins using the Freedman-Diaconis rule
    num_bins <- nclass.FD(data)
    cat("Number of bins:", num_bins, "\n")

    ## Number of bins: 62

    # Calculate the bin edges
    bin_edges <- seq(min(data), max(data), length.out = num_bins + 1)

    # Calculate the binned data based on the mean of each bin
    bins <- cut(data, breaks = bin_edges, include.lowest = TRUE)

    # Calculate the bin means --- it can have NAs as some bins may have 0 entries
    bin_means <- tapply(data, bins, mean)

    # Map data points to the corresponding bin means
    binned_data <- bin_means[as.numeric(bins)]

    # Calculate the mean squared error (MSE)
    mse <- mean((data - binned_data)^2)
    cat("Mean Squared Error (MSE) for Freedman-Diaconis binning:", mse, "\n")

    ## Mean Squared Error (MSE) for Freedman-Diaconis binning: 0.001284303

Let’s find the optimal number of bins using Sturges’ method. And then do
linear binning for MSE.

    # Calculate the number of bins using Sturges' formula
    num_bins <- ceiling(log2(length(data)) + 1)

    cat("Number of bins:", num_bins, "\n")

    ## Number of bins: 15

    # Calculate the bin edges using the range of the data
    bin_edges <- seq(min(data), max(data), length.out = num_bins + 1)

    # Slightly diff. way ...
    bin_means <- sapply(1:(length(bin_edges) - 1), function(i) {
      mean(data[data >= bin_edges[i] & data < bin_edges[i + 1]])
    })

    binned_data <- sapply(data, function(value) {
      bin_index <- findInterval(value, bin_edges, rightmost.closed = TRUE)
      bin_means[bin_index]
    })

    mse_sturges <- mean((data - binned_data)^2)
    print(paste("Mean Squared Error (MSE) for Sturges' formula:", mse_sturges))

    ## [1] "Mean Squared Error (MSE) for Sturges' formula: 0.0213204737111141"

DBScan.

DBScan is a disaster. Not only is it in that hyperparams like eps etc.
need to be carefully chosen, we have to deal with noise points and find
their bins.

    ### DBScan
    library(dbscan)  # Load the dbscan package

    ## 
    ## Attaching package: 'dbscan'

    ## The following object is masked from 'package:stats':
    ## 
    ##     as.dendrogram

    # Convert data to a matrix format
    data_matrix <- matrix(data, ncol = 1)

    # Set the DBSCAN parameters
    eps <- 0.2  # Maximum distance between points in a cluster
    minPts <- 5  # Minimum number of points to form a cluster

    # Apply the DBSCAN algorithm
    dbscan_result <- dbscan(data_matrix, eps = eps, minPts = minPts)

    # Get the cluster assignments and noise points
    cluster_assignments <- dbscan_result$cluster
    noise_points <- dbscan_result$cluster == 0

    # Calculate the cluster means (excluding noise points)
    cluster_means <- tapply(data[!noise_points], cluster_assignments[!noise_points], mean)

    # Calculate the distance between each data point and its corresponding cluster mean
    distances <- mapply(function(d, b) (d - b)^2, data[!noise_points], cluster_means[cluster_assignments[!noise_points]])

    # Convert noise data to a matrix format
    noise_data_matrix <- matrix(data[noise_points], ncol = 1)

    # Apply the DBSCAN algorithm on noise data
    noise_to_cluster <- dbscan::dbscan(noise_data_matrix, eps = eps, minPts = 1)$cluster
    noise_to_cluster_means <- tapply(data[noise_points], noise_to_cluster, mean)

    # Calculate the distance between each noise point and its assigned cluster mean
    noise_distances <- mapply(function(d, b) (d - b)^2, data[noise_points], noise_to_cluster_means[noise_to_cluster])

    # Calculate the mean squared error (MSE)
    mse <- mean(c(distances, noise_distances))
    cat("Mean Squared Error (MSE) with DBScan:", mse, "\n")

    ## Mean Squared Error (MSE) with DBScan: 0.9928345
