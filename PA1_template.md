# Reproducible Research: Peer Assessment 1


## Loading and preprocessing the data


```r
activityData <- read.csv(unz("activity.zip", "activity.csv"), stringsAsFactors=FALSE)
activityData$date <- strptime(activityData$date, format = "%Y-%m-%d")
```


## What is mean total number of steps taken per day?



## What is the average daily activity pattern?



## Imputing missing values



## Are there differences in activity patterns between weekdays and weekends?
