---
title: "Forecasting Flood Inundation"
author: "Leila Bahrami"
date: "2024-10-16"
output: 
  html_document:
    keep_md: yes
    toc: yes
    theme: flatly
    toc_float: yes
    code_folding: hide
    number_sections: yes
  pdf_document:
    toc: yes
---
According to the World Health Organization, floods are the most frequent natural disaster affecting countries across the globe. Floods have the potential to leave communities devastated not only because of the direct loss of life due to drowning and infrastructure damage, but also because of indirect impacts such as increased transmission of disease, higher risk of injury and hypothermia, disrupted or disabled infrastructure systems, and increased likelihood of causing other natural disasters. In the past ten years, climate change has exacerbated the effects of natural disasters like flooding, drought, sea level rise, and extreme precipitation and their frequency and intensity are expected to continue to rise if left unchecked.

City planners have the utmost responsibility to ensure their cities are prepared to respond to such natural disasters and minimize harm to their most vulnerable communities. The following document outlines a machine learning algorithm that predicts which areas of a city are at the highest risk of flooding disasters. An Emergency Management team can use the resulting model of this algorithm to inform their preparedness tactics to minimize damage and respond to future flooding disasters more efficiently. Ultimately, the algorithm is a helpful tool for individuals in Public Works, Public Health, City Planning, and Community-Based Organizations to proactively plan for resilience.

The model is created using past flood data from the City of Calgary in Canada to measure accuracy and then is applied to the City of Denver in the United States to measure generalizability. This document explains each step of the process such that City Planning departments can input their city’s data and accurately interpret the resulting model. We also wanted to provide transparency into the feature engineering steps of the process, which is why the procedures in ArcGIS and R are described in detail.

# SetUp
The algorithm was created using ArcGIS Pro and R. First, the necessary R libraries are loaded and themes are defined for the maps and plots that will be displayed later in the process.


``` r
# load libraries
library(caret)
library(pscl)
library(plotROC)
library(pROC)
library(sf)
library(tidyverse)
library(knitr)
library(kableExtra)
library(FNN)
library(scales)
library(jtools)
library(viridis)
library(gridExtra)


# load themes and palettes
mapTheme <- function(base_size = 12) {
  theme(
    text = element_text( color = "black"),
    plot.title = element_text(size = 14,colour = "black"),
    plot.subtitle=element_text(face="italic"),
    plot.caption=element_text(hjust=0),
    axis.ticks = element_blank(),
    panel.background = element_blank(),axis.title = element_blank(),
    axis.text = element_blank(),
    axis.title.x = element_blank(),
    axis.title.y = element_blank(),
    panel.grid.minor = element_blank(),
    panel.border = element_rect(colour = "black", fill=NA, size=2),
    strip.text.x = element_text(size = 14))
}

plotTheme <- function(base_size = 12) {
  theme(
    text = element_text( color = "black"),
    plot.title = element_text(size = 14,colour = "black"),
    plot.subtitle = element_text(face="italic"),
    plot.caption = element_text(hjust=0),
    axis.ticks = element_blank(),
    panel.background = element_blank(),
    panel.grid.major = element_line("grey80", size = 0.1),
    panel.grid.minor = element_blank(),
    panel.border = element_rect(colour = "black", fill=NA, size=2),
    strip.background = element_rect(fill = "grey80", color = "white"),
    strip.text = element_text(size=12),
    axis.title = element_text(size=12),
    axis.text = element_text(size=10),
    plot.background = element_blank(),
    legend.background = element_blank(),
    legend.title = element_text(colour = "black", face = "italic"),
    legend.text = element_text(colour = "black", face = "italic"),
    strip.text.x = element_text(size = 14)
  )
}
```
# Calgary Data

## Initial Data

The fishnet was generated in ArcGIS Pro using the Create Fishnet tool in order to generate a grid of the appropriate extent with 200 meter by 200 meter cells as the unit of measurement. This cell size was determined using the overall extent of the city boundary and the size of the resulting dataset. Cell size will vary depending on the city of focus and the computing power of the machine being used.

The inundation raster data was also generated in ArcGIS Pro using the Reclassify tool to indicate pixels within the flooding extent and pixels outside of the flooding extent. Inundated pixels are assigned a value of 1 and not inundated pixels are assigned a value of 0. Zonal Statistics were then used to calculate the maximum level of flooding per fishnet grid cell in order to visualize and identify which grid cells are at the highest risk of flooding.


``` r
# CALGARY: Load Calgary fishnet generated in ArcGIS Pro (200x200m cells).
getwd()
setwd("C:/Users/Leila'Laptop/Documents/GitHub/Calgary-Flood-Inundation-Modeling/Data")
Calgary <- st_read("calgary_criteria.shp")
#plot(Calgary, Inundation)
ggplot(data = Calgary) +
  geom_sf(aes(fill = Inundation)) +  # Color by 'Inundation'
  scale_fill_viridis_c() +           # Optional: Use a color scale from viridis
  theme_minimal() +                  # Apply a minimal theme
  labs(title = "Inundation Map", fill = "Inundation Level") +  # Add title and legend label
  mapTheme()   # Adjust legend position
```

![](Calgary-Flood-Inundation_files/figure-html/Load Data-1.png)<!-- -->

## Significant Features

Next, data for features that contribute to flooding and flood risk is loaded into ArcGIS Pro for calculations.Knowledge about the characteristics of the land such as topography, land cover, and hydrology can indicate how susceptible the area is to floods.  

### Elevation

Elevation generally describes the topography of the land. Areas at lower levels of elevation are at higher risk for floods than areas at higher elevations. We explored calculating the lowest degree of elevation per fishnet grid cell in Calgary, but found that using the mean helped the performance of our model. Mean elevation is stored in the elevation_mean feature variable, measured in meters.

To calculate our elevation_mean feature variable, a Digital Elevation Model (DEM) was loaded into ArcGIS. Zonal Statistics were used to calculate the mean per fishnet grid cell.

``` r
#Maps of All Significant Features included in the Model
map.elevation <- ggplot() +
  geom_sf(data = Calgary, aes(fill = elevation), color=NA) +
  scale_fill_viridis()+
  labs(title = "Elevation (Mean)", fill="Distance \n(meters)") + mapTheme()
```
### Distance to Nearest Stream
Understanding the structure of streams helps inform the intensity of potential flooding. If the water level of the stream exceeds the stream channel, then flooding occurs. Areas that are closer to streams are more vulnerable to floods than areas that are farther away.

The hydrology tools in ArcGIS Pro are used to turn the Calgary DEM into a stream network. First, the Fill tool was used to fill any sinks. The resulting surface is input in the Flow Direction tool to generate a raster showing the direction of flow out of each pixel. Direction is used to calculate Flow Accumulation per pixel - this step is described further in the next feature section. Assuming a drainage threshold of 25 square kilometers, Set Null is used to calculate the number of pixels that represent a drainage of this area or more, assign these cells a value of 1 and all other pixels a value of NODATA. The resulting Calgary stream network is displayed below. Lastly, the Generate Near Table tool is used to calculate the distance of each fishnet grid cell to a stream, stored as the dist_stream feature variable, measured in meters.

``` r
map.water <- ggplot() +
  geom_sf(data = Calgary, aes(fill = dist_water), color=NA) +
  scale_fill_viridis()+
  labs(title = "Distance to \n Nearest Stream", fill="Distance \n(meters)") + mapTheme()
```
### Maximum Flow Accumulation
Flooding occurs when water levels exceed the stream channel. Based on the direction of the streams calculated above, Flow Accumulation tool is used to calculate the accumulated weight of all pixels flowing into each downslope pixel in the raster. Zonal Statistics (maximum) is used to calculate the maximum flow accumulation per each fishnet grid cell. Using the maximum value allows us to evaluate at the highest risk level. In other words, the resulting model will present a “worst-case-scenario” picture enabling planners to be over-prepared rather than under-prepared. 

``` r
map.flowAccu <- ggplot() +
  geom_sf(data = Calgary, aes(fill = Flow_Accum), color=NA) +
  scale_fill_viridis()+
  labs(title = "Flow Accumulation \n (Maximum)", fill="Distance \n(meters)") + mapTheme()
```
### Distance to Nearest Steep Slope
Steep slopes in the land can contribute to both flow direction and flow accumulation. Additionally, a steeper slope likely accelerates the speed of flow and can intensify in the case of a flood. Thus, it is important to be aware of the location of steep slopes in the topography and their relative distance to past floods.

