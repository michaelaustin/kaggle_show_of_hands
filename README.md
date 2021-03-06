<h1>Kaggle: Show of Hands -- Predicting Happiness</h1>

<h2>Introduction</h2>
Recently, I got to compete in my first <a href='www.kaggle.com' target='_blank'>Kaggle</a> competition. It was a private competition for students taking the Analytics Edge course on <a href='www.edx.org' target='_blank'>EdX</a>.

It was an exhilarating learning experience. The goal of the competition was to predict whether poll respondents were happy or unhappy. The data was collected and distributed for the competition by a polling app called <a href='http://www.showofhands.mobi/' target='_blank'>Show of Hands</a>.

The data comprised of the following demographic information:
<ul>
<li>year of birth</li>
<li>gender</li>
<li>income</li>
<li>household status</li>
<li>education level</li>
<li>party preference</li>
</ul>

In addition, the data contained poll respondents' answers to 101 questions with binary answers. For example:<br />
<ul>
<li>Are you good at math?</li>
<li>Have you cried in the past 60 days?</li>
<li>Do you brush your teeth two or more times every day?</li>
<li>Mac or PC?</li>
<li>Do you drink the unfiltered tap water in your home?</li>
<li>Were you an obedient child?</li>
<li>Do you personally own a gun?</li>
</ul>

The first few rows and columns of the sample data are shown below.

    > head(initial_train_test)
      UserID  YOB Gender             Income  HouseholdStatus        EducationLevel       Party Happy Q124742 Q124122 Q123464 Q123621 Q122769 Q122770 Q122771
    1      1 1938   Male               <NA> Married (w/kids)                  <NA> Independent     1      No    <NA>      No      No      No     Yes  Public
    2      2 1985 Female  $25,001 - $50,000 Single (no kids)       Master's Degree    Democrat     1    <NA>     Yes      No     Yes      No      No  Public
    3      5 1963   Male      over $150,000 Married (w/kids)                  <NA>        <NA>     0      No     Yes      No     Yes      No      No Private
    4      6 1997   Male $75,000 - $100,000 Single (no kids)   High School Diploma  Republican     1    <NA>     Yes     Yes      No    <NA>     Yes Private
    5      7 1996   Male  $50,000 - $74,999 Single (no kids)          Current K-12        <NA>     1      No      No      No      No     Yes     Yes  Public
    6      8 1991 Female      under $25,000 Single (no kids) Current Undergraduate        <NA>     1     Yes     Yes      No    <NA>    <NA>    <NA>  Public

A significant portion (about a quarter) of the data was missing (marked yellow).
![Alt text](./figures/missingDataMap.png)

<h2>First Approach</h2>

<h3>1. Data Pre-processing</h3>
First, I created a new column Age, using the YOB (year of birth) column. I then removed the UserID column in the initial

I could easily spot that there were obvious invalid data entries. People with "negative" numbers for their ages, people over 100, etc. I decided to just remove them commpletely.

    > initial_train_test <- subset(initial_train_test, Age > 13 & Age < 100)
    > final_test <- subset(final_test, Age > 13 & Age < 100)

As for those missing values, my first reaction was imputation. I utilized complete() and mice() functions in the <strong>mice</strong> package.

    > library(mice)

    > # for the initial_train_test dataset
    > set.seed(123)
    > initial_train_test.dep <- subset(initial_train_test, select=Happy)
    > initial_train_test.indep <- subset(initial_train_test, select=-Happy)
    > initial_train_test.indep.imputed <- complete(mice(initial_train_test.indep))
    > initial_train_test.imputed <- cbind(initial_train_test.dep, 
    +                                     initial_train_test.indep.imputed)

    > # for the final_test dataset
    > set.seed(123)
    > final_test.imputed <- complete(mice(final_test))

I saved the imputed datasets by exporting them as separate CSV files.

    > write.csv(initial_train_test.imputed, 
    +	        file='data/imputed/initial_train_test_imputed.csv',
    + 	        row.names = FALSE)
    > write.csv(final_test.imputed, 
    +           file='data/imputed/final_test_imputed.csv',
    +           row.names = FALSE)

<h3>2. Data Modeling</h3>
I decided to test several prediction models to see which performed the best: logistic regression model, regression tree model with 10-fold validation, random forest model, linear discriminant model, and quadratic discriminant model.

I decided to use all variables to build my models, but really, I should have only selected variables that were helpful for predicting poll respondents' happiness. The number of variables used to build models was reduced in the second approach.

<h6>Considering data transformation</h6>
There were two continuous variables in the initial training/testing set: Age and votes columns. I considered data transformation but it didn't seem necessary. The data wasn't too skewed and was "normal enough".

    > hist(initial_train_test$Age)
    > hist(initial_train_test$votes) 

    > library(psych)
    > describe(initial_train_test$Age)
      var    n  mean    sd median trimmed   mad min max range skew kurtosis   se
    1   1 3927 34.87 15.17     32   33.38 16.31  14  82    68  0.7    -0.38 0.24
    > describe(initial_train_test$votes)
      var    n  mean    sd median trimmed   mad min max range  skew kurtosis   se
    1   1 3927 75.01 27.39     87   78.14 20.76  20 101    81 -0.69    -1.01 0.44

<h6>Spliting data into training and testing sets</h6>

    > library(caTools)
    > set.seed(123)
    > split <- sample.split(initial_train_test$Happy, SplitRatio=0.75)
    > initial_train <- initial_train_test[split==TRUE, ]
    > initial_test <- initial_train_test[split==FALSE, ]
    > nrow(initial_train)
    [1] 2946
    > nrow(initial_test)
    [1] 981

