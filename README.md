# Midterm Report

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo=TRUE)
knitr::opts_chunk$set(warning=F)
knitr::opts_chunk$set(message=F)
```

## Introduction

While median wages are remaining stagnant, housing prices in Taipei, Taiwan are increasing exponentially. Using a historical data set of real estate valuation from the Sindian District and New Taipei City in Taiwan, we decided to explore which predictors had the most impact on housing prices in Taipei. 

We outlined our analysis plan so that our models excluded observations that had missing values. However, we did not have any observations with missing information and therefore a sensitivity analysis was not necessary. The variable “no” in the original dataset was dropped, as it was an observation id, which would not be relevant in our analysis. 

This dataset contains 414 observations, with the 6 predictors as follows:

* X1=the transaction date (for example, 2013.250=2013 March, 2013.500=2013 June, etc.)
* X2=the house age (unit: year) 
* X3=the distance to the nearest MRT station (unit: meter) 
* X4=the number of convenience stores in the living circle on foot (integer) 
* X5=the geographic coordinate, latitude. (unit: degree) 
* X6=the geographic coordinate, longitude. (unit: degree) 

The outcome is as follows: 

* Y= house price of unit area (10000 New Taiwan Dollar/Ping, where Ping is a local unit, 1 Ping = 3.3 meter squared) 

## Exploratory Data Analysis

```{r packages, echo=FALSE, include=FALSE}
library(caret)
library(readxl)
library(tidyverse)
library(glmnet)
library(ISLR)
library(corrplot)
library(pastecs)
library(earth)
library(splines)
library(mgcv)
library(knitr)
library(vip)
```

```{r data cleaning, message=F, include=FALSE}
taipei_data <- read_excel("Real_estate_valuation_data_set.xlsx") %>%
  janitor::clean_names()

taipei_data <- na.omit(taipei_data) 
taipei_data$no <- NULL

taipei_data  =
  taipei_data %>%
  rename(house_price = y_house_price_of_unit_area)
taipei_data  = 
 taipei_data %>%
  rename(transaction_date = x1_transaction_date,
         house_age = x2_house_age,
         distance_mrt = x3_distance_to_the_nearest_mrt_station,
         conv_stores = x4_number_of_convenience_stores,
         latitude = x5_latitude,
         longitude = x6_longitude
         )
```

#### Descriptive Statistics
```{r, echo=FALSE}
stat.desc(taipei_data) %>% round() %>%  kable(full_width = F, font_size=8)
```




```{r, include=FALSE}
### Designating the Predictors and Outcome
taipei_data <- na.omit(taipei_data) 
taipei_data$no <- NULL

x <- model.matrix(house_price~. ,taipei_data)[,-1]
y <- taipei_data$house_price
```


```{r, echo=FALSE, include=FALSE}
###Observing Correlation of Predictors
corrplot(cor(x))
```


### Scatterplot
```{r, echo=FALSE}
theme1 <- trellis.par.get()
theme1$plot.symbol$col <- rgb(.2, .4, .2, .5)
theme1$plot.symbol$pch <- 16
theme1$plot.line$col <- rgb(.8, .1, .1, 1)
theme1$plot.line$lwd <- 2
theme1$strip.background$col <- rgb(.0, .2, .6, .2)
trellis.par.set(theme1)
featurePlot(x, y, plot = "scatter", labels = c("","Y"),
            type = c("p"), layout = c(4, 2))
```

We can see that there may be a non-linear relationship between house price and distance to MRT station, and between house price and house age. We will test both of these terms independently and together as spline terms in GAM models. Transaction date and distance to convenience stores appear to be categorical predictors.

Some of the liminitations of these predictors include incorporating spatial data (latitude and longitude) as continuous variables, rather than incorporating a separate spatial analysis. Additionally, transaction date was treated as a continuous predictor, which may not be appropriate for these models as none of these models use survival analysis. 

## Models
We used a total of six different modeling techniques to estimate house prices. GAMS, MARS, and KNN were nonparametric while ridge regression, lasso regression, and linear regression were parametric. We started with the most flexible model and progressed to the most stringent. All six predictors mentioned in the Introduction were used in our analysis. The results are summarized in the following table:

### Summary of Models Used
Model | R^2 | RMSE 
------------- | ------------- | ------------- 
GAM  | 0.687 | N/A
MARS | 0.7005081 | 7.568432 
KNN | 0.6545951 | 8.208905 
Ridge Regression | 0.599 | 8.799873 
Lasso | 0.599 | 8.810508 
Linear Regression | 0.5824 | 8.782313 

#### Partitioning the Data Set 
Before we ran our analyses, we partitioned our dataset into training and test sets. 80% of observations were randomly sampled and partitioned into the training set while the remaining 20% was allocated for the test set. If a model's accuracy was tested using this method, it will be specified.

```{r, echo=FALSE, message=F}
#Partitioning the dataset
data(taipei_data)

