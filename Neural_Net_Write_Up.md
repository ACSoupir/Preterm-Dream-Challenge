SDSU-DC Preterm Birth Prediction Challenge, Transcriptomics: Part 1
================
James Young, Dakota York, Bikram Das, Amrit Koiala, and Alex Soupir
South Dakota State University, Brookings, South Dakota, United States of America
16 August 2019

This document may be made public.

Background/Intro
================

The approach used in this document was merely first experience trial and error. Initial thoughts were to try several different approaches to see what would work and what would not work, trying to get the lowest RMSE from a train/test split of the data with gestational age provided. Using models with the lowest RMSE, and those with the lowest difference between the train/test split RMSE values, the unknown gestational age data is predicted in an ensemble. The gestational ages were weighted according to the differences between the train/test split RMSE values. Rather than using a single model, the thoughts were that an ensemble would balance those gestational ages that models were uncertain of.

Benefits and the Novelty of this approach is unknown. This exercise was the first introduction to predictive statistics for several in the group, therefore, where our model fits in the larger picture is unknown.

<!--
This document is an explanation of how data was processed and use to predict the gestational ages of women using whole blood transcriptome data. Both Python and R/R Studio was used for this analysis: Python was used for determining best probesets with XGBoost, and R/R Studio was used for training Keras models. 

Models were trained on gestational age probeset data, with a train/test split of 80%/20%. Models were looped through using a Keras model with 2 layers and 10,000 epochs: layer 1 containing values between 1 and 25, and layer 2 containing values between 0 and 50. Best models were determined based on those with a testing RMSE less than 4, and a difference between the train/test RMSE values of 1. Further, based on the difference between the train/test RMSE values, each model was given a weight, with the model resulting in the lowest difference having more weight than the model with a greater difference. The final gestational age values were submitted, receiving a submission RMSE of 5.6853.
-->
Methods
=======

Python codes were run using Google Colab and the on campus HPC cluster. The entirety of the probeset data was imported and split into 8 data frames with 100,000 features, and 1 data frame with the remaining 125,030 features. Each data frame was than looped through, splitting into training and testing data for the XGBoost model. XGBoost determined the important features of the data frame in predicting the gestational age. These features were then concatenated to the previous data frames best features for a list of features from *all* data frames that are important. With the list of all of the important features, the main probeset data frame was filtered to include only the important features, and then written to a *.csv* file, along with the submission data lacking the gestational age.

R was run on a personal computer. In order to run Keras and Tensorflow, first Tensorflow had to be installed using *conda install* through anaconda prompt. Then after *install.packages('keras')* in R, *install\_keras()* had to be executed. Since an NVIDIA GPU was not available, *version = 'gpu'* was not used as a parameter while installing Keras. (Chollet, Allaire, and others 2017)

Data from the XGBoost filtering was imported into R and then scaled based on the mean and standard deviation using the *scale()* fuction. The submission data was scaled based on the mean and standard deviation of the training data to maintain consistency in predictions from the models. Keras models were run with 1 and 2 layers. The first layer was looped through with 1 to 25 nodes and the second layer was looped through with 0 to 50 nodes to determine which network architecture performed best in predicting the gestational age of the test set.

Best models were determined based on the testing split RMSE and the difference between the training RMSE and testing RMSE. Of the models with a testing RMSE less than 4 and a difference between the training RMSE and the testing RMSE of 1, and ensemble was created weighing the gestational ages of each model based on the difference between the training and testing RMSE (smaller difference between training RMSE and testing RMSE was given higher weights than those models with greater differences).

### Data Filtering with Python

``` python
import numpy as np
np.random.seed(333)
import pandas as pd
import xgboost
from sklearn.model_selection import train_test_split

#Load dataset
df = pd.read_csv("HTA20_probeset_Train_NT.csv", index_col=0)
df = df.transpose()


#Separate the whole dataset into subsets of 100k with last set taking up all extras
X1 = df[df.columns[0:100000]]
X2 = df[df.columns[100000:200000]]
X3 = df[df.columns[200000:300000]]
X4 = df[df.columns[300000:400000]]
X5 = df[df.columns[400000:500000]]
X6 = df[df.columns[500000:600000]]
X7 = df[df.columns[600000:700000]]
X8 = df[df.columns[700000:800000]]
X9 = df[df.columns[800000:925030]]
XList = [X1, X2, X3, X4, X5, X6, X7, X8, X9]

y1 = df[df.columns[925032]]

#For each chunk in XList
for i in range(0, len(XList)):

  #Split that chunk into train and test set
    X_train, X_test, y_train, y_test = train_test_split(XList[i], y1,test_size=0.20,
                                                        random_state=333)
    eval_set = [(X_test, y_test)]
    
    #Run xgboost on chunk
    xgb = xgboost.XGBRegressor(n_estimators=30000, learning_rate=0.15, gamma=0,
                               subsample=1, colsample_bytree=1, max_depth=1)
    xgb.fit(X_train, y_train, early_stopping_rounds=50, eval_metric="rmse",
            eval_set=eval_set, verbose=True)
    
    #Extract only the features with importances > 0 and reform DataFrame
    features = pd.DataFrame(xgb.feature_importances_)
    
    XList[i] = pd.DataFrame(XList[i])
    XList[i] = XList[i].T
    featuresIndex = XList[i].index.values
    features.reset_index(drop=True, inplace=True)
    numbersIndex = features.index.values
    XList[i].reset_index(drop=True, inplace=True)
    frames = [XList[i], features]
    XList[i] = pd.concat(frames, axis=1)
    dictionarySamples = dict(zip(numbersIndex, featuresIndex))
    
    XList[i].rename(index=dictionarySamples, inplace=True)
    XList[i] = XList[i][(XList[i] != 0).all(1)]
    XList[i] = XList[i].drop(XList[i].columns[367], axis=1)
    XList[i] = XList[i].T
    
#Add all filtered chunkes back together
FinalTrainSet = pd.concat(XList, axis=1)

#Load the test set
FinalTestSet = pd.read_csv("HTA20_probeset_Test_NT.csv", index_col=0)
FinalTestSet = FinalTestSet.transpose()

#Filter out columns from the test set so it matches train set and save
FinalTestSet = FinalTestSet[FinalTrainSet.columns]
TestHandle = FinalTestSet.to_csv("filteredprobeset100k_test_7_4.csv",
                                 index = True, header=True)

#Add y1 back into train set and save
FinalTrainSet = pd.concat([FinalTrainSet, y1], axis=1)
TrainHandle = FinalTrainSet.to_csv('filteredprobeset100k_train_7_4.csv',
                                   index = True, header=True)
```

### Importing Keras Library to R

``` r
if (!requireNamespace("keras", quietly = TRUE))
  install.packages("keras")
library(keras)

#run the first time running Keras
install_keras()
```

### Importing and Scaling Data

Data that was previously determined using Python XGBoost to be of greatest importance in predicting gestational age was imported into R. Data with a gestational age provided was split for training the Keras models, with 80% used for training the model and 20% for testing or validating the model. The submission data was scaled based on the testing data.

``` r
#import training dataset
probs = read.csv('filteredprobeset15lr_train_7_4.csv', row.names = 1)

#set seed for reproducibility
set.seed(333)

#create list for splitting training set to train/test of 80/20
ind = sample(2, nrow(probs), replace = TRUE, prob = c(0.8, 0.2))

#with validation split, train.nt is training and train.nv is validation
train.nt = probs[ind == 1,]
train.nv = probs[ind == 2,]

#pull training data as well as the response GA
p.train_data = train.nt[,1:length(train.nt)-1]
p.train_labels = train.nt$GA

#pull testing data as well as the response GA
p.test_data = train.nv[,1:length(train.nv)-1]
p.test_labels = train.nv$GA

#view the first few training labels
p.train_labels[1:10]

p.train_data = p.train_data %>% select(row.names(skew2)[1:200])
p.test_data = p.test_data %>% select(row.names(skew2)[1:200])

#scale
p.train_data = scale(p.train_data)

#use means and std from training to normalize testing
p.col_means_train = attr(p.train_data, 'scaled:center')
p.col_stddevs_train = attr(p.train_data, 'scaled:scale')
p.test_data = scale(p.test_data, center = p.col_means_train, scale = p.col_stddevs_train)

#submission set
subs = read.csv('filteredprobeset15lr_test_7_4.csv', row.names = 1)

#scale the submission set
p.submission_data = scale(subs, center = p.col_means_train, scale = p.col_stddevs_train)
```

### Create table for storing RMSE and architecture

Rather than storing every model, a table was created to store the RMSE values of the train/test split data. This table contained the run count, the number of nodes that were in the first layer, number of nodes in the second layer, and the RMSE values of the training data and the testing data. This will allow for determining which models are the best later on.

``` r
#create empty table to store model RMSE values in
rootmse = data.frame(matrix(NA, nrow = 10000, ncol = 5))
colnames(rootmse)[1] = 'Run'
colnames(rootmse)[2] = 'H1Nodes'
colnames(rootmse)[3] = 'H2Nodes'
colnames(rootmse)[4] = 'TrainRMSE'
colnames(rootmse)[5] = 'TestRMSE'

rowz = 1
```

### Looping through Keras

Keras was used for creating the model, and each time through the loop the seed was set to ensure that data was reproducible. For models where there wasn't a second layer involved the second for loop was commented out, as well as the second dense call in building the model.

``` r
for(H1N in 1:25){
  for(H2N in 1:50){
    use_session_with_seed(333, disable_gpu = FALSE, disable_parallel_cpu = FALSE)
    p.build_model <- function() {
      
      #create the model
      model <- keras_model_sequential() %>%
        layer_dense(units = H1N, activation = "relu",
                    input_shape = dim(p.train_data)[2]) %>%
        layer_dense(units = H2N, activation = 'relu') %>%
        layer_dropout(rate = 0.2) %>%
        layer_dense(units = 1)
      
      #compile the model
      model %>% compile(
        loss = "mse",
        optimizer = optimizer_rmsprop(),
        metrics = list("mean_squared_error")
      )
      
      model
    }
    
    #set the built and compiled model
    p.model = p.build_model()
    
    #train the model
    p.history = p.model %>% fit(
      p.train_data,
      p.train_labels, 
      epochs = 10000,
      validation_split = 0.2,
      verbose = 0
    )
    
    #calculating RMSE of the training data
    p.train_predictions = p.model %>% predict(p.train_data)
    p.train.rmse = rmse(p.train_labels, p.train_predictions)
    p.train.rmse
    
    #calculating the RMSE of the testing data
    p.test_predictions = p.model %>% predict(p.test_data)
    p.test.rmse = rmse(p.test_labels, p.test_predictions)
    p.test.rmse
    
    rootmse$Run[rowz] = rowz
    rootmse$H1Nodes[rowz] = H1N
    rootmse$H2Nodes[rowz] = H2N
    rootmse$TrainRMSE[rowz] = p.train.rmse
    rootmse$TestRMSE[rowz] = p.test.rmse
    rowz = rowz + 1
    
    write.csv(rootmse, 'Root Mean Square Error DL.csv')
  }
}
```

### Finding top models

Lowest RMSE between train/test

With the table created previously, the difference between the training RMSE and the testing RMSE was calculated. Those models with a testing RMSE less than 4 and difference less than 1 were subset into a new table. This new table was used to calculate the weight of each model. This will give the models with the lowest difference the greatest weight in the ensemble results.

The models that are in the new table are stored in 2 variables with the node counts. These will be used in the next section.

``` r
#read in RMSE table
rmseResults = read.csv('Root Mean Square Error DL - Copy.csv')
rmseResults$Difference = rmseResults$TestRMSE - rmseResults$TrainRMSE

#subset by test RMSE less than 4 and difference between train/test RMSE of 1
rmseL3 = subset(rmseResults, TestRMSE < 4)
rmseL3D1 = subset(rmseL3, Difference < 1)

#calculate weights
rmseL3D1$W1 = 1/rmseL3D1$Difference
w1.sum = sum(rmseL3D1$W1)
rmseL3D1$Weight = rmseL3D1$W1/w1.sum
```

``` r
#set hidden layer nodes of the top models
H1N.L = rmseL3D1$H1Nodes
H2N.L = rmseL3D1$H2Nodes

#make new tables to store ensemble model values
allPredicts.train = data.frame(matrix(NA, nrow = nrow(p.train_data), ncol = length(H1N.L)))
colnames(allPredicts.train) = rmseL3D1$Run
rownames(allPredicts.train) = row.names(p.train_data)
allPredicts.test = data.frame(matrix(NA, nrow = nrow(p.test_data), ncol = length(H1N.L)))
colnames(allPredicts.test) = rmseL3D1$Run
rownames(allPredicts.test) = row.names(p.test_data)
allPredicts.subs = data.frame(matrix(NA, nrow = nrow(p.submission_data), ncol = length(H1N.L)))
colnames(allPredicts.subs) = rmseL3D1$Run
rownames(allPredicts.subs) = row.names(p.submission_data)
```

### Rerunning top models and predicting GA on submission data

The models that were filtered as the best models, training is rerun storing the predictions for the submission set.

``` r
for(i in seq(H1N.L)){
  use_session_with_seed(333, disable_gpu = FALSE, disable_parallel_cpu = FALSE)
  p.build_model <- function() {
    
    #create the model
    model <- keras_model_sequential() %>%
      layer_dense(units = H1N.L[i], activation = "relu",
                  input_shape = dim(p.train_data)[2]) %>%
      layer_dense(units = H2N.L[i], activation = 'relu') %>%
      layer_dropout(rate = 0.2) %>%
      layer_dense(units = 1)
    
    #compile the model
    model %>% compile(
      loss = "mse",
      optimizer = optimizer_rmsprop(),
      metrics = list("mean_squared_error")
    )
    
    model
  }
  
  #set the built and compiled model
  p.model = p.build_model()
  
  #train the model
  p.history = p.model %>% fit(
    p.train_data,
    p.train_labels, 
    epochs = 10000,
    validation_split = 0.2,
    verbose = 0
    #,callbacks = list(print_dot_callback)
  )
  
  #calculating RMSE of the training data
  p.train_predictions = p.model %>% predict(p.train_data)
  p.train.rmse = rmse(p.train_labels, p.train_predictions)
  p.train.rmse
  
  #calculating the RMSE of the testing data
  p.test_predictions = p.model %>% predict(p.test_data)
  p.test.rmse = rmse(p.test_labels, p.test_predictions)
  p.test.rmse
  
  allPredicts.train[,i] = p.train_predictions
  allPredicts.test[,i] = p.test_predictions
  
  p.sub_predictions = p.model %>% predict(p.submission_data)
  allPredicts.subs[,i] = p.sub_predictions
}
```

### Calculating weights based on models with closest train/test RMSE

The table that was produced in the previous section is used along with the weights to calculated an ensembled gestational age. After the individual models are weighted, they are added together for the final gestational age prediction. The values that are greater than 42 are reassigned a gestational age of 41.99 and the values that are less than 8 are reassigned a gestational ages of 8.01. Lastly, gestational ages are rounded to 1 decimal place before submission and then written to a csv file.

``` r
#make new data frames for calcuting weighted gestational ages
allPredicts.train.weighted = allPredicts.train
allPredicts.test.weighted = allPredicts.test
allPredicts.subs.weighted = allPredicts.subs

weights = rmseL3D1$Weight

#finding weighted gestational ages
for(i in seq(weights)){
  allPredicts.train.weighted[,i] = allPredicts.train[,i]*weights[i]
  allPredicts.test.weighted[,i] = allPredicts.test[,i]*weights[i]
  allPredicts.subs.weighted[,i] = allPredicts.subs[,i]*weights[i]
}

means.train = rowSums(allPredicts.train.weighted)
means.test = rowSums(allPredicts.test.weighted)
means.subs = rowSums(allPredicts.subs.weighted)

#Viewing ensemble RMSE values 
ensemble.train.rmse = rmse(p.train_labels, means.train)
ensemble.test.rmse = rmse(p.test_labels, means.test)

means.subs[means.subs >= 42] = 41.99
means.subs[means.subs <= 8] = 8.01

#p.submission_predictions = p.model %>% predict(p.submission_data)
submission_dataframe = data.frame(SampleID = row.names(allPredicts.subs),
                                  GA = round(means.subs, 1))

#submission one is 27, 17, 0.2 dropout, with 10k epochs
#submission two is 27, 19, 0.2 dropout, with 10k epochs
write.csv(submission_dataframe,
          file = 'soupir_predictions_2_SC2.csv',
          row.names = FALSE)
```

Conclusion/Discussion
=====================

After submitting the predicted gestational ages, the scored RMSE was 5.6853. This RMSE value was very different from the ensemble RMSE of the training and testing data. The RMSE fo the training split was 2.9915 and the RMSE of the testing split was 3.7729. These differences may be due to the submission data having different skews in the expression data, or the train/test split having different skews simply by chance since the train/test split was made randomly.

Another source that may have caused the large difference between the training data and submission data might originate in the XGBoost filtering of subset data instead of filtering the all of the data at one time. This was difficult, however, because initiallly the data worked on within Google Colab where storage and RAM are limited, as well as all of the models in R having to be trained on a personal computer which takes a great deal of time. If we were to run everything again from the beginning, this is a potential area for improvement.

Authors statement
=================

-   James Young: Writing Python code in Google Colab for filtering features
-   Dakota York: Working with the SDSU HPC and refining Python code
-   Alex Soupir: Work in R

Acknoledgements
===============

We would like to thank those from the SDSU high performance computing team who helped get the modules needed to run the python filtering script.

References
==========

Chollet, François, JJ Allaire, and others. 2017. “R Interface to Keras.” <https://github.com/rstudio/keras>; GitHub.
