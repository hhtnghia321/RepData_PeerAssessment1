---
title: 'Reproducible Research: Peer Assessment 1'
output:
  html_document:
    keep_md: yes
  pdf_document: default
  word_document: default
header-includes:
- \usepackage{color}
- \usepackage{xcolor}
- \usepackage{framed}
---
## Setting global options and loading packages

```r
library(knitr)
library(dplyr)
library(ggplot2)
library(lubridate)
library(data.table)
library(VIM)
library(scales)
opts_chunk$set(cache = FALSE, message = FALSE)
```
* Turn off *cache* and *message* in global option   
* Load package:  
  + kinitr: for translating Markdown format  
  + dplyr: for cleaning data  
  + ggplot2: for plotting my chart  
  + libridate: for reformating datedata format  
  + data.table: for reformating datedata format   
  + impute: for imputing missing value   
  + sacles: for scaling the Time format in the plot for better intepretation
  

## Loading and preprocessing the data

```r
data <- read.csv("activity.csv", header = TRUE, )
```


## What is mean total number of steps taken per day?
for this part of the analysis, I temporary ignore the NA value of the step column


This is how I calculate the total steps each day

```r
group.Total1 <- group_by(data, date) %>% 
  summarise(Totalstep = sum(steps, na.rm = TRUE))
```

> * The Mean of the Total step is 9354.2295082.
  * The Median of the Total step is 10395. 

Ploting the total each day by histogram to see the distribution of steps in 2 months


```r
group.Total1$date <- as.Date(group.Total1$date)
ggplot(data = group.Total1, aes(x= Totalstep)) + 
  geom_histogram() +
  labs(title = "Histogram of Total steps each days form Oct to Nov", y = "Days") +
  theme(plot.title = element_text(hjust=0.5))
```

![](PA1_template_files/figure-html/histogram-1.png)<!-- -->

> * We can see that there are more than 10 days of missing value 
  * the density is discrete in some value
  * It can easily infer that the Total step each date follow the *Normal Distribution*

## What is the average daily activity pattern?

Before ploting time series line chart by 5-minute of step, I have to add one more columns of Time data with the interval of increment of 5-minutes. Then I add its to the original dataset. I also add one more interval columns as the Time format (HH:MM:SS)

```r
Time <- seq.POSIXt(from = as.POSIXct(ymd_hms("2012-10-01 00:00:01")),to = as.POSIXct(ymd_hms("2012-11-30 23:55:01")), by = "5 min")
data$Time <- Time
data$intervaltime <- as.ITime(data$Time)
```

calculating the average numbers of step by each interval  

```r
groupave <- group_by(data, intervaltime) %>% summarise(Averagestep = mean(steps, na.rm = TRUE))
```

Ploting the time series based on the 5-minute time series

```r
ggplot(groupave, aes(x =as.POSIXct(groupave$intervaltime, format = "%H:%M:%S"), y = Averagestep)) +
  geom_line() +
  scale_x_datetime(breaks = date_breaks("5 hours"), labels = date_format("%H:%M")) +
  labs(x = "Times of a day", title = "The Allocation Of Average Step During a Day") +
  theme(plot.title = element_text(hjust = 0.5))
```

![](PA1_template_files/figure-html/unnamed-chunk-5-1.png)<!-- -->

Time of a day which have maximum average step 

```r
subset(groupave, groupave$Averagestep == max(groupave$Averagestep))
```

```
## # A tibble: 1 x 2
##   intervaltime Averagestep
##   <ITime>            <dbl>
## 1 08:35:01            206.
```

> * It can be seen that the walking activity often occure at late morning and early noon
  * More specifically, the largest amount of step occured at **08:35:01**

## Imputing missing values

There are numbers of missing value in this data. We had ignore those missing value from the beginning of the analysis. Now, take a look at the number of those missing value and find the way to improve this

Counting the totla number of missing value:

```r
sum(is.na(data$steps))
```

