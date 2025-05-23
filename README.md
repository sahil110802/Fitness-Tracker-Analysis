**The Business Task:**
  * To analyze data to gain insights on how consumers use smart devices.
  * Use the discovered insights to help guide marketing strategies for the company.
___

## Preparing Data for Analysis

For this project, I've used the 18 datasets dated from 03/12/2016 - 05/12/2016 provided on kaggle. Click on this [FitBit Fitness Tracker Data](https://www.kaggle.com/arashnic/fitbit) (CC0: Public Domain, dataset made available through Mobius) to access the website and download the datasets provided as .csv files. This data set contains personal fitness data from 30 fitbit users on minute-level output for physical activity, heart rate, and sleep monitoring. It also includes information on daily activity, steps, and heart rate that can be used to explore users’ habits. Thirty Fitbit users consented to the submission of personal tracker data and the data provided in this website is made available to access to the public. 

Or you could access and download the data from this repository named as "Raw Data".

I am using Microsoft SQL Server Management Studio in this part of the project to help process and analyze the datasets.

First make sure to import all of the dataset as a .csv file to the database server.
In order to solve this business task, only 6 of the given 18 datasets was used. Many of the files are either redundant, is not essential, or relevant, or lacking sufficient data to perform an analysis upon.

___


## Processing of Data
Here, I will be transforming and organizing data by adding new columns, extracting information and removing bad data and duplicates.

**In order to get accurate analysis, validate and make sure the dataset does not include any bias, incorrect data, and duplicates.**

