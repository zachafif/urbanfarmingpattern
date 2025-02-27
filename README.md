# Urban Farming Spatial Pattern
Portfolio project about spatial autocorellation of urban farming location in Jakarta

## Introduction

_“The first law of geography: Everything is related to everything else, but near things are more related than distant things.” Waldo R. Tobler (Tobler 1970)_

### Spatial Autocorrelation
Tobler stated that the first law of geography, observation of certain phenomenon has similarity within neighbours. With closer observation tend to be more related than farther observation. Naturally, human visual can easily determine spatial pattern, however the distinction between pattern sometimes are not so obvious.Therefore, spatial pattern can be determined by using spatial autocorrelation analysis, a spatial analysis that can help to quantitatively determine whether observations are spatially correlated with one another. The conclusion from this analysis can reveals the pattern of the observation such as clustered, dispersed, or random. 

### Urban Agriculture
Urban agriculture has gained increasing attention in recent years as a strategy for addressing food security, climate change, and sustainable development challenges in urban areas such as Jakarta. As there are limited spaces to do conventional farming and increasing demands for food production in Indonesia due to it constant increase of population in Jakarta, urban agriculture can serve as an alternative solution for this problem.

Efforts to encourage urban agriculture has been made by Jakarta’s government and urban farms has been emerging in Jakarta for the past years. In this post, we will analyze on the spatial pattern of urban farms and discuss the result further.


## Data

Data used in this analysis is sourced from the national geospatial agency (BIG). It consists of geospatial features containing the administrative boundaries of districts in Jakarta in a polygon format. This data will be accessed via ArcGIS rest service. 
```
https://geoservices.big.go.id/rbi/rest/services
```

Another data is urban farming spot census data provided by JakartaSatu portal containing a CSV file along with the address of the urban farming spots.
  ```
 https://satudata.jakarta.go.id/backend/restapi/v1/export-dataset/filedata_org13_74937d9685038c3cad791243544f5498?secret_key=sdibKbc83EIJG4dRy7Bgi520MWkE7rxwQZgsKJjyJNJXmLq1alaFLWwYtfExADae
  ```


## Method

One of the most popular methods to determine spatial autocorrelation is by calculating the Moran’s I coefficient. The Moran’s I statistic is the correlation coefficient for the relationship between a variable and its neighboring values. Moran's I statistic has a unique difference in compare to other spatial autocorellation method as it incorporate both spatial location and attribute values in its analysis.

The formula are as follows,