```
## [1] 2304
```
>There are **2304** missing value in this data set which accounting for 
0.1311475 of the total data

I using the impute package of bioconduct to impute the missing value by the ` kNN()` function. The strategy of imputing missing value is followed by the system of ` kNN()` function. It is that the function will try to define the nearest K-centroid of the existed value. The method is used to find the K-centroid is Euclidean. So, a K was stick with a defined numbers of existed value. The missing will be filled with the average of non-missing values in that K. 


Now, We will impute the missing value. I have to specify the dataset needed to be imputed and the variables ( within that dataset ), then specify the number of neighbor K. In this case, I specify the K to be 500 because there more than 288 missing value in the first day.


```r
imputedata <- kNN(data = data, variable = "steps", k= 400)
```

Then I will try to histogram this dataset (with imputed data) to seen if there is any difference in the Total number of steps each day, and report its mean and median


```r
group.Total2 <- group_by(imputedata, date) %>% summarise(Totalstep = sum(steps))
ggplot(group.Total2, aes(x = Totalstep)) + 
  geom_histogram() +
  labs(title = "Histogram of Total steps each days form Oct to Nov with imputed data", y = "Days") +
  theme(plot.title = element_text(hjust=0.5))
```

![](PA1_template_files/figure-html/unnamed-chunk-9-1.png)<!-- -->

We have try to summarise the newly generated value for replacing the missing value to see what is difference in those new value


```r
summary(imputedata$steps[is.na(data$steps)] )
```

```
##    Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
##   0.000   0.000   0.000   3.441   0.000  37.000
```

> * As can We see there not much diffence between this plot and the plot of the first part. The most obsvious difference is that the number of Days with 0 step is significantly reduced form 10 to under 2 days and there are some slightly change in other numbers of Total steps which often around the median. Also, the newly generated value are near 0 than older value because the maximum value of the newly imputed value is 37.
  * The Mean of the Total step is 9484.1967213.
  * The Median of the Total step is 10395. 


## Are there differences in activity patterns between weekdays and weekends?

Answer this question, We create a new factor column which distincguish weekdays and weekend. We also using that data with imputed missing value for this question


```r
#convert into date/POSIXct class
imputedata$date <- as.Date(imputedata$date)

#classify which day is weekdays and weekend
weekdays <- weekdays(imputedata$date, abbreviate = TRUE)
for ( i in 1: length(weekdays)) {
  if (weekdays[i] == "Fri"|weekdays[i] ==  "Mon" |
      weekdays[i] ==  "Thu" | weekdays[i] == "Tue" | 
      weekdays[i] == "Wed" ) {
    weekdays[i] <- "weekdays"
  } else {
    weekdays[i] <- "weeekend"
  }
}

#convert the object into factor
weekdays <- factor(weekdays)

#create a new column in the dataset with imputed value
imputedata$weekdays <-  weekdays
```

Calculating the average numbers of step by each interval by weekday and weekend  

```r
groupweekday <- group_by(imputedata, weekdays , intervaltime) %>% summarise(Averagestep = mean(steps, na.rm = TRUE))
```

Ploting the time series based on the 5-minute time series

```r
ggplot(groupweekday, aes(x =as.POSIXct(groupweekday$intervaltime, format = "%H:%M:%S"), y = Averagestep)) +
  facet_grid(weekdays ~.) +
  geom_line() +
  scale_x_datetime(breaks = date_breaks("5 hours"), labels = date_format("%H:%M")) +
  labs(x = "Times of a day", title = "The Allocation Of Average Step During a Day of weekend and weekdays") +
  theme(plot.title = element_text(hjust = 0.5))
```

![](PA1_template_files/figure-html/unnamed-chunk-13-1.png)<!-- -->

> As, can be seen there are a little bit difference of averager step between weekdays and weekend. The averagesteps peak could reach above 200 in the range of 8AM to 10AM in weekdays, while this only reach 150 in weekend. However, in other time of the day in weekend, the average step took more and more stable from 12AM to 18PM compared to weekdys