<h6>Baseline model</h6> 

    > table(initial_test$Happy)

      0   1 
    428 553 
    > 553 / (428 + 553)  # 56.37% accuracy on test
    [1] 0.5637105

<h6>Logistic regression model</h6>
The logistic regression model yielded 66.46% accuracy on the initial test dataset.

    > logModel <- glm(Happy ~ ., data = initial_train, family = binomial)
    > predPerc.logModel.initial_test <- predict(logModel, newdata = initial_test, type = 'response')
    > pred.logModel.initial_test <- ifelse(predPerc.logModel.initial_test > 0.5, 1, 0)
    > table(pred.logModel.initial_test, initial_test$Happy)

    pred.logModel.initial_test   0   1
			     0 242 143
			     1 186 410
 

<h6>Random forest</h6>
The random forest model yielded 67.58% accuracy on the initial test dataset.

Loading the library:

    > library(randomForest)

Random forest requires the outcome variable to be a factor variable.

    > initial_train$Happy <- as.factor(initial_train$Happy)
    > initial_test$Happy <- as.factor(initial_test$Happy)

Building the model and calculating the accuracy:

    > set.seed(123)
    > RFmodel <- randomForest(Happy ~ ., data = initial_train, ntrees=200)
    > predPerc.RFmodel.initial_test <- predict(RFmodel, newdata = initial_test, type = 'prob')[ , 2]
    > pred.RFmodel.initial_test <- ifelse(predPerc.RFmodel.initial_test > 0.5, 1, 0)
    > table(pred.RFmodel.initial_test, initial_test$Happy)

    pred.RFmodel.initial_test   0   1
			    0 209  99
			    1 219 454

<h6>Regression tree with 10-fold cross validation</h6>
The regression tree model with 10-fold cross validation yielded 64.83% accuracy on the inital test dataset.

Loading the necessary libraries:

    > library(rpart)  # for rpart()
    > library(rpart.plot)  # for prp()
    > library(caret)  # for cross-validation
    > library(e1071)  # for cross-validation

Finding complex parameter through training:

    > trControl <- trainControl(method = 'cv', number = 10)
    > tuneGrid <- expand.grid(.cp = (1:50) * 0.01)
    > set.seed(123)
    > train(Happy ~ . - YOB, 
    +       data = initial_train, 
    +       method = 'rpart', 
    +       trControl = trControl,
    +       tuneGrid = tuneGrid)

    .
    .
    .

    RMSE was used to select the optimal model using  the smallest value.
    The final value used for the model was cp = 0.01. 

Building the model and calculating the accuracy:

    > CARTmodel <- rpart(Happy ~ ., data = initial_train, cp = 0.01)
    > predPerc.CARTmodel.initial_test <- predict(CARTmodel, newdata = initial_test)
    > pred.CARTmodel.initial_test <- ifelse(predPerc.CARTmodel.initial_test > 0.5, 1, 0)
    > table(pred.CARTmodel.initial_test, initial_test$Happy)

    pred.CARTmodel.initial_test   0   1
			      0 183 100
			      1 245 453


<h6>Linear discriminant analysis</h6>

The LDA model yielded 66.87% accuracy on the initial test dataset. 

    > library(MASS)  # for lda()
    > LDAmodel <- lda(Happy ~ ., data = initial_train)
    Warning message:
    In lda.default(x, grouping, ...) : variables are collinear
    > predPerc.LDAmodel.initial_test <- predict(LDAmodel, newdata = initial_test)$posterior[ , 2]
    > pred.LDAmodel.initial_test <- ifelse(predPerc.LDAmodel.initial_test > 0.5, 1, 0)
    > table(pred.LDAmodel.initial_test, initial_test$Happy)

    pred.LDAmodel.initial_test   0   1
			     0 244 141
			     1 184 412


<h6>Averaging prediction probabilities to make predictions</h6>

One known method for improving prediction accuracy is model averaging. Here, we will average the prediction probabilities obtained from GLM, random forest, and LDA models to make our predictions on the initial_test dataset. 

We will not include our regression tree model in the model averaging process because regression tree and random forest both utilize decision trees to make predictions. Model averaging works best when we use different models that do not utilize the same algorithm.

    > predPerc.averaged.initial_test <- (predPerc.logModel.initial_test + predPerc.LDAmodel.initial_test + 
    +                                      predPerc.RFmodel.initial_test) / 3
    > pred.averaged.initial_test <- as.integer(predPerc.averaged.initial_test > 0.5)
    > table(pred.averaged.initial_test, initial_test$Happy)

    pred.averaged.initial_test   0   1
			     0 244 129
			     1 184 424

Averaging prediction probabilities yielded 68.09% accuracy, which was higher than any single model's prediction accuracy. Let's see what would have happened if we included the regression tree model in the model averaging process.

    > predPerc.averaged.initial_test <- (predPerc.logModel.initial_test + predPerc.LDAmodel.initial_test + 
    +                                      predPerc.RFmodel.initial_test + predPerc.CARTmodel.initial_test) / 4
    > pred.averaged.initial_test <- as.integer(predPerc.averaged.initial_test > 0.5)
    > table(pred.averaged.initial_test, initial_test$Happy)

    pred.averaged.initial_test   0   1
			     0 221 112
			     1 207 441

Including regression tree model yielded 67.48% accuracy, which was slightly lower than that of random forest model (67.58%).