## 75% of the sample size
smp_size <- floor(0.80 * nrow(taipei_data))

## set the seed to make your partition reproducible
set.seed(123)
train_taipei <- sample(seq_len(nrow(taipei_data)), size = smp_size)
train <- taipei_data[train_taipei, ]
test <- taipei_data[-train_taipei, ]
```

#### GAM
###### Assumptions: None (non-parametric approach)
###### Methods: We initially fit a GAM model with all six predictors and no spline terms. Then distance to MRT station was used as a spline term (forming model 2) and house age was used as a spline term (model 3). Finally, we created a model in which both predictors were splined (model 4). After running an ANOVA test and examining the R-Squared values, we confirmed that  GAM M4 has best R^2 value in addition to being statistically significant. The model uses house age and distance to the nearest MRT station as spline predictors, while also including transaction date, latitude, longitude, and the number of convenience stores in the living circle on foot.

```{r GAM calc, echo=FALSE, include=FALSE, message=F}
gam.m1 <- gam(house_price ~ transaction_date + house_age + distance_mrt + conv_stores + latitude + longitude, data = taipei_data)

gam.m2 <- gam(house_price ~ transaction_date + house_age + s(distance_mrt) + conv_stores + latitude + longitude, data = taipei_data)

#Spline term applied to distance to mrt station

gam.m3 <- gam(house_price ~ transaction_date + s(house_age) + distance_mrt + conv_stores + latitude + longitude, data = taipei_data)

#spline term applied to house age

gam.m4 <- gam(house_price ~ transaction_date + s(house_age) + s(distance_mrt) + conv_stores + latitude + longitude, data = taipei_data)

#Both house age and distance to mrt station are splined
```

```{r, echo=FALSE, include=FALSE}
anova(gam.m1, gam.m2, gam.m3, gam.m4, test = "F")
#summary(gam.m2)
#summary(gam.m3)
#summary(gam.m4)


plot(gam.m2)
plot(gam.m3)
```


##### MARS
###### Assumptions: None (non-parametric approach)
###### Methods: MARS requires two tuning parameters: product degree and number of terms. In order to minimize the MSE, 1 product degree and 6 retained terms were used (as depicted on the plot). The top three predictors that were most influential were distance to MRT station, latitutde, and house age. 

```{r, echo=FALSE, fig.width=4, fig.height=4}
set.seed(2)
mars_grid <- expand.grid(degree = 1:2, nprune = 2:6)
ctrl1 <- trainControl(method = "cv", number = 10)

mars.fit <- train(x, y, method = "earth", tuneGrid = mars_grid, trControl = ctrl1)

ggplot(mars.fit)

```

```{r, include=FALSE}
print(mars.fit$bestTune)
print(coef(mars.fit$finalModel))
print(mars.fit)
```

```{r marspredictors, echo=FALSE, include=FALSE}
p1 <- vip(mars.fit, num_features = 10, bar = FALSE, value = "gcv") + ggtitle("GCV")
p2 <- vip(mars.fit, num_features = 10, bar = FALSE, value = "rss") + ggtitle("RSS")
gridExtra::grid.arrange(p1, p2, ncol = 2)
``` 

##### KNN
###### Assumptions: None (non-parametric approach)
###### Methods: The training data was used to fit the initial kNN model. Then, the model RMSE was plotted against different values of k. From there, we decided that k=17 was the best tuning parameter. We then calculated the RSME using the actual test values vs the predicted test values. 

```{r, include=FALSE}

trainX <- train[,names(train) != "house_price"]
preProcValues <- preProcess(x = trainX,method = c("center", "scale"))
preProcValues
set.seed(1)
trctrl <- trainControl(method = "repeatedcv", number = 10, repeats = 5)
knn_fit <- train(house_price ~., data = train, method = "knn",
            trControl = trctrl,
            preProcess = c("center", "scale"),
            tuneLength = 10)
knn_fit
```
```{r, echo=FALSE, include=TRUE, fig.width=4, fig.height=4}
# Plot model error RMSE vs different values of k
ggplot(knn_fit)
```
```{r, include=FALSE}

# Best tuning parameter k that minimizes the RMSE
knn_fit$bestTune
# Make predictions on the test data
knn_predict <- knn_fit %>% predict(test)

