# ames_housing
Analysis and price prediction of the Ames Housing Prices data set.


## Purpose

As housing prices continue to raise, making an informed investment is even more important. Many features of houses can be appealing for you as a buyer and for future potential buyers when you decide to sell. The goal of this project was to examine which features contribute to the sale price to help you make a more informed decsiions when buying or preparing to sell a home.


## The Ames Housing Data Set
The dataset we will be using to answer this questions was developed by Dean De Cock at Truman State University. Seeking to build a dataset complementary to the Boston Housing Data Set he was trained on, De Cock worked with the City of Ames Iowa who released several years of housing sales data to him. After some loose data preparation he released it to the public for academic use. You can read the whole story in the [Original Paper.](http://jse.amstat.org/v19n3/decock.pdf)

The data set includes data on the following aspects of the sale.
- Location 
    - Zoning 
    - Streets & Alleys
    - Neighborhood 
    - Lot Area & Shape
- Building
    - Basements & Garages
    - Square Footage
    - Quaility & Condition
    - Materials
    - Ammenities (baths, fireplaces, pools)
- Meta-Data
    - Functionality
    - Miscellaneous Features
    - Sale Type & Condition (family sale, adjoining land purchase)
    - Sale Date

We can use all of these features to predict the Sale Price of the home. Predicting the price well will give us a sense of the importance of each feature and how much it adds to the price of the home. 

## Data Preparation
Before training a model on the data it needs to be cleaned, engineered, and standardized.

**1. Outliers**

Outliers can severly effect how a model estimates coefficients and determines what is important. Some outliers are easy to find such as the one home with non-standard Utilities. Other outliers are more open to interpretation. Above Ground Living Area includes two values at ~4500 and ~5600 square feet which are clearly outliers when GrLivArea is plotted against Sale Price. Another two sales appear appropriately priced for their square footage but also represent a large z-score and were removed. Similar to GrLivArea, LotArea shows several extreme outliers with several sales having lot areas greater than 100,000 square feet (2+ acres!). These four lots were removed from the training data. 

In total, 9 sales were removed from the data set, less than 1% of the total training data. 


**2. Null Values**

On first glance the data appears to be full of missing values. All but 7 values are missing from PoolQC, many houses are missing information about their basements, garages, alleys, and fences. However, by examining the provided data key it becomes obvious that most of these missing values are intentional. Most PoolQC values are null because most houses lack a pool. The same is true for basements, garages, fences, and alleys. 

When this intentional missingness is corrected only three features display missing information: LotFrontage (n=259), MasVnrType (n=8), and Electrical (n=1).

Electrical was imputed with the mode of the feature. MasVnrType was imputed to "None". LotFrontage was imputed by the intersection of Neighborhood and LotConfig. This is based on the planned nature of mid-west towns. Most construction follows a top-down approach leaving features like LotArea & LotFrontage standardized across Neighborhoods. The LotConfig supplies important information as well, a house on a cul-de-sac has much less frontage than a house on a corner lot. 


**3. Data Types and Misspellings**

Some features which appear as numbers are actually categories, MSSubClass appears as a number but actually represents discrete categories. 

Additionally, the Exterior2nd features includes misspelled materials. "CmentBd" should be "CemntBd"

**4. Feature Engineering**

For basement and Proch features I pivoted the data. 

BsmtFinType and square footage was stored in 4 columns (2 types, 2 areas). I pivoted the data so that each type (ALQ, Rec, Unf) received an individual column listing the square footage.

Porch features were organized the opposite way, each porch type had it's own column which I pivoted into a PorchType column (Screen, 3 Season, Multiple) and TotalPorchSF column which is the sum of all porch square feet for each home. 

Additionally I attempted to include features which might impact the price such as calculating the area of the yard, if the home has been remoded recently, the number of floors & the total finished SF of the home.

Optionally, Condition1/2 (proximity to roads/railroads/parks) and Exterior1st/2nd (Material used on the side of the house) can be dummified for the linear models. Since data is stored across multiple columns. I manually construct these columns instead of using One-Hot-encoding. 



**5. Model Preparation for Models**

For the models to make sense of the data, they need to be in specific format. For both linear & tree based models, all ordinal features were encoded None->Excellent as 0->5. 

For linear models:
- Categorical Features were One-Hot encoded, providing a 0/1 column for each category in each categorical column.
- Numeric Features were normalized via the PowerTransformer function with the "yeo-johnson" method. This allowed negative and 0 values to be normalized where log or box-cox would fail.

For tree models:
- Categorical Features were Label encoded via a custom function. This allows the sorting of the labels so that a OldTown with label 3 has a lower sale price than Timber with a label of 21. 
- Numeric Features were left unmodified as they do not effect tree model function. 


Finally the target variable SalePrice was also encoded with the PowerTransformer function (for both linear & tree models.)


## Model Creation

In order to avoid issues with multicollinearity, I constructed multiple feature sets, using mutual information with the SalePrice. This provided 5 feature lists, top 10%, top 20%, top 35%, features with non-zero mutual information, and the full feature set. 

Another step to avoiding multicollinearity was only selecting models with strong penalization functions. I used the following models
    - Lasso
    - Ridge
    - ElasticNet
    - SVR
    - RandomForest
    - GradientBoost
    - XGBoost


Before training the data was split 2:1 into training and validation data (I am using validation instead of test because the kaggle competition provides a test.csv)

Each of these models and feature set combinations was run through a hyperparameter tuning function which used cross-validation (GridSearchCV) on the training data. This produced the best hyperparameters for each model-feature combo. I then used this best model+hyperparameters+features to create a validation score from the partition of data I had withheld earlier.

Finally we can plot all the cross-validation test scores (one for each fold) and the validation score (from the non-training data) of the best hyperparameters for each model & feature set. 

I then select the best performing linear & tree models for further analysis.


## Results & Analysis

The best linear model and feature set (lasso with all features) and the best tree model (GradientBoost with all features) have a the following train, validation, and kaggle scores.


| Model | Train Error | Validation Error | Kaggle (RMSLE)  |
|-------|------|------|----|
| Lasso | $18,200 | $20,700 | 0.210  |
| Gradient Boost | $10,600 | $21,500 | 0.135  |


The residuals of both models suggest that predictive power starts to deteriorte for the higher priced homes, especially the lasso model.

Feature importance suggests that many obvious features are important, such as OverallQuality, Total Finished SF, Neighborhood, and Kitchen Quality.


This information can be used to better select a home to buy or improve your home to increase the sale price.

**Purchasing:**
Some variables are baked into the home at time of purchase. In order of importance based on our model:
- Total Square Feet (Especially Finished)
- Neighborhood
- LotArea
- Zoning
- Garage


**Selling:**
While updating your home is a personal choice, some target area's our model reveals are the following:
- Overall Quality & Condition
- Kitchen Quality
- Central Air
- Recent Remodeling

**Seemingly Unimportant**
Not only does our model tell us what is important, it lets us know what might not impact the price at all:
- Roofing Materials
- Lot Shape
- Paved Streets
- Building Type
- Fence
- Number of floors


Hopefully this can help you make an informed decision regarding you home purchasing or selling. 

## Credits:
Photo by <a href="https://unsplash.com/@brenoassis?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Breno Assis</a> on <a href="https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
 
 
 