<h6>Picking more frequent prediction outcomes from different models</h6>
We can make prediction models by selecting a more common prediction outcome for each of our prediction case.

    > head(pred.logModel.initial_test, 10)
     2  5  6 13 17 25 27 29 33 34 
     0  1  1  0  0  0  0  1  0  1 
    > head(pred.LDAmodel.initial_test, 10)
     2  5  6 13 17 25 27 29 33 34 
     0  1  1  0  0  0  0  1  0  1 
    > head(pred.RFmodel.initial_test, 10)
     2  5  6 13 17 25 27 29 33 34 
     0  1  1  1  1  0  1  1  1  1 
    >
    > predPerc.freq.initial_test <- (pred.logModel.initial_test + pred.LDAmodel.initial_test + 
    +                                  pred.RFmodel.initial_test) / 3
    > pred.freq.initial_test <- round(pred.freq.initial_test, 0)
    > table(pred.freq.initial_test, initial_test$Happy)

    pred.freq.initial_test   0   1
			 0 221 106
			 1 207 447 

This method, similar to averaging prediction probabilities, yielded 68.09% accuracy, which was higher than that of any single prediction model.  

<h3>3. Dummy-Coding and Normalization</h3>
One technique that data scientists utilize to improve  prediction accuracies is clustering. By building prediction models on clustered data, prediction models deal with more similar data points, which can improve accuracy.

<h6>Dummy coding binary variables (to 0's and 1's)</h6>

    > initial_train_test.dc <- initial_train_test
    > for (i in c(3, 8:(ncol(initial_train_test)-2))) {  
    +   initial_train_test.dc[[i]] <- as.integer(initial_train_test.dc[[i]]) - 1
    + }
    > colnames(initial_train_test.dc)[3] <- 'Male'

<h6>Dummy coding categorical variables (to 0's and 1's)</h6>

    > # dummy coding: Income
    > dcIncome.df <- as.data.frame(model.matrix(~initial_train_test.dc$Income))[-1]
    > colnames(dcIncome.df) <- c('inc.25001_50000', 'inc.50000_74999',
    +                            'inc.75000_100000', 'inc.100001_150000',
    +                            'inc.over.150000')
    > head(dcIncome.df)
      inc.25001_50000 inc.50000_74999 inc.75000_100000 inc.100001_150000 inc.over.150000
    1               0               0                0                 1               0
    2               1               0                0                 0               0
    3               0               0                0                 1               0
    4               0               0                1                 0               0
    5               0               1                0                 0               0
    6               0               0                0                 0               1
    > initial_train_test.dc <- cbind(initial_train_test.dc, dcIncome.df)

    > # dummy coding: HouseholdStatus
    > dcHouseholdStatus.df <- as.data.frame(model.matrix(~initial_train_test.dc$HouseholdStatus))[-1]
    > colnames(dcHouseholdStatus.df) <- c('Single.w.kids', 'DomParts.wo.kids', 
    +                                     'DomParts.w.kids', 'Married.wo.kids', 
    +                                     'Married.w.kids')
    > head(dcHouseholdStatus.df)
      Single.w.kids DomParts.wo.kids DomParts.w.kids Married.wo.kids Married.w.kids
    1             0                0               1               0              0
    2             0                0               0               1              0
    3             0                0               1               0              0
    4             0                0               0               1              0
    5             0                0               0               1              0
    6             0                0               0               1              0
    > initial_train_test.dc <- cbind(initial_train_test.dc, dcHouseholdStatus.df)

    > # dummy coding: EducationLevel
    > dcEducation.df <- as.data.frame(model.matrix(~initial_train_test.dc$EducationLevel))[-1]
    > colnames(dcEducation.df) <- c('edu.HS_diploma', 'edu.associate', 
    +                               'edu.current_undergrad', 'edu.bachelor',
    +                               'edu.master', 'edu.doctoral')
    > head(dcEducation.df)
      edu.HS_diploma edu.associate edu.current_undergrad edu.bachelor edu.master edu.doctoral
    1              0             0                     0            0          0            0
    2              0             0                     0            0          0            1
    3              1             0                     0            0          0            0
    4              0             0                     0            0          1            0
    5              0             1                     0            0          0            0
    6              0             0                     1            0          0            0
    > initial_train_test.dc <- cbind(initial_train_test.dc, dcEducation.df)

    > # dummy coding: Party
    > dcParty.df <- as.data.frame(model.matrix(~initial_train_test.dc$Party))[-1]
    > colnames(dcParty.df) <- c('party.independent', 'party.libertarian',
    +                           'party.other', 'party.republican')
    > head(dcParty.df)
      party.independent party.libertarian party.other party.republican
    1                 1                 0           0                0
    2                 0                 0           0                0
    3                 0                 0           0                1
    4                 0                 0           0                1
    5                 1                 0           0                0
    6                 1                 0           0                0
    > initial_train_test.dc <- cbind(initial_train_test.dc, dcParty.df)

<h6>Removing categorical variables that have been dummy-coded</h6>

    > initial_train_test.dc$Income <- NULL
    > initial_train_test.dc$HouseholdStatus <- NULL
    > initial_train_test.dc$EducationLevel <- NULL
    > initial_train_test.dc$Party <- NULL

<h6>Split the dummy-coded initial_train_test dataset into training and testing</h6>

    > library(caTools)
    > set.seed(123)
    > split <- sample.split(initial_train_test.dc$Happy, SplitRatio=0.75)
    > initial_train.dc <- initial_train_test.dc[split==TRUE, ]
    > initial_test.dc <- initial_train_test.dc[split==FALSE, ]

