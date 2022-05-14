---
title: "Covid Response Policy among Countries with Different Happiness Levels"
author: "Xiaolin Zheng, Sheng Cao, Xiangting Ye, Tianjiao Gao"
date: "5/13/2022"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = FALSE, warning = FALSE, message = FALSE)
```

```{r load the package, message=FALSE, warning=FALSE, include=FALSE}
library(tidyverse)
library(lubridate)
library(sf)
library(rnaturalearthdata)
library(rnaturalearth)
library(munsell)
library(patchwork)
library(tidymodels)
library(factoextra)
library(corrplot)
library(ggthemes)
library(plotly)
```


#### Data

```{r load the data, message=FALSE, warning=FALSE}
# Load the data
whs2019 <- read_csv("2019.csv")
whs2020 <- read_csv("2020.csv")
covid2020 <- read_csv("OxCGRT_withnotes_2020.csv")
```

```{r data cleaning for whs2019 dataset, message=FALSE, warning=FALSE}
# Data cleaning for whs2019 dataset
  
  # Clean the variable names
  whs2019 <- whs2019 %>%
  janitor::clean_names()

  # Select the variables
  whs2019 <- whs2019 %>%
    select(country_name, regional_indicator, ladder_score, logged_gdp_per_capita, social_support, healthy_life_expectancy, freedom_to_make_life_choices, generosity, perceptions_of_corruption) %>%
    rename("happiness" = "ladder_score")
  
  # Delete the missing values
  whs2019 <- whs2019 %>%
    na.omit()
```

```{r data cleaning for whs2020 dataset, message=FALSE, warning=FALSE}
# Data cleaning for whs2020 dataset
  
  # Clean the variable names
  whs2020 <- whs2020 %>%
  janitor::clean_names()

  # Select the variables
  whs2020 <- whs2020 %>%
    select(country_name, regional_indicator, logged_gdp_per_capita, social_support, healthy_life_expectancy, freedom_to_make_life_choices, generosity, perceptions_of_corruption) 

  # Delete the missing values
  whs2020 <- whs2020 %>%
    na.omit()
```

```{r data cleaning for covid2020 dataset, message=FALSE, warning=FALSE}
# Data cleaning for covid2020 dataset

  # clean the data variable name
  covid2020 <- covid2020 %>%
    janitor::clean_names()
  
  # select and rename variables
  covid2020 <- covid2020 %>%
    select(c("country_name", "country_code", "region_name", "stringency_index_for_display", "government_response_index_for_display", "containment_health_index_for_display", "economic_support_index_for_display")) %>%
    rename(
      "SI" = "stringency_index_for_display",
      "GRI" = "government_response_index_for_display",
      "CHI" = "containment_health_index_for_display",
      "ESI" = "economic_support_index_for_display"
        )
  
  # group by country
  covid2020 <- covid2020 %>%
    group_by(country_name, country_code) %>%
    summarize(
      SI = mean(SI, na.rm = TRUE),
      GRI = mean(GRI, na.rm = TRUE),
      CHI = mean(CHI, na.rm = TRUE),
      ESI = mean(ESI, na.rm = TRUE)
    )
```

```{r create the country code index}
# create the country code index
code_index <- covid2020 %>%
  select(country_name, country_code)
```

```{r join country code index to whs2019}
# add country code column to whs2019 so that we can match whs2019 to the geometry dataset using the country code
whs2019 <- whs2019 %>%
  left_join(code_index, by = "country_name")
```

```{r join country code index to whs2020}
# add country code column to whs2020 so that we can match whs2019 to the geometry dataset using the country code
whs2020 <- whs2020 %>%
  left_join(code_index, by = "country_name")