Slopes can be calculated using the Slope tool in ArcGIS. To distinguish which slopes are steep, Reclassify is used to assign slopes greater than or equal to 10 a value of 1 and all other slopes a value of NODATA. These steep slopes are converted from raster data to polygon features using the Raster to Polygon tool. Then, Near can be used once again to calculate the distance of each fishnet grid cell to a steep slope. 

``` r
map.steepslopedist <- ggplot() +
  geom_sf(data = Calgary, aes(fill = dist_slop), color=NA) +
  scale_fill_viridis()+
  labs(title = "Distance to \nNearest Steep Slope", fill="Distance \n(meters)") + mapTheme()
```
### Distance to Nearest Park
Land cover or soil type can be indicative of areas prone to flooding. The more permeable the surface is, the more risky it is for floods. Initially, land cover data for Calgary was used to identify which fishnet grid cells were impervious based on their land cover type; however, this did not help the model perform well. Instead, distance to parks is incorporated into the model. Parks are generally covered in grass which is fairly permeable. Thus, the closer a fishnet grid cell is to a park, the more likely it is to become inundated.

After bringing in the complete parks dataset in Calgary from the city’s open data portal, the Near tool in ArcGIS is used to perform this calculation for the distance to parks feature variable.

``` r
map.distparks <- ggplot() +
  geom_sf(data = Calgary, aes(fill = dist_park), color=NA) +
  scale_fill_viridis()+
  labs(title = "Distance to \nNearest Park", fill="Distance \n(meters)") + mapTheme()
```

### Tree Density
Higher tree density increases water absorption through root systems and promotes evapotranspiration, which can significantly reduce surface runoff. Trees also stabilize soil, preventing erosion and increasing water infiltration.


``` r
map.hydrolength <- ggplot() +
  geom_sf(data = Calgary, aes(fill = tree_dens), color=NA) +
  scale_fill_viridis()+
  labs(title = "Tree Density", fill="Distance \n(meters)") + mapTheme()
```
### Maps of All Significant Features included in the Model


``` r
grid.arrange(map.elevation, map.water,map.flowAccu,map.steepslopedist, map.hydrolength, map.distparks ,  
             ncol=3,
             top = "")
```

![](Calgary-Flood-Inundation_files/figure-html/allfeatures-1.png)<!-- -->

# Plotting Features per Inundation Outcome

These six selected significant features are shown below in the plots to show differences in across fishnet grid cells that flooded and those that did not, according to the classification from the 2013 inundation extent.

``` r
#Plotting Features per Inundation Outcome
CalgaryPlotVariables <- 
  Calgary %>%
  as.data.frame() %>%
  select(Inundation, elevation, dist_water, tree_dens, 
         Flow_Accum, dist_slop, dist_park,
         flowlength, dev_sum, PopDens) %>%
  gather(key, value, elevation:PopDens) %>%
  mutate(value = ifelse(key == "Inundation", value*1000, value))


# Do again to scale the bars
CalgaryPlotVariables <- 
  CalgaryPlotVariables %>%
  mutate(value = ifelse(key == "Inundation", value*1000, value))

ggplot(CalgaryPlotVariables, aes(as.factor(Inundation), value, fill=as.factor(Inundation))) + 
  geom_bar(stat="identity") + 
  facet_wrap(~key, scales="free") +
  scale_fill_manual(values = c("dodgerblue4", "darkgreen"),
                    labels = c("Not Inundation","Inundation"),
                    name = "") +
  labs(x="Inundation", y="Value")
```

![](Calgary-Flood-Inundation_files/figure-html/Plot-1.png)<!-- -->

# Model Development & Testing on Calgary

Now that the features have been prepared, we can begin to build the predictive model. First, we split up our known data set into a training and a test set.  

## Iterative Development Process

A linear regression is performed on the training set to determine the correlation between the set of selected features for the training data and the inundation data for the training data. This process was repeated multiple times to determine which combination of engineered features performed the best. For instance, recall from the Distance to Parks section that an impervious surfaces feature had been engineered but ultimately was not included in the final model. Reviewing the results from multiple regressions helped us determine which combination of features would provide the best results. The results of the final model are displayed and discussed below:

``` r
# set Training vs Testing
set.seed(3456)
trainIndex <- createDataPartition(Calgary$Inundation, p = .70,
                                  list = FALSE,
                                  times = 1)
CalgaryTrain <- Calgary[ trainIndex,]
CalgaryTest  <- Calgary[-trainIndex,]

InundationModel <- glm(Inundation ~ . - Id, 
                    family="binomial"(link="logit"), data = CalgaryTrain %>%
                      as.data.frame() %>%
                      select(-geometry, -basin, -dist_deve, dist_fores, -Residentia, -dist_fores))
```
## Results  

### Model Summary
The output displays various statistics that indicate the quality of the model. For instance, the p-value listed for each feature describes how significant the feature is to the outcome of whether or not a flood occurs. All of these features are statistically significant as their p-values are <0.05. Additionally, measures like the Pseudo-R^2 and the AIC score describe how closely related our features are to the outcome. This is the point at which features can be removed, added, or adjusted within the model to enhance performance.

``` r
summary(InundationModel)
```

```
## 
## Call:
## glm(formula = Inundation ~ . - Id, family = binomial(link = "logit"), 
##     data = CalgaryTrain %>% as.data.frame() %>% select(-geometry, 
##         -basin, -dist_deve, dist_fores, -Residentia, -dist_fores))
## 
## Coefficients:
##               Estimate Std. Error z value Pr(>|z|)    
## (Intercept)  6.456e+01  6.502e+00   9.929  < 2e-16 ***
## elevation   -6.321e-02  6.476e-03  -9.760  < 2e-16 ***
## PopDens      3.469e+02  6.451e+01   5.377 7.56e-08 ***
## dist_water  -2.303e-03  3.902e-04  -5.902 3.60e-09 ***
## tree_dens    1.017e-09  1.393e-08   0.073  0.94182    
## Flow_Accum  -3.405e-07  2.424e-07  -1.405  0.16005    
## dev_sum     -1.261e-02  1.835e-03  -6.869 6.46e-12 ***
## flowlength   4.368e-05  1.335e-05   3.271  0.00107 ** 
## dist_slop   -2.999e-04  1.561e-04  -1.922  0.05464 .  
## dist_park   -3.077e-04  7.136e-05  -4.312 1.62e-05 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## (Dispersion parameter for binomial family taken to be 1)
## 
##     Null deviance: 1951.05  on 5757  degrees of freedom
## Residual deviance:  943.48  on 5748  degrees of freedom
## AIC: 963.48
## 
## Number of Fisher Scoring iterations: 10
```

### Histogram of classProbs

Once the features have been finalized, we can run the regression model on our test data to further examine its accuracy. The results are displayed in the histogram below showing the frequency of each probability level that a flood will occur.

``` r
classProbs <- predict(InundationModel, CalgaryTest, type="response")

hist((classProbs), main = paste("Histogram of classProbs"), col = "blue", xlab = "Inundation Probability") + plotTheme()
```

![](Calgary-Flood-Inundation_files/figure-html/Histogram-1.png)<!-- -->


### Distribution of Probabilities Visualization

The visualization below shows the distribution of outcomes for the model. It is clear that the model is better at predicting 0s (no inundation) than predicting 1s (inundation).

``` r
#Distribution of Probabilities Visualization
testProbs <- data.frame(obs = as.numeric(CalgaryTest$Inundation),
                        pred = classProbs)


ggplot(testProbs, aes(x = pred, fill=as.factor(obs))) + geom_density() +
  facet_grid(obs ~ .) + xlab("Probability") + geom_vline(xintercept = .53) +
  scale_fill_manual(values = c("darkgreen", "navy"),
                    labels = c("Not Flooded","Flooded"),
                    name="") +
  labs(title = "Distribution of Probabilities") + plotTheme()
```

![](Calgary-Flood-Inundation_files/figure-html/Probability-1.png)<!-- -->

