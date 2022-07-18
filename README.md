# Analysis of San Francisco Police Department Incident data
*08/07/2022*

### Methodology
- Data taken from the following link: http://2016.padjo.org/tutorials/sqlite-data-starterpacks/#toc-sfpd-incidents-2012-through-2015
- Queried using SQLite (Git Bash)
- Independently developed & answered a series of questions around the data 
- Comments on the results are highlighted with an arrow symbol (:arrow_forward:)

### Schema overview ('incidents' table) 

| Column | Example | Comments |
| ------------- | ------------- | ------------- |
| datetime  | 2014-05-01 00:01:00  | *Runs from 2012-01-01 to 2015-12-31* |
| category  | LARCENY/THEFT  |  |
| description  | PETTY THEFT OF PROPERTY  |  |
| resolution  | EXCEPTIONAL CLEARANCE  | *```NULL``` where resolution is pending* |
| pd_district  | CENTRAL  |  |
| address  | 900 BLOCK OF BAY ST  | *Table also contains longitude and latitude* |

### Index of questions posed (self-driven) 

A) **2015 Crime Snapshot**
1. Number of incidents
2. Number of incidents, by district
3. Number of incidents per capita, by district
4. Top-3 incident categories
5. Top-3 incident categories, by district
6. Months with most/least incidents
7. Hours of the day with most/least incidents
8. Resolution analysis (% prosecuted, % booked or cited, % cleared, % unresolved)

B) **2012-15 Crime trends over time**
1. Number of incidents by year
2. Districts with the fastest growing / fastest declining number of incidents (% change)
3. Resolutions - clearance rate by year (defined as # cleared / # resolved)
4. Day of the year with most incidents, by year

### Solutions and findings

A) 2015 Crime Snapshot

1. Number of incidents

```sql
SELECT COUNT(*)  
FROM incidents  
WHERE strftime('%Y',datetime) = '2015';
```  
> 156224

2. Number of incidents, by district

```sql
SELECT pd_district,
  COUNT(*)  
FROM incidents  
WHERE strftime('%Y',datetime) = '2015'  
GROUP BY 1  
ORDER BY 2 DESC;
```
    
>SOUTHERN|30032  
NORTHERN|20055  
CENTRAL|18537  
MISSION|18515  
BAYVIEW|14672  
INGLESIDE|13390  
TARAVAL|11926  
TENDERLOIN|10706  
PARK|9318  
RICHMOND|9073

>:arrow_forward: 10 districts, with incident numbers varying from 9k (Richmond) to 30k (Southern)  
:arrow_forward: Top 4 districts made up 56% of 2015 incidents 

3. Number of incidents per capita, by district

```sql
/* Creating a new population by district table ('pop') */
CREATE TABLE pop (pd_district TEXT, pop INTEGER);

INSERT INTO pop (pd_district, pop)  
  VALUES ('SOUTHERN', 75368),      -- Mock-up population data; actual population by police district unavailable   
    ('NORTHERN', 75071),  
    ('CENTRAL', 89486),  
    ('MISSION', 97043),  
    ('BAYVIEW', 84585),  
    ('INGLESIDE', 116125),  
    ('TARAVAL', 115179),  
    ('TENDERLOIN', 120051),  
    ('PARK', 101965),  
    ('RICHMOND', 86543);
```

```sql
/* Joining the new table and analysing the results */ 
WITH table_join AS (  
  SELECT *  
  FROM incidents i  
  LEFT JOIN pop p  
    ON i.pd_district = p.pd_district),  
aggregate_stats AS (  
  SELECT pd_district AS 'district',  
    count(*) AS 'number_incidents',  
    AVG(pop) AS 'population'  
  FROM table_join  
  WHERE strftime('%Y',datetime) = '2015'  
  GROUP BY 1)  
SELECT district,  
  1.0 * number_incidents/population  
FROM aggregate_stats  
GROUP BY 1  
ORDER BY 2 DESC;
```

>SOUTHERN|0.398471499840781  
NORTHERN|0.267147100744628  
CENTRAL|0.207149721744183  
MISSION|0.190791710891048  
BAYVIEW|0.173458651061063  
INGLESIDE|0.115306781485468  
RICHMOND|0.10483805738188  
TARAVAL|0.103543180614522  
PARK|0.0913842985338106  
TENDERLOIN|0.0891787656912479  

>:arrow_forward: High variation in incidents per capita between districts in 2015, ranging from 0.09 to 0.40

4. Top 3 incident categories
```sql
SELECT category,  
  count(*)  
FROM incidents  
WHERE strftime('%Y',datetime) = '2015'  
GROUP BY 1  
ORDER BY 2 DESC  
LIMIT 3;
```
>LARCENY/THEFT|42034  
OTHER OFFENSES|20321  
NON-CRIMINAL|19127