```

```{r sum the number of countries in each dataset, include=FALSE}
# sum the number of countries left in each dataset after data cleaning
nrow(whs2019)
nrow(whs2020)
nrow(covid2020)
```

For this analysis, we combine two data sources. The first one is [World Happiness Report 2019 and 2020](https://worldhappiness.report/), a landmark survey of the state of global happiness that ranks worldwide countries by how happy their citizens perceive themselves to be. Among the variables, the happiness score is a national average of the responses to the main life evaluation question asked in the Gallup World Poll (GWP), which uses the Cantril Ladder. Happiness score can be explained by the other six factors, namely  GDP per capita, Healthy Life Expectancy, Social support, Freedom to make life choices, Generosity, and Corruption Perception, which are also included in the dataset.

The second source is the database of variation in government responses to COVID-19 from 2020 to now collected by the [Oxford COVID-19 Government Response Tracker (OXCGRT)](https://www.bsg.ox.ac.uk/research/research-projects/covid-19-government-response-tracker), providing a systematic cross-national, cross-temporal measure to understand how government responses have evolved over the full period of the disease's spread. It also offered the composition and calculation of several policy indices, among which we took advantage of 4 indices, namely government response index, containment and health index, stringency index and economic support index.

After deleting missing values and selecting variables of interest, we are left with 153 countries in World Happiness Report 2019, 136 countries in World Happiness Report 2020 and 187 countries in Oxford COVID-19 Government Response Tracker 2020.

**Full Github Repository:** [PPOL670 Intro to Data Science Final Project](https://vladimircao.github.io/final_project/)

The full analysis can also be [read alongside the code that produced the figures.](/final-project.html)


# Part I. Exploratory Data Analysis


The correlation matrix displays the correlation between six happiness factors, of which the color intensity is proportional to the coefficients. As shown above, except for generosity, all other five factors show strong correlation. There are strong positive correlation between GDP, social support, healthy life expectancy and freedom to make life choices and negative correlation between perceptions of corruption and other variables.

```{r prepare for correlation matrix}
whs2019cor <- whs2019 %>%
  # select numeric columns
  select_if(is.numeric) %>%
  # select six happiness factors
  select(-c("happiness"))
```

```{r draw the correlation matrix plot}
# form the correlation matrix table of happiness factors
cor_whs <- cor(whs2019cor) 

# rename column and row names
colnames(cor_whs) <- c("GDP per capita", "Social Support", "Healthy Life Expectancy", "Freedom to Make Life Choices", "Generosity", "Perceptions of Corruption")
rownames(cor_whs) <- c("GDP per capita", "Social Support", "Healthy Life Expectancy", "Freedom to Make Life Choices", "Generosity", "Perceptions of Corruption")

# draw the correlation matrix plot
corrplot(cor_whs,
         tl.col = c("black"),
         title = "Correlation Matrix of Happiness Factors in 2019",
          mar=c(0,0,2,0))
```

```{r load the world map data, include=FALSE}
# load the world map data
world <- ne_countries(scale = "medium", returnclass = "sf")
```

```{r clean the world map dataest, include=FALSE}
# clean the world map dataset
world_map <- world %>%
  select(admin, adm0_a3, geometry) %>%
  rename(
    "country_name" = "admin",
    "country_code" = "adm0_a3"
  ) %>%
  st_transform(crs = 4326)
```

```{r match two datasets, include=FALSE}
# match two datasets
whs2019_geometry <- left_join(world_map, whs2019, by = "country_code") %>%
  na.omit()
```

```{r draw happiness scores map}
# draw the happiness socres map
P1 <- ggplot(whs2019_geometry) +
  geom_sf(aes(fill = happiness)) +
  scale_fill_distiller(palette = "YlOrBr") +
  theme_void() +
  labs(
    title = "Developed Countries Represent Higher Levels of Happiness",
    subtitle = "World Happiness Score by Country in 2019",
    caption = "SOURCE: World Happiness Report 2019 ",
    fill = "Happiness Score"
  )
```

As we can see in the map, dark orange stands for lower level of happiness, and vice versa. We can see that in 2019, people in North America, Western Europe, Australia and New Zealand report higher happiness scores, corresponding to most developed countries in the world. In contrast, low-income countries like Southern Asia and Africa represent the lowest happiness scores worldwide, corresponding to a darker color.

```{r, warning=FALSE, message=FALSE}
P1
```





```{r split the  data, message=FALSE, warning=FALSE, include=FALSE}
# set the seed
set.seed(20211101)

# split the data
split <- initial_split(whs2019)
whs19_train <- training(split)
whs19_test <- testing(split)
```

```{r set up folds, message=FALSE, warning=FALSE, include=FALSE}
# set the seed
set.seed(20211101)

# set up v-fold cross-validation
folds <- vfold_cv(data = whs19_train, v = 10)
```

```{r create a recipe, include=FALSE}
# create a recipe
whs_rec <- 
  recipe(happiness ~ ., data = whs19_train) %>%
  # select all numeric variables and remove covid-related variables
  step_rm(country_name, regional_indicator, country_code) %>%
  
  # center and scale predictors
  step_center(all_numeric_predictors()) %>%
  step_scale(all_numeric_predictors()) %>%
  prep()
  
# see the engineered training data
bake(prep(whs_rec, data = whs19_train), new_data = whs19_train)
```


```{r set up for the model, message=FALSE, warning=FALSE, include=FALSE}
# number of features in the dataset
n_features <- length(setdiff(names(whs19_train), "happiness"))

# create hyperparameter grid for random forest model
rf_grid <- expand.grid(
  mtry = floor(n_features * c(.05, .15, .25, .333, .4)),
  min_n = c(1, 3, 5, 10)
)
```

```{r create a Random Forest model, message=FALSE, warning=FALSE, include=FALSE}
# create a model
rf_mod <- rand_forest(
  mtry = tune(),
  trees = n_features * 10,
  min_n = tune()
) %>%
  set_engine("ranger") %>%
  set_mode("regression")
```

```{r create the workflow for Random Forest model, message=FALSE, warning=FALSE, include=FALSE}
# create a workflow
rf_wf <- workflow() %>%
  add_recipe(whs_rec) %>%
  add_model(rf_mod)
```

```{r fit the Random Forest model, message=FALSE, warning=FALSE, include=FALSE}
# fit the model
rf_cv <- rf_wf %>%
  tune_grid(
    resamples = folds,
    grid = rf_grid
  )
```

```{r select the best parameter for Random Forest model, message=FALSE, warning=FALSE, include=FALSE}
# select the best model based on the "rmse" metric
rf_best <- rf_cv %>%
  select_best(metric = "rmse")
```

```{r update the workflow for the Random Forest model, message=FALSE, warning=FALSE, include=FALSE}
# use the best model to update the workflow
rf_final <- finalize_workflow(
  rf_wf,
  parameters = rf_best
)
```

```{r fit the entire training data for Random Forest model, message=FALSE, warning=FALSE, include=FALSE}
# use the best parameter of Random Forest model to fit the entire training data
rf_fittrain <-
  rf_wf %>%
  finalize_workflow(parameters = rf_best) %>%
  fit(data = whs19_train) 
```

```{r make predictions using testing data, message=FALSE, warning=FALSE, include=FALSE}
# make predictions with the testing data
pred_testrf <-
  bind_cols(
    whs19_test,
    predict(rf_fittrain, new_data = whs19_test)
  )
```

```{r calculate the RMSE, message=FALSE, warning=FALSE, include=FALSE}
# calculate the RMSE on the testing data
rmse(data = pred_testrf, truth = happiness, estimate = .pred)
```


```{r set the tuning grid, include=FALSE}
# set the tuning grid for lasso model
lasso_grid <- grid_regular(penalty(), levels = 10)
```

```{r create a Lasso model, include=FALSE}
# create a lasso model
lasso_mod <- linear_reg(
  penalty = tune(),
  mixture = 1
  ) %>%
  set_engine("glmnet")
```

```{r create the workflow for Lasso model}
# create the workflow for lasso model
lasso_wf <- workflow() %>%
  add_recipe(whs_rec) %>%
  add_model(lasso_mod)
```

```{r fit the Lasso model, message=FALSE, warning=FALSE}
# fit the model
lasso_cv <- lasso_wf %>%
  tune_grid(
    resample = folds,
    grid = lasso_grid
    )
