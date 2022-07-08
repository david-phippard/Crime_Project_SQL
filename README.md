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
  VALUES ('southern', 75368),
    ('northern', 75071),
    ('central', 89486),
    ('mission', 97043),
    ('bayview', 84585),
    ('ingleside', 116125),
    ('taraval', 115179),
    ('tenderloin', 120051),
    ('park', 101965),
    ('richmond', 86543);



