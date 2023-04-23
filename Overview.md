# How Does a Bike-Share Navigate Speedy Success?

## Introduction
  For the Google Data Analytics Capstone, I decided to complete the Cyclistic bike-share analysis case study. I used Google’s data analysis process (ask, prepare, process, analyze, share, and act) as a guideline for this case study. In this scenario, I was a junior data analyst in the Cyclistic marketing analytics team being managed by the director of marketing, Lily Moreno. Cyclistic is a company which launched a bike-share program last 2016 that has grown to a fleet of 5,824 bicycles that are geotracked and locked into a network of 692 stations across Chicago. Bikes can be unlocked and returned to any station within the network at any time.
  
  The company’s general marketing objectives were to increase brand awareness and gain customers from a broad consumer segment.One strategy was their flexible pricing plan: single-ride passes, full-day passes, and annual memberships. Customers who purchase single-ride or full-day passes were referred to as casual riders while customers who purchased annual memberships were Cyclistic members. 

  Cyclistic’s finance analysts concluded that annual members were much more profitable than casual riders. With this information, Lily Moreno believed that maximizing annual memberships would lead to the company’s success and thus planned to design a marketing strategy that would make casual riders avail annual memberships. The director believed that casual riders have a high chance to avail annual membership as they were already aware about and chose the company’s services for their mobility needs. 

  In order to devise an effective marketing program, the following questions must be investigated by the marketing analytics team:
* How do annual members and casual riders use Cyclistic bikes differently?
* Why would casual riders buy Cyclistic annual memberships?
* How can Cyclistic use digital media to influence casual riders to become members?

  Lily Moreno assigned the question “How do annual members and casual riders use Cyclistic bikes differently?” to me. This question was the focus of the entire case study.

## Business Task
  Identify differentiating factors of annual members and casual riders based on the latest 12 months of historical trip data and create an effective business plan to convert casual riders into annual members.

## Key Stakeholders
* Cyclistic Executive Team
  * Notoriously detail-oriented executive team who will decide whether to approve the recommended marketing program.
* Lily Moreno
  * The director of marketing of the company and direct supervisor.
  * Responsible for the development of marketing campaigns and initiatives of the bike-share program which may include email, social media, and other channels. 
  * Aims to devise a marketing strategy to convert casual riders to annual members 

## Data Source
  For the purpose of this case study, the public dataset of Divvy by Motivate International Inc will be used as Cyclistic’s historical trip data. The study used the latest available 12 months worth of data from the cloud through AWS. For data-privacy, no identifiable information of the rider is provided such as credit card information and home address.

## Data Cleaning
  This case study used the dataset for the year of 2022 as it was the available dataset at the time of download. The dataset initially came as monthly CSV files that contain 5,667,717 rows when combined with 13 columns each.

| Column Name | Description |
| ----------- | ----------- |
| ride_id | Unique identifier of the recorded trip and serves as the primary ID.|
| rideable_type | Type of bike used: classic bike, electric bike, or docked bike. |
| started_at | Time and date the bike was unlocked from the station. |
| ended_at | Time and date the bike was returned to the station. |
| start_station_name | Name of the station where the bike was unlocked. |
| start_station_id | Unique identifier for the station where the bike was unlocked. |
| end_station_name | Name of the station where the bike was returned. |
| end_station_id | Unique identifier for the station where the bike was returned. |
| start_lat | Latitude of the station where the bike was unlocked. |
| start_lng | Longitude of the station where the bike was unlocked. |
| end_lat | Latitude of the station where the bike was returned. |
| end_lng | Longitude of the station where the bike was returned. |
| member_casual | Type of membership: casual rider or annual member. |


  SQLite with DB Browser was used to clean the data. SQL was used due to the size of the consolidated 12 months worth of data which spreadsheet software may have trouble processing.
  
  The CSV files were imported to a SQLite database as individual tables for each month with ride_id as the primary key. The table schema for all months was verified to be identical and all tables were then combined into a single table named consolidated_2022.

  Each station id was checked if it referenced a unique station name.
  
 ```
SELECT COUNT (DISTINCT start_station_name),
	COUNT (DISTINCT start_station_id)
FROM consolidated_2022;

SELECT COUNT (DISTINCT end_station_name),
	COUNT (DISTINCT end_station_id)
FROM consolidated_2022;
```
  There were 1,692 unique end station names with only 1,317 unique station IDs and 1,674 unique start station names with only 1,313 unique station IDs. To investigate, the associated station names of each station IDs were counted.