### Finding the Optimal Threshold
In order to best optimize this model for use, identifying the threshold that provides the greatest accuracy is important. This is done by running the iterateThresholds function and looking at the results for all thresholds the table of thresholds 0.01 through 0.99. The optimal threshold in this case is 0.53.

``` r
# iterateThresholds function
iterateThresholds <- function(data, group) {
  group <- enquo(group)
  x = .01
  all_prediction <- data.frame()
  while (x <= 1) {
    
    this_prediction <-
      testProbs %>%
      mutate(predOutcome = ifelse(pred > x, 1, 0)) %>%
      group_by(!!group) %>%
      dplyr::count(predOutcome, obs) %>%
      dplyr::summarize(sum_TN = sum(n[predOutcome==0 & obs==0]),
                       sum_TP = sum(n[predOutcome==1 & obs==1]),
                       sum_FN = sum(n[predOutcome==0 & obs==1]),
                       sum_FP = sum(n[predOutcome==1 & obs==0]),
                       total=sum(n)) %>%
      mutate(True_Positive = sum_TP / total,
             True_Negative = sum_TN / total,
             False_Negative = sum_FN / total,
             False_Positive = sum_FP / total,
             Accuracy = (sum_TP + sum_TN) / total, Threshold = x)
    
    all_prediction <- rbind(all_prediction, this_prediction)
    x <- x + .01
  }
  return(all_prediction)
}


whichThreshold <- iterateThresholds(testProbs)

allThresholds<-kable(whichThreshold) %>%
  kable_styling(bootstrap_options = "striped", full_width = FALSE)%>%
  scroll_box(width = "100%", height = "500px")

allThresholds
```