<h6>Normalize data</h6>

    > initial_train.dc.dep <- subset(initial_train.dc, select = Happy)
    > initial_train.dc.indep <- subset(initial_train.dc, select = - Happy)
    > 
    > initial_test.dc.dep <- subset(initial_test.dc, select = Happy)
    > initial_test.dc.indep <- subset(initial_test.dc, select = - Happy)
    > 
    > library(caret)
    > preproc <- preProcess(initial_train.dc.indep)
    > initial_train.dc.indep.norm <- predict(preproc, initial_train.dc.indep)
    > initial_test.dc.indep.norm <- predict(preproc, initial_test.dc.indep)
    >
    > initial_train.dc.norm <- cbind(initial_train.dc.dep, initial_train.dc.indep.norm)
    > initial_test.dc.norm <- cbind(initial_test.dc.dep, initial_test.dc.indep.norm)

<h3>4. Clustering</h3>

<h6>Hierarchical clustering</h6>

Looking at the dendrogram, I decided that clustering the dataset into two clusters was appropriate.

![Alt text](./figures/hClust.png)

    > d.initial_train.dc.indep.norm <- dist(initial_train.dc.indep.norm, method='euclidean')
    > hclust.initial_train <- hclust(d.initial_train.dc.indep.norm, method='ward')
    > plot(hclust.initial_train)

<h6>Clustering paradigm with training and testing datasets</h6>
With the cluster-and-predict method, the goal is to create cluster groups in the training dataset and to assign observersations in the test dataset to appropriately matching clusters. We do NOT want to run two separate cluster algorithms, one for the training dataset and another one for the testing dataset (especially for k-means clustering method) because there is no guarantee that a cluster from the training dataset shares similar qualities as a cluster formed from the testing dataset.

In R, there is a package that allows one to create clusters from the training dataset and appropriately categorize observations from the testing dataset into the clusters: the <strong>flexclust</strong> package.

After I decided on the number of clusters by looking at the dendrogram, decided to run k-means algorithm (instead of hierarchical clustering algorithm). 

<h6>Loading the <strong>flexclust</strong> library</h6>

    > library(flexclust)