```
WITH cte_id AS
	(SELECT DISTINCT start_station_name,start_station_id
	FROM consolidated_2022
	)
SELECT start_station_name,start_station_id, COUNT (start_station_id) AS station_cnt
FROM cte_id
GROUP BY start_station_id
HAVING COUNT (start_station_id) > 1
ORDER BY station_cnt;

WITH cte_id AS
	(SELECT DISTINCT end_station_name,end_station_id
	FROM consolidated_2022
	)
SELECT end_station_name,end_station_id, COUNT (end_station_id) AS station_cnt
FROM cte_id
GROUP BY end_station_id
HAVING COUNT (end_station_id) > 1
ORDER BY station_cnt;
```
  Some station IDs had 2 to 4 station names attached to them, there were 339 end station IDs and 329 start station IDs with multiple station names attached to them. For this reason, station ID was decided to be used for trip data analysis. On initial inspection, it was found out that some station names with an “&” symbol had a variation with “amp;” possibly due to an error. This was corrected and reflected onto the table.

```
UPDATE consolidated_2022
SET start_station_name = 
  replace(start_station_name, 'amp;','')
  WHERE start_station_name like '%amp;%';

UPDATE consolidated_2022
SET end_station_name = 
  replace(end_station_name, 'amp;','')
  WHERE end_station_name like '%amp;%';
```
  Unfortunately, it only affected 2 station names on the start_station_name column.
  
  Each column was checked for null values.
```
SELECT start_station_id
FROM consolidated_2022
WHERE start_station_id IS NULL;

SELECT start_lat, start_lng
FROM consolidated_2022
WHERE start_lng IS NULL OR start_lat IS NULL;

SELECT end_station_id
FROM consolidated_2022
WHERE end_station_id IS NULL;

SELECT end_lat, end_lng, end_station_id
FROM consolidated_2022
WHERE end_lng IS NULL OR end_lat IS NULL;

SELECT end_lat, end_lng, end_station_id
FROM consolidated_2022
WHERE end_lng IS NOT NULL AND end_lat IS NULL;

SELECT *
FROM consolidated_2022
WHERE ended_at IS  NULL;

SELECT *
FROM consolidated_2022
WHERE started_at IS  NULL;

SELECT *
FROM consolidated_2022
WHERE rideable_type IS  NULL;

SELECT *
FROM consolidated_2022
WHERE member_casual IS  NULL;
```
  There were null latitude and longitude values for end stations without corresponding end station IDs while some rows were only missing end station IDs. Records without end station latitude, longitude, and ID were deleted while those with only missing station IDs would be cross-referenced later. All other columns had no null values. 

```
DELETE FROM consolidated_2022
WHERE end_lng IS NULL OR end_lat IS NULL;
```
  There were 5,858 rows deleted. Before addressing the remaining null station IDs, an initial count was done to verify the number of null values for start and end station IDs.

```
SELECT COUNT (start_station_id) AS not_null, SUM(CASE WHEN start_station_id IS NULL THEN 1 ELSE 0 END) AS null_count
FROM consolidated_2022;

SELECT COUNT (end_station_id) AS not_null, SUM(CASE WHEN end_station_id IS NULL THEN 1 ELSE 0 END) AS null_count
FROM consolidated_2022;
```
  There were initially 833,064 null start station IDs and 886,884 end station IDs. To identify the missing station IDs, the latitude and longitude of the stations were combined and cross referenced with complete entries to fill out the missing station IDs.The following columns were created to identify missing station IDs:
* start_xy		
* end_xy  
  An index utilizing the new columns was also created to improve querying performance
```
SELECT start_lat || '_' || start_lng AS start_xy,
		end_lat|| '_' || end_lng AS end_xy 
FROM consolidated_2022;

ALTER TABLE consolidated_2022 ADD COLUMN start_xy TEXT;
ALTER TABLE consolidated_2022 ADD COLUMN end_xy TEXT;

UPDATE consolidated_2022
SET start_xy = start_lat || '_' || start_lng;
UPDATE consolidated_2022
SET end_xy = end_lat|| '_' || end_lng;

CREATE INDEX 'coordinates'
ON consolidated_2022(ride_id,start_station_id,start_xy,end_station_id,end_xy);
```
  The missing station IDs were then cross-referenced using complete records from the consolidated_2022 table. A new table named xy_station_id was created to store the cross-referenced values obtained through a Common Table Expression by utilizing the coalesce function with window functions. The query would partition the result by coordinates and if the row has a missing station ID, it would replace it with the max value which in this case would be the station ID with the same coordinate as the station ID with a null value.

