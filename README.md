# taxi hack

## A: Preparing the Database

1. download and set up PostGIS database for taxi 
adapted from [https://github.com/toddwschneider/nyc-taxi-data](https://github.com/toddwschneider/nyc-taxi-data)

- create table
```sql
CREATE TABLE yellow_tripdata_2017 (
  id serial primary key,
  vendor_id varchar,
  tpep_pickup_datetime varchar,
  tpep_dropoff_datetime varchar,
  passenger_count varchar,
  trip_distance varchar,
  rate_code_id varchar,
  store_and_fwd_flag varchar,
  pickup_location_id varchar,
  dropoff_location_id varchar,
  payment_type varchar,
  fare_amount varchar,
  extra varchar,
  mta_tax varchar,
  tip_amount varchar,
  tolls_amount varchar,
  improvement_surcharge varchar,
  total_amount varchar
);

```

- copy csv to database
```
\copy yellow_tripdata_2017 (vendor_id,tpep_pickup_datetime,tpep_dropoff_datetime,passenger_count,trip_distance,rate_code_id,store_and_fwd_flag,pickup_location_id,dropoff_location_id,payment_type,fare_amount,extra,mta_tax,tip_amount,tolls_amount,improvement_surcharge,total_amount) FROM 'C:/Users/base/Desktop/taxi hack/data/2017_Yellow_Taxi_Trip_Data.csv' CSV HEADER;
```
- optimize database and add columns
```sql
VACUUM ANALYZE yellow_tripdata_2017;

ALTER TABLE yellow_tripdata_2017
ADD COLUMN tstamp timestamp;

UPDATE yellow_tripdata_2017 SET tstamp = to_timestamp(tpep_pickup_datetime, 'MM/DD/YYYY HH:MI:SS AM');
```
it takes a long time for a query, wait
![it takes a long time for a query](images/bigdata.png)

data queries adapted from [queries_2017.sql](https://github.com/toddwschneider/nyc-taxi-data/blob/master/analysis/2017_update/queries_2017.sql)

```sql
--get hour counts 
select extract(hour from tstamp) as hour , count(id) from yellow_tripdata_2017 group by hour
--get day of week counts
select extract(dow from tstamp) as doy , count(id) from yellow_tripdata_2017 group by doy
--get day of year counts
select extract(doy from tstamp) as doy , count(id) from yellow_tripdata_2017 group by doy
```
![amount of yellow taxi rides by hour, 2017](images/taxi_hour.png)
![amount of yellow taxi rides by day of week, 2017](images/taxi_dow.png)
![amount of yellow taxi rides by day of year, 2017](images/taxi_doy.png)

## B : Preparing the zones and routes

1. exporting to centroid to csv (qgis)
![qgis - exporting to csv](images/1.png)

2. build distanceMatrix for query (python) , filtering out only rows that go from or to congestion zones to reduce values from 34452 to 11661
![python - build distanceMatrix for query](images/2.png)

3. setting up google directions queries (python) , 4 time buckets 
- 8am Weekdays (6/6) - (morning rush-hours 6 - 10am)
- 2pm Weekdays (6/6) - (non-rush hours)
- 6pm Weekdays (6/6) - (night rush-hours 4 - 8pm)
- 2pm Weekends (6/9) - (non-rush hours)

![python - setting up google directions queries](images/3.png)


5. Running api queries and

Queries are done in chunks so it won't go over the 2500 limit (api resets at 3am) will take 4 days

Queries are joined back with distanceMatrix to get fromTo Ids
![results from queries](images/4.png)

4. ZERO_RESULTS Error

![errors](images/5.png)

some routes returned with ZERO_RESULTS, they are ignored because those zones are islands

![island](images/6.png)


5. Processing route/leg information and uploading it to the database

Convert from Encoded Polyline to WKT Line because PostGIS [ST_LineFromEncodedPolyline](https://postgis.net/docs/ST_LineFromEncodedPolyline.html) doesn't work so well. Using [ST_GeomFromText](http://postgis.net/docs/ST_GeomFromText.html)

![upload from csv to qgis then to postgis](images/7.png)

<!-- 
```sql
CREATE TABLE legs (
  id serial primary key,
  distance bigint,
  duration bigint,
  fromID integer,
  instructions varchar,
  polyline varchar,
  timeBucketId integer,
  toID integer,
  geom geometry
);

\copy legs (id,distance,duration,fromID,instructions,polyline,timeBucketId,toID) FROM 'C:/Users/base/Desktop/taxi hack/data/2017_Yellow_Taxi_Trip_Data.csv' CSV HEADER;

UPDATE legs SET geom = ST_GeomFromText(polyline,4269);

CREATE INDEX index_legs_on_geom ON legs USING geom;
VACUUM ANALYZE legs;
``` -->


## C: Preparing the database for joining

1. upload shapefile of the congestion zone using QGIS

![dissolved congestion zone](images/zone.png)

2. add column for which time bucket each trip belongs to

```sql
ALTER TABLE yellow_tripdata_2017
ADD COLUMN bucket integer;



```
3. creating a sql view for only taxi trips with in and out of zones

```sql
CREATE VIEW yellow_tripdata_2017_fromtozones AS 
  SELECT id, bucket, pickup_location_id, dropoff_location_id, trip_distance FROM yellow_tripdata_2017 
  WHERE 
    pickup_location_id in (4,12,13,43,45,48,50,68,79,87,88,90,100,107,113,114,125,137,140,141,142,143,144,148,158,161,162,163,164,170,186,209,211,224,229,230,231,232,233,234,236,237,238,239,246,249,261,262,263) 
    OR dropoff_location_id in (4,12,13,43,45,48,50,68,79,87,88,90,100,107,113,114,125,137,140,141,142,143,144,148,158,161,162,163,164,170,186,209,211,224,229,230,231,232,233,234,236,237,238,239,246,249,261,262,263) 

```


## D. Analysis

1. get counts group by bucket, pickup_location_id, dropoff_location_id

```sql

```

2. calcaute percentage in congestion zone

```sql
Using the SQL group by function and Sum will get the total dwell time and distance of each route
SELECT route_id, SUM(percent * leg_time) as dwell_time, SUM(percent * leg_distance) as dwell time FROM leg_table GROUP BY route_id
```

3. calcaute number crosses per zone

```sql
SELECT (count(st_intersection(st_boundary(congestion.geom),route_table.route_geom))).geom AS points FROM route_table, congestion WHERE st_intersects(congestion.geom,route_table.route_geom);
```