```

```{r select the best parameter for Lasso model}
# select the best model based on the "rmse" metric
lasso_best <- lasso_cv %>%
  select_best(metric = "rmse")
```

```{r update the workflow for the Lasso model}
# use the best model to update the workflow
lasso_final <- finalize_workflow(
  lasso_wf,
  parameters = lasso_best
)
```

```{r fit the entire training data for Lasso model}
# use the best parameter of Lasso model to fit the entire training data
lasso_fittrain <- lasso_wf %>%
  finalize_workflow(parameters = lasso_best) %>%
  fit(data = whs19_train)
```

```{r make predictions using the testing data}
# make predictions with the testing data
pred_testlasso <-
  bind_cols(
    whs19_test,
    predict(lasso_fittrain, new_data = whs19_test)
  )
```

```{r calculate RMSE, include=FALSE}
# calculate the RMSE on the testing data
rmse(data = pred_testlasso, truth = happiness, estimate = .pred)
```





```{r implement using Random Forest model , message=FALSE, warning=FALSE, include=FALSE}
# make predictions for 2020 happiness score
happi_2020 <- bind_cols(
    whs2020,
    predict(object = rf_fittrain, new_data = whs2020)
  )

# rename the prediction column
happi_2020 <- happi_2020 %>%
  rename("happiness" = ".pred")

# show the prediction results
select(happi_2020, country_name, happiness)

# add covid policies index to 2020 predicted dataset
happi_2020 <- happi_2020 %>%
  select(-country_name) %>%
  inner_join(covid2020, by = "country_code") %>%
  na.omit()

```

# Part II. Machine Learning

We constructed Random Forest and LASSO model with data in 2019, and choose the model with smaller RMSE as the best model to predict the world happiness score in 2020. As the Random Forest Model returns a lower RMSE, we choose it as our best model to predict the happiness score in 2020. After Predicting the happiness score for each country in 2020, we conducted our analysis based on this new predicted dataset.

### A. Pre-analysis: OLS regression 

Before doing cluster analysis, we use OLS regression to see the relationship between covid response policy index and predicted happiness scores in 2020. According to the graphs, we can see that with the exception of stringency index, other three indices
are associated with happiness scores. Thus, we infer that after we cluster happiness factors, covid response policy index and weighted happiness scores, index of economic support, government response and containment health would show difference between each cluster.

```{r run OLS regression}
# regress economic support index on happiness scores
p1 <- happi_2020 %>% ggplot(aes(x = ESI, y = happiness)) +
  geom_point() +
  geom_smooth(method = "lm",formula = y ~ x) + 
  labs(x = "Economic Support Index")

# regress stringency index on happiness scores
p2 <- happi_2020 %>% ggplot(aes(x = SI, y = happiness)) +
  geom_point() +
  geom_smooth(method = "lm",formula = y ~ x) + 
  labs(x = "Stringency Index")

# regress government response index on happiness scores
p3 <- happi_2020 %>% ggplot(aes(x = GRI, y = happiness)) +
  geom_point() +
  geom_smooth(method = "lm",formula = y ~ x) +
  labs(x = "Government Response Index")
 
# regression containment health index on happiness scores 
p4 <- happi_2020 %>% ggplot(aes(x = CHI, y = happiness)) +
  geom_point() +
  geom_smooth(method = "lm",formula = y ~ x) +
  labs(x = "Containment Health Index")

p1 + p2 + p3 + p4 & theme_minimal() & labs(y = "Happiness")
```

### B. Cluster Analysis

After we scaling the data to ensure all variables are on the same unit, we put more weight on happiness so that the clustered groups are able to reflect different levels of happiness. The next step is to conduct a cluster analysis, finding which countries are in each group accordingly with a world map and an interactive cluster plot. The suggested number of clusters is 2 and 7. However, for the purpose of better interpretation and visualization, we choose k = 3 as the number of clusters.

```{r prepare for cluster analysis, message=FALSE, warning=FALSE, include=FALSE}
# Select numeric value
happi_2020_num <- happi_2020 %>%
  select(-c(country_name, 
            regional_indicator, 
            country_code))