>:arrow_forward: Larceny/theft the clear #1 incident category, but only forms c.25% of total; long tail of other categories

5. Top 3 incident categories, by district
```sql
/* Solution below relies on multiple temporary tables & uses of UNION/EXCEPT - possible to optimise? */
WITH aggregate_stats AS (  
  SELECT pd_district,  
    category,  
    count(*) AS 'cnt'  
  FROM incidents  
  WHERE strftime('%Y',datetime) = '2015'  
  GROUP BY 1, 2),  
district_category_largest1 AS (  
  SELECT pd_district,  
   category,  
    max(cnt)  
  FROM aggregate_stats  
  GROUP BY 1),  
aggregate_stats_2 AS (  
  SELECT *
  FROM aggregate_stats  
  EXCEPT  
  SELECT *
  FROM district_category_largest1),  
district_category_largest2 AS(  
  SELECT pd_district,  
   category,  
    max(cnt)  
  FROM aggregate_stats_2  
  GROUP BY 1),  
aggregate_stats_3 AS (  
  SELECT *
  FROM aggregate_stats_2  
  EXCEPT  
  SELECT *
  FROM district_category_largest2),  
district_category_largest3 AS(  
   SELECT pd_district,  
    category,  
    max(cnt)  
  FROM aggregate_stats_3  
  GROUP BY 1)  
SELECT *
FROM district_category_largest1  
UNION  
SELECT *
FROM district_category_largest2  
UNION  
SELECT *
FROM district_category_largest3  
ORDER BY 1, 3 DESC;
```
>BAYVIEW|OTHER OFFENSES|2659  
BAYVIEW|LARCENY/THEFT|2208  
BAYVIEW|ASSAULT|1647  
CENTRAL|LARCENY/THEFT|6912  
CENTRAL|NON-CRIMINAL|2472  
CENTRAL|OTHER OFFENSES|1666  
INGLESIDE|OTHER OFFENSES|2143  
INGLESIDE|LARCENY/THEFT|2004  
INGLESIDE|ASSAULT|1446  
MISSION|OTHER OFFENSES|2931  
MISSION|LARCENY/THEFT|2405  
MISSION|NON-CRIMINAL|2352  
NORTHERN|LARCENY/THEFT|7463  
NORTHERN|NON-CRIMINAL|2158  
NORTHERN|OTHER OFFENSES|2067  
PARK|LARCENY/THEFT|2377  
PARK|NON-CRIMINAL|1367  
PARK|OTHER OFFENSES|1132  
RICHMOND|LARCENY/THEFT|2977  
RICHMOND|NON-CRIMINAL|1349  
RICHMOND|OTHER OFFENSES|1110  
SOUTHERN|LARCENY/THEFT|10921  
SOUTHERN|NON-CRIMINAL|3518  
SOUTHERN|OTHER OFFENSES|3295  
TARAVAL|LARCENY/THEFT|2729  
TARAVAL|OTHER OFFENSES|1955  
TARAVAL|NON-CRIMINAL|1658  
TENDERLOIN|LARCENY/THEFT|2038  
TENDERLOIN|NON-CRIMINAL|1426  
TENDERLOIN|OTHER OFFENSES|1363  

>:arrow_forward: Across all districts, top incident category is either 'Larceny/theft' or 'Other offenses'  
:arrow_forward: 'Non-criminal' and 'Assault' are the only other top-3 categories 


6. Months with most/least incidents 
```sql
WITH month_count AS (  
  SELECT strftime ('%m', datetime) AS 'month',  
    count (*) AS 'cnt'  
  FROM incidents  
  WHERE strftime ('%Y', datetime) = '2015'  
  GROUP BY 1  
  ORDER BY 2 DESC)  
SELECT month,  
  max(cnt)  
FROM month_count  
UNION  
SELECT month,  
  min(cnt)  
FROM month_count;  
```
>03|13924  
12|11381

>:arrow_forward: Limited variation in number of incidents by month

7. Hours of the day with most/least incidents
```sql
WITH hour_count AS (  
  SELECT strftime ('%H', datetime) AS 'hour',  
    count (*) AS 'cnt'  
  FROM incidents  
  WHERE strftime ('%Y', datetime) = '2015'  
  GROUP BY 1  
  ORDER BY 2 DESC)  
SELECT hour,  
  max(cnt)  
FROM hour_count  
UNION  
SELECT hour,  
  min(cnt)  
FROM hour_count  
ORDER BY 2 DESC;
```
>18|10521  
05|1708

