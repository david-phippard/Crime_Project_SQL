# Analysis of San Francisco Police Department Incident data
*08/07/2022*

### Schema overview ('incident' table) 
- **datetime** // *e.g. 2014-05-01 00:01:00; runs from 2012-01-01 to 2015-12-31*
- **category** // *e.g. 'Larceny/Theft', 'Fraud'*
- **description** // *e.g. 'Petty theft of property'* 
- **resolution** // *e.g. 'Exceptional clearance'; null where resolution pending (xx% of cases)*
- **pd_district** // *e.g. 'Central', 'Taraval'*
- **address** // *e.g. '900 block of Bay St' - table also contains address longitude & latitude* 

### Index of questions posed (self-driven) 

A) 2015 Crime Snapshot
1. Number of incidents
2. Number of incidents, by district
3. Number of incidents per capita, by district
4. Top-3 incident categories
5. Top-3 incident categories, by district
6. Months with most/least incidents
7. Days of the week with most/least incidents
8. Hours of the day with most/least incidents
9. Resolution analysis (% prosecuted, % booked or cited, % cleared, % unresolved)

B) 2012-15 Crime trends over time
1. Number of incidents by year
2. Districts with the fastest growing / fastest declining number of incidents (% change)
3. Resolutions - clearance rate by year (defined as # cleared / # resolved)
4. Day of the year with most incidents, by year

C) Repeat incidents
1. *see analysis for definition*

### Solutions and findings

A) 2015 Crime Snapshot

1. Number of incidents

SELECT COUNT(*)
  FROM incidents
  WHERE strftime('%Y',datetime) = '2015';
  
  Total: 156,224

2. Number of incidents, by district

SELECT pd_district, count(*)
  FROM incidents
  WHERE strftime('%Y',datetime) = '2015'
  GROUP BY 1
  ORDER BY 2 DESC;
    
SOUTHERN|30032
NORTHERN|20055
CENTRAL|18537
MISSION|18515
BAYVIEW|14672
INGLESIDE|13390
TARAVAL|11926
TENDERLOIN|10706
PARK|9318
RICHMOND|9073

3. Number of incidents per capita, by district

*Requires creation of a new 'district_pop' table (NB - using mock-up data)*

CREATE TABLE pd_district_pop (pd_district TEXT, pop INTEGER);

INSERT INTO pd_district_pop (pd_district, pop)
  VALUES ('SOUTHERN', 75368),
    ('NORTHERN', 75071),
    ('CENTRAL', 89486),
    ('MISSION', 97043),
    ('BAYVIEW', 84585),
    ('INGLESIDE', 116125),
    ('TARAVAL', 115179),
    ('TENDERLOIN', 120051),
    ('PARK', 101965),
    ('RICHMOND', 86543);
    
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

SOUTHERN|0.398471499840781
NORTHERN|0.267147100744628
CENTRAL|0.207149721744183
MISSION|0.190791710891048
BAYVIEW|0.173458651061063
INGLESIDE|0.115306781485468
RICHMOND|0.10483805738188
TARAVAL|0.103543180614522
PARK|0.0913842985338106
TENDERLOIN|0.0891787656912479

4. Top 3 incident categories

SELECT category,
  count(*)
FROM incidents
WHERE strftime('%Y',datetime) = '2015'
GROUP BY 1
ORDER BY 2 DESC
LIMIT 3;

LARCENY/THEFT|42034
OTHER OFFENSES|20321
NON-CRIMINAL|19127

5. Top 3 incident categories, by district

WITH aggregate_stats AS (
  SELECT pd_district,
    category,
    count(*) AS 'cnt'
  FROM incidents
  WHERE strftime('%Y',datetime) = '2015'
  GROUP BY 1,2),
district_category_largest1 AS (
  SELECT pd_district,
   category,
    max(cnt)
  FROM aggregate_stats
  GROUP BY 1),
aggregate_stats_2 AS (
  SELECT * FROM aggregate_stats
  EXCEPT
  SELECT * FROM district_category_largest1),
district_category_largest2 AS(
  SELECT pd_district,
   category,
    max(cnt)
  FROM aggregate_stats_2
  GROUP BY 1),
aggregate_stats_3 AS (
  SELECT * FROM aggregate_stats_2
  EXCEPT
  SELECT * FROM district_category_largest2),
district_category_largest3 AS(
   SELECT pd_district,
    category,
    max(cnt)
  FROM aggregate_stats_3
  GROUP BY 1)
SELECT * FROM district_category_largest1
UNION
SELECT * FROM district_category_largest2
UNION 
SELECT * FROM district_category_largest3
ORDER BY 1, 3 DESC;

BAYVIEW|OTHER OFFENSES|2659
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

6. Months with most/least incidents 

SELECT strftime ('%m', datetime),
  count (*)
FROM incidents
WHERE strftime ('%Y', datetime) = '2015'
GROUP BY 1
ORDER BY 2 DESC;

03|13924
05|13711
08|13672
01|13594
07|13342
06|13287
10|13115
04|12950
09|12871
02|12320
11|12057
12|11381

  