# Scale the dataset to be on the same unit
happi_2020_scale <- as.data.frame(scale(happi_2020_num))

# add more weight to the happiness score
happi_2020_scale <- happi_2020_scale %>%
  mutate(
    happiness = happiness * 3
  )

# Create a scaled dataframe with country_name
happi_2020_scale2 <- bind_cols(
  happi_2020_scale,
  happi_2020$country_name) %>%
  rename("country_name" = "...12")
```

```{r pick the number of clusters, message=FALSE, warning=FALSE}
# pick the number of clusters
set.seed(20220501)

# Calculate total within sum of squares
fviz_nbclust(happi_2020_scale, FUN = kmeans, method = "wss")

# Calculate silhouette distance
fviz_nbclust(happi_2020_scale, FUN = kmeans, method = "silhouette")

# Calculate gap statistics
fviz_nbclust(happi_2020_scale, FUN = kmeans, method = "gap_stat")
```



```{r run kmeans model, include=FALSE}
set.seed(12305812)
# run the kmeans model
whs2020_kmeans3 <- kmeans(happi_2020_scale, centers = 3, nstart = 100)

# combine the cluster column to th happi_2020_scale2
whs2020_clusters <- 
  bind_cols(happi_2020_scale2,
            cluster3 = whs2020_kmeans3$cluster)
```

```{r run PCA, include=FALSE}
# run PCA
set.seed(1019278012)
happi_2020_pca <- happi_2020_scale %>%
  prcomp()

# extract the principal components
happi_pca_pc1pc2 <- as_tibble(happi_2020_pca$x) %>%
  select(PC1, PC2)

# combine the first two PCs to whs2020_clusters
happi2020_cluster_pca <-  bind_cols(
  whs2020_clusters,
  happi_pca_pc1pc2
)
summary(happi_2020_pca)
```



```{r find the names of most central countries in each cluster, include=FALSE}
# find the names of most central countries in each cluster
central_happi_names <- happi2020_cluster_pca %>%
  group_by(cluster3) %>%
  mutate(dist = sqrt((PC1 - mean(PC1)) ^ 2 + (PC2 - mean(PC2)) ^ 2)) %>%
  slice_min(dist) %>%
  ungroup()

select(central_happi_names, country_name, cluster3, happiness)
```

After running PCA. The first principal component explains  0.65 of the variance. The second principal component explains 0.15 of the variance. To get a preliminary understanding the characteristics of each cluster, we find the names of most central countries in each cluster. In the first cluster, Germany is the most central country, corresponding to a happiness score of 4.44. In the second cluster, South Korea is the most central country, of which its happiness score is 0.45. In the third cluster, Senegal is the most central country and its happiness score is -3.27. So this provides us with evidence that the first to the third cluster group represent the highest to the least happiness scores accordingly.

```{r Create a plot of the clusters with PC1 and PC2 as the x and y axis, warning=FALSE}
Pcluster <- ggplot() +
  geom_point(
    data = happi2020_cluster_pca,
    mapping = aes(PC1, PC2,colour = factor(cluster3), text = country_name),
    alpha = 0.8,
  ) +
  labs(
    title = "K-Means with K = 3 and PCA",
    colour = "Cluster",
    x = "PC1 (0.65 of Variation)",
    y = "PC2 (0.15 of Variation)"
  ) + 
  theme_minimal() + 
  guides(text = NULL)

ggplotly(Pcluster, tooltip = "text")
```

```{r map the cluster group,warning=FALSE}
# add country_code back to the dataset 
whs2020_clusters <- left_join(whs2020_clusters, code_index, by = "country_name")

# add geometry to the dataset
whs2020_geometry <- left_join(world_map, whs2020_clusters, by = "country_code") %>%
  na.omit()

# map the cluster group
C1<- ggplot(data = whs2020_geometry) +
  geom_sf(aes(fill = factor(cluster3))) +
  scale_fill_brewer(palette = "Set4") +
  theme_void() +
  labs(
    fill = "Cluster",
    title = "Each Cluster is Representative of Happiness Levels",
    subtitle = "Geographical Distribution of Each Cluster",
    caption = "SOURCE: World Happiness Report 2020"
  )
