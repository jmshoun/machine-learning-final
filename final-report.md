# Machine Learning Final
J. Mark Shoun  
11/25/2016  

### Synopsis

In this document, I detail the model selection and fitting process for a machine learning model to predict the manner in which a weightlifting exercise was performed on the basis of sensor data. The final model has an estimated out-of-sample error rate of around 0.4%.

### Data Pre-Processing

#### Data Loading

Our first order of business is to load the data into R and partition it into train and test data sets. For this application, we'll do a 70/30 split between training and validation.


```r
library(dplyr)
library(magrittr)
library(caret)
library(xgboost)
library(reshape2)
library(scales)
```


```r
## Download train and test data sets
download.file("https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv",
              "weight-train.csv")
download.file("https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv",
              "weight-test.csv")

weight.supervised <- read.csv("weight-train.csv")
weight.test <- read.csv("weight-test.csv")

set.seed(12345)
train.rows <- createDataPartition(weight.supervised$classe, p=0.7, list=FALSE)
weight.train <- weight.supervised[train.rows, ]
weight.validation <- weight.supervised[-train.rows, ]
```

We'll do all of our model fitting with `weight.train`, saving the validation and test sets until the end of the process.

#### Variable Selection

Before we do any modeling, we should first do some pre-processing of the data. It turns out that the majority of the columns contain `NA`s for the vast majority of observations. This is because these rows represent quantities that are summaries of many observations within a window of time. Only those observations with `new_window == "yes"` (273 out of 13737 training observations) have the summary attributes. Since these attributes are so rarely observed (and since none of the test data observations have non-`NA` values for these attributes), we'll exclude them from the data.


```r
is_mostly_missing <- function(x, cutoff=0.95) {
    missing <- is.na(x) | (x == "")
    mean(missing) > cutoff
}

mostly.missing <- sapply(weight.train, is_mostly_missing)
mostly.missing.names <- names(weight.train)[mostly.missing]
```

There are some other variables that should be excluded from the data. The first column is just the observation number, and should be excluded. `user_name` is the name of the user who performed the exercise. In some circumstances, it could make sense to retain this variable, but we assume that our goal is to be able to classify new observations from new users who might not have been a part of the training corpus, so we exclude it. `raw_timestamp_part_1`, `raw_timestamp_part_2`, and `cvtd_timestamp` are when the observation was recorded, and will no no predictive power for out-of-sample observations, so we exclude them. Finally, `new_window` and `num_window` are ID variables that are only meaningful on the training, so they are excluded.


```r
control.names <- c("X", "user_name", "raw_timestamp_part_1", 
                   "raw_timestamp_part_2", "cvtd_timestamp", "new_window",
                   "num_window")
response.name <- "classe"
predictor.names <- setdiff(names(weight.train), 
                           c(mostly.missing.names, control.names, 
                             response.name))
```

#### Variable Transformation

We don't need to do any further transformation of the data because the output of the machine learning technique I'll be using is invariant to monotone transformations of the predictor variables. (That being said, I would obtain different results were I to perform PCA on the inputs.)

### Modeling

#### Modeling Strategy

Now, we can move on to the fun of fitting models. My go-to model choice for this type of application is gradient-boosted models as provided by the `xgboost` package. There are several dozen tuning parameters available in `xgboost`, but for the sake of expediency, I'll focus on `min.child.weight` (the minimum weight in the terminal nodes of each decision tree) and `lambda` (the L2 regularization penalty applied to the terminal node weights).

My strategy is as follows:

* Use prior knowledge of the package to select a few potential values for `min.child.weight` and `lambda`. Set `nrounds` (the number of trees in the model) and `eta` high enough so that test error will almost certainly have reach its minimum for any values of the tuning parameters. (Note that we're subsampling the training data on each round, which essentially eliminates the risk of overfitting.)
* For each combination of `min.child.weight` and `lambda`, use 5-fold cross-validation to estimate the out-of-sample classification performance of the model. My chosen metric for performance is multinomial log-loss (which is equivalent to the classification cross-entropy or log-likelihood). I prefer this metric to misclassificaiton error for a couple of reasons. First, the metric is continuous, unlike misclassification error, which can only take on a finite number of values. Second, the metric accounts for the confidence the model has in each of its predictions, not just the single most likely outcome. In other words, it distinguishes between a model that puts a 30% chance on the most likely outcome and a model that puts a 99% chance on the most likely outcome.
* Select the combination of tuning parameters with the best estimated performance, then fit a model to the entire training data set with a higher value of `nrounds` and lower value of `eta` to get an estimate of the out-of-sample classification performance of the final selected model. (The effect of our tuning parameters on fit quality should be fairly insensitive to raising `nrounds` and lowering `eta`. The smaller the value of `eta`, the better the model in the end, but the longer the training process takes.)

#### Tuning Parameter Evaluation

The R code that follows implements this strategy. First, we'll create a list with the train and test data sets for each cross-validation fold, with the data converted into the format required by the `xgboost` package.


```r
set.seed(1234)
NUM.FOLDS <- 5
train.folds <- createFolds(weight.train$classe, k=NUM.FOLDS)

create_xgb_data <- function(df) {
    data.matrix <- as.matrix(df[, predictor.names])
    if ("classe" %in% names(df)) {
        response = as.integer(df$classe) - 1
        xgb.DMatrix(data.matrix, label=response)
    } else {
        xgb.DMatrix(data.matrix)
    }
}
```

Next, we'll set up the grid of tuning parameters we'll evaluate. Note that we're defining `lambda.relative` instead of `lambda` directly. This is because of the way `lambda` is defined within the `xgboost` package -- we expect setting a uniform set of values for `lambda.relative` and then defining `lambda = lambda.relative * min.child.weight` to be more informative than fixing the values of `lambda` regardless of `min.child.weight`.


```r
parameter.tuning <- expand.grid(fold=1:5, min.child.weight=c(.2, .5, 1, 2, 5),
                                lambda.relative=c(.2, .5, 1, 2, 5)) %>%
    mutate(lambda = lambda.relative * min.child.weight)
```

Now, we'll evaluate our models. `cv.data` is a list of `xgb.DMatrix` data objects. `test_entropy` calculates the (base 2) cross-entropy of the model predictions on the test data, and `cv_test_entropy` fits a GBM model and calculates the cross-entropy on the test set.

The last line of code in this chunk actually does all of the model fitting and performance evaluation. Patience is essential -- this took a couple of hours to run on my laptop.


```r
cv.data <- lapply(1:NUM.FOLDS, function(k) {
    train.indices <- unlist(train.folds[-k])
    test.indices <- train.folds[[k]]
    list(train=create_xgb_data(weight.train[train.indices, ]),
         test=create_xgb_data(weight.train[test.indices, ]))
})

test_entropy <- function(model, test.data) {
    true.labels <- getinfo(test.data, "label")
    prediction.raw <- predict(model, test.data)
    prediction.matrix <- matrix(prediction.raw, ncol=5, byrow=TRUE)
    correct.indices <- cbind(1:length(true.labels), true.labels + 1)
    correct.predictions <- prediction.matrix[correct.indices]
    -1 * mean(log2(correct.predictions))
}

cv_test_entropy <- function(fold, min.child.weight, lambda) {
    message(sprintf("Fold: %d, MinChild: %f, Lambda: %f", 
                    fold, min.child.weight, lambda))
    model <- xgb.train(data=cv.data[[fold]]$train, nrounds=500,
                       objective="multi:softprob", eval.metric="mlogloss",
                       subsample=0.5, num.class=5, lambda=lambda,
                       min.child.weight=min.child.weight)
    test_entropy(model, cv.data[[fold]]$test)
}

parameter.tuning$metric <- mapply(cv_test_entropy,
                                  parameter.tuning$fold,
                                  parameter.tuning$min.child.weight,
                                  parameter.tuning$lambda)
```

#### Tuning Parameter Selection

Finally, we can summarize our results and plot them. The plot below is a heatmap of cross-validated model performance across our grid of tuning parameters.


```r
parameter.summary <- aggregate(metric ~ lambda.relative + min.child.weight, 
                               parameter.tuning, mean)

ggplot(parameter.summary) +
    aes(x=as.factor(lambda.relative), y=as.factor(min.child.weight)) +
    geom_tile(aes(fill=metric)) +
    geom_text(aes(label=sprintf("%.04f", metric))) +
    scale_fill_gradient2(low=muted("blue"), high=muted("red"), midpoint=0.03) +
    labs(x="Lambda (Relative)", y="Minimum Child Weight",
         title="Cross-Validated Model Performance") +
    theme_bw()
```

![](final-report_files/figure-html/unnamed-chunk-7-1.png)<!-- -->

It looks like our best bet is `min.child.weight=0.2` and `lambda=0.4`.

#### Final Model Fitting

Now, we can crank up `nrounds`, lower `eta`, and fit a final model to all of the training data.


```r
full.train <- create_xgb_data(weight.train)
full.validation <- create_xgb_data(weight.validation)
full.test <- create_xgb_data(weight.test)
model <- xgb.train(data=full.train, nrounds=5000, eta=0.04,
                   objective="multi:softprob", eval.metric="mlogloss",
                   subsample=0.5, num.class=5, lambda=0.4,              
                  min.child.weight=0.2)
validation.entropy <- test_entropy(model, full.validation)

validation.prediction.matrix <- predict(model, full.validation) %>%
    matrix(ncol=5, byrow=TRUE)
test.prediction.matrix <- predict(model, full.test) %>%
    matrix(ncol=5, byrow=TRUE)
```

### Model Performance

#### Overall Performance

The cross-entropy on the validation set is 0.0171. This is a pretty good result! Maximum entropy for a discrete variable with five levels is 2.3219 bits per observation. Essentially, this means that our model captures more than 99% of the information contained in the class labels.

How accurate is that in more normal terms? There are a couple of ways we can answer that. First, let's look at the confusion matrix.


```r
validation.predictions <- validation.prediction.matrix %>%
    apply(1, which.max) %>%
    factor(labels=LETTERS[1:5])

validation.confusion <- confusionMatrix(validation.predictions, 
                                        weight.validation$classe)
validation.confusion
```

```
## Confusion Matrix and Statistics
## 
##           Reference
## Prediction    A    B    C    D    E
##          A 1673    4    0    0    0
##          B    1 1134    2    0    0
##          C    0    1 1022    7    0
##          D    0    0    2  956    3
##          E    0    0    0    1 1079
## 
## Overall Statistics
##                                           
##                Accuracy : 0.9964          
##                  95% CI : (0.9946, 0.9978)
##     No Information Rate : 0.2845          
##     P-Value [Acc > NIR] : < 2.2e-16       
##                                           
##                   Kappa : 0.9955          
##  Mcnemar's Test P-Value : NA              
## 
## Statistics by Class:
## 
##                      Class: A Class: B Class: C Class: D Class: E
## Sensitivity            0.9994   0.9956   0.9961   0.9917   0.9972
## Specificity            0.9991   0.9994   0.9984   0.9990   0.9998
## Pos Pred Value         0.9976   0.9974   0.9922   0.9948   0.9991
## Neg Pred Value         0.9998   0.9989   0.9992   0.9984   0.9994
## Prevalence             0.2845   0.1935   0.1743   0.1638   0.1839
## Detection Rate         0.2843   0.1927   0.1737   0.1624   0.1833
## Detection Prevalence   0.2850   0.1932   0.1750   0.1633   0.1835
## Balanced Accuracy      0.9992   0.9975   0.9972   0.9953   0.9985
```

It looks like we achieved 99.64% accuracy on the validation data set. This is the first time we've used the validation data in the moeling process, so this should be an unbiased estimate of the out-of-sample predictive performance.

#### Test Set Performance

The predictions we obtained earlier for the 20 test cases (results not shown here) matched the actual class labels for 20 out of 20 observations.

### Conclusion

The model we fit appears to perform very well. Without knowing precisely the end use-case in mind for this model, it's impossible to say whether an error rate of around 0.4% is sufficiently low, but it seems likely that this would be the case.