<div style="border: 1px solid #ddd; padding: 0px; overflow-y: scroll; height:500px; overflow-x: scroll; width:100%; "><table class="table table-striped" style="width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:right;position: sticky; top:0; background-color: #FFFFFF;"> sum_TN </th>
   <th style="text-align:right;position: sticky; top:0; background-color: #FFFFFF;"> sum_TP </th>
   <th style="text-align:right;position: sticky; top:0; background-color: #FFFFFF;"> sum_FN </th>
   <th style="text-align:right;position: sticky; top:0; background-color: #FFFFFF;"> sum_FP </th>
   <th style="text-align:right;position: sticky; top:0; background-color: #FFFFFF;"> total </th>
   <th style="text-align:right;position: sticky; top:0; background-color: #FFFFFF;"> True_Positive </th>
   <th style="text-align:right;position: sticky; top:0; background-color: #FFFFFF;"> True_Negative </th>
   <th style="text-align:right;position: sticky; top:0; background-color: #FFFFFF;"> False_Negative </th>
   <th style="text-align:right;position: sticky; top:0; background-color: #FFFFFF;"> False_Positive </th>
   <th style="text-align:right;position: sticky; top:0; background-color: #FFFFFF;"> Accuracy </th>
   <th style="text-align:right;position: sticky; top:0; background-color: #FFFFFF;"> Threshold </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 1805 </td>
   <td style="text-align:right;"> 104 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 557 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0421565 </td>
   <td style="text-align:right;"> 0.7316579 </td>
   <td style="text-align:right;"> 0.0004054 </td>
   <td style="text-align:right;"> 0.2257803 </td>
   <td style="text-align:right;"> 0.7738143 </td>
   <td style="text-align:right;"> 0.01 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 1932 </td>
   <td style="text-align:right;"> 104 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 430 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0421565 </td>
   <td style="text-align:right;"> 0.7831374 </td>
   <td style="text-align:right;"> 0.0004054 </td>
   <td style="text-align:right;"> 0.1743008 </td>
   <td style="text-align:right;"> 0.8252939 </td>
   <td style="text-align:right;"> 0.02 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2014 </td>
   <td style="text-align:right;"> 103 </td>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:right;"> 348 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0417511 </td>
   <td style="text-align:right;"> 0.8163762 </td>
   <td style="text-align:right;"> 0.0008107 </td>
   <td style="text-align:right;"> 0.1410620 </td>
   <td style="text-align:right;"> 0.8581273 </td>
   <td style="text-align:right;"> 0.03 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2065 </td>
   <td style="text-align:right;"> 102 </td>
   <td style="text-align:right;"> 3 </td>
   <td style="text-align:right;"> 297 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0413458 </td>
   <td style="text-align:right;"> 0.8370490 </td>
   <td style="text-align:right;"> 0.0012161 </td>
   <td style="text-align:right;"> 0.1203891 </td>
   <td style="text-align:right;"> 0.8783948 </td>
   <td style="text-align:right;"> 0.04 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2103 </td>
   <td style="text-align:right;"> 99 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 259 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0401297 </td>
   <td style="text-align:right;"> 0.8524524 </td>
   <td style="text-align:right;"> 0.0024321 </td>
   <td style="text-align:right;"> 0.1049858 </td>
   <td style="text-align:right;"> 0.8925821 </td>
   <td style="text-align:right;"> 0.05 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2129 </td>
   <td style="text-align:right;"> 98 </td>
   <td style="text-align:right;"> 7 </td>
   <td style="text-align:right;"> 233 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0397244 </td>
   <td style="text-align:right;"> 0.8629915 </td>
   <td style="text-align:right;"> 0.0028375 </td>
   <td style="text-align:right;"> 0.0944467 </td>
   <td style="text-align:right;"> 0.9027158 </td>
   <td style="text-align:right;"> 0.06 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2153 </td>
   <td style="text-align:right;"> 96 </td>
   <td style="text-align:right;"> 9 </td>
   <td style="text-align:right;"> 209 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0389137 </td>
   <td style="text-align:right;"> 0.8727199 </td>
   <td style="text-align:right;"> 0.0036482 </td>
   <td style="text-align:right;"> 0.0847183 </td>
   <td style="text-align:right;"> 0.9116336 </td>
   <td style="text-align:right;"> 0.07 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2176 </td>
   <td style="text-align:right;"> 94 </td>
   <td style="text-align:right;"> 11 </td>
   <td style="text-align:right;"> 186 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0381030 </td>
   <td style="text-align:right;"> 0.8820430 </td>
   <td style="text-align:right;"> 0.0044589 </td>
   <td style="text-align:right;"> 0.0753952 </td>
   <td style="text-align:right;"> 0.9201459 </td>
   <td style="text-align:right;"> 0.08 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2191 </td>
   <td style="text-align:right;"> 90 </td>
   <td style="text-align:right;"> 15 </td>
   <td style="text-align:right;"> 171 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0364816 </td>
   <td style="text-align:right;"> 0.8881232 </td>
   <td style="text-align:right;"> 0.0060803 </td>
   <td style="text-align:right;"> 0.0693150 </td>
   <td style="text-align:right;"> 0.9246048 </td>
   <td style="text-align:right;"> 0.09 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2202 </td>
   <td style="text-align:right;"> 89 </td>
   <td style="text-align:right;"> 16 </td>
   <td style="text-align:right;"> 160 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0360762 </td>
   <td style="text-align:right;"> 0.8925821 </td>
   <td style="text-align:right;"> 0.0064856 </td>
   <td style="text-align:right;"> 0.0648561 </td>
   <td style="text-align:right;"> 0.9286583 </td>
   <td style="text-align:right;"> 0.10 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2212 </td>
   <td style="text-align:right;"> 88 </td>
   <td style="text-align:right;"> 17 </td>
   <td style="text-align:right;"> 150 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0356709 </td>
   <td style="text-align:right;"> 0.8966356 </td>
   <td style="text-align:right;"> 0.0068910 </td>
   <td style="text-align:right;"> 0.0608026 </td>
   <td style="text-align:right;"> 0.9323064 </td>
   <td style="text-align:right;"> 0.11 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2228 </td>
   <td style="text-align:right;"> 86 </td>
   <td style="text-align:right;"> 19 </td>
   <td style="text-align:right;"> 134 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0348602 </td>
   <td style="text-align:right;"> 0.9031212 </td>
   <td style="text-align:right;"> 0.0077017 </td>
   <td style="text-align:right;"> 0.0543170 </td>
   <td style="text-align:right;"> 0.9379814 </td>
   <td style="text-align:right;"> 0.12 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2238 </td>
   <td style="text-align:right;"> 84 </td>
   <td style="text-align:right;"> 21 </td>
   <td style="text-align:right;"> 124 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0340495 </td>
   <td style="text-align:right;"> 0.9071747 </td>
   <td style="text-align:right;"> 0.0085124 </td>
   <td style="text-align:right;"> 0.0502635 </td>
   <td style="text-align:right;"> 0.9412242 </td>
   <td style="text-align:right;"> 0.13 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2245 </td>
   <td style="text-align:right;"> 84 </td>
   <td style="text-align:right;"> 21 </td>
   <td style="text-align:right;"> 117 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0340495 </td>
   <td style="text-align:right;"> 0.9100122 </td>
   <td style="text-align:right;"> 0.0085124 </td>
   <td style="text-align:right;"> 0.0474260 </td>
   <td style="text-align:right;"> 0.9440616 </td>
   <td style="text-align:right;"> 0.14 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2251 </td>
   <td style="text-align:right;"> 83 </td>
   <td style="text-align:right;"> 22 </td>
   <td style="text-align:right;"> 111 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0336441 </td>
   <td style="text-align:right;"> 0.9124443 </td>
   <td style="text-align:right;"> 0.0089177 </td>
   <td style="text-align:right;"> 0.0449939 </td>
   <td style="text-align:right;"> 0.9460884 </td>
   <td style="text-align:right;"> 0.15 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2256 </td>
   <td style="text-align:right;"> 81 </td>
   <td style="text-align:right;"> 24 </td>
   <td style="text-align:right;"> 106 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0328334 </td>
   <td style="text-align:right;"> 0.9144710 </td>
   <td style="text-align:right;"> 0.0097284 </td>
   <td style="text-align:right;"> 0.0429672 </td>
   <td style="text-align:right;"> 0.9473044 </td>
   <td style="text-align:right;"> 0.16 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2263 </td>
   <td style="text-align:right;"> 80 </td>
   <td style="text-align:right;"> 25 </td>
   <td style="text-align:right;"> 99 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0324281 </td>
   <td style="text-align:right;"> 0.9173085 </td>
   <td style="text-align:right;"> 0.0101338 </td>
   <td style="text-align:right;"> 0.0401297 </td>
   <td style="text-align:right;"> 0.9497365 </td>
   <td style="text-align:right;"> 0.17 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2271 </td>
   <td style="text-align:right;"> 79 </td>
   <td style="text-align:right;"> 26 </td>
   <td style="text-align:right;"> 91 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0320227 </td>
   <td style="text-align:right;"> 0.9205513 </td>
   <td style="text-align:right;"> 0.0105391 </td>
   <td style="text-align:right;"> 0.0368869 </td>
   <td style="text-align:right;"> 0.9525740 </td>
   <td style="text-align:right;"> 0.18 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2278 </td>
   <td style="text-align:right;"> 78 </td>
   <td style="text-align:right;"> 27 </td>
   <td style="text-align:right;"> 84 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0316173 </td>
   <td style="text-align:right;"> 0.9233887 </td>
   <td style="text-align:right;"> 0.0109445 </td>
   <td style="text-align:right;"> 0.0340495 </td>
   <td style="text-align:right;"> 0.9550061 </td>
   <td style="text-align:right;"> 0.19 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2284 </td>
   <td style="text-align:right;"> 78 </td>
   <td style="text-align:right;"> 27 </td>
   <td style="text-align:right;"> 78 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0316173 </td>
   <td style="text-align:right;"> 0.9258208 </td>
   <td style="text-align:right;"> 0.0109445 </td>
   <td style="text-align:right;"> 0.0316173 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.20 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2286 </td>
   <td style="text-align:right;"> 77 </td>
   <td style="text-align:right;"> 28 </td>
   <td style="text-align:right;"> 76 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0312120 </td>
   <td style="text-align:right;"> 0.9266315 </td>
   <td style="text-align:right;"> 0.0113498 </td>
   <td style="text-align:right;"> 0.0308066 </td>
   <td style="text-align:right;"> 0.9578435 </td>
   <td style="text-align:right;"> 0.21 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2290 </td>
   <td style="text-align:right;"> 77 </td>
   <td style="text-align:right;"> 28 </td>
   <td style="text-align:right;"> 72 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0312120 </td>
   <td style="text-align:right;"> 0.9282529 </td>
   <td style="text-align:right;"> 0.0113498 </td>
   <td style="text-align:right;"> 0.0291852 </td>
   <td style="text-align:right;"> 0.9594649 </td>
   <td style="text-align:right;"> 0.22 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2292 </td>
   <td style="text-align:right;"> 75 </td>
   <td style="text-align:right;"> 30 </td>
   <td style="text-align:right;"> 70 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0304013 </td>
   <td style="text-align:right;"> 0.9290636 </td>
   <td style="text-align:right;"> 0.0121605 </td>
   <td style="text-align:right;"> 0.0283745 </td>
   <td style="text-align:right;"> 0.9594649 </td>
   <td style="text-align:right;"> 0.23 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2294 </td>
   <td style="text-align:right;"> 73 </td>
   <td style="text-align:right;"> 32 </td>
   <td style="text-align:right;"> 68 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0295906 </td>
   <td style="text-align:right;"> 0.9298743 </td>
   <td style="text-align:right;"> 0.0129712 </td>
   <td style="text-align:right;"> 0.0275638 </td>
   <td style="text-align:right;"> 0.9594649 </td>
   <td style="text-align:right;"> 0.24 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2298 </td>
   <td style="text-align:right;"> 71 </td>
   <td style="text-align:right;"> 34 </td>
   <td style="text-align:right;"> 64 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0287799 </td>
   <td style="text-align:right;"> 0.9314957 </td>
   <td style="text-align:right;"> 0.0137819 </td>
   <td style="text-align:right;"> 0.0259424 </td>
   <td style="text-align:right;"> 0.9602756 </td>
   <td style="text-align:right;"> 0.25 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2303 </td>
   <td style="text-align:right;"> 70 </td>
   <td style="text-align:right;"> 35 </td>
   <td style="text-align:right;"> 59 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0283745 </td>
   <td style="text-align:right;"> 0.9335225 </td>
   <td style="text-align:right;"> 0.0141873 </td>
   <td style="text-align:right;"> 0.0239157 </td>
   <td style="text-align:right;"> 0.9618970 </td>
   <td style="text-align:right;"> 0.26 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2307 </td>
   <td style="text-align:right;"> 70 </td>
   <td style="text-align:right;"> 35 </td>
   <td style="text-align:right;"> 55 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0283745 </td>
   <td style="text-align:right;"> 0.9351439 </td>
   <td style="text-align:right;"> 0.0141873 </td>
   <td style="text-align:right;"> 0.0222943 </td>
   <td style="text-align:right;"> 0.9635184 </td>
   <td style="text-align:right;"> 0.27 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2309 </td>
   <td style="text-align:right;"> 66 </td>
   <td style="text-align:right;"> 39 </td>
   <td style="text-align:right;"> 53 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0267531 </td>
   <td style="text-align:right;"> 0.9359546 </td>
   <td style="text-align:right;"> 0.0158087 </td>
   <td style="text-align:right;"> 0.0214836 </td>
   <td style="text-align:right;"> 0.9627077 </td>
   <td style="text-align:right;"> 0.28 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2312 </td>
   <td style="text-align:right;"> 65 </td>
   <td style="text-align:right;"> 40 </td>
   <td style="text-align:right;"> 50 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0263478 </td>
   <td style="text-align:right;"> 0.9371707 </td>
   <td style="text-align:right;"> 0.0162140 </td>
   <td style="text-align:right;"> 0.0202675 </td>
   <td style="text-align:right;"> 0.9635184 </td>
   <td style="text-align:right;"> 0.29 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2314 </td>
   <td style="text-align:right;"> 65 </td>
   <td style="text-align:right;"> 40 </td>
   <td style="text-align:right;"> 48 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0263478 </td>
   <td style="text-align:right;"> 0.9379814 </td>
   <td style="text-align:right;"> 0.0162140 </td>
   <td style="text-align:right;"> 0.0194568 </td>
   <td style="text-align:right;"> 0.9643291 </td>
   <td style="text-align:right;"> 0.30 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2316 </td>
   <td style="text-align:right;"> 63 </td>
   <td style="text-align:right;"> 42 </td>
   <td style="text-align:right;"> 46 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0255371 </td>
   <td style="text-align:right;"> 0.9387921 </td>
   <td style="text-align:right;"> 0.0170247 </td>
   <td style="text-align:right;"> 0.0186461 </td>
   <td style="text-align:right;"> 0.9643291 </td>
   <td style="text-align:right;"> 0.31 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2317 </td>
   <td style="text-align:right;"> 62 </td>
   <td style="text-align:right;"> 43 </td>
   <td style="text-align:right;"> 45 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0251317 </td>
   <td style="text-align:right;"> 0.9391974 </td>
   <td style="text-align:right;"> 0.0174301 </td>
   <td style="text-align:right;"> 0.0182408 </td>
   <td style="text-align:right;"> 0.9643291 </td>
   <td style="text-align:right;"> 0.32 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2321 </td>
   <td style="text-align:right;"> 61 </td>
   <td style="text-align:right;"> 44 </td>
   <td style="text-align:right;"> 41 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0247264 </td>
   <td style="text-align:right;"> 0.9408188 </td>
   <td style="text-align:right;"> 0.0178354 </td>
   <td style="text-align:right;"> 0.0166194 </td>
   <td style="text-align:right;"> 0.9655452 </td>
   <td style="text-align:right;"> 0.33 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2321 </td>
   <td style="text-align:right;"> 60 </td>
   <td style="text-align:right;"> 45 </td>
   <td style="text-align:right;"> 41 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0243210 </td>
   <td style="text-align:right;"> 0.9408188 </td>
   <td style="text-align:right;"> 0.0182408 </td>
   <td style="text-align:right;"> 0.0166194 </td>
   <td style="text-align:right;"> 0.9651398 </td>
   <td style="text-align:right;"> 0.34 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2322 </td>
   <td style="text-align:right;"> 60 </td>
   <td style="text-align:right;"> 45 </td>
   <td style="text-align:right;"> 40 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0243210 </td>
   <td style="text-align:right;"> 0.9412242 </td>
   <td style="text-align:right;"> 0.0182408 </td>
   <td style="text-align:right;"> 0.0162140 </td>
   <td style="text-align:right;"> 0.9655452 </td>
   <td style="text-align:right;"> 0.35 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2323 </td>
   <td style="text-align:right;"> 58 </td>
   <td style="text-align:right;"> 47 </td>
   <td style="text-align:right;"> 39 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0235103 </td>
   <td style="text-align:right;"> 0.9416295 </td>
   <td style="text-align:right;"> 0.0190515 </td>
   <td style="text-align:right;"> 0.0158087 </td>
   <td style="text-align:right;"> 0.9651398 </td>
   <td style="text-align:right;"> 0.36 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2323 </td>
   <td style="text-align:right;"> 55 </td>
   <td style="text-align:right;"> 50 </td>
   <td style="text-align:right;"> 39 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0222943 </td>
   <td style="text-align:right;"> 0.9416295 </td>
   <td style="text-align:right;"> 0.0202675 </td>
   <td style="text-align:right;"> 0.0158087 </td>
   <td style="text-align:right;"> 0.9639238 </td>
   <td style="text-align:right;"> 0.37 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2325 </td>
   <td style="text-align:right;"> 55 </td>
   <td style="text-align:right;"> 50 </td>
   <td style="text-align:right;"> 37 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0222943 </td>
   <td style="text-align:right;"> 0.9424402 </td>
   <td style="text-align:right;"> 0.0202675 </td>
   <td style="text-align:right;"> 0.0149980 </td>
   <td style="text-align:right;"> 0.9647345 </td>
   <td style="text-align:right;"> 0.38 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2327 </td>
   <td style="text-align:right;"> 52 </td>
   <td style="text-align:right;"> 53 </td>
   <td style="text-align:right;"> 35 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0210782 </td>
   <td style="text-align:right;"> 0.9432509 </td>
   <td style="text-align:right;"> 0.0214836 </td>
   <td style="text-align:right;"> 0.0141873 </td>
   <td style="text-align:right;"> 0.9643291 </td>
   <td style="text-align:right;"> 0.39 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2328 </td>
   <td style="text-align:right;"> 50 </td>
   <td style="text-align:right;"> 55 </td>
   <td style="text-align:right;"> 34 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0202675 </td>
   <td style="text-align:right;"> 0.9436563 </td>
   <td style="text-align:right;"> 0.0222943 </td>
   <td style="text-align:right;"> 0.0137819 </td>
   <td style="text-align:right;"> 0.9639238 </td>
   <td style="text-align:right;"> 0.40 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2329 </td>
   <td style="text-align:right;"> 48 </td>
   <td style="text-align:right;"> 57 </td>
   <td style="text-align:right;"> 33 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0194568 </td>
   <td style="text-align:right;"> 0.9440616 </td>
   <td style="text-align:right;"> 0.0231050 </td>
   <td style="text-align:right;"> 0.0133766 </td>
   <td style="text-align:right;"> 0.9635184 </td>
   <td style="text-align:right;"> 0.41 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2330 </td>
   <td style="text-align:right;"> 48 </td>
   <td style="text-align:right;"> 57 </td>
   <td style="text-align:right;"> 32 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0194568 </td>
   <td style="text-align:right;"> 0.9444670 </td>
   <td style="text-align:right;"> 0.0231050 </td>
   <td style="text-align:right;"> 0.0129712 </td>
   <td style="text-align:right;"> 0.9639238 </td>
   <td style="text-align:right;"> 0.42 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2332 </td>
   <td style="text-align:right;"> 47 </td>
   <td style="text-align:right;"> 58 </td>
   <td style="text-align:right;"> 30 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0190515 </td>
   <td style="text-align:right;"> 0.9452777 </td>
   <td style="text-align:right;"> 0.0235103 </td>
   <td style="text-align:right;"> 0.0121605 </td>
   <td style="text-align:right;"> 0.9643291 </td>
   <td style="text-align:right;"> 0.43 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2333 </td>
   <td style="text-align:right;"> 45 </td>
   <td style="text-align:right;"> 60 </td>
   <td style="text-align:right;"> 29 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0182408 </td>
   <td style="text-align:right;"> 0.9456830 </td>
   <td style="text-align:right;"> 0.0243210 </td>
   <td style="text-align:right;"> 0.0117552 </td>
   <td style="text-align:right;"> 0.9639238 </td>
   <td style="text-align:right;"> 0.44 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2334 </td>
   <td style="text-align:right;"> 45 </td>
   <td style="text-align:right;"> 60 </td>
   <td style="text-align:right;"> 28 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0182408 </td>
   <td style="text-align:right;"> 0.9460884 </td>
   <td style="text-align:right;"> 0.0243210 </td>
   <td style="text-align:right;"> 0.0113498 </td>
   <td style="text-align:right;"> 0.9643291 </td>
   <td style="text-align:right;"> 0.45 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2335 </td>
   <td style="text-align:right;"> 45 </td>
   <td style="text-align:right;"> 60 </td>
   <td style="text-align:right;"> 27 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0182408 </td>
   <td style="text-align:right;"> 0.9464937 </td>
   <td style="text-align:right;"> 0.0243210 </td>
   <td style="text-align:right;"> 0.0109445 </td>
   <td style="text-align:right;"> 0.9647345 </td>
   <td style="text-align:right;"> 0.46 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2335 </td>
   <td style="text-align:right;"> 44 </td>
   <td style="text-align:right;"> 61 </td>
   <td style="text-align:right;"> 27 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0178354 </td>
   <td style="text-align:right;"> 0.9464937 </td>
   <td style="text-align:right;"> 0.0247264 </td>
   <td style="text-align:right;"> 0.0109445 </td>
   <td style="text-align:right;"> 0.9643291 </td>
   <td style="text-align:right;"> 0.47 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2337 </td>
   <td style="text-align:right;"> 43 </td>
   <td style="text-align:right;"> 62 </td>
   <td style="text-align:right;"> 25 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0174301 </td>
   <td style="text-align:right;"> 0.9473044 </td>
   <td style="text-align:right;"> 0.0251317 </td>
   <td style="text-align:right;"> 0.0101338 </td>
   <td style="text-align:right;"> 0.9647345 </td>
   <td style="text-align:right;"> 0.48 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2338 </td>
   <td style="text-align:right;"> 41 </td>
   <td style="text-align:right;"> 64 </td>
   <td style="text-align:right;"> 24 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0166194 </td>
   <td style="text-align:right;"> 0.9477098 </td>
   <td style="text-align:right;"> 0.0259424 </td>
   <td style="text-align:right;"> 0.0097284 </td>
   <td style="text-align:right;"> 0.9643291 </td>
   <td style="text-align:right;"> 0.49 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2341 </td>
   <td style="text-align:right;"> 40 </td>
   <td style="text-align:right;"> 65 </td>
   <td style="text-align:right;"> 21 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0162140 </td>
   <td style="text-align:right;"> 0.9489258 </td>
   <td style="text-align:right;"> 0.0263478 </td>
   <td style="text-align:right;"> 0.0085124 </td>
   <td style="text-align:right;"> 0.9651398 </td>
   <td style="text-align:right;"> 0.50 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2342 </td>
   <td style="text-align:right;"> 39 </td>
   <td style="text-align:right;"> 66 </td>
   <td style="text-align:right;"> 20 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0158087 </td>
   <td style="text-align:right;"> 0.9493312 </td>
   <td style="text-align:right;"> 0.0267531 </td>
   <td style="text-align:right;"> 0.0081070 </td>
   <td style="text-align:right;"> 0.9651398 </td>
   <td style="text-align:right;"> 0.51 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2342 </td>
   <td style="text-align:right;"> 39 </td>
   <td style="text-align:right;"> 66 </td>
   <td style="text-align:right;"> 20 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0158087 </td>
   <td style="text-align:right;"> 0.9493312 </td>
   <td style="text-align:right;"> 0.0267531 </td>
   <td style="text-align:right;"> 0.0081070 </td>
   <td style="text-align:right;"> 0.9651398 </td>
   <td style="text-align:right;"> 0.52 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2345 </td>
   <td style="text-align:right;"> 39 </td>
   <td style="text-align:right;"> 66 </td>
   <td style="text-align:right;"> 17 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0158087 </td>
   <td style="text-align:right;"> 0.9505472 </td>
   <td style="text-align:right;"> 0.0267531 </td>
   <td style="text-align:right;"> 0.0068910 </td>
   <td style="text-align:right;"> 0.9663559 </td>
   <td style="text-align:right;"> 0.53 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2345 </td>
   <td style="text-align:right;"> 37 </td>
   <td style="text-align:right;"> 68 </td>
   <td style="text-align:right;"> 17 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0149980 </td>
   <td style="text-align:right;"> 0.9505472 </td>
   <td style="text-align:right;"> 0.0275638 </td>
   <td style="text-align:right;"> 0.0068910 </td>
   <td style="text-align:right;"> 0.9655452 </td>
   <td style="text-align:right;"> 0.54 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2345 </td>
   <td style="text-align:right;"> 36 </td>
   <td style="text-align:right;"> 69 </td>
   <td style="text-align:right;"> 17 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0145926 </td>
   <td style="text-align:right;"> 0.9505472 </td>
   <td style="text-align:right;"> 0.0279692 </td>
   <td style="text-align:right;"> 0.0068910 </td>
   <td style="text-align:right;"> 0.9651398 </td>
   <td style="text-align:right;"> 0.55 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2345 </td>
   <td style="text-align:right;"> 36 </td>
   <td style="text-align:right;"> 69 </td>
   <td style="text-align:right;"> 17 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0145926 </td>
   <td style="text-align:right;"> 0.9505472 </td>
   <td style="text-align:right;"> 0.0279692 </td>
   <td style="text-align:right;"> 0.0068910 </td>
   <td style="text-align:right;"> 0.9651398 </td>
   <td style="text-align:right;"> 0.56 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2347 </td>
   <td style="text-align:right;"> 36 </td>
   <td style="text-align:right;"> 69 </td>
   <td style="text-align:right;"> 15 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0145926 </td>
   <td style="text-align:right;"> 0.9513579 </td>
   <td style="text-align:right;"> 0.0279692 </td>
   <td style="text-align:right;"> 0.0060803 </td>
   <td style="text-align:right;"> 0.9659505 </td>
   <td style="text-align:right;"> 0.57 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2348 </td>
   <td style="text-align:right;"> 34 </td>
   <td style="text-align:right;"> 71 </td>
   <td style="text-align:right;"> 14 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0137819 </td>
   <td style="text-align:right;"> 0.9517633 </td>
   <td style="text-align:right;"> 0.0287799 </td>
   <td style="text-align:right;"> 0.0056749 </td>
   <td style="text-align:right;"> 0.9655452 </td>
   <td style="text-align:right;"> 0.58 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2350 </td>
   <td style="text-align:right;"> 34 </td>
   <td style="text-align:right;"> 71 </td>
   <td style="text-align:right;"> 12 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0137819 </td>
   <td style="text-align:right;"> 0.9525740 </td>
   <td style="text-align:right;"> 0.0287799 </td>
   <td style="text-align:right;"> 0.0048642 </td>
   <td style="text-align:right;"> 0.9663559 </td>
   <td style="text-align:right;"> 0.59 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2351 </td>
   <td style="text-align:right;"> 30 </td>
   <td style="text-align:right;"> 75 </td>
   <td style="text-align:right;"> 11 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0121605 </td>
   <td style="text-align:right;"> 0.9529793 </td>
   <td style="text-align:right;"> 0.0304013 </td>
   <td style="text-align:right;"> 0.0044589 </td>
   <td style="text-align:right;"> 0.9651398 </td>
   <td style="text-align:right;"> 0.60 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2351 </td>
   <td style="text-align:right;"> 29 </td>
   <td style="text-align:right;"> 76 </td>
   <td style="text-align:right;"> 11 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0117552 </td>
   <td style="text-align:right;"> 0.9529793 </td>
   <td style="text-align:right;"> 0.0308066 </td>
   <td style="text-align:right;"> 0.0044589 </td>
   <td style="text-align:right;"> 0.9647345 </td>
   <td style="text-align:right;"> 0.61 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2351 </td>
   <td style="text-align:right;"> 25 </td>
   <td style="text-align:right;"> 80 </td>
   <td style="text-align:right;"> 11 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0101338 </td>
   <td style="text-align:right;"> 0.9529793 </td>
   <td style="text-align:right;"> 0.0324281 </td>
   <td style="text-align:right;"> 0.0044589 </td>
   <td style="text-align:right;"> 0.9631131 </td>
   <td style="text-align:right;"> 0.62 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2351 </td>
   <td style="text-align:right;"> 24 </td>
   <td style="text-align:right;"> 81 </td>
   <td style="text-align:right;"> 11 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0097284 </td>
   <td style="text-align:right;"> 0.9529793 </td>
   <td style="text-align:right;"> 0.0328334 </td>
   <td style="text-align:right;"> 0.0044589 </td>
   <td style="text-align:right;"> 0.9627077 </td>
   <td style="text-align:right;"> 0.63 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2352 </td>
   <td style="text-align:right;"> 22 </td>
   <td style="text-align:right;"> 83 </td>
   <td style="text-align:right;"> 10 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0089177 </td>
   <td style="text-align:right;"> 0.9533847 </td>
   <td style="text-align:right;"> 0.0336441 </td>
   <td style="text-align:right;"> 0.0040535 </td>
   <td style="text-align:right;"> 0.9623024 </td>
   <td style="text-align:right;"> 0.64 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2353 </td>
   <td style="text-align:right;"> 21 </td>
   <td style="text-align:right;"> 84 </td>
   <td style="text-align:right;"> 9 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0085124 </td>
   <td style="text-align:right;"> 0.9537900 </td>
   <td style="text-align:right;"> 0.0340495 </td>
   <td style="text-align:right;"> 0.0036482 </td>
   <td style="text-align:right;"> 0.9623024 </td>
   <td style="text-align:right;"> 0.65 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2356 </td>
   <td style="text-align:right;"> 19 </td>
   <td style="text-align:right;"> 86 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0077017 </td>
   <td style="text-align:right;"> 0.9550061 </td>
   <td style="text-align:right;"> 0.0348602 </td>
   <td style="text-align:right;"> 0.0024321 </td>
   <td style="text-align:right;"> 0.9627077 </td>
   <td style="text-align:right;"> 0.66 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2357 </td>
   <td style="text-align:right;"> 18 </td>
   <td style="text-align:right;"> 87 </td>
   <td style="text-align:right;"> 5 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0072963 </td>
   <td style="text-align:right;"> 0.9554114 </td>
   <td style="text-align:right;"> 0.0352655 </td>
   <td style="text-align:right;"> 0.0020268 </td>
   <td style="text-align:right;"> 0.9627077 </td>
   <td style="text-align:right;"> 0.67 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2358 </td>
   <td style="text-align:right;"> 17 </td>
   <td style="text-align:right;"> 88 </td>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0068910 </td>
   <td style="text-align:right;"> 0.9558168 </td>
   <td style="text-align:right;"> 0.0356709 </td>
   <td style="text-align:right;"> 0.0016214 </td>
   <td style="text-align:right;"> 0.9627077 </td>
   <td style="text-align:right;"> 0.68 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2358 </td>
   <td style="text-align:right;"> 16 </td>
   <td style="text-align:right;"> 89 </td>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0064856 </td>
   <td style="text-align:right;"> 0.9558168 </td>
   <td style="text-align:right;"> 0.0360762 </td>
   <td style="text-align:right;"> 0.0016214 </td>
   <td style="text-align:right;"> 0.9623024 </td>
   <td style="text-align:right;"> 0.69 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2359 </td>
   <td style="text-align:right;"> 14 </td>
   <td style="text-align:right;"> 91 </td>
   <td style="text-align:right;"> 3 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0056749 </td>
   <td style="text-align:right;"> 0.9562221 </td>
   <td style="text-align:right;"> 0.0368869 </td>
   <td style="text-align:right;"> 0.0012161 </td>
   <td style="text-align:right;"> 0.9618970 </td>
   <td style="text-align:right;"> 0.70 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2361 </td>
   <td style="text-align:right;"> 12 </td>
   <td style="text-align:right;"> 93 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0048642 </td>
   <td style="text-align:right;"> 0.9570328 </td>
   <td style="text-align:right;"> 0.0376976 </td>
   <td style="text-align:right;"> 0.0004054 </td>
   <td style="text-align:right;"> 0.9618970 </td>
   <td style="text-align:right;"> 0.71 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2361 </td>
   <td style="text-align:right;"> 10 </td>
   <td style="text-align:right;"> 95 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0040535 </td>
   <td style="text-align:right;"> 0.9570328 </td>
   <td style="text-align:right;"> 0.0385083 </td>
   <td style="text-align:right;"> 0.0004054 </td>
   <td style="text-align:right;"> 0.9610863 </td>
   <td style="text-align:right;"> 0.72 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2361 </td>
   <td style="text-align:right;"> 8 </td>
   <td style="text-align:right;"> 97 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0032428 </td>
   <td style="text-align:right;"> 0.9570328 </td>
   <td style="text-align:right;"> 0.0393190 </td>
   <td style="text-align:right;"> 0.0004054 </td>
   <td style="text-align:right;"> 0.9602756 </td>
   <td style="text-align:right;"> 0.73 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2361 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 99 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0024321 </td>
   <td style="text-align:right;"> 0.9570328 </td>
   <td style="text-align:right;"> 0.0401297 </td>
   <td style="text-align:right;"> 0.0004054 </td>
   <td style="text-align:right;"> 0.9594649 </td>
   <td style="text-align:right;"> 0.74 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2361 </td>
   <td style="text-align:right;"> 6 </td>
   <td style="text-align:right;"> 99 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0024321 </td>
   <td style="text-align:right;"> 0.9570328 </td>
   <td style="text-align:right;"> 0.0401297 </td>
   <td style="text-align:right;"> 0.0004054 </td>
   <td style="text-align:right;"> 0.9594649 </td>
   <td style="text-align:right;"> 0.75 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 5 </td>
   <td style="text-align:right;"> 100 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0020268 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0405351 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9594649 </td>
   <td style="text-align:right;"> 0.76 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 101 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0016214 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0409404 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9590596 </td>
   <td style="text-align:right;"> 0.77 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 101 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0016214 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0409404 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9590596 </td>
   <td style="text-align:right;"> 0.78 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:right;"> 103 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0008107 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0417511 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9582489 </td>
   <td style="text-align:right;"> 0.79 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:right;"> 103 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0008107 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0417511 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9582489 </td>
   <td style="text-align:right;"> 0.80 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:right;"> 103 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0008107 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0417511 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9582489 </td>
   <td style="text-align:right;"> 0.81 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:right;"> 103 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0008107 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0417511 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9582489 </td>
   <td style="text-align:right;"> 0.82 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:right;"> 103 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0008107 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0417511 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9582489 </td>
   <td style="text-align:right;"> 0.83 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 104 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0004054 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0421565 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9578435 </td>
   <td style="text-align:right;"> 0.84 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 104 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0004054 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0421565 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9578435 </td>
   <td style="text-align:right;"> 0.85 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 104 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0004054 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0421565 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9578435 </td>
   <td style="text-align:right;"> 0.86 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 105 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0425618 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.87 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 105 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0425618 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.88 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 105 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0425618 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.89 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 105 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0425618 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.90 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 105 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0425618 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.91 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 105 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0425618 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.92 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 105 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0425618 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.93 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 105 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0425618 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.94 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 105 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0425618 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.95 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 105 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0425618 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.96 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 105 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0425618 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.97 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 105 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0425618 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.98 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2362 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 105 </td>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 2467 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.0425618 </td>
   <td style="text-align:right;"> 0.0000000 </td>
   <td style="text-align:right;"> 0.9574382 </td>
   <td style="text-align:right;"> 0.99 </td>
  </tr>