```

Comparing the geographic distribution of each cluster group and happiness levels, we find significant similarities. The first group, mainly distributed in North America, Western Europe, Australia and New Zealand, corresponds to countries with the highest happiness levels whereas The third group, mainly in Africa and Southern Asia, represents the least happiness levels. The second group corresponds to countries with medium happiness level.

```{r, warning=FALSE}
C1 + P1+plot_layout(ncol=1)
```


### C. Compare Mean Covid Response Policy Index by Cluster Group

```{r get the mean of covid response policy index}
# get the mean of covid response policy index
happi2020_cluster_pca_m<- happi2020_cluster_pca %>%
  group_by(cluster3) %>%
  summarise_at(.vars = c("ESI", "SI", "GRI", "CHI"), .funs = mean)
```

```{r construct a function to compare the mean of covid-related index within each cluster group}
# construct a function that draw the graph to compare the covid response index
cluster_mean_graph <- function(df, var) {
  df %>% 
    ggplot() +
    geom_bar(aes(x = factor(cluster3),
                 y = var,
                 fill = factor(cluster3)),
             stat = "identity",
             legend = FALSE) +
    scale_fill_brewer(palette = "Set2") +
    theme_minimal() +
    labs(x = "",
         fill = "Cluster")
}
```

```{r message=FALSE, warning=FALSE, include=FALSE}
# draw the graph to compare the covid response index
pic1 <- cluster_mean_graph(happi2020_cluster_pca_m, happi2020_cluster_pca_m$SI) + labs(y = "Stringency Index")
pic2 <- cluster_mean_graph(happi2020_cluster_pca_m, happi2020_cluster_pca_m$GRI) + labs(y = "Government Response Index")
pic3 <- cluster_mean_graph(happi2020_cluster_pca_m, happi2020_cluster_pca_m$CHI)+ labs(y = "Conkktainment and Health Index")
pic4 <- cluster_mean_graph(happi2020_cluster_pca_m, happi2020_cluster_pca_m$ESI) + labs(y = "Economic Support Index")
```

As we can see from the graph above, cluster 1 and 3 have negative mean stringency index, imply that those countries are less likely to implement rigorous closure policy. While cluster 2 countries are prone to enforce stern closure policy. Stringency index’s difference between group is relatively indistinctive. 

This finding may be explained by two different reasons. The first explanation is that covid closure policy would not heavily affect people’s happiness. While given the fact that there are many researches in psychology have demonstrated that isolation from ordinary social life can produce significant negative impact on people’s mental health, this argument may be questionable. Another explanation is that closure policy would significantly affect people’s happiness, but its effects can be partly offset by economic support and health investment provided by governments. Considering the evidence in the fields of sociology and psychology, the second explanation may be more reasonable. 

As of the government response index, cluster 3 enjoys negative government response index, while cluster 1 and 2 countries' governments are more responsive compared with cluster 3 countries.

Government response index’s difference between groups is relatively insignificant. Considering that this index is the aggregation of other 3 indexes, the between group variation of economic support index and containment and health index may be partly offset by stringency index. The total effect of covid relevant policies appears to have moderate impact on people’s happiness index. 

Cluster 3 have negative containment and health index while cluster 1 countries' mean index is also nearly 0. Compared with cluster 2 countries, they are less prone to provide additional health investment. Cluster 3 countries' economic support on citizens are relatively weak, while countries for cluster 1 and 2 more prone to provide economic support for citizens, especially for countries in cluster 1.

These facts indicate that economic support index and containment and health index have larger difference between group based on happiness index compared with stringency index and government response index. This phenomenon may be explained by the relationship between countries’ economic condition and citizens’ happiness. Developed countries usually enjoy higher happiness index, and also have the ability to provide greater economic support and health investment. As countries are divided mainly based on happiness score, it is reasonable to discover that these two indexes are significantly vary between groups. 


```{r}
pic1+pic2+pic3+pic4 + plot_layout(guides = "collect")
```



