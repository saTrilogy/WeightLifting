---
title: "Weight Lifting Predictions"
author: "Chris Shattock"
date: "February 17, 2019"
output: 
  html_document: 
    fig_caption: yes
    keep_md: yes
    highlight: pygments
references:
- type: article-journal
  id: vel2013
  title: Qualitative activity recognition of weight lifting exercises
  author:
  - family: Velloso
    given: Eduardo
  - family: Bulling
    given: Andreas
  - family: Gellersen
    given: Hans
  - family: Ugulino
    given: Wallace
  - family: Fuks
    given: Hugo
  issued:
    date-parts:
    - - 2013
  container-title: Proceedings of the 4th Augmented Human International Conference
  page: 116-123
abstract: |
    In determining inappropriate weight-lifting practices a number of variables are proposed as being 
    potentially capable of predicting a test subject's correct exercise regime. Given sets of training 
    and test data, the former are used to model a random forest predictor with an accuracy in excess of
    67% to determine an appropriate, or otherwise, undertaking of an exericse by a test subject. Whilst
    boosting yields predication results with an estimated accuracy of over 65%, we show that a linear
    discriminant analysis is an inappropriate model. Flaws in the structure of the test data do not
    permit evaluation of the accuracy of the prediction outcomes, nor the use of a composite, stacked
    model approach.
---

<!-- Override the code font to Fira Sans as it supports ligatures
     and add formatting for tables -->
<style type="text/css">
  body{ font-size: 12pt; }
  code.r{ font-size: 10pt; font-family: "Fira Code"  !important;}
  p{text-align: justify; }
  p.abstract{ display: none; }
  div.abstract {
    font-style: italic;
    font-size: 11pt;
    margin-left: 0pt;
    border-bottom: 1px solid;
    border-top: 1px solid;
    width:100% !important;
    text-align: justify;}
  h1.title {
    font-size: 22pt;
    color: DarkRed;
  }
  h1 { font-size: 18pt; }
  h2 { font-size: 16pt; }
  h3 { font-size: 14pt; }  
  h4.author, h4.date  { font-size: 11pt; }  
  table{
    text-align: center;
    margin: auto; }
  td,th{padding-right:12px;}
</style>







# Introduction
The study of @vel2013 considers the classification of appropriate versus inappropriate methods of undertaking a variety of weight lifting exercises by classifying inertial sensor data into one of five classes; **A** being the 'correct' methodology, with erroneous activities classified as one of: **B** - throwing the elbows to the front, **C** - lifting the dumbbell only halfway, **D** - lowering the dumbbell only halfway and **E** - throwing the hips to the front. The task is to predict the efficacy of the exercise regime, given training data of known class methodology via a wide variety of inertial sensor data which may be used as predictors of determining the appropriateness of a subject's exercise in testing data. The test and training data are supplied as pre-sliced datasets.

It so happens that the sliced test and training data do not have a common structure and, furthermore, the outcome to be employed - the class designating the efficacy of the activity, does not appear in the test data. In modelling and predicting tasks, it is usual that the structure of the test and training data are *identical*, and such permits the determination of the accuracy of any prediction and an assessment of the out-of-process error as opposed to the in-process error. In this case, we cannot therefore assess the accuracy nor error inherent in any prediction against the testing data based upon a confusion matrix. Moreover, given the different structure based upon encoding of empty numerical fields in the test data being encoded as `""`, such are parsed as logical fields with a value of `NA`. 

This being the case, for this exercise, we adopt the unusual strategy of determining which columns in the test data exhibit valid numeric data and thereby only select these columns for the training data in order to build our classifiers. There is little point in building classifiers upon the 'richer' content of the training data when such column targets are no present in the test data.

# Parsing the Testing and Training Data
Given the foregoing considerations, we first parse the supplied training data and use the numeric columns available therein to apply a column filter against the training data whilst including in the training data our outcome variable, which we name `Outcome`, thus:

```r
testData <-
  data.table(fread("pml-testing.csv",header=T,strip.white=T,blank.lines.skip=T,
                   check.names=F,key=NULL,stringsAsFactors=F,na.strings="NA")) %>%
  select_if(is.double)
trainData <-
  data.table(fread("pml-training.csv",header=T,strip.white=T,blank.lines.skip=T,
                   check.names=F,key=NULL,stringsAsFactors=T)) %>%
  filter(new_window=="yes") %>%
  select_at(vars("classe",colnames(testData))) %>% na.omit %>%
  rename_at(vars("classe"),funs(str_replace(.,"classe","Outcome")))
```

Note that, in the event that training data columns are not filtered in this manner, there are many occurrences of `NA` in the training data which presence will compromise model efficacy as one is then required to either omit `NA` data from the training process or to estimate a replacement value - not necessarily a sensible approach. As it happens, with this column pre-selection, no `NA` data occurs in the training data, so the addition of the pipe to `na.omit` is superfluous.