# Compute the prediction error RMSE
RMSE(knn_predict, test$house_price)
```

##### Ridge Regression
###### Assumptions: Multicollinearity 
###### Limitations: Estimators will have greater bias than least squares regression, but lower variance (bias-variance tradeoff).
###### Methods: First we fit the ridge regression model using the training data. Lambda (i.e. the tuning parameter that controls the amount of shrinkage) was determined after using ten fold cross-validation and assessing which value minimized the MSE (lambda = 0.8007374).   

```{r, echo=FALSE, include=FALSE}
ctrl1 <- trainControl(method = "repeatedcv", number = 10, repeats = 5)
set.seed(123)
ridge.fit <- train(x, y,
                     method = "glmnet",
                     tuneGrid = expand.grid(alpha = 0, 
                                            lambda = exp(seq(-1, 10, length = 100))),
                   # preProc = c("center", "scale"),
                     trControl = ctrl1)
plot(ridge.fit, xTrans = function(x) log(x))
ridge.fit$bestTune
coef(ridge.fit$finalModel,ridge.fit$bestTune$lambda)
```
```{r ridgereg, echo=FALSE, include=FALSE}
p111 <- vip(ridge.fit, num_features = 10, bar = FALSE, value = "gcv") + ggtitle("GCV")
p211 <- vip(ridge.fit, num_features = 10, bar = FALSE, value = "rss") + ggtitle("RSS")
gridExtra::grid.arrange(p111, p211, ncol = 2)
``` 

##### Lasso
###### Assumptions: Only a small number of n variables are actually relevant to the outcome. The relevant variables must not be correlated with the irrelevant ones.
###### Limitations: Saturates easily, limited by small n value. If there are highly correlated variables/grouped variables, Lasso will select one from each group and ignore the others.
###### Methods: Just like in ridge regression, we fit the model using the training data. The lasso penalty term was determined after using ten fold cross-validation and assessing which value minimized the MSE (lambda = 0.3678794).   

```{r, include=FALSE}
set.seed(123)
lasso.fit <- train(x, y, method = "glmnet", tuneGrid = expand.grid(alpha = 1, lambda = exp(seq(-1, 5, length=100))), trControl = ctrl1)

plot(lasso.fit, xTrans = function(x) log(x))

lasso.fit$bestTune

coef(lasso.fit$finalModel,lasso.fit$bestTune$lambda)
```

```{r, echo=FALSE, fig.width=4, fig.height=4}
plot(lasso.fit, xTrans = function(x) log(x))
```

##### Linear
###### Assumptions: Linear relationship, Residuals normality, No or little multicollinearity, No auto-correlation, Homoscedasticity
###### Methods: We fit a linear model using all predictors. Then, we guaged via R^2 and MSE, how accurate the model was. Of all the predictors, the only non-significant predictor was longitude.

```{r, echo=FALSE, include=FALSE}
set.seed(2)
lm.fit <- train(x, y,
                method = "lm",
                trControl = ctrl1)
summary(lm.fit)
```

```{r lmpred, echo=FALSE, include=FALSE}
p11 <- vip(lm.fit, num_features = 10, bar = FALSE, value = "gcv") + ggtitle("GCV")
p21 <- vip(lm.fit, num_features = 10, bar = FALSE, value = "rss") + ggtitle("RSS")
gridExtra::grid.arrange(p11, p21, ncol = 2)
``` 

```{r calculations, include=FALSE}
sqrt(mean(residuals(lm.fit)^2))
summary(gam.m4)
summary(mars.fit)
sqrt(mean(residuals(ridge.fit)^2))
sqrt(mean(residuals(lasso.fit)^2))
```

## Conclusion
```{r, include=FALSE}
#Comparison using MSE
resamp <- resamples(list(lasso = lasso.fit, ridge = ridge.fit, lm = lm.fit, knn = knn_fit))
summary(resamp)
parallelplot(resamp, metric = "RMSE")
bwplot(resamp, metric = "RMSE")
```


All of the analyses conducted were well rationalized given the nature of the Taipei real estate dataset. To find the best possible model to predict real estate value (per unit area) there were two key metrics that were used. Primarily, the Mean Squared Error (MSE), which represents the average squared difference between the estimated value and actual value, was a key indicator in model selection. Secondarily, because the Generalized Adaptive Model (GAM) did not yield an MSE value, the R-Squared value was used to incorporate the GAM into the model selection process. An R-Squared value represents the proportion of variance in the outcome (price per unit area) that is explained by the predictor variables. When comparing the GAM to the MARS model using the R-Squared value, it underperforms in explaining the variance in the outcome. Therefore, when disregarding the GAM (since it does not have the highest R-Squared), the MSE of the other models can be used to select the best model since it is a better metric for model selection. MARS has the lowest MSE, thus is the best model to predict the cost of real estate in Taipei. One way to see if this model is biased (or if it has good external validity) based on the local real estate market, is to do a confirmatory analysis using the same model with data in other cities of similar land area, population, and economic size. 

