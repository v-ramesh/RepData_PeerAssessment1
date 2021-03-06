# Reproducible Research: Peer Assessment 1


## Loading and preprocessing the data


```r
activityData <- read.csv(unz("activity.zip", "activity.csv"), colClasses = c("integer", "Date", "integer"))
```
By using the `colClasses` argument, values in the `date` column (second column) are automatically processed to be in a suitable date format.

## What is mean total number of steps taken per day?

### Calculation of total number of steps taken per day
The default value of the argument `na.action` in the `aggregate` function ignores all rows having missing (`NA`) values as required in this part of the assignment.

```r
stepsByDay <- aggregate(steps ~ date, activityData, sum)
```

### Histogram of total number of steps taken per day

```r
library(ggplot2)
h <- ggplot(stepsByDay, aes(x = steps)) + geom_histogram(binwidth = 2500, aes(fill = ..count..))
h + labs(x = "Daily Total Steps", y = "Number of Days", fill = "Number of Days")
```

![](PA1_template_files/figure-html/unnamed-chunk-3-1.png) 

### Mean and Median number of steps taken each day

```r
with(stepsByDay, data.frame(Mean = mean(steps), Median = median(steps), row.names = "Daily Total Steps"))
```

```
##                       Mean Median
## Daily Total Steps 10766.19  10765
```

## What is the average daily activity pattern?

### Time Series Plot of average number of steps vs 5-minute intervals

```r
meanStepsByInterval <- aggregate(steps ~ interval, activityData, mean)
p <- qplot(interval, steps, data=meanStepsByInterval, geom = "line")
p + labs(x = "Time of Day", y = "Average Number of Steps (5-min Interval)")
```

![](PA1_template_files/figure-html/unnamed-chunk-5-1.png) 

### Interval with maximum average number of steps

```r
with(meanStepsByInterval, interval[which.max(steps)])
```

```
## [1] 835
```
The values in the **interval** column seem to have been obtained by concatenating the hour:minute representation of time of day into a single number. For example, the interval number 1530 corresponds to the time 15:30 or in other words 3:30 PM. From the above calculation, we therefore see that, on average, the maximum number of steps is taken during the 5-minute interval at **8:35 AM**. 

## Imputing missing values

### Total number of missing values
We calculate the total number of missing values in each column of the dataset.

```r
colSums(is.na(activityData))
```

```
##    steps     date interval 
##     2304        0        0
```
We see that the missing values are only in the **steps** column and
that there are **2304** such rows.

### Imputation strategy
From the previous section, all missing values appear in rows of the form `NA d i`, where `d` is the date and `i` the interval,  and we need to devise a strategy for filling in the number of steps for such rows. We first examine the distribution of missing values in the dataset.

```r
datesMissingSteps <- with(activityData, unique(date[is.na(steps)]))
datesMissingSteps
```

```
## [1] "2012-10-01" "2012-10-08" "2012-11-01" "2012-11-04" "2012-11-09"
## [6] "2012-11-10" "2012-11-14" "2012-11-30"
```

```r
datesHavingSteps <- with(activityData, unique(date[!is.na(steps)]))
datesMissingSteps %in% datesHavingSteps
```

```
## [1] FALSE FALSE FALSE FALSE FALSE FALSE FALSE FALSE
```
The above calculations show that there are 8 dates with missing step values, and none of these dates appear in a row having a (non-missing) step value, *i.e.*, for these 8 dates **all** intervals are missing values for the step count. For a row `NA d i`, there are no step values for that day (having date `d`) and we therefore cannot use the the mean/median step count for that day as our imputation strategy. The two most straightforward possibilities we are left with for using as the step count for such a row are the following:

1. The mean/median total number of steps per day divided by the number of intervals in a day
2. The mean number of steps for the interval `i` (averaged across all days)

Option 1. ensures that the total number of steps for the days with missing values is the same as the mean/median value in the dataset. It also has the advantage of being easier to code in R since all the missing values need to be replaced with the same constant value. However, it would be insensitive to the variation of activities across intervals which, from the time series plot of the previous section, can be observed to be significant. We therefore use Option 2. as our imputation strategy which would take into account the particular interval for which the missing value is being filled. 

### Creation of a dataset with missing data filled in
The average number of steps for each interval were previously calculated and stored in the dataframe `meanStepsByInterval` - we make use of this dataframe in implementing our imputation strategy. 

```r
# rename column named "steps"
names(meanStepsByInterval)[2] <- "meanSteps"
# after renaming, merging will be done on the only common column "interval"
imputedData <- merge(activityData, meanStepsByInterval)
# use "meanSteps" column in each row of merged set to fill in missing value
imputedData$steps[is.na(imputedData$steps)] <- imputedData$meanSteps[is.na(imputedData$steps)]
```

### Updated Histogram of total number of steps taken per day


```r
stepsByDayImputed <- aggregate(steps ~ date, imputedData, sum)
h2 <- ggplot(stepsByDayImputed, aes(x = steps)) + geom_histogram(binwidth = 2500, aes(fill = ..count..))
h2 + labs(x = "Daily Total Steps", y = "Number of Days", fill = "Number of Days")
```

![](PA1_template_files/figure-html/unnamed-chunk-10-1.png) 

Compared to our original histogram (before imputation), the only change is that the frequency count of the central bin of 10000-12500 has increased from 18 to 26 with all other bins remaining exactly the same. This increase corresponds directly with the 8 days with missing values - using the average number of steps for each interval resulted in a total number for the day falling into this central bin. 


```r
rbind(with(stepsByDay, data.frame(Mean = mean(steps), Median = median(steps),
                                  row.names = "Daily Total Steps (Original)")),
      with(stepsByDayImputed, data.frame(Mean = mean(steps), Median = median(steps), 
                                         row.names = "Daily Total Steps (Imputed)")))
```

```
##                                  Mean   Median
## Daily Total Steps (Original) 10766.19 10765.00
## Daily Total Steps (Imputed)  10766.19 10766.19
```
Thus, the mean total steps per day has been unaffected by our imputation strategy and the median has shifted slightly to coincide exactly with the mean. We can therefore infer that filling intervals with their average number of steps resulted in a total number for the day that was exactly the same as the mean of the total number of steps per day. 

## Are there differences in activity patterns between weekdays and weekends?


```r
# create a new column for classifying the type of day
imputedData$dayType <- factor(ifelse(weekdays(imputedData$date) %in% c("Saturday", "Sunday"), 
                                     "Weekend", "Weekday"))
meanByDayIntvl <- aggregate(steps ~ interval + dayType, imputedData, mean)
p2 <- qplot(interval, steps, data=meanByDayIntvl, geom = "line", facets = dayType ~ .)
p2 + labs(x = "Time of Day", y = "Average Number of Steps (5-min Interval)")
```

![](PA1_template_files/figure-html/unnamed-chunk-12-1.png) 

The main difference seems to be that on weekdays there is a much sharper peak in activity level in the morning while on weekends the peak in activity level is more evenly spread through the morning to afternoon. This might be traceable to the fact that more of the day is available for physical activities during the weekends while on weekdays they are confined to the period immediately before the beginning or at the end of the workday.
