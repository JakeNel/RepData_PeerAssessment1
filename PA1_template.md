# Reproducible Research: Peer Assessment 1


## Loading and preprocessing the data

First, I loaded the data in using read.csv().  The code assumes that the folder containing the dataset is your working directory, and is called "activity.zip".  Because it is a zip file, the unzip() function is used to open a connection inside the zip file and extract the csv file.  


```r
amd <- read.csv(unzip("activity.zip", "activity.csv"))
```
Here is an overview of what the data looks like:


```r
head(amd)
```

```
##   steps       date interval
## 1    NA 2012-10-01        0
## 2    NA 2012-10-01        5
## 3    NA 2012-10-01       10
## 4    NA 2012-10-01       15
## 5    NA 2012-10-01       20
## 6    NA 2012-10-01       25
```

```r
str(amd)
```

```
## 'data.frame':	17568 obs. of  3 variables:
##  $ steps   : int  NA NA NA NA NA NA NA NA NA NA ...
##  $ date    : Factor w/ 61 levels "2012-10-01","2012-10-02",..: 1 1 1 1 1 1 1 1 1 1 ...
##  $ interval: int  0 5 10 15 20 25 30 35 40 45 ...
```

The date variable then needs to be converted into a date format, using the as.Date() function.    

```r
amd$date <- as.Date(amd$date, "%Y-%m-%d")

##Verify that reformatting was successful

class(amd$date)
```

```
## [1] "Date"
```

Lastly, group the data by date and time interval, using the group_by() function in the dplyr package. Each type of grouping will be used later.  


```r
library(dplyr)
grp_by_date <- group_by(amd, date)
grp_by_interval <- group_by(amd, interval)
```

## What is mean total number of steps taken per day?

Using the summarize function from dplyr, a new dataframe is created which gives the total number of steps on each date in the sample.   


```r
totalSteps <- summarize(grp_by_date, sum=sum(steps))
head(totalSteps)
```

```
## Source: local data frame [6 x 2]
## 
##         date   sum
##       (date) (int)
## 1 2012-10-01    NA
## 2 2012-10-02   126
## 3 2012-10-03 11352
## 4 2012-10-04 12116
## 5 2012-10-05 13294
## 6 2012-10-06 15420
```

This histogram shows the distribution of total number of steps that we just calculated:


```r
library(ggplot2)
ggplot() + aes(na.omit(totalSteps$sum)) + geom_histogram(binwidth = 1000, color = "black", fill = "blue") + labs(title = "Distribution of the Total Number of Steps ", x = "Total number of Steps")
```

![](PA1_template_files/figure-html/unnamed-chunk-6-1.png) 

The mean and median of the total steps taken per day can calculated fom the same dataset:


```r
mn <- mean(totalSteps$sum, na.rm = TRUE)
md <- median(totalSteps$sum, na.rm = TRUE)
mn
```

```
## [1] 10766.19
```

```r
md
```

```
## [1] 10765
```


## What is the average daily activity pattern?

Again, using the summarize() function in dplyr, a new dataframe is created which gives the average number of steps recorded for each time interval:


```r
averageSteps <- summarize(grp_by_interval, mean = mean(steps, na.rm = TRUE))
```

Here is the time series plot by time interval, which represents the data we just calculated:


```r
  ggplot() + aes(averageSteps$interval, averageSteps$mean) + geom_line(size = 1.5, color = "purple") + labs(title = "Time Series of Average Steps by 5-minute Interval", x = "Interval", y = "Average # of Steps")
```

![](PA1_template_files/figure-html/unnamed-chunk-9-1.png) 

The interval with the most number of steps is given in the following code:


```r
as.numeric(averageSteps[which.max(averageSteps$mean), 1])
```

```
## [1] 835
```


## Imputing missing values

The original dataset has many missing values, which can potentially bias our analysis in unknown ways.  The number of missing values for number of steps is calculated in the following code:


```r
sum(is.na(amd$steps))
```

```
## [1] 2304
```

My strategy for dealing with missing values is to replace the NA value with the average number of steps for each time interval that was taken from the rest of the dataset:


```r
amd_modified <- amd
for(i in 1:nrow(amd)) {
    if(is.na(amd[i, "steps"])) {
        interval <- amd[i, "interval"]
        newValue <- averageSteps[which(averageSteps$interval == interval), "mean"]
        amd_modified[i, "steps"] <- newValue 
    }
}

## New grouped data
mod_group_date <- group_by(amd_modified, date)
```

With this new dataset, we can simply repeat the histogram for total steps that we made before:


```r
totalSteps_mod <- summarize(mod_group_date, sum=sum(steps))
ggplot() + aes(na.omit(totalSteps_mod$sum)) + geom_histogram(binwidth = 1000, color = "black", fill = "blue") + labs(title = "Distribution of the Total Number of Steps with Imputed Values ", x = "Total number of Steps")
```

![](PA1_template_files/figure-html/unnamed-chunk-13-1.png) 

The new mean and median are given in the following:  

```r
mn <- mean(totalSteps_mod$sum, na.rm = TRUE)
md <- median(totalSteps_mod$sum, na.rm = TRUE)
mn
```

```
## [1] 10766.19
```

```r
md
```

```
## [1] 10766.19
```

We see that under the new imputed values...the mean and median don't change much, but as we might expect, the distribution is much more centrally concentrated.  

## Are there differences in activity patterns between weekdays and weekends?

We can select out the weekdays and weekends with the weekdays() function, grouping it by interval.  


```r
Weekday.dat <- amd_modified [weekdays(totalSteps_mod$date) %in% c("Monday", "Tuesday", "Wednesday", "Thursday", "Friday"),]

Weekend.dat <- amd_modified [weekdays(totalSteps_mod$date) %in% c("Saturday", "Sunday"),]

## Group by interval

Weekday.dat <- group_by(Weekday.dat, interval)
Weekend.dat <- group_by(Weekend.dat, interval)

## Create Average Steps datasets

AverageWeekdaySteps <- summarize(Weekday.dat, mean = mean(steps))
AverageWeekendSteps <- summarize(Weekend.dat, mean = mean(steps))

## Combine datasets into one, with a new factored variable indicating the day, weekday or weekend.  

Average_byDay <- rbind(AverageWeekdaySteps, AverageWeekendSteps)
Day <- as.factor(c(rep("Weekday", nrow(AverageWeekdaySteps)), rep("Weekend", nrow(AverageWeekendSteps))))
Average_byDay <- cbind(Average_byDay, Day)
```

Then, I perform the same time series as before, but this time with two lines, one for weekdays, and one for weekends.  The README file example suggests putting the two time series graphs on the same grid with two panels. In ggplot, this is done with the facet_grid() function.


```r
ggplot(Average_byDay, aes(interval, mean)) + geom_line(size = 1.2, color = "brown") + facet_grid(Day~.) + labs(x= "interval", y = "Average # of Steps", title= "Weekday vs Weekend Steps Taken per 5 Minute Interval")
```

![](PA1_template_files/figure-html/unnamed-chunk-16-1.png) 