</tbody>
</table></div>

### Confusion Matrix

There are four possible scenarios that will result from the use of this model: 1. Model predicts inundation and there is inundation (True Positive) 2. Model predicts no inundation and there is no inundation (True Negative) 3. Model predicts inundation and there is no inundation (False Positive) 4. Model predicts no inundation and there is inundation (False Negative)

The results of each scenario at the optimal threshold for accuracy (0.53) is displayed in the table below (with the confusion matrix of 0 and 1 directly below the table). These results support our above observation that the model predicts well for areas of no inundation than areas of inundation.

``` r
testProbs$predClass  = ifelse(testProbs$pred > .53 ,1,0)

xtab.regCalgary <- caret::confusionMatrix(reference = as.factor(testProbs$obs),
                                          data = as.factor(testProbs$predClass),
                                          positive = "1")

as.matrix(xtab.regCalgary) %>% kable(caption = "Confusion Matrix") %>% kable_styling("striped", full_width = T, font_size = 14, position = "left")
```

<table class="table table-striped" style="font-size: 14px; ">
<caption style="font-size: initial !important;">Confusion Matrix</caption>
 <thead>
  <tr>
   <th style="text-align:left;">  </th>
   <th style="text-align:right;"> 0 </th>
   <th style="text-align:right;"> 1 </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> 0 </td>
   <td style="text-align:right;"> 2345 </td>
   <td style="text-align:right;"> 66 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> 1 </td>
   <td style="text-align:right;"> 17 </td>
   <td style="text-align:right;"> 39 </td>
  </tr>