<h6>Creating 2 cluster groups through k-means</h6>

    > set.seed(123)
    > km2 <- kmeans(initial_train.dc.indep.norm, center=2)
    > km2.kcca <- as.kcca(km2, initial_train.dc.indep.norm)
    > kClust.2Groups.initial_train <- km2$cluster
    > kClust.2Groups.initial_test <- predict(km2.kcca, newdata = initial_test.dc.ind

It was evident that Cluster 1 group was overall happier than Cluster 2.

    > tapply(initial_train$Happy, kClust.2Groups.initial_train, mean)
	    1         2 
    0.5991903 0.5127362 

In addition, Cluster 1 group was comprised of older poll respondents than was Cluster 2, both in the training and testing set.

    > tapply(initial_train$Age, kClust.2Groups.initial_train, mean)
	   1        2 
    44.05899 21.77075 
    > tapply(initial_test$Age, kClust.2Groups.initial_test, mean)
	   1        2 
    44.44580 21.61125 

<h6>Split the training and testing datasets into clusters</h6>

    > initial_train1 <- subset(initial_train, kClust.2Groups.initial_train==1)
    > initial_train2 <- subset(initial_train, kClust.2Groups.initial_train==2)
    > initial_test1 <- subset(initial_test, kClust.2Groups.initial_test==1)
    > initial_test2 <- subset(initial_test, kClust.2Groups.initial_test==2)

<h6>Building models on the clustered groups</h6>

    > set.seed(123)
    > RFmodel1 <- randomForest(Happy ~ ., data = initial_train1, ntree = 200)
    > RFmodel2 <- randomForest(Happy ~ ., data = initial_train2, ntree = 200)
    > 
    > ## on the initial_test set
    > # prediction probabilities
    > predPerc.RFmodel1.initial_test1 <- predict(RFmodel1, newdata = initial_test1, type = 'prob')[ , 2]
    > predPerc.RFmodel2.initial_test2 <- predict(RFmodel2, newdata = initial_test2, type = 'prob')[ , 2]
    > 
    > # predictions
    > pred.RFmodel1.initial_test1 <- as.integer(predPerc.RFmodel1.initial_test1 > 0.5)
    > pred.RFmodel2.initial_test2 <- as.integer(predPerc.RFmodel2.initial_test2 > 0.5)

Similarly, other models (logistic regression, LDA, and regression tree with 10-fold cross validation) were created and accuracies measured.

<h6>Averaging prediction probabilities to make predictions</h6>

    > ## prediction percentages on initial_testing sets averaged across GLM and RF
    > predPerc.avgClust1.initial_test1 <- (predPerc.logModel1.initial_test1 + predPerc.RFmodel1.initial_test1) / 2
    > predPerc.avgClust2.initial_test2 <- (predPerc.logModel2.initial_test2 + predPerc.RFmodel2.initial_test2) / 2
    > 
    > ## predictions from averaged probabilities on initial_test sets
    > pred.avgClust1.initial_test1 <- as.integer(predPerc.avgClust1.initial_test1 > 0.5)
    > pred.avgClust2.initial_test2 <- as.integer(predPerc.avgClust2.initial_test2 > 0.5)
    > 
    > ## accuracy on the initial_testing set
    > table(pred.avgClust1.initial_test1, initial_test1$Happy)

    pred.avgClust1.initial_test1   0   1
			       0 106  54
			       1 122 290
    > table(pred.avgClust2.initial_test2, initial_test2$Happy)

    pred.avgClust2.initial_test2   0   1
			       0 123  71
			       1  77 138
<h6>Selecting more frequent prediction outcomes to make predictions</h6>

    > predPerc.freqClust1.initial_test1 <- (pred.logModel1.initial_test1 + pred.RFmodel1.initial_test1) / 2
    > head(predPerc.freqClust1.initial_test1, 10)
     [1] 1.0 0.0 0.5 1.0 0.0 0.5 1.0 0.5 0.0 0.0
    > predPerc.freqClust2.initial_test2 <- (pred.logModel2.initial_test2 + pred.RFmodel2.initial_test2) / 2
    > head(predPerc.freqClust2.initial_test2, 10)
     [1] 1.0 1.0 1.0 0.5 0.0 0.0 1.0 0.0 0.0 1.0

    > RFvalues.clust1 <- pred.RFmodel1.initial_test1[predPerc.freqClust1.initial_test1==0.5]
    > RFvalues.clust2 <- pred.RFmodel2.initial_test2[predPerc.freqClust2.initial_test2==0.5]
    > pred.freqClust1.initial_test1 <- replace(predPerc.freqClust1.initial_test1,
    + predPerc.freqClust1.initial_test1==0.5,
    + RFvalues.clust1)
    > pred.freqClust2.initial_test2 <- replace(predPerc.freqClust2.initial_test2,
    + predPerc.freqClust2.initial_test2==0.5,
    + RFvalues.clust2)

    > pred.freqClust1.initial_test1 <- round(pred.freqClust1.initial_test1, 0)
    > pred.freqClust2.initial_test2 <- round(pred.freqClust2.initial_test2, 0)
    > head(pred.freqClust1.initial_test1, 10)
     [1] 1 0 0 1 0 1 1 1 0 0
    > head(pred.freqClust2.initial_test2, 10)
     [1] 1 1 1 1 0 0 1 0 0 1

<h3>5. Model Performance Assessment</h3>
<h6>No clustering</h6>
<ul>
<li>GLM accuracy: 66.46%</li>
<li>RT accuracy: 64.83%</li>
<li>RF accuracy: 67.58%</li>
<li>LDA accuracy: 66.87%</li>
<li>Probabilities averaged (GLM + RF + LDA): 68.09%</li>
<li>Frequent outcome picked (GLM + RF + LDA): 68.09%</li>
</ul>

<h6>2-group clustering</h6>

<ul>
<li>GLM accuracy: 65.85%</li>
<li>RT accuracy: 65.95%</li>
<li>RF accuracy: 67.89%</li>
<li>LDA accuracy: 43.63%</li>
<li>Probabilities averaged (GLM + RF): 66.97%</li>
<li>Frequent outcome picked (GLM + RF): 67.89%</li>
</ul>

<h2>Second Approach</h2>

<h3>1. Data Pre-processing</h3>
First, as I did in my first approach, I created a new variable Age.

    > initial_train_test$Age <- 2013 - initial_train_test$YOB  # this data is from 2013
    > final_test$Age <- 2013 - final_test$YOB  # this data is from 2013

Instead of removing observations with invalid age information, I changed the cell values for age and year of birth to NA.

    > initial_train_test$Age[initial_train_test$Age > 100 | initial_train_test$Age < 10] <- NA
    > initial_train_test$YOB[initial_train_test$Age > 100 | initial_train_test$Age < 10] <- NA
    > final_test$Age[final_test$Age > 100 | final_test$Age < 10] <- NA
    > final_test$YOB[final_test$Age > 100 | final_test$Age < 10] <- NA

In addition, I created a new categorical variable, AgeGroup.

    > initial_train_test$AgeGroup <- cut(initial_train_test$Age,
    +                                    breaks=c(12, 21, 31, 44, 82),
    +                                    right=F,
    +                                    dig.lab=2)
    > final_test$AgeGroup <- cut(final_test$Age,
    +                            breaks=c(12, 21, 31, 44, 82),
    +                            right=F,
    +                            dig.lab=2)
 
Then I removed the Age and YOB (year of birth) columns.

    > initial_train_test <- subset(initial_train_test, select = c(- YOB, - Age))
    > final_test <- subset(final_test, select = c(- YOB, - Age))

This time, instead of imputing for missing data, I decided to code the missing cells with a unique categorical value, "Skipped."

    > for (i in names(initial_train_test)) {
    +   levels(initial_train_test[ , i]) <- c(levels(initial_train_test[ , i]), 'Skipped')
    + }
    > initial_train_test[is.na(initial_train_test)] <- 'Skipped'
    > for (i in names(final_test)) {
    +   levels(final_test[ , i]) <- c(levels(final_test[ , i]), 'Skipped')
    + }
    > final_test[is.na(final_test)] <- 'Skipped'

Below shows the first 10 rows and columns of the resulting data frame:

    > initial_train_test[1:10, 1:10]
       UserID  YOB Gender              Income  HouseholdStatus        EducationLevel       Party Happy Q124742 Q124122
    1       1 1938   Male             Skipped Married (w/kids)               Skipped Independent     1      No Skipped
    2       2 1985 Female   $25,001 - $50,000 Single (no kids)       Master's Degree    Democrat     1 Skipped     Yes
    3       5 1963   Male       over $150,000 Married (w/kids)               Skipped     Skipped     0      No     Yes
    4       6 1997   Male  $75,000 - $100,000 Single (no kids)   High School Diploma  Republican     1 Skipped     Yes
    5       7 1996   Male   $50,000 - $74,999 Single (no kids)          Current K-12     Skipped     1      No      No
    6       8 1991 Female       under $25,000 Single (no kids) Current Undergraduate     Skipped     1     Yes     Yes
    7       9 1995   Male  $75,000 - $100,000 Single (no kids)          Current K-12  Republican     1 Skipped Skipped
    8      11 1983   Male $100,001 - $150,000 Married (w/kids)     Bachelor's Degree Independent     1      No     Yes
    9      12 1984 Female   $50,000 - $74,999 Married (w/kids)   High School Diploma  Republican     0      No     Yes
    10     13 1997 Female       over $150,000 Single (no kids)          Current K-12    Democrat     0 Skipped Skipped

<h3>2. Selecting Significant Variables with ANOVA</h3>
Another approach I took to improve the prediction accuracies of my models was selecting important variables. This is a crucial step. Excluding unimportant variables not only reduces computational burden (which can be notable in case of significant multicolinearity) but also avoids potential overfitting.

I wrote a function that can go through all categorical variables and conduct analysis of variance (ANOVA) testing for each categorical variable. If a categorical variable yielded a p-value of less than 0.05, then it would register the as "significant"

    > findSigVariables <- function(df, dep.var, threshold) {
    +   colnames <- names(df)  # save all column names
    +   sig.colnames <- c()
    +   p.values <- c()
    +   for (colname in colnames) {
    +     if (tolower(colname) != tolower(dep.var)) {  
    +       aov.formula <- as.formula(paste('Happy~', colname))
    +       aov <- aov(aov.formula, data = df)  # perform ANOVA
    +       p.value <- summary(aov)[[1]]$'Pr(>F)'[1]  # extract p-value from ANOVA
    +       if (p.value <= threshold) {
    +         print(paste('p-value (', colname, '): ', p.value, sep = ''))
    +         sig.colnames <- append(sig.colnames, colname)
    +         p.values <- append(p.values, p.value)
    +       }
    +     }
    +   }
    +   sigVariables <- as.data.frame(cbind(sig.colname = sig.colnames, p.value = p.values), 
    +                                 stringsAsFactors = FALSE)  # create a return data frame
    +   sigVariables$p.value <- as.numeric(as.character(sigVariables$p.value))  # convert p.value column to numeric
    +   sigVariables <- sigVariables[order(sigVariables$p.value), ]  # order df
    +   return(sigVariables)
    + }

Then, I ran the function.

    > sigVarsT.initial_train <- findSigVariables(initial_train,
    +                                            dep.var = 'Happy',
    +                                            threshold = 0.05)
    > sigVars <- sigVarsT.initial_train$sig.colname
    > sigVars
     [1] "Q118237"         "Q101162"         "Q107869"         "Q102289"         "Q98869"          "Q102906"        
     [7] "Q106997"         "HouseholdStatus" "Q108855"         "Q119334"         "Q115610"         "Q108856"        
    [13] "Q120014"         "Q108343"         "Q116197"         "Q98197"          "Q116448"         "Q102687"        
    [19] "Q114961"         "Q108342"         "Q113181"         "Income"          "Q117186"         "Party"          
    [25] "Q115390"         "Q112512"         "Q102089"         "Q116953"         "Q115611"         "Q121011"        
    [31] "Q111580"         "Q99716"          "Q106993"         "Q109367"         "Q114152"         "Q106389"        
    [37] "Q116441"         "Q123621"         "Q113584"         "Q124742"         "Q108617"         "Q116881"        
    [43] "Q117193"         "Q100689"         "Q115602"         "Q98578"          "Q120012"         "Q100680"        
    [49] "Q112478"         "Q106272"         "Q99982"          "Q109244"         "Q98059"          "EducationLevel" 
    [55] "Q96024"          "Q111848"         "Q119851"         "Q118233"         "Q119650"         "UserID"         
    [61] "Q108950"         "Q102674"     

Looking at the list, I saw that UserID was listed as one of the "significant" variables for predicting happiness. Intuitively, that was non-sense. However, understanding how ANOVA works and how the function was written, it made sense. The output was suggesting that there exists a significant difference in happiness among difference users (who share separate user IDs). Because we coudn't infer anything about a user from his/her user ID in the test dataset, I removed the UserID variable, which left us with 61 "significant" variables (reduced from the original dataset's 110 independent variables).