>:arrow_forward: 6-7pm sees the highest number of incidents (10.5k per year, or c.30/day); significantly higher than 5-6am (1.7k per year, or c.5/day)

8. Resolution breakdown (prosecuted, booked or cited, cleared, unresolved)

Take the set of resolution states
```sql
SELECT DISTINCT resolution
FROM incidents;
```

We map these as follows, into either **Prosecuted**, **Booked or cited**, **Cleared** or **Unresolved**

| Resolution | Type |
| ------------- | ------------- |
| UNFOUNDED  | Cleared  |
| EXCEPTIONAL CLEARANCE  | Cleared  |
| LOCATED  | Cleared  |
| NOT PROSECUTED  | Cleared  |
| ARREST, CITED  | Booked or cited  |
| ARREST, BOOKED  | Booked or cited  |
| JUVENILE BOOKED  | Booked or cited  |
| DISTRICT ATTORNEY REFUSES TO PROSECUTE  | Cleared  |
| PSYCOPATHIC CASE  | Booked or cited  |
| PROSECUTED BY OUTSIDE AGENCY  | Prosecuted  |
| JUVENILE CITED  | Booked or cited  |
| JUVENILE ADMONISHED  | Booked or cited  |
| CLEARED-CONTACT JUVENILE FOR MORE INFO  | Cleared  |
| JUVENILE DIVERTED  | Booked or cited  |
| PROSECUTED FOR LESSER OFFENSE  | Prosecuted  |
| ```NULL```  | Unresolved  |

We recall the total number of incidents from A1 as 156,224;
```sql
SELECT CASE 
    WHEN resolution = 'UNFOUNDED' THEN 'CLEARED'  
    WHEN resolution = 'EXCEPTIONAL CLEARANCE' THEN 'CLEARED'   
    WHEN resolution = 'COMPLAINANT REFUSES TO PROSECUTE' THEN 'CLEARED'  
    WHEN resolution = 'LOCATED' THEN 'CLEARED'  
    WHEN resolution = 'NOT PROSECUTED' THEN 'CLEARED'  
    WHEN resolution = 'ARREST, CITED' THEN 'BOOKED OR CITED'  
    WHEN resolution = 'ARREST, BOOKED' THEN 'BOOKED OR CITED'  
    WHEN resolution = 'JUVENILE BOOKED' THEN 'BOOKED OR CITED'  
    WHEN resolution = 'DISTRICT ATTORNEY REFUSES TO PROSECUTE' THEN 'CLEARED'  
    WHEN resolution = 'PSYCHOPATHIC CASE' THEN 'BOOKED OR CITED'  
    WHEN resolution = 'PROSECUTED BY OUTSIDE AGENCY' THEN 'PROSECUTED'  
    WHEN resolution = 'JUVENILE CITED' THEN 'BOOKED OR CITED'  
    WHEN resolution = 'JUVENILE ADMONISHED' THEN 'BOOKED OR CITED'  
    WHEN resolution = 'CLEARED-CONTACT JUVENILE FOR MORE INFO' THEN 'CLEARED'  
    WHEN resolution = 'JUVENILE DIVERTED' THEN 'BOOKED OR CITED'  
    WHEN resolution = 'PROSECUTED FOR LESSER OFFENSE' THEN 'PROSECUTED'  
    ELSE 'UNRESOLVED'  
    END AS 'status',  
  100.0 * COUNT(*) / 156224  
FROM incidents  
WHERE strftime('%Y',datetime) = '2015'  
GROUP BY 1  
ORDER BY 2 DESC;  
```
>UNRESOLVED|70.84%  
BOOKED OR CITED|26.87%  
CLEARED|2.28%

>:arrow_forward: Majority (71%) of 2015 cases unresolved  
:arrow_forward: Fewer than 10% of resolved cases result in clearance (>90% resulting in booking/citation)  
:arrow_forward: **No** 2015 cases have (yet?) resulted in a prosecution

Constrast the 2015 results to 2012:
```sql
SELECT CASE  
    WHEN resolution = 'UNFOUNDED' THEN 'CLEARED'  
    ... *(as before)*  
    WHEN resolution = 'PROSECUTED FOR LESSER OFFENSE' THEN 'PROSECUTED'  
    ELSE 'UNRESOLVED'  
    END AS 'status',  
  100.0 * COUNT(*) / 140855 -- total 2012 cases (140,855) is calculated later in B1  
FROM incidents  
WHERE strftime('%Y',datetime) = '2012'  
GROUP BY 1  
ORDER BY 2 DESC;  
```
>UNRESOLVED|63.88%  
BOOKED OR CITED|31.54%  
CLEARED|4.33%  
PROSECUTED|0.24%