</tbody>
</table>

### Confusion Matrix - Statistics

Further statistics associated with the confusion matrix are displayed in the table below. Of particular note are the sensitivity (rate of true positives) and specificity (rate of true negatives). While the results indicate that the model performs very well for true negatives (a specificity of 98%), it also predicts the true positives fairly well (a sensitivity of ~72%).

Since it is better to be overprepared for a flooding event than underprepared, we concluded that the number of false positive predictions was not a concern, especially given the precarious and unexpected nature of extreme weather events during an era of climate change.

``` r
as.matrix(xtab.regCalgary, what="classes") %>% kable(caption = "Confusion Matrix - Statistics") %>% kable_styling(font_size = 14, full_width = T,
                                                                                                                  bootstrap_options = c("striped", "hover"))
```

<table class="table table-striped table-hover" style="font-size: 14px; margin-left: auto; margin-right: auto;">
<caption style="font-size: initial !important;">Confusion Matrix - Statistics</caption>
 <thead>
  <tr>
   <th style="text-align:left;">  </th>
   <th style="text-align:right;">  </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> Sensitivity </td>
   <td style="text-align:right;"> 0.3714286 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Specificity </td>
   <td style="text-align:right;"> 0.9928027 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Pos Pred Value </td>
   <td style="text-align:right;"> 0.6964286 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Neg Pred Value </td>
   <td style="text-align:right;"> 0.9726255 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Precision </td>
   <td style="text-align:right;"> 0.6964286 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Recall </td>
   <td style="text-align:right;"> 0.3714286 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> F1 </td>
   <td style="text-align:right;"> 0.4844720 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Prevalence </td>
   <td style="text-align:right;"> 0.0425618 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Detection Rate </td>
   <td style="text-align:right;"> 0.0158087 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Detection Prevalence </td>
   <td style="text-align:right;"> 0.0226996 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Balanced Accuracy </td>
   <td style="text-align:right;"> 0.6821156 </td>
  </tr>