Instead of manually typing in all 61 variables, I wrote a function that can take the sigVars object and convert to a formula string, which could easily be converted to a formula object.

    > createFormulaStr <- function(sigVariables) {
    +   formula <- 'Happy~'
    +   for (var.name in sigVariables) {
    +     formula <- paste(formula, var.name, '+', sep='')
    +   }
    +   formula <- substr(formula, 1, nchar(formula) - 1)
    + }
    > 
    > formulaStr <- createFormulaStr(sigVars)
    > formula <- as.formula(formulaStr)
    > formula
    Happy ~ Q118237 + Q101162 + Q107869 + Q102289 + Q98869 + Q102906 + 
	Q106997 + HouseholdStatus + Q108855 + Q119334 + Q115610 + 
	Q108856 + Q120014 + Q108343 + Q116197 + Q98197 + Q116448 + 
	Q102687 + Q114961 + Q108342 + Q113181 + Income + Q117186 + 
	Party + Q115390 + Q112512 + Q102089 + Q116953 + Q115611 + 
	Q121011 + Q111580 + Q99716 + Q106993 + Q109367 + Q114152 + 
	Q106389 + Q116441 + Q123621 + Q113584 + Q124742 + Q108617 + 
	Q116881 + Q117193 + Q100689 + Q115602 + Q98578 + Q120012 + 
	Q100680 + Q112478 + Q106272 + Q99982 + Q109244 + Q98059 + 
	EducationLevel + Q96024 + Q111848 + Q119851 + Q118233 + Q119650 + 
	UserID + Q108950 + Q102674  

<h3>3. Data Modeling</h3>
As before, I decided to test several prediction models to measure and compare their performances: logistic regression model, regression tree model with 10-fold validation, random forest, and linear discriminant analysis.

<h6>Logistic regression</h6>

Accuracy: 68.4%

    > logModel <- glm(formula, data = initial_train, family = binomial)
    > predProb.logModel.initial_test <- predict(logModel, newdata = initial_test, type = 'response')
    > pred.logModel.initial_test <- as.integer(predProb.logModel.initial_test > 0.5)
    > table(pred.logModel.initial_test, initial_test$Happy)
    > table(pred.logModel.initial_test, initial_test$Happy)

    pred.logModel.initial_test   0   1
			     0 358 191
			     1 247 590

<h6>Regression tree with 10-fold cross-validation</h6>

Accuracy: 64.72%

    > library(rpart)
    > library(rpart.plot)
    > library(caret)
    > library(e1071)
    > 
    > set.seed(123)
    > trainControl <- trainControl('cv', 10)
    > tuneGrid <- expand.grid(.cp = (1:30 * 0.01))
    > train(formula, data = initial_train, method = 'rpart', 
    +       trControl = trainControl, tuneGrid = tuneGrid)

    .
    .
    .

    RMSE was used to select the optimal model using  the smallest value.
    The final value used for the model was cp = 0.01. 

    > CARTmodel <- rpart(formula, data = initial_train, cp = 0.01, method = 'class')
    > predProb.CARTmodel.initial_test <- predict(CARTmodel, newdata = initial_test)[ , 2]
    > pred.CARTmodel.initial_test <- as.integer(predProb.CARTmodel.initial_test > 0.5)
    > table(pred.CARTmodel.initial_test, initial_test$Happy)

    pred.CARTmodel.initial_test   0   1
			      0 267 151
			      1 338 630

<h6>Random forest</h6>

Accuracy: 68.11%

    > initial_train$Happy <- factor(initial_train$Happy)
    > initial_test$Happy <- factor(initial_test$Happy)
    > set.seed(123)
    > RFmodel <- randomForest(formula, data = initial_train, ntrees = 200)
    > predProb.RFmodel.initial_test <- predict(RFmodel, newdata = initial_test, type = 'prob')[ , 2]
    > pred.RFmodel.initial_test <- as.integer(predProb.RFmodel.initial_test > 0.5)
    > table(pred.RFmodel.initial_test, initial_test$Happy)

    pred.RFmodel.initial_test   0   1
			    0 326 163
			    1 279 618

<h6>Linear discriminant analysis</h6>

Accuracy: 68.12%

    > library(MASS)
    > LDAmodel <- lda(formula, data = initial_train)
    > predProb.LDAmodel.initial_test <- predict(LDAmodel, newdata = initial_test)$posterior[ , 2]
    > pred.LDAmodel.initial_test <- as.integer(predProb.LDAmodel.initial_test > 0.5)
    > table(pred.LDAmodel.initial_test, initial_test$Happy)

    pred.LDAmodel.initial_test   0   1
			     0 349 186
			     1 256 595

<h6>Averaging prediction probabilities to make predictions</h6>

Accuracy: 69.19%

    > predProb.averaged.initial_test <- 
    +   (predProb.logModel.initial_test + 
    +      predProb.RFmodel.initial_test + 
    +      predProb.LDAmodel.initial_test) / 3
    > pred.averaged.initial_test <- as.integer(predProb.averaged.initial_test > 0.5)
    > table(pred.averaged.initial_test, initial_test$Happy)

    pred.averaged.initial_test   0   1
			     0 355 177
			     1 250 604

<h6>Selecting more frequent outcomes to make predictions</h6>

Accuracy: 68.47%

    > predProb.freq.initial_test <- (pred.logModel.initial_test + 
    +                                  pred.RFmodel.initial_test + 
    +                                  pred.LDAmodel.initial_test) / 3
    > pred.freq.initial_test <- round(predProb.freq.initial_test, 0)
    > table(pred.freq.initial_test, initial_test$Happy)

    pred.freq.initial_test   0   1
			 0 355 187
			 1 250 594

<h3>3. Selecting 17 most significant variables (out of 61)</h3>

<h6>Plotting 25 important variables (according to random forest)</h6>

![Alt text](./figures/varImpPlot_25vars.png)

    > varImpPlot(RFmodel, n.var = 25, 
    +            main = 'Importance of Variables',
    +            cex = 0.7)  # cex controls the size of label texts

