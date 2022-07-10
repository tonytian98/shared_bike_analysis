# SQL_Analysis

## Business Background
In 2016, Cyclistic launched a successful bike-share offering. Since then, the program has grown to a fleet of 5,824 bicycles that
are geotracked and locked into a network of 692 stations across Chicago. The bikes can be unlocked from one station and
returned to any other station in the system anytime.
Its pricing plans: single-ride passes, full-day passes, and annual memberships. Customers who purchase single-ride or full-day passes are referred to as **casual riders**. Customers who purchase annual memberships are Cyclistic **members**.
## Business Task
The marketing team has set a clear goal: Design marketing strategies aimed at converting casual riders into annual members. In order to
do that, however, the analyst team needs to **better understand how annual members and casual riders differ**,

[Data Source](https://divvy-tripdata.s3.amazonaws.com/index.html) *Download those YYYYMM-divvy-tripdata.zip as monthly data then merge them together. 12 months of data is about 1 Million rows*

## SQL Analysis
*Because the data has more than 1Million rows, it's better to analyse it on a MySQL Server.*

The data header lookes like this.
![alt text](https://github.com/tonytian98/shared_bike_analysis/blob/main/SQL1.png)

### 1.We may use the *started_at*, *ended_at* fields to calculate trip duration to see how it differs between members and casual users
#### First just calculate the average of duration in different groups (may also use MIN(), MAX() to see each user type's min and max)

##### SQL Code:
```
SELECT 
  member_casual,
  COUNT(*) AS total,
  AVG(TIMESTAMPDIFF(minute, started_at, ended_at)) AS average_duration 
FROM bike.12_months_tripdata
GROUP BY member_casual;
```
Their averages look like this:
![alt text](https://github.com/tonytian98/shared_bike_analysis/blob/main/SQL3.png)
It turns our members rides shorter in duration.

#### Let's dig deeper to actually see the distribution of duration from each group
##### SQL Code:
```
ALTER TABLE bike.tripdata
ADD duration INT;

UPDATE bike.12_months_tripdata 
SET duration_min =
(TIMESTAMPDIFF(minute, started_at, ended_at));

/* Turns out that some duration is negative, meaning those rows' data is not valid, 
   we need to delete them and redo our analysis. 
*/
 
SET SQL_SAFE_UPDATES=0;
DELETE  FROM  bike.12_months_tripdata WHERE duration_min <0;

SELECT 
  member_casual AS type,
  duration_min,
  COUNT(*) AS total
FROM 12_months_tripdata
GROUP BY member_casual, duration_min

/* Since we will reference the temp table twice under one command later, MySQL doesn't allow that for temporary tables
   So we need to create a proper table then delete it manually after it served its purpose. 
*/

CREATE TABLE IF NOT EXISTS temp(
SELECT 
  member_casual AS type,
	CASE 
    WHEN duration_min BETWEEN 0 AND 10 THEN "less than 10 mins"
    WHEN duration_min BETWEEN 11 AND 20 THEN "11 to 20 mins"
    WHEN duration_min BETWEEN 21 AND 30 THEN "21 to 30 mins"
    WHEN duration_min BETWEEN 31 AND 60 THEN "31 to 60 mins"
    WHEN duration_min BETWEEN 61 AND 120 THEN "1 to 2 hours"
    ELSE "More than 2 hours"
    END AS duration_type
FROM 12_months_tripdata);

SELECT 
  type,
  duration_type,
  CONCAT(ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (PARTITION BY type), 2), '%') AS percentage
FROM temp
GROUP BY type, duration_type;

DROP TABLE IF EXISTS temp;
```

Their distribution look like this:
![alt text](https://github.com/tonytian98/shared_bike_analysis/blob/main/SQL2.png)

**Notice** Members have a significantly large porpotion in the "less than 10 minutes" duration (64%) 

### 2. We can use the categorical field *start_station_name* combined with start_lat, start_lng (latitude and longitude) to investigate where do members and casuals concentrate at the start of rides, especially with the help of Tableau

##### SQL Code:
```
SELECT 
  member_casual AS type,
  start_station_name AS start_station,
  start_lat,
  start_lng,
  COUNT(*) AS count
FROM bike.12_months_tripdata
GROUP BY  member_casual, start_station_name
ORDER BY count DESC
```
Here we get some of the most popular start stations.
![alt text](https://github.com/tonytian98/shared_bike_analysis/blob/main/SQL4.png)


### 3. We can use the field *started_at* to see when do users like to ride our shared bikes, maybe most members use it to go to work and go back home (maybe 7:00-9:00 and 17:00-19:00 will have large amount of member usages)

##### SQL Code:
```
ALTER TABLE bike.tripdata
ADD start_time INT;

UPDATE bike.12_months_tripdata 
SET start_time =
(
CASE
    WHEN HOUR(started_at) BETWEEN 5 AND 6 THEN "5~6"
    WHEN HOUR(started_at) BETWEEN 7 AND 8 THEN "7~8"
    WHEN HOUR(started_at) BETWEEN 9 AND 10 THEN "9~10"
    WHEN HOUR(started_at) BETWEEN 11 AND 12 THEN "11~12"
    WHEN HOUR(started_at) BETWEEN 13 AND 14 THEN "13~14"
    WHEN HOUR(started_at) BETWEEN 15 AND 16 THEN "15~16"
    WHEN HOUR(started_at) BETWEEN 17 AND 18 THEN "17~18"
    WHEN HOUR(started_at) BETWEEN 19 AND 20 THEN "19~20"
    WHEN HOUR(started_at) BETWEEN 21 AND 22 THEN "21~22"
	ELSE "Midnight & Before Dawn"
END);


SELECT member_casual AS type,
	start_time,
	CONCAT(ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (PARTITION BY member_casual), 2), '%') AS percentage
FROM 12_months_tripdata
GROUP BY type, start_time
```
The result looks like this ![alt text](https://github.com/tonytian98/shared_bike_analysis/blob/main/SQL5.png)

There are indeed more members using bikes at 7:00 to 9:00

### 4. We can use the field *started_at* to see what day of the week do users like to ride our shared bikes, maybe most members ride bike from Monday to Friday, casuals ride those mostly on weekends.

##### SQL Code:
```
SELECT member_casual AS type,
  DAYOFWEEK(started_at) AS day_of_week,
  CONCAT(ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (PARTITION BY member_casual), 2), '%') AS percentage
FROM bike.12_months_tripdata
GROUP BY member_casual, DAYOFWEEK(started_at) 
ORDER BY CAST(percentage AS FLOAT) DESC
```

The result looks like this ![alt text](https://github.com/tonytian98/shared_bike_analysis/blob/main/SQL7.png)

Indeed, from Tuesday to Thurday, members percentages are higher, casuals like to ride bikes on Sunday and Monday, Saturday bike usage is surprisingly low for both types.


### 5. We can use the field *start_station_name* and *end_station_name* to see if there are certain types like to do round trips (meaning the start and end station is the same one).

##### SQL Code:
```
SELECT
  member_casual AS type,
  CONCAT(ROUND(avg( start_station_id = end_station_id ) * 100,2), "%") AS percentage_of_round_trip
FROM bike.12_months_tripdata
GROUP BY member_casual
```

The result looks like this ![alt text](https://github.com/tonytian98/shared_bike_analysis/blob/main/SQL6.png)

About 10 percentage of casuals do round trips, whereas only 3% of members do round trips