```TSQL

--Checking Number of Rows on dailyActivities

Select Count(*)
From [dbo].[dailyActivity_merged]


--Checking for duplicates in dailyActivity dataset

Select Id, ActivityDate, TotalSteps, Count(*)
From [dbo].[dailyActivity_merged]
group by id, ActivityDate, TotalSteps
Having Count(*) > 1


--Modify date format for better understaning in sleepDay

Update sleepDay_merged
Set SleepDay = Convert(date, SleepDay, 101)


--Modify date format for better understaning in dailyActivity

Update dailyActivity_merged
Set ActivityDate = Convert(date, ActivityDate, 101)


--Add day_0f_week column on dailyActivities

Alter Table [dbo].[dailyActivity_merged]
ADD day_of_week nvarchar(50)


--Extract datename from ActivityDate

Update dailyActivity_merged
SET day_of_week = DATENAME(DW, ActivityDate)


--Add sleep data columns on dailyActivity table

Alter Table [dbo].[dailyActivity_merged]
ADD total_mins_sleep int,
total_mins_bed int


--Add sleep records into dailyActivity table

UPDATE dailyActivity_merged
Set total_mins_sleep = t2.TotalMinutesAsleep,
total_mins_bed = t2.TotalTimeInBed 
From [dbo].[dailyActivity_merged] as t1
Full Outer Join sleepDay_merged as t2
on t1.id = t2.id and t1.ActivityDate = t2.SleepDay


--Adding specific date format to [dailyActivity_merged] table

Alter table dailyActivity_merged
Add Date_d date

Update dailyActivity_merged
Set Date_d = CONVERT( date, ActivityDate, 103 )


--Split date and time seperately for [hourlyCalories_merged] table

Alter Table [dbo].[hourlyCalories_merged]
ADD time_h int

Update [dbo].[hourlyCalories_merged]
Set time_h = DATEPART(hh, Date_d)

Update [dbo].[hourlyCalories_merged]
Set Date_d = SUBSTRING(Date_d, 1, 9)


--Split date and time seperately for [hourlyIntensities_merged]

Alter Table [dbo].[hourlyIntensities_merged]
ADD time_h int

Update [dbo].[hourlyIntensities_merged]
Set time_h = DATEPART(hh, ActivityHour)

Update [dbo].[hourlyIntensities_merged]
Set ActivityHour = SUBSTRING(ActivityHour, 1, 9)


--Split date and time seperately for [hourlySteps_merged]

Alter Table [dbo].[hourlySteps_merged]
ADD time_h int

Update [dbo].[hourlySteps_merged]
Set time_h = DATEPART(hh, Date_d)

Update [dbo].[hourlySteps_merged]
Set Date_d = SUBSTRING(Date_d, 1, 9)


--Split date and time seperately for [minuteMETsNarrow_merged]

Alter Table [dbo].[minuteMETsNarrow_merged]
ADD time_t time

Update [dbo].[minuteMETsNarrow_merged]
Set time_t = CAST(Date_d as time)

Update [dbo].[minuteMETsNarrow_merged]
Set time_t = Convert(varchar(5), time_t, 108)

Update [dbo].[minuteMETsNarrow_merged]
Set Date_d = SUBSTRING(Date_d, 1, 9)


--Create new table to merge hourlyCalories, hourlyIntensities, and hourlySteps

Create table hourly_cal_int_step_merge(
Id numeric(18,0),
Date_d nvarchar(50),
time_h int,
Calories numeric(18,0),
TotalIntensity numeric(18,0),
AverageIntensity float,
StepTotal numeric (18,0)
)


--Insert corresponsing data and merge multiple table into one table

Insert Into hourly_cal_int_step_merge (Id, Date_d, time_h, Calories, TotalIntensity, AverageIntensity, StepTotal)
(Select t1.Id, t1.Date_d, t1.time_h, t1.Calories, t2.TotalIntensity, t2.AverageIntensity, t3.StepTotal
From [dbo].[hourlyCalories_merged] as t1
Inner Join [dbo].[hourlyIntensities_merged] as t2
ON t1.Id = t2.Id and t1.Date_d = t2.ActivityHour and t1.time_h = t2.time_h
Inner Join [dbo].[hourlySteps_merged] as t3
ON t1.Id = t3.Id and t1.Date_d = t3.Date_d and t1.time_h = t3.time_h)


--Checking for duplicates

/*Select Id, time_h, Calories, TotalIntensity, AverageIntensity, StepTotal, Count(*) as duplicates
From [dbo].[hourly_cal_int_step_merge]
Group by Id, time_h, Calories, TotalIntensity, AverageIntensity, StepTotal
Having Count(*) > 1*/


--Checking for duplicates

/*Select sum(duplicates) as sum_s
from (Select Id, Date_d time_h, Calories, TotalIntensity, AverageIntensity, StepTotal, Count(*) as duplicates
From [dbo].[hourly_cal_int_step_merge]
Group by Id, Date_d, time_h, Calories, TotalIntensity, AverageIntensity, StepTotal
Having Count(*) > 1
Order by duplicates DESC) as cte*/


--Query in hh:mm time format for better understanding on MET Table 

select Id, Cast(Date_d as date) as date_d, METs, Convert(varchar(5), time_t, 108) as time_t
From [dbo].[minuteMETsNarrow_merged]


--Change date type nvarchar to date on MET table to join properly with other table

Alter table minuteMETsNarrow_merged
Add dates_d date

Update minuteMETsNarrow_merged
Set dates_d = Cast(Date_d as date)
```
___


## Analysis of Data

Here, I will be analysing the consumer data to discover trends and patterns. 

```TSQL
                                                      --Analysis--
--Calculate average met per day per user, and compare with the calories burned

Select Distinct t1.Id, t1.dates_d, sum(t1.METs) as sum_mets, t2.Calories
From [dbo].[minuteMETsNarrow_merged] as t1
inner join dailyActivity_merged as t2
on t1.Id = t2.Id and t1.dates_d = t2.Date_d
Group By t1.Id, t1.dates_d, t2.Calories
Order by dates_d
```

Result: *Data contains 934 rows, this table limits to view the top 50 rows*