</tbody>
</table>

### Map of Predictions for Calgary Testing Set

To spatially investigate the distribution of these four predicted outcomes, the map here shows the four confusion matrix results associated with each fishnet grid cell in the testing set. We elected to only show the predicts for the testing set since the amount of observations was already at a large number and patterns are already discernible.

Note how there are few false positives throughout the city, indicating that there are few places for which the model did not accurately predict inundation. The true positives are concentrated around the area where we saw major hydrological features and the stream network.

``` r
test_predictions <- testProbs %>%
  mutate(TN = predClass==0 & obs==0,
         TP = predClass==1 & obs==1,
         FN =  predClass==0 & obs==1,
         FP = predClass==1 & obs==0)


test_predictions <- test_predictions %>%
  mutate(confResult=case_when(TN == TRUE ~ "True_Negative",
                              TP == TRUE ~ "True_Positive",
                              FN == TRUE ~ "False_Negative",
                              FP == TRUE ~ "False_Positive"))


#join with geometry for the map of prediction classes
cal_test_predictions_mapdata <- cbind(CalgaryTest, test_predictions, by= "ID_FISHNET") %>% st_as_sf()


ggplot() +
  geom_sf(data=cal_test_predictions_mapdata, aes(fill=confResult), colour=NA)+
  scale_fill_discrete()+
  mapTheme() +
  labs(title="Map of Prediction Classes on Calgary Test")
```