## Principle Components Analysis of the Training Data
Having curtailed the data available in the training data, we should verify the importance of the remaining variables via a Principle Components Analysis (PCA). We may construct a PCA model via:

```r
trainPCA <- FactoMineR::PCA(select_if(trainData,is.double),scale.unit=T,graph=F)
```

Even though we have reduced the independent variables to 24 in the training data, the variable factor map is still visually uninformative with this number of variables. However, the individuals factor map - that is, row-oriented impact upon determining principle components, shows no evident principal component clustering. The following charts depict, from left-to-right, the primary principle components highlighted by our `Outcome` class (A,B,C,D,E), the contribution of our filtered variables to the primary dimension of the PCA components with a vertical line showing a threshold of significance (of contribution) of 4%, and a correlation plot for all variables with respect to the reduced dimensions of the data.  


![](PCA.png)
Given this significance of how relatively well the remaining 24 variable explain the variance in class outcome with no extreme bias in any one or more variable, we will conduct modelling using all of the variable terms encoded in the training data.

# Modelling & Prediction
Another aspect of the lack of presence of the outcome variable in the testing data is that we may not conduct a stacked analysis. Consequently, we instead choose to use three three algorithms to compare their outcomes individually. 

## Decision Tree
Given the foregoing PCA analysis, it is also helpful to construct a decision tree regarding potential outcome prediction based upon ranges of values undertaken via the variables in the test data. Such a decision tree is as follows:

<img src="Project_files/figure-html/unnamed-chunk-7-1.png" style="display: block; margin: auto;" />

Given the lack of discrimination in principal components, this decision tree is not very specific - that is, despite what is shown, there are many of the 24 variables available whose values are not incorporated into a decision to traverse the tree the arrive at a distinct outcome. This is an indicator, in general, that outcome discrimination will not be highly accurate except in the instances where the depicted variable value ranges provide for node traversal.

## Modelling Algorithms 
We choose three algorithms; the Random Forest, Boosting and a Linear Discriminant Analysis (LDA). Note that the latter also undertakes dimension reduction steps in evaluating an optimal projection of outcome evaluation. It may, in this case, be considered better to use a Mixture Discriminant Analysis (MDA), however, given the lack of clustering exhibited (as in the foregoing figure) by the primary principal components, it is unlikely that an MDA analysis will produce any 'better' result than an LDA and, even then, the LDA outcome is likely to have limited ability in discriminating outcomes. Our selected models are evaluated thus:

```r
set.seed(1287)
modelRF <- train(Outcome~.,data=trainData,method="rf")
modelBOOST <- train(Outcome~.,data=trainData,method="gbm")
modelLDA <- train(Outcome~.,data=trainData,method="lda")
```

## Predicting Outcomes for Test Data
For the evaluated models, we undertake prediction against out training data via:

```r
predictionRF <- predict(modelRF,testData)
predictionBOOST <- predict(modelBOOST,testData)
predictionLDA <- predict(modelLDA,testData)
```

As discussed above, since the test data does not encode the *actual* outcome, we cannot now construct confusion matrices nor provide estimates of the out-of-process, predicted values for the test data. Nevertheless, we tabulate the predictions for each of the rows in the test data, and the in-process accuracy and variance for each of our models.



<table>
 <thead>
  <tr>
   <th style="text-align:left;"> Model </th>
   <th style="text-align:left;"> Predicted Test Data Outcomes </th>
   <th style="text-align:right;"> Accuracy % </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> Random Forest </td>
   <td style="text-align:left;"> {B,A,B,A,A,B,D,B,A,A,B,C,B,A,E,B,A,E,A,B} </td>
   <td style="text-align:right;"> 67.19 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Boost </td>
   <td style="text-align:left;"> {C,C,B,A,C,E,D,B,A,A,B,C,B,A,E,E,A,B,D,B} </td>
   <td style="text-align:right;"> 65.06 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> LDA </td>
   <td style="text-align:left;"> {E,A,A,A,A,B,D,D,A,C,B,A,A,A,A,A,A,D,D,B} </td>
   <td style="text-align:right;"> 41.55 </td>
  </tr>
</tbody>
</table>

Note that for the Random Forest model, the best accuracy is yielded when the number of randomly selected variables was 2 with a standard deviation in the accuracy of 
0.04. This result, alongside the decision tree, is highlighted by a plot of the decreasing importance of the variables used by the Random Forest model:  


<img src="Project_files/figure-html/unnamed-chunk-13-1.png" style="display: block; margin: auto;" />

Clearly `roll_belt` is a key indicator and it's likely that in the selection of two variables for optimal accuracy, if `roll_belt` is one, then many of the others can significantly contribute to improved accuracy.

For the boosted model, the reported, best accuracy of 65.06 has a standard deviation of 0.03 and is achieved when considered 150 trees with a maximum interaction depth of 3. For the LDA model, despite the standard deviation of the accuracy being reported as 0.03, it's clear that the linear partitioning of data given the visualisations to date, is unable to accurately discriminate such and predict a 'reasonable' outcome.

# References