<h6>Pick the first 17 variables</h6>

    > formula.17vs <- as.formula('Happy ~ EducationLevel + Income + Q118237 + Party + 
    +                            HouseholdStatus + Q101162 + Q107869 + Q119334 + 
    +                            Q124742 + Q108855 + Q102289 + Q121011 + Q98869 + 
    +                            Q123621 + Q120014 + Q102906 + Q106997')

<h6>Logistic regression</h6>

Accuracy: 67.75%

    > logModel17vs <- glm(formula.17vs, data = initial_train, family = binomial)
    > predProb.logModel17vs.initial_test <- predict(logModel17vs, newdata = initial_test, type = 'response')
    > pred.logModel17vs.initial_test <- as.integer(predProb.logModel17vs.initial_test > 0.5)
    > table(pred.logModel17vs.initial_test, initial_test$Happy)

    pred.logModel17vs.initial_test   0   1
				 0 337 179
				 1 268 602

<h6>Regression tree</h6>

Accuracy: 64.72%

    > set.seed(123)
    > trainControl <- trainControl('cv', 10)
    > tuneGrid <- expand.grid(.cp = (1:50 * 0.01))
    > train(formula.17vs, data = initial_train, method = 'rpart', 
    +       trControl = trainControl, tuneGrid = tuneGrid)

    .
    .
    .

    RMSE was used to select the optimal model using  the smallest value.
    The final value used for the model was cp = 0.01. 

    > CARTmodel17vs <- rpart(formula.17vs, data = initial_train, cp = 0.01, method = 'class')
    > predProb.CARTmodel17vs.initial_test <- predict(CARTmodel17vs, newdata = initial_test)[ , 2]
    > pred.CARTmodel17vs.initial_test <- as.integer(predProb.CARTmodel17vs.initial_test > 0.5)
    > table(pred.CARTmodel17vs.initial_test, initial_test$Happy)

    pred.CARTmodel17vs.initial_test   0   1
				  0 267 151
				  1 338 630

<h6>Random forest</h6>

Accuracy: 66.81%

    > set.seed(123)
    > RFmodel17vs <- randomForest(formula.17vs, data = initial_train, ntrees = 200)
    > predProb.RFmodel17vs.initial_test <- predict(RFmodel17vs, newdata = initial_test, type = 'prob')[ , 2]
    > pred.RFmodel17vs.initial_test <- as.integer(predProb.RFmodel17vs.initial_test > 0.5)
    > table(pred.RFmodel17vs.initial_test, initial_test$Happy)

    pred.RFmodel17vs.initial_test   0   1
				0 330 185
				1 275 596

<h6>Linear discriminant analysis</h6>

Accuracy: 68.25%

    > LDAmodel17vs <- lda(formula.17vs, data = initial_train)
    > predProb.LDAmodel17vs.initial_test <- predict(LDAmodel17vs, newdata = initial_test)$posterior[ , 2]
    > pred.LDAmodel17vs.initial_test <- as.integer(predProb.LDAmodel17vs.initial_test > 0.5)
    > table(pred.LDAmodel17vs.initial_test, initial_test$Happy)

    pred.LDAmodel17vs.initial_test   0   1
				 0 336 171
				 1 269 610

<h6>Averaging prediction probabilities to make predictions</h6>

Accuracy: 67.68%

    > predProb.averaged17vs.initial_test <- 
    +   (predProb.logModel17vs.initial_test + 
    +      predProb.RFmodel17vs.initial_test + 
    +      predProb.LDAmodel17vs.initial_test) / 3
    > pred.averaged17vs.initial_test <- as.integer(predProb.averaged17vs.initial_test > 0.5)
    > table(pred.averaged17vs.initial_test, initial_test$Happy)

    pred.averaged17vs.initial_test   0   1
				 0 337 180
				 1 268 601

<h6>Selecting more frequent outcomes to make predictions</h6>

Accuracy: 68.47%

    > predProb.freq17vs.initial_test <- (pred.logModel.initial_test + 
    +                                      pred.RFmodel.initial_test + 
    +                                      pred.LDAmodel.initial_test) / 3
    > pred.freq17vs.initial_test <- round(predProb.freq17vs.initial_test, 0)
    > table(pred.freq17vs.initial_test, initial_test$Happy)

    pred.freq17vs.initial_test   0   1
			     0 355 187
			     1 250 594

<h3>4. Model Performance Assessment</h3>
<h6>61 variables</h6>
<ul>
<li>GLM accuracy: 68.4%</li>
<li>RT accuracy: 64.72%</li>
<li>RF accuracy: 68.11%</li>
<li>LDA accuracy: 68.12%</li>
<li>Probabilities averaged (GLM + RF + LDA): 69.19%</li>
<li>Frequent outcome picked (GLM + RF + LDA): 68.47%</li>
</ul>

<h6>17 variables</h6>
<ul>
<li>GLM accuracy: 67.75%</li>
<li>RT accuracy: 64.72%</li>
<li>RF accuracy: 66.81%</li>
<li>LDA accuracy: 68.25%</li>
<li>Probabilities averaged: 67.68%</li>
<li>Frequent outcome picked: 68.47%</li>
</ul>

<h2>Conclusion</h2>
There are several lessons I've learned from participating in this competition.<br />

<strong>1. Imputing</strong><br />
Do <strong>NOT</strong> always impute all missing values. Imputing missing values inevitably creates errors. If one tries to impute missing values in Column A, and impute missing values in Column B using A and other variables, the errors incurred from imputing Column A is going to propagate to Column B.<br />

Therefore, the reasoning goes, one should impute for missing values only for:<br />
a. The most important variables to your final model<br />
b. The variables with the least number of missing values<br />
c. The variables that you can impute with a minimal error<br />

<strong>2. Variable Selection</strong><br />
Do <strong>NOT</strong> put all independent variables when creating a prediction model. Pick out the most important variables by pre-selecting them. One can further remove variables by removing variables that are highly colinear (usually around r > 0.75). Another technique of selecting important variables for building prediction models is plotting their importance as recognized by random forest as we did above.

<strong>3. Clustering</strong><br />
Clustering can actually decrease the accuracy rate, as we saw in our case. This could have been very much due to the fact we did not have enough observations to create sizeable clusters. With less data in each cluster, we had less data with which we could train our models--therefore a drop in the accuracy of the models.

<strong>4. Model Averaging</strong><br />
Model averaging is a simple yet powerful technique. Instead of making predictions based on a single prediction model, one should combine outputs (whether they be actual predicted outcomes or prediction probabilities) from different prediction models. 

This ensemble technique works best if models are vastly different each other. Averaging a pack of nearly identical models (logistic model with another logistic model, or random forest model with a regression tree model, etc.) will hardly improve the accuracy. 

<strong>5. Having Fun</strong><br />
Most importantly, have fun. Kaggle is a great platform for data enthusiathiasts to practice, learn, and compete.