| Id         | dates_d    | sum_METs | Calories |
|------------|------------|----------|----------|
| 1503960366 | 12-04-2016 | 25241    | 1985     |
| 1624580081 | 12-04-2016 | 17234    | 1432     |
| 1644430081 | 12-04-2016 | 22768    | 3199     |
| 1844505072 | 12-04-2016 | 21704    | 2030     |
| 1927972279 | 12-04-2016 | 15599    | 2220     |
| 2022484408 | 12-04-2016 | 23035    | 2390     |
| 2026352035 | 12-04-2016 | 18830    | 1459     |
| 2320127002 | 12-04-2016 | 23090    | 2124     |
| 2347167796 | 12-04-2016 | 24924    | 2344     |
| 2873212765 | 12-04-2016 | 22570    | 1982     |
| 3372868164 | 12-04-2016 | 19118    | 1788     |
| 3977333714 | 12-04-2016 | 20644    | 1450     |
| 4020332650 | 12-04-2016 | 26521    | 3654     |
| 4057192912 | 12-04-2016 | 18434    | 2286     |
| 4319703577 | 12-04-2016 | 14400    | 2115     |
| 4388161847 | 12-04-2016 | 14400    | 2955     |
| 4445114986 | 12-04-2016 | 18392    | 2113     |
| 4558609924 | 12-04-2016 | 20576    | 1909     |
| 4702921684 | 12-04-2016 | 21040    | 2947     |
| 5553957443 | 12-04-2016 | 22118    | 2026     |
| 5577150313 | 12-04-2016 | 26431    | 3405     |
| 6117666160 | 12-04-2016 | 14400    | 1496     |
| 6290855005 | 12-04-2016 | 17890    | 2560     |
| 6775888955 | 12-04-2016 | 14400    | 1841     |
| 6962181067 | 12-04-2016 | 22065    | 1994     |
| 7007744171 | 12-04-2016 | 27171    | 2937     |
| 7086361926 | 12-04-2016 | 24496    | 2772     |
| 8053475328 | 12-04-2016 | 26168    | 3186     |
| 8253242879 | 12-04-2016 | 20620    | 2044     |
| 8378563200 | 12-04-2016 | 24210    | 3635     |
| 8583815059 | 12-04-2016 | 18896    | 2650     |
| 8792009665 | 12-04-2016 | 17433    | 2044     |
| 8877689391 | 12-04-2016 | 32010    | 3921     |
| 1503960366 | 13-04-2016 | 22859    | 1797     |
| 1624580081 | 13-04-2016 | 16990    | 1411     |
| 1644430081 | 13-04-2016 | 20656    | 2902     |
| 1844505072 | 13-04-2016 | 19880    | 1860     |
| 1927972279 | 13-04-2016 | 15014    | 2151     |
| 2022484408 | 13-04-2016 | 25074    | 2601     |
| 2026352035 | 13-04-2016 | 19636    | 1521     |
| 2320127002 | 13-04-2016 | 21782    | 2003     |
| 2347167796 | 13-04-2016 | 21675    | 2038     |
| 2873212765 | 13-04-2016 | 22820    | 2004     |
| 3372868164 | 13-04-2016 | 22380    | 2093     |
| 3977333714 | 13-04-2016 | 21286    | 1495     |
| 4020332650 | 13-04-2016 | 14400    | 1981     |
| 4057192912 | 13-04-2016 | 18566    | 2306     |
| 4319703577 | 13-04-2016 | 17299    | 2135     |
| 4388161847 | 13-04-2016 | 19246    | 3092     |
| 4445114986 | 13-04-2016 | 18230    | 2095     |

___

```TSQL
                                                             
--Activities and colories comparison

Select Id,
SUM(TotalSteps) as total_steps,
SUM(VeryActiveMinutes) as total_Vactive_mins,
Sum(FairlyActiveMinutes) as total_Factive_mins,
SUM(LightlyActiveMinutes) as total_Lactive_mins,
SUM(Calories) as total_calories
From [dbo].[dailyActivity_merged]
Group By Id
```

Result: *Strong correlation between activity intense time and calories burned*