![image](https://github.com/user-attachments/assets/f0583e13-b823-4c04-947b-ea0d8158cbaf)

Moran's I coefficient ranges from -1 to 1 with positive value means that the data is clustered while negative value means that the data is dispersed. Otherwise, a near zero result can be intepreted as a random distribution. Furthermore, the calculated result needs to undergo a significance testing (inferential statistics) to determine whether the value of Moran's I is statistically significant to randomness. 

![image](https://github.com/user-attachments/assets/a90dca3b-43df-4107-91f2-11cc56763f89)

## Analysis

### Library Import
The analysis was done using R with the following packages,

  ```
library(sf)
library(httr)
library(tmap)
library(sp)
library(dplyr)
library(tidyr)
library(spdep)
  ```

### Data Import

Data import for district boundary can be done directly from the ArcGIS REST Services link,
```
url <- parse_url("https://geoservices.big.go.id/rbi/rest/services")
url$path <- paste(url$path, "BATASWILAYAH/Administrasi_AR_Kecamatan_10K/MapServer/0/query", sep = "/")
url$query <- list(where = "WADMPR='DKI Jakarta'",
                  outFields = "*",
                  returnGeometry = "true",
                  f = "geojson")

request <- build_url(url)
```

Next, we can import the boundary data as a polygon,

```
jakarta<-st_read(request)
```

The urban farming spot data imported as a CSV, 

```
dat_urbfarm<-read.delim(".../data Urban Farming.csv", sep=",")
```

### Data Checking

The spot data then check using summary() function to get an overview of the columns and data types,

```
> summary(dat_urbfarm)
  periode_data    wilayah           kecamatan          kelurahan            alamat         
 Min.   :2023   Length:286         Length:286         Length:286         Length:286        
 1st Qu.:2023   Class :character   Class :character   Class :character   Class :character  
 Median :2023   Mode  :character   Mode  :character   Mode  :character   Mode  :character  
 Mean   :2023                                                                              
 3rd Qu.:2023                                                                              
 Max.   :2023
```

Then, we check whether the data has a NaN (no data) values,

```
> dat_urbfarm[rowSums(is.na(dat_urbfarm))==1,]
[1] periode_data wilayah      kecamatan    kelurahan    alamat      
<0 rows> (or 0-length row.names)
```

Clearly there is none so we can proceed to the data processing.

### Data Processing

In the data processing we want to analyze the data in district (Kecamatan) level. The spot will be counted per district,

```
data<-dat_urbfarm %>%count(kecamatan)
```

Next we want to join the urban farming count data with boundary data, but first we must rename the column,

```
colnames(data)[1]<-"WADMKC"
colnames(data)[2]<-"n"
```
Then, we can join both data by attribute "WADMKC" or simply the district name,

```
join<-left_join(jakarta,data,by="WADMKC")
```

Same with the step before, we check the data for NaN values and replace it with 0 (zero) values,

```
join<-join %>% replace(is.na(.), 0)
```

The joined data can be visualized with this code,

```
tm_shape(join) + tm_fill(col="n", style="jenks", n=4, palette="YlOrBr") +
  tm_legend(outside=TRUE)
```
View of Jakarta (land)
![Rplot](https://github.com/user-attachments/assets/3823b702-3aec-43b3-9e7c-a22f7f6bd373)

View of Jakarta (Kep.Seribu Isles)
![Rplot01](https://github.com/user-attachments/assets/45c73891-ce00-456a-a9b8-1fb929512f3d)

View per Cities
![image](https://github.com/user-attachments/assets/d1a556b4-ba94-4026-8ba4-423b7bdbd5df)


### Global Moran I Coefficient determination

#### Defining Neighbour

First step of determining Moran I coefficient is to define the neighbour spatially. For this project, we will define the neighbour as neighbouring polygons that at least share the same vertex,

```
neighbour <- poly2nb(join, queen=TRUE)
```

#### Assiging Weight of neighbouring polygons

Each neighboring polygon will be multiplied by the weight equal to 1/(num of neighbours)
```
weight <- nb2listw(neighbour, style="W", zero.policy=TRUE)
```
#### Calculate Moran I coefficient and significance test

There are two approach that can be done to assess Moran I coefficient, analytical approach and monte carlo method.

First, the analytical approach can be done by doing a significance test using the moran.test function,

```
> moran.test(join$n,weight, alternative="greater")

	Moran I test under randomisation

data:  join$n  
weights: weight  
n reduced by no-neighbour observations  

Moran I statistic standard deviate = 1.4771, p-value = 0.06983
alternative hypothesis: greater
sample estimates:
Moran I statistic       Expectation          Variance 
       0.11263217       -0.02439024        0.00860548 
```

The result is 0.113! A positive result that might suggest the data has a clustered pattern.

However, we need additional tests to determine whether the result is significantly difference from zero, thus we can say that it is not random (reject null hypothesis). Note that we use one-tailed hypothethic test by assign "greater" as alternative (means we hypothesize that the data is clustered as an alternative). The result is that we got a p-value of 0.0683, meaning that we have a 6% chance that the null hypothesis is correct (the data has random pattern).

Second, another approach to assess the Moran I coefficient we can do a Monte Carlo simulation by randomly permute the count values across all polygons, then we compute a Moran’s I coefficient for each permuted set of values.

```
> moran.mc(join$n, weight, nsim=999, alternative="greater")

	Monte-Carlo simulation of Moran I

data:  join$n 
weights: weight 
number of simulations + 1: 1000 

statistic = 0.11263, observed rank = 924, p-value = 0.076
alternative hypothesis: greater
```

Then we can plot the histogram of the permutated Moran I value,
![image](https://github.com/user-attachments/assets/38b0f40c-aa41-49cb-b663-29eaf532c76e)

From the monte carlo simulations, we can conclude that our observed Moran’s I value of 0.113 is not a value we would expect to compute if the income values were randomly distributed across each district. As we got a p-value of 0.058 from this simulation, meaning there is a 6% chance of error if we reject the null hypothesis (declare the pattern is random).

### Discussion

From both tests, we got p-value of 0.0683 from analytical test and p-value of 0.076 from Monte Carlo test. Thus, _there is less than a 6% likelihood that the observed pattern could be the result of random chance_, hence with a standard significance level of 0.05 (confidence level 95%), we can conclude that there is not enough statistical evidence to reject the null hypothesis (the data pattern being random).

In conclusion, while there is a visual clustered-like pattern and positive Moran's I coefficient, the quantitative calculation told us that the pattern of urban farming location in Jakarta is random. However, the significance test result had a very close call from being statistically significant, therefore further tests such as Local Moran's I are needed.

## Reference

https://mgimond.github.io/Spatial/spatial-autocorrelation.html

https://mgimond.github.io/es214_support_tutorials/moranI/Mapping_and_Morans.html

https://pro.arcgis.com/en/pro-app/latest/tool-reference/spatial-statistics/spatial-autocorrelation.htm

Lembo, A. J., Jr., & McGrew, J. C., Jr. (2023). *An introduction to statistical problem solving in geography* (4th ed.). Waveland Press.