>:arrow_forward: Fewer 2012 cases are unresolved (64%, vs. 71% in 2015) - possibly due to data time lag  
:arrow_forward: Higher clearance rate in 2012 (12%, vs. 8% in 2015)  
:arrow_forward: Some 2012 cases resulted in prosecutions, vs. none in 2015 - but still minor (0.24% of total) 



B) 2012-15 Crime trends over time

1. Number of incidents by year
```sql
SELECT strftime('%Y',datetime),  
  count(*)  
FROM incidents  
GROUP BY 1;
```
>2012|140855  
2013|152806  
2014|150143  
2015|156224

>:arrow_forward: Significant jump in incident numbers between 2012-13 (+8%); overall 2012-15 CAGR of +4%

2. Districts with the fastest growing / fastest declining number of incidents (% change)
```sql
WITH aggregate_stats AS (  
  SELECT pd_district,  
    SUM(CASE  
      WHEN strftime('%Y',datetime) = '2012' THEN 1  
      ELSE 0  
      END) AS 'incidents_2012',  
    SUM(CASE  
      WHEN strftime('%Y',datetime) = '2015' THEN 1  
      ELSE 0  
      END) AS 'incidents_2015'  
    FROM incidents  
    GROUP BY 1)  
SELECT pd_district,  
  1.0 * incidents_2015 / incidents_2012  
FROM aggregate_stats  
GROUP BY 1  
ORDER BY 2 DESC;
```
>CENTRAL|1.32095774246419  
NORTHERN|1.22728107214981  
RICHMOND|1.1772414687946  
SOUTHERN|1.15998455001931  
TARAVAL|1.13418925344746  
INGLESIDE|1.06957424714434  
PARK|1.06260691070818  
BAYVIEW|1.01151327128576  
MISSION|0.985574363888002  
TENDERLOIN|0.908057675996607  

>:arrow_forward: 8 districts have seen incident numbers rise (2012-15), vs. 2 with reductions  
:arrow_forward: Central has seen the sharpest increase in incident numbers, at +32% increase over 3 years

3. Resolutions - clearance rate by year (defined as # cleared / # resolved)

*Recall the approach taken in A8 - we take the same approach to classification*
```sql
WITH aggregate_stats AS (  
  SELECT strftime('%Y',datetime) AS 'year',  
    SUM(CASE  
      WHEN resolution = 'UNFOUNDED' THEN 1  
      WHEN resolution = 'EXCEPTIONAL CLEARANCE' THEN 1  
      WHEN resolution = 'COMPLAINANT REFUSES TO PROSECUTE' THEN 1  
      WHEN resolution = 'LOCATED' THEN 1  
      WHEN resolution = 'NOT PROSECUTED' THEN 1  
      WHEN resolution = 'DISTRICT ATTORNEY REFUSES TO PROSECUTE' THEN 1  
      WHEN resolution = 'CLEARED-CONTACT JUVENILE FOR MORE INFO' THEN 1  
      ELSE 0  
      END) AS 'cleared_count',  
    SUM(CASE  
      WHEN resolution IS NOT NULL THEN 1  
      ELSE 0  
      END) AS 'resolved_count'  
    FROM incidents  
    GROUP BY 1)  
  SELECT year,  
    1.0 * cleared_count / resolved_count  
  FROM aggregate_stats  
  GROUP BY 1  
  ORDER BY 1;  
```
>2012|0.120002358768723  
2013|0.156249499046184  
2014|0.108239095315024  
2015|0.0782251690524282

>:arrow_forward: Significant variation in clearance rate between years (from 16% in 2013, then decreasing to 8% in 2013); general downward trend

4. Day of the year with most/least incidents, by year
```sql
WITH date_count AS (  
  SELECT strftime('%d',datetime) AS 'day',  
    strftime('%m',datetime) AS 'month',  
    strftime('%Y',datetime) AS 'year',  
    count(*) AS 'cnt'  
  FROM incidents  
  GROUP BY 1, 2, 3)  
SELECT year,  
  day,  
  month,  
  max(cnt)  
FROM date_count  
GROUP BY 1
UNION
SELECT year,  
  day,  
  month,  
  min(cnt)  
FROM date_count  
GROUP BY 1
ORDER BY 1, 4 DESC;
```
>2012|01|10|547  
2012|25|12|217  
2013|01|01|627  
2013|25|12|152  
2014|11|10|521  
2014|25|12|257  
2015|28|06|598  
2015|19|12|254

>:arrow_forward: Across 2012-15, worst days saw c.500-600 incidents, vs. best days with c.200  
:arrow_forward: The day with fewest incidents tends to be Christmas Day (2012, 2013, 2014); day with most incidents varies year-on-year 


  

  



    





  