| id         | total_steps | total_Vactive_m | total_Factive_m | total_Lactive_m | calories |
|------------|-------------|-----------------|-----------------|-----------------|----------|
| 8583815059 | 223154      | 300             | 688             | 4287            | 84693    |
| 6117666160 | 197308      | 44              | 57              | 8074            | 63312    |
| 8053475328 | 457662      | 2640            | 297             | 4680            | 91320    |
| 4319703577 | 225334      | 111             | 382             | 7092            | 63168    |
| 4388161847 | 335232      | 718             | 631             | 7110            | 95910    |
| 6290855005 | 163837      | 80              | 110             | 6596            | 75389    |
| 2320127002 | 146223      | 42              | 80              | 6144            | 53449    |
| 1644430081 | 218489      | 287             | 641             | 5354            | 84339    |
| 2873212765 | 234229      | 437             | 190             | 9548            | 59426    |
| 1503960366 | 375619      | 1200            | 594             | 6818            | 56309    |
| 4702921684 | 265734      | 159             | 807             | 7362            | 91932    |
| 4445114986 | 148693      | 205             | 54              | 6482            | 67772    |
| 4057192912 | 15352       | 3               | 6               | 412             | 7895     |
| 1624580081 | 178061      | 269             | 180             | 4758            | 45984    |
| 8792009665 | 53758       | 28              | 117             | 2662            | 56907    |
| 2026352035 | 172573      | 3               | 8               | 7956            | 47760    |
| 3977333714 | 329537      | 567             | 1838            | 5243            | 45410    |
| 5577150313 | 249133      | 2620            | 895             | 4438            | 100789   |
| 4558609924 | 238239      | 322             | 425             | 8834            | 63031    |
| 8378563200 | 270249      | 1819            | 318             | 4839            | 106534   |
| 1844505072 | 79982       | 4               | 40              | 3579            | 48778    |
| 7007744171 | 294409      | 807             | 423             | 7299            | 66144    |
| 8253242879 | 123161      | 390             | 272             | 2221            | 33972    |
| 2347167796 | 171354      | 243             | 370             | 4545            | 36782    |
| 4020332650 | 70284       | 161             | 166             | 2385            | 73960    |
| 6962181067 | 303639      | 707             | 574             | 7620            | 61443    |
| 6775888955 | 65512       | 286             | 385             | 1044            | 55426    |
| 5553957443 | 266990      | 726             | 403             | 6392            | 58146    |
| 1927972279 | 28400       | 41              | 24              | 1196            | 67357    |
| 7086361926 | 290525      | 1320            | 786             | 4459            | 79557    |
| 3372868164 | 137233      | 183             | 82              | 6558            | 38662    |
| 8877689391 | 497241      | 2048            | 308             | 7276            | 106028   |
| 2022484408 | 352490      | 1125            | 600             | 7981            | 77809    |

___

```TSQL

--Average Sleep Time per user

Select Id, Avg(TotalMinutesAsleep)/60 as avg_sleep_time_h,
Avg(TotalTimeInBed)/60 as avg_time_bed_h,
AVG(TotalTimeInBed - TotalMinutesAsleep) as wasted_bed_time_m
from sleepDay_merged
Group by Id

```
Result: 

| Id         | avg_sleep_time_h | avg_time_bed_h | wasted_bed_time_m |
|------------|------------------|----------------|-------------------|
| 1503960366 | 6.004666         | 6.386666       | 22.92             |
| 1644430081 | 4.9              | 5.766666       | 52                |
| 1844505072 | 10.866666        | 16.016666      | 309               |
| 1927972279 | 6.95             | 7.296666       | 20.8              |
| 2026352035 | 8.436309         | 8.960714       | 31.464285         |
| 2320127002 | 1.016666         | 1.15           | 8                 |
| 2347167796 | 7.446666         | 8.188888       | 44.533333         |
| 3977333714 | 4.894047         | 7.685714       | 167.5             |
| 4020332650 | 5.822916         | 6.329166       | 30.375            |
| 4319703577 | 7.94423          | 8.366025       | 25.307692         |
| 4388161847 | 6.71875          | 7.103472       | 23.083333         |

___

```TSQL

--Sleep and calories comparison	

Select t1.Id, SUM(TotalMinutesAsleep) as total_sleep_m,
SUM(TotalTimeInBed) as total_time_inbed_m,
SUM(Calories) as calories
From [dbo].[dailyActivity_merged] as t1
Inner Join [dbo].[sleepDay_merged] as t2
ON t1.Id = t2.Id and t1.ActivityDate = t2.SleepDay
Group By t1.Id
```

Result:

| Id         | total_sleep_m | total_time_inbed_m | calories |
|------------|---------------|--------------------|----------|
| 1503960366 | 9007          | 9580               | 46807    |
| 1644430081 | 1176          | 1384               | 11911    |
| 1844505072 | 1956          | 2883               | 5029     |
| 1927972279 | 2085          | 2189               | 11581    |
| 2026352035 | 14173         | 15054              | 43142    |
| 2320127002 | 61            | 69                 | 1804     |
| 2347167796 | 6702          | 7370               | 29570    |
| 3977333714 | 8222          | 12912              | 43691    |
| 4020332650 | 2795          | 3038               | 25560    |
| 4319703577 | 12393         | 13051              | 52642    |
| 4388161847 | 9675          | 10229              | 75159    |
| 4445114986 | 10785         | 11671              | 61128    |
| 4558609924 | 638           | 700                | 10985    |
| 4702921684 | 11792         | 12375              | 85211    |
| 5553957443 | 14368         | 15682              | 58146    |
| 5577150313 | 11232         | 11976              | 92019    |
| 6117666160 | 8618          | 9183               | 44295    |
| 6775888955 | 1049          | 1107               | 7034     |
| 6962181067 | 13888         | 14450              | 61443    |
| 7007744171 | 137           | 143                | 4301     |
| 7086361926 | 10875         | 11194              | 63783    |
| 8053475328 | 891           | 905                | 9928     |
| 8378563200 | 14187         | 15466              | 110539   |
| 8792009665 | 6535          | 6807               | 34490    |

___

```TSQL

--Daily Sum Analysis - No trends/patterns found

Select SUM(TotalSteps) as total_steps,
SUM(TotalDistance) as total_dist,
SUM(Calories) as total_calories,
day_of_week
From [dbo].[dailyActivity_merged]
Group By  day_of_week

```
Result: *No trends or patterns found*

| total_steps | total_dist | total_calories | day_of_week |
|-------------|------------|----------------|-------------|
| 1133906     | 763        | 345393         | Wednesday   |
| 1010969     | 676        | 292016         | Saturday    |
| 933704      | 613        | 278905         | Monday      |
| 838921      | 554        | 273823         | Sunday      |
| 938477      | 614        | 293805         | Friday      |
| 1088658     | 718        | 323337         | Thursday    |
| 1235001     | 817        | 358114         | Tuesday     |

___

```TSQL

--Daily Average analysis - No trends/patterns found

Select AVG(TotalSteps) as avg_steps,
AVG(TotalDistance) as avg_dist,
AVG(Calories) as avg_calories,
day_of_week
From [dbo].[dailyActivity_merged]
Group By  day_of_week
```
Result: *No trends or patterns found*

| avg_steps   | avg_dist | avg_calories | day_of_week |
|-------------|----------|--------------|-------------|
| 7559.373333 | 5.086666 | 2302.62      | Wednesday   |
| 8152.975806 | 5.451612 | 2354.967741  | Saturday    |
| 7780.866666 | 5.108333 | 2324.208333  | Monday      |
| 6933.231404 | 4.578512 | 2263         | Sunday      |
| 7448.230158 | 4.873015 | 2331.785714  | Friday      |
| 7405.836734 | 4.884353 | 2199.571428  | Thursday    |
| 8125.006578 | 5.375    | 2356.013157  | Tuesday     |

___

```TSQL
--Time Expenditure per day
Select Distinct Id, SUM(SedentaryMinutes) as sedentary_mins,
SUM(LightlyActiveMinutes) as lightly_mins,
SUM(FairlyActiveMinutes) as fairlyactive_mins, 
SUM(VeryActiveMinutes) as veryactive_mins
From [dbo].[dailyActivity_merged]
where total_mins_bed IS NOT NULL
Group by Id
```
Result:

![time spent in a day](https://github.com/sahil110802/Fitness-Tracker-Analysis/blob/main/Tableau%20Visualizations/Time%20spent%20in%20a%20day.PNG)

___


## Visualizing Data

In this phase, we will be visualizing the data analyzed and tables created using Tableau Public.

### Activity Durations and Calories Relationship:

![Activitydurationcalories](https://github.com/sahil110802/Fitness-Tracker-Analysis/blob/main/Tableau%20Visualizations/Activity%20Duration%20and%20Calories%20Burned.PNG)

Key Findings:
* The R-Squared value for Low Active graph is 0.0118
* The R-Squared value for Fairly Active graph is 0.0391
* The R-Squared value for Very Active graph is 0.3865

There is a strong correlation between Very Active minutes and the amount of calories burned. The r-squared value seems to rise as the intensity and the duration is the activity increases.

Thus by inferring to the r-squared values of the respective trend lines of the graphs, we can conclude that the higher the intensity and the duration of the activity, the more calories is burned.

### METs and Average Calories Burned: 

**A brief overview on METs:**

Metabolic Equivalent of Task (MET) is the ratio of your working metabolic rate relative to your resting metobolic rate. It is a way to describe the intensity of an activity or workout. METs provide a simple and practical way to easily understand the body's energy expenditure with metabolic equivalents.

One MET is defined as resting metabolic rate, it is approximately 3.5 milliliters of oxygen consumed per kilogram of body weight per minute. One MET is the energy a person spends while resting or sitting idle. 

So an MET of 4 means a person is spending 4 times the energy they are when sitting idle. The METs for each person is not constant and can have slight variations depending on their weight, age or other physical parameters. 

To calculate the amount of calories burned per minute, we can use the formula:

**Calories burned per minute = (METs x 3.5 x (your body weight in Kg)) / 200**

To better understand METs more, the approximate METs values while performing certain excersices are depicted below:

| Light < 3.0 METs              | Moderate 3.0–6.0 METs                  | Vigorous > 6.0 METs                       |
|-------------------------------|----------------------------------------|-------------------------------------------|
| Sitting at a desk: 1.3        | Housework (cleaning, sweeping): 3.5    | Walking at very brisk pace (4.5 mph): 6.3 |
| Sitting, playing cards: 1.5   | Weight training (lighter weights): 3.5 | Bicycling 12–14 mph (flat terrain): 8     |
| Standing at a desk: 1.8       | Golf (walking, pulling clubs): 4.3     | Circuit training (minimal rest): 8        |
| Strolling at a slow pace: 2.0 | Brisk walking (3.5–4 mph): 5           | Singles tennis: 8                         |
| Washing dishes: 2.2           | Weight training (heavier weights): 5   | Shoveling, digging ditches: 8.5           |
| Hatha yoga: 2.5               | Yard work (mowing, moderate effort): 5 | Competitive soccer: 10                    |
| Fishing (sitting): 2.5        | Swimming laps (leisurely pace): 6      | Running (7 mph): 11.5                     |




![METs vs Average Calories](https://github.com/sahil110802/Fitness-Tracker-Analysis/blob/main/Tableau%20Visualizations/METs%20vs%20Avg.Calories%20Burned.png)

Key Findings:
* The R-Squared value is 0.5504
* Strong positive corellation between METs and average calories burned.

The amount of calories burned for every user is highly dependent on their MET values they spend every day. This can be seen by the high r-squared value suggesting that the trend line has strong relation with the data points. 


**Sleep and Calories Comparison **


![sleep and calories](https://github.com/sahil110802/Fitness-Tracker-Analysis/blob/main/Tableau%20Visualizations/Sleep%20and%20Calories%20Burned%20Comparison.png)

Key Findings:
* The R-Squared value is 0.8727
* Strong positive corellation between amount of sleep and calories burned. 

Higher duration of sleep is associated with higher amount of calories burned. An adequate duration and good quality of sleep constitutes to higher calories burned during the sleeping process. 

However sleeping more than the required range doesn't seem to burn more calories and in fact causes the opposite to occur, which is burn fewer calories. 


**Popular Time for Activities**

![popular time for workouts](https://github.com/sahil110802/Fitness-Tracker-Analysis/blob/main/Tableau%20Visualizations/Popular%20Time%20for%20Workout.png)

From the graph above, we can infer that the most popular time people are active throughout the day is between **5:00 AM - 9:00PM**

**Tableau Dashboard**

Visualizations built in a dashboard. 

![dashboard](https://github.com/sahil110802/Fitness-Tracker-Analysis/blob/main/Tableau%20Visualizations/FitBit%20Case%20Study%20Visualized.png)

___

## Conclusion


We can infer that the duration and the level of intensity of the activities performed are greatly in dependence to the amount of calories burned. METs provide a great insight on the intensity of activities performed and the amount of calories burned per minute. While most of the consumers attain adequate amounts of sleep, it is noticed that a small fraction of the users either oversleep or undersleep. Consumers are also more likely to perform low-high intensity activities in the range of **5:00 AM - 9:00PM** throughout the day.