![](Calgary-Flood-Inundation_files/figure-html/matrixstat-1.png)<!-- -->

### ROC Curve

The Receiver Operating Characteristic (ROC) Curve for the model is shown below. This curve is a helpful goodness of fit indicator, while helping to visualize trade-offs between true positive and false positive metrics at each threshold from 0.01 to 1. A line going “over” the curve indicates a useful fit. The area under the curve (AUC) here is about 0.96, indicating a useful fit. A reasonable AUC is between 0.5 and 1.

``` r
#ROC Curve
ggplot(testProbs, aes(d = obs, m = pred)) +
  geom_roc(n.cuts = 50, labels = FALSE) +
  style_roc(theme = theme_grey) +
  geom_abline(slope = 1, intercept = 0, size = 1.5, color = 'grey') + labs(title="ROC Curve", subtitle="AUC ~ 0.96") + plotTheme()
```

![](Calgary-Flood-Inundation_files/figure-html/ROC Curve-1.png)<!-- -->

``` r
auc_calgary <- auc(testProbs$obs, testProbs$pred)
print(auc_calgary)
```

### Cross-Validation

While the ROC indicates a relatively good fit, cross-validation is key to ensure that the model generalizes well across contexts. A k-fold cross validation (CV) is performed here.

``` r
#convert inundation to factor
Calgary$Inundation <-as.factor(as.character(Calgary$Inundation))

ctrl <- trainControl(method = "cv",
                     number = 100,
                     savePredictions = TRUE)

cvFit_InundationModel <- train(as.factor(Inundation) ~ ., data = Calgary %>%
                                 as.data.frame() %>%
                                 select(-geometry, -basin, -dist_deve, dist_fores),
                               method="glm", family="binomial", trControl=ctrl)

cvFit_InundationModel
```

```
## Generalized Linear Model 
## 
## 8225 samples
##   12 predictor
##    2 classes: '0', '1' 
## 
## No pre-processing
## Resampling: Cross-Validated (100 fold) 
## Summary of sample sizes: 8142, 8142, 8143, 8143, 8143, 8142, ... 
## Resampling results:
## 
##   Accuracy   Kappa    
##   0.9679263  0.4840761
```

### Accuracy and Kappa

To evaluate the algorithm further, the accuracy an kappa metrics shed light on goodness of fit. Accuracy is the percentage of instances classified correctly out of the total. While this does not provide the nuance provided in the confusion matrix (showing the breakdown of accuracy across the four classes), it is useful in defending the model’s success at a high level.

Kappa (also known as Cohen’s Kappa) is normalized accuracy. This is useful here because we have an imbalance in the 0s and 1s for no inundation and inundation, respectively. While the imbalance is expected given that the previous flood extent showed that the majority of Calgary was not inundated, Kappa is useful in its comparison of observed accuracy to expected accuracy, taking into account random chance. It is still a relatively high value with a mean of over 50%. According to best practices in statistics, a kappa value of 0.5 falls within the range of “substantial agreement.”

``` r
dplyr::select(cvFit_InundationModel$resample, -Resample) %>%
  gather(metric, value) %>%
  left_join(gather(cvFit_InundationModel$results[2:4], metric, mean)) %>%
  ggplot(aes(value)) +
  geom_histogram(bins=35, fill = "navy") +
  facet_wrap(~metric) +
  geom_vline(aes(xintercept = mean), colour = "red", linetype = 3, size = 1.5) +
  scale_x_continuous(limits = c(0, 1)) +
  labs(x="Goodness of Fit", y="Count", title="Accuracy and Kappa",
       subtitle = "Across-fold mean represented as dotted lines") +plotTheme()
```

![](Calgary-Flood-Inundation_files/figure-html/Accuracy-1.png)<!-- -->

### Mapping Calgary Predictions

The final predictions for inundation across Calgary are mapped for each fishnet grid cell. We elected to display the predictions on a probabilities scale to achieve a more continuous surface.

``` r
allPredictions <-
  predict(cvFit_InundationModel, Calgary, type="prob")[,2]

calgary_fishnet_final_preds <- Calgary %>%
  cbind(Calgary,allPredictions) %>%
  mutate(allPredictions = round(allPredictions * 100)) 

ggplot() +
  geom_sf(data=calgary_fishnet_final_preds, aes(fill=(allPredictions)), colour=NA) +
  scale_fill_viridis(name = "Probability")+
  mapTheme() +
  labs(title="Predictions for Inundation in Calgary")
```

![](Calgary-Flood-Inundation_files/figure-html/Mapping-1.png)<!-- -->
This second map shows the predicted probabilities, but with the 2013 flood extent overlay. This helps to compare the predicted inundation outcomes with the actual data used to train the model in the first place.

``` r
ggplot() +
  geom_sf(data=calgary_fishnet_final_preds, aes(fill=(allPredictions)), colour=NA) +
  scale_fill_viridis(name = "Probability")+
  geom_sf(data = calgary_fishnet_final_preds %>%
            filter(Inundation=="1"),
          aes(), color="transparent", fill="red", alpha=0.5)+
  mapTheme() +
  labs(title="Predictions for Inundation in Calgary", subtitle = "2013 flood extent in red overlay")
```

![](Calgary-Flood-Inundation_files/figure-html/Mapping2013-1.png)<!-- -->

# Conclusion 

The results of this model are crucial to informing planners and allowing a city to be prepared for flood disasters. Successfully leveraging the knowledge and insight gained can save lives, cities, homes, and money. There are many implementation challenges when it comes to appropriately executing emergency preparedness programs such as funding, political and community support, and collaboration across departments. However, we are confident that the story being told through these data visualizations can be an effective tool in convincing relevant decision-makers
