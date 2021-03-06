---
title: "Humana Competition"
author: "Kevin Kusuma"
date: "10/14/2020"
output: html_document
---

# Packages
```{r, include=F}
library(tidyverse)
library(caret)
library(janitor)
library(ROSE)
library(caTools)
library(doSNOW)
library(glmnet)
library(ROCR)
library(psych)
library(doSNOW)
cl <- makeCluster(7, type = "SOCK")
library(C50)
library(RANN)
library(rminer)
library(devtools)
library(DomoR)
DomoR::init('domo_instance', 'alpha_numeric_code') #the necessary inputs of this line are not included for proprietary purposes
library(dataPreparation)
library(xgboost)
```

# Data
```{r, Data}
# Pulling Data from DOMO
combined_data <- fetch('f686d532-ae94-4baf-b0be-6cab8c29db22') %>% 
  arrange(desc(Source))

id_and_source <- combined_data[ ,c(1,826)]

data <- combined_data %>% 
  filter(Source == 'Training') %>% 
  select(-person_id_syn, -Source, -ZIP_STRING, -FIPS, -FIPS.String, -State, -City, -cnty_cd, -ID) %>% 
  mutate(transportation_issues = factor(ifelse(transportation_issues==1, 'Yes','No'), levels = c("No", "Yes")),
         lang_spoken_cd = ifelse(lang_spoken_cd=='E','ENG',lang_spoken_cd),
         Cluster_2 = factor(Cluster_2),
         Cluster_3 = factor(Cluster_3),
         Cluster_4 = factor(Cluster_4),
         Cluster_5 = factor(Cluster_5)) 
```

# Data Cleaning & Preparation
```{r, Cleaning}
# Creating Dummy Variables
dummy_imputed <- dummyVars(transportation_issues~., data, fullRank = T) %>% 
  predict(newdata=data)

# Median Impute and Data Frame
dummy_imputed <- preProcess(dummy_imputed, method = c('medianImpute')) %>% 
  predict(newdata = dummy_imputed) %>% 
  data.frame()

# Removing Near Zero Variance Variables
registerDoSNOW(cl)
set.seed(212)
nzv_variables <- nearZeroVar(dummy_imputed, allowParallel = TRUE, foreach = TRUE)
on.exit(stopCluster(cl))

data_nzv <- dummy_imputed[ , -nzv_variables]
```

# Train/Test Partitioning
```{r, Train/Test}
# Adding back the target variable
data_nzv$transportation_issues <- data$transportation_issues

# Subseting the Data
set.seed(212)
partition <- createDataPartition(data_nzv$transportation_issues, p=0.7, list = F)
training_data <- data_nzv[partition, ]
testing_data <- data_nzv[-partition, ]

```

# Center and Scale
```{r, Center/Scale}
# Center and Scale
scales <- build_scales(dataSet = training_data[,-435])

scaled_data <- fastScale(dataSet = training_data[,-435], scales = scales)

scaled_data <- cbind(transportation_issues = training_data$transportation_issues,
                     scaled_data)
```

# Balancing the Data
```{r, Balance}
# Balancing the Data
n_glmnet <- scaled_data %>% 
  filter(transportation_issues=='Yes') %>% 
  nrow()

ovun_glmnet <- ovun.sample(transportation_issues~.,
                           data = scaled_data,
                           method = "under",
                           N = n_glmnet*2,
                           seed = 212)

balanced_glmnet <- ovun_glmnet$data
balanced_glmnet <- balanced_glmnet%>% 
  mutate(transportation_issues = factor(transportation_issues, levels=c('No','Yes')))
```

# GLMNET for Feature Selection
```{r, glmnet}
# GLMNET Model with Lasso
set.seed(212)
glmnet_model <- cv.glmnet(x = as.matrix(balanced_glmnet[ ,-1]), y = as.factor(balanced_glmnet$transportation_issues), 
                         alpha=1, family="binomial",
                         nfolds=7, type.measure = "auc", parallel = TRUE)

# Extracting the non-zero coefficients
glmnet_coef <- coef(glmnet_model) %>% as.matrix() %>% 
  data.frame() %>% 
  filter(X1!=0)

glmnet_coef$variable <- rownames(glmnet_coef)

glmnet_coef <- glmnet_coef %>% 
  filter(variable != '(Intercept)')

glmnet_training <- balanced_glmnet %>% 
  select(transportation_issues, glmnet_coef$variable)

# Looking at the variable importance
head(glmnet_coef, n = nrow((glmnet_coef)))
```

# Base Logistic Regression Model
```{r, Logistic}
registerDoSNOW(cl)
set.seed(212)
base_logistic <- train(transportation_issues~.,
                       glmnet_training,
                       method = 'glm',
                       metric = 'ROC',
                       trControl = trainControl(
                         method = "cv",
                         number = 7,
                         summaryFunction = twoClassSummary,
                         classProbs = T,
                         verboseIter = T))

on.exit(stopCluster(cl))


# ROC on the Training Data
p1 <- predict(base_logistic, training_data, type = 'prob')
colAUC(p1, training_data$transportation_issues)

# ROC on the Testing Data
p2 <- predict(base_logistic, testing_data, type = 'prob')
colAUC(p2, testing_data$transportation_issues)

```

# Gradient Boosting Machine
```{r, GBM}
# Time the code execution
start.time0 <- Sys.time()

registerDoSNOW(cl)
set.seed(212)
GBM_model <- train(transportation_issues~.,
                           glmnet_training,
                           method = 'gbm',
                           metric = 'ROC',
                           tuneGrid = expand.grid(
                             n.trees = 150,
                             interaction.depth = 2,
                             shrinkage = 0.1,
                             n.minobsinnode = 500),
                           trControl = trainControl(
                             method = "cv",
                             number = 7,
                             summaryFunction = twoClassSummary,
                             classProbs = T))

on.exit(stopCluster(cl))
# Total time of execution
total.time0 <- Sys.time() - start.time0
total.time0

# ROC on the Training Data
p7 <- predict(GBM_model, training_data, type = 'prob')
colAUC(p7, training_data$transportation_issues)

# ROC on the Testing Data
p8 <- predict(GBM_model, testing_data, type = 'prob')
colAUC(p8, testing_data$transportation_issues, plotROC = T)

```