```
CREATE TABLE xy_station_id AS
WITH sid_cte AS (
SELECT ride_id, coalesce(start_station_id, max(start_station_id) OVER (PARTITION BY start_xy)) AS xy_start_station_id, coalesce(end_station_id, max(end_station_id) OVER (PARTITION BY end_xy)) AS xy_end_station_id
FROM consolidated_2022
)
SELECT * FROM sid_cte;

UPDATE consolidated_2022 AS c
SET start_station_id = x.xy_start_station_id
FROM xy_station_id AS x
WHERE c.ride_id = x.ride_id
AND c.start_station_id IS NULL;

UPDATE consolidated_2022 AS c
SET end_station_id = x.xy_end_station_id
FROM xy_station_id AS x
WHERE c.ride_id = x.ride_id
AND c.end_station_id IS NULL;

SELECT *
FROM consolidated_2022
where start_station_id is NULL;

SELECT *
FROM consolidated_2022
where end_station_id is NULL;
```
  After cross referencing, there were still some null station IDs. The number of null station IDs were recounted.

```
SELECT COUNT (start_station_id) AS not_null, SUM(CASE WHEN start_station_id IS NULL THEN 1 ELSE 0 END) AS null_count
FROM consolidated_2022;

SELECT COUNT (end_station_id) AS not_null, SUM(CASE WHEN end_station_id IS NULL THEN 1 ELSE 0 END) AS null_count
FROM consolidated_2022;
```
 There were still 476,870 start station IDs and 553,680 end station IDs with null values. The incomplete 856,074 rows were deleted.

```
DELETE FROM consolidated_2022
where start_station_id is NULL;

DELETE FROM consolidated_2022
where end_station_id is NULL;
```
 Each column was verified if they only contained appropriate entries (proper spelling, no excess spaces, uniform capitalization, proper format, and within the year 2022). 
 
 ```
SELECT DISTINCT member_casual
FROM consolidated_2022;

SELECT DISTINCT rideable_type
FROM consolidated_2022;

SELECT started_at
FROM consolidated_2022
order by started_at DESC;

SELECT started_at, ended_at
FROM consolidated_2022
order by ended_at DESC;

SELECT length(started_at), length(ended_at)
FROM consolidated_2022
WHERE length(started_at) <> 19 OR length(ended_at) <> 19;
```
  No error was detected upon inspection on every column. All dates were conveniently formatted based on ISO 8601. 
  
  The following factors would be investigated on the analysis phase of the case study: 
* Preference in rideable type
* Ride length
* Rider population by month
* Rider population by week
* Rider population by hour
* Station popularity
* Route popularity

  A new column named ride_length was created for ride length analysis.

```
ALTER TABLE consolidated_2022 ADD COLUMN ride_length REAL;

UPDATE consolidated_2022
SET ride_length = round(((julianday(ended_at) - julianday(started_at))*24*60),3);
```
  The new column was tested for zero and negative values and returned 380 rows. The records were deleted.  

```
SELECT * FROM consolidated_2022 WHERE ride_length <= 0; 

DELETE FROM consolidated_2022
WHERE ride_length <= 0;
```
  A new column named day_of_week was created to investigate weekly trends.

```
ALTER TABLE consolidated_2022 ADD COLUMN day_of_week TEXT;

UPDATE consolidated_2022
SET day_of_week = (select
  case cast(strftime('%w',started_at) as INTEGER)
  when 0 then 'sun'
  when 1 then 'mon'
  when 2 then 'tue'
  when 3 then 'wed'
  when 4 then 'thu'
  when 5 then 'fri'
  else 'sat' end);

SELECT DISTINCT day_of_week
FROM consolidated_2022;
```
  A new column named trip_id was created to identify popular routes.
  
```
ALTER TABLE consolidated_2022 ADD COLUMN trip_id TEXT;
UPDATE consolidated_2022
SET trip_id = start_station_id || '_' || end_station_id;
```
  To reduce the size of the database and improve performance, unnecessary columns  and the xy_station_id table were deleted via DB Browser’s interface. Only the following 12 columns were retained:
* ride_id
* rideable_type
* started_at
* ended_at
* start_station_name
* start_station_id
* end_station_name
* end_station_id
* member_casual
* ride_length
* day_of_week
* trip_id

## Analysis
  R with RStudio was used for the analysis phase of the case study. R is a much more capable programming language for analysis as it has statistical and data visualization tools and is capable of connecting to relational databases.The database was imported to R as a data frame through the RSQLite package. The following packages were loaded into R to expand its capabilities for the analysis: 
* Pacman
* DBI
* RSQLite
* Dbplyr
* Tidyverse
* Lubridate
* Psych
* RColorBrewer
* Scales

  To accommodate color blind people, RColorBrewer palette “Set2” was used in all visualizations in this case study. “Set2” is also a qualitative palette which is ideal for categorical data.
 ![set2 pallete]
