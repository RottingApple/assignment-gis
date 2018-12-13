# Overview

This application shows hotels in Bratislava on a map. Most important features are:
- search by proximity to my current location
- search by hotel name
- intelligent ordering - by proximity and by hotel features
- hotels on the map are color coded by their quality assigned in stars (standard)

This application shows Bicycle Tracks in central Slovakia on map. Most important features are:
- Place Marker on a Map and search nearest tracks
- Search Tracks which are near selected amenities
- Show only tracks which can be completed in selected time
- Show nature reserves areas
- Reset map to original state
- Each track has color atribute class which represents ratio of intersection of length in nature reserve to whole length, blue means almost 0 ratio and red means 80% and more
This is it in action:
- Initial State

![Screenshot](Screenshot_2.png)

- Showing some Tracks

![Screenshot](Screenshot_3.png)
The application has 2 separate parts, the client which is a [frontend web application](#frontend) using leaflet API and landing.js and [Bootstrap framework](https://getbootstrap.com). [Backend application](#backend) written in [node.js](https://nodejs.org/en/) with [Express](https://expressjs.com/) framework, backed by PostGIS and Postgresql. The frontend application communicates with backend using a [REST API](#api).

# Frontend

The frontend application is a static HTML page (`landing.ejs`), which shows a leaflet.js widget. It is displaying bicycle tracks. Each track has color class based on the ratio of its length in nature reserve.

All relevant javascript code is in `landing.js` which is referenced from `landing.ejs`. The frontend code is very simple, its only responsibilities are:
- displaying the leaflet map which shows relevant bicycle tracks
- displaying the sidebar panel user can interact with if he wishes to use features described above like Search for tracks which are near selected amenities, or display nature reserves
- Make Requests on Server to get relevant tracks
- Display Tracks on the map which come from server response

# Backend

The backend application is written in node.js and is responsible for querying geo data, formatting the geojson and data for the map.

## Data

Tracks data is coming directly from [Open Street Maps](https://www.openstreetmap.org/#map=8/48.674/19.709). I downloaded an extent covering middle part of Slovakia and imported it using the `osm2pgsql` tool into the standard OSM schema in WGS 84 with hstore enabled. To speedup the queries I created multiple indexes on geometry columns (amenity, route, ST_Length(ST_Transform(way, 4326)::geography)) in tables point and line. The route methods are inside `routes/index.js` which responds to the requests sent by (#frontend). In order to speed up queries some additional collumns were created so they dont need to be calculated each time the request is send (mainly column crossing_area).

## Api

**Find tracks in proximity to marker**
Query:
```
SELECT ST_AsGeoJSON(ST_Transform(routes.way,4326)) as lines, routes.name, routes.crossing_area, ST_distance(ST_Transform(routes.way,3857),ST_Transform(ST_SetSRID(ST_Point(19.175380, 48.744962),4326), 3857)) as distance
FROM planet_osm_line as routes
where(routes.route='bicycle' or routes.bicycle='yes')
and ST_Intersects( ST_Transform(way,3857), ST_Buffer(ST_Transform(ST_SetSRID(ST_Point(19.175380, 48.744962),4326), 3857), 5000))
ORDER BY distance asc 
Limit 10;
```
Explain:
```
- Plan: 
    Node Type: "Limit"
    Startup Cost: 23425.25
    Total Cost: 23425.27
    Plan Rows: 10
    Plan Width: 412
    Plans: 
      - Node Type: "Sort"
        Parent Relationship: "Outer"
        Startup Cost: 23425.25
        Total Cost: 23425.29
        Plan Rows: 17
        Plan Width: 412
        Sort Key: 
          - "(st_distance(st_transform(way, 3857), '0101000020110F00008075C8C420494041DC55BA43A4C55741'::geometry))"
        Plans: 
          - Node Type: "Seq Scan"
            Parent Relationship: "Outer"
            Relation Name: "planet_osm_line"
            Alias: "routes"
            Startup Cost: 0.00
            Total Cost: 23424.90
            Plan Rows: 17
            Plan Width: 412
            Filter: "(((route = 'bicycle'::text) OR (bicycle = 'yes'::text)) AND (st_transform(way, 3857) && '0103000020110F000001000000210000008075C8C4E4524041DC55BA43A4C55741DDCC5CBBB4524041E427D366B0C45741CD10AF7726524041598E07E9C5C3574117BD55713F514041AFD840CDEDC25741147D1F890850404112D28E6130C25741DA6FBBB18D4E404110B273ED94C1574187042E7ADD4C40413508476A21C157416FD1967E084B40412D2A7048DAC057418075C8C420494041DC55BA43C2C057419119FA0A394740412D2A7048DAC0574179E6620F644540413508476A21C15741267BD5D7B343404110B273ED94C15741EC6D71003942404112D28E6130C25741E92D3B1802414041AFD840CDEDC2574133DAE1111B404041598E07E9C5C35741231E34CE8C3F4041E427D366B0C457418075C8C45C3F4041DC55BA43A4C55741231E34CE8C3F4041D483A12098C6574133DAE1111B4040415F1D6D9E82C75741E92D3B180241404109D333BA5AC85741EC6D710039424041A6D9E52518C95741267BD5D7B3434041A8F9009AB3C9574179E6620F6445404183A32D1D27CA57419119FA0A394740418B81043F6ECA57418075C8C420494041DC55BA4386CA57416FD1967E084B40418B81043F6ECA574187042E7ADD4C404183A32D1D27CA5741DA6FBBB18D4E4041A8F9009AB3C95741147D1F8908504041A6D9E52518C9574117BD55713F51404109D333BA5AC85741CD10AF77265240415F1D6D9E82C75741DDCC5CBBB4524041D483A12098C657418075C8C4E4524041DC55BA43A4C55741'::geometry) AND _st_intersects(st_transform(way, 3857), '0103000020110F000001000000210000008075C8C4E4524041DC55BA43A4C55741DDCC5CBBB4524041E427D366B0C45741CD10AF7726524041598E07E9C5C3574117BD55713F514041AFD840CDEDC25741147D1F890850404112D28E6130C25741DA6FBBB18D4E404110B273ED94C1574187042E7ADD4C40413508476A21C157416FD1967E084B40412D2A7048DAC057418075C8C420494041DC55BA43C2C057419119FA0A394740412D2A7048DAC0574179E6620F644540413508476A21C15741267BD5D7B343404110B273ED94C15741EC6D71003942404112D28E6130C25741E92D3B1802414041AFD840CDEDC2574133DAE1111B404041598E07E9C5C35741231E34CE8C3F4041E427D366B0C457418075C8C45C3F4041DC55BA43A4C55741231E34CE8C3F4041D483A12098C6574133DAE1111B4040415F1D6D9E82C75741E92D3B180241404109D333BA5AC85741EC6D710039424041A6D9E52518C95741267BD5D7B3434041A8F9009AB3C9574179E6620F6445404183A32D1D27CA57419119FA0A394740418B81043F6ECA57418075C8C420494041DC55BA4386CA57416FD1967E084B40418B81043F6ECA574187042E7ADD4C404183A32D1D27CA5741DA6FBBB18D4E4041A8F9009AB3C95741147D1F8908504041A6D9E52518C9574117BD55713F51404109D333BA5AC85741CD10AF77265240415F1D6D9E82C75741DDCC5CBBB4524041D483A12098C657418075C8C4E4524041DC55BA43A4C55741'::geometry))"
```
`POST /clicked_nearest`

In body part of request is the information about the minimal distance, maximum number of requested tracks, marker coordinates.

**Find tracks which have selected amenities in area**
Query:
```
explain (format YAML)
with cesta as (
	SELECT ST_Transform(way,3857) as way, crossing_area as crossing_area
	from planet_osm_line
	where (route='bicycle' or bicycle='yes')
	)
SELECT distinct ST_AsGeoJSON(ST_Transform(p.way,4326)) as point_data, ST_AsGeoJSON(ST_Transform(cesta.way,4326)) as track_data, p.amenity, cesta.crossing_area
from cesta
cross join (Select p.amenity, p.way
			from planet_osm_point p
where (p.amenity = 'drinking_water' or p.amenity = 'hospital')) as p
where ST_Dwithin(ST_Transform(p.way,3857),cesta.way, 500)
Limit 1000;
```
Explain:
```
- Plan: 
    Node Type: "Limit"
    Startup Cost: 15138.20
    Total Cost: 15143.22
    Plan Rows: 1
    Plan Width: 81
    Plans: 
      - Node Type: "Seq Scan"
        Parent Relationship: "InitPlan"
        Subplan Name: "CTE cesta"
        Relation Name: "planet_osm_line"
        Alias: "planet_osm_line"
        Startup Cost: 0.00
        Total Cost: 6157.97
        Plan Rows: 253
        Plan Width: 398
        Filter: "((route = 'bicycle'::text) OR (bicycle = 'yes'::text))"
      - Node Type: "Aggregate"
        Strategy: "Hashed"
        Parent Relationship: "Outer"
        Startup Cost: 8980.24
        Total Cost: 8985.25
        Plan Rows: 1
        Plan Width: 81
        Group Key: 
          - "st_asgeojson(st_transform(p.way, 4326), 15, 0)"
          - "st_asgeojson(st_transform(cesta.way, 4326), 15, 0)"
          - "p.amenity"
          - "cesta.crossing_area"
        Plans: 
          - Node Type: "Nested Loop"
            Parent Relationship: "Outer"
            Join Type: "Inner"
            Startup Cost: 9.56
            Total Cost: 8980.23
            Plan Rows: 1
            Plan Width: 81
            Join Filter: "((st_transform(p.way, 3857) && st_expand(cesta.way, '500'::double precision)) AND (cesta.way && st_expand(st_transform(p.way, 3857), '500'::double precision)) AND _st_dwithin(st_transform(p.way, 3857), cesta.way, '500'::double precision))"
            Plans: 
              - Node Type: "CTE Scan"
                Parent Relationship: "Outer"
                CTE Name: "cesta"
                Alias: "cesta"
                Startup Cost: 0.00
                Total Cost: 5.06
                Plan Rows: 253
                Plan Width: 40
              - Node Type: "Materialize"
                Parent Relationship: "Inner"
                Startup Cost: 9.56
                Total Cost: 327.99
                Plan Rows: 122
                Plan Width: 41
                Plans: 
                  - Node Type: "Bitmap Heap Scan"
                    Parent Relationship: "Outer"
                    Relation Name: "planet_osm_point"
                    Alias: "p"
                    Startup Cost: 9.56
                    Total Cost: 327.38
                    Plan Rows: 122
                    Plan Width: 41
                    Recheck Cond: "((amenity = 'drinking_water'::text) OR (amenity = 'hospital'::text))"
                    Plans: 
                      - Node Type: "BitmapOr"
                        Parent Relationship: "Outer"
                        Startup Cost: 9.56
                        Total Cost: 9.56
                        Plan Rows: 122
                        Plan Width: 0
                        Plans: 
                          - Node Type: "Bitmap Index Scan"
                            Parent Relationship: "Member"
                            Index Name: "index_point_amenity"
                            Startup Cost: 0.00
                            Total Cost: 4.75
                            Plan Rows: 61
                            Plan Width: 0
                            Index Cond: "(amenity = 'drinking_water'::text)"
                          - Node Type: "Bitmap Index Scan"
                            Parent Relationship: "Member"
                            Index Name: "index_point_amenity"
                            Startup Cost: 0.00
                            Total Cost: 4.75
                            Plan Rows: 61
                            Plan Width: 0
                            Index Cond: "(amenity = 'hospital'::text)"
```
`POST /nearest_amenities`

In body part of request is the information about the maximum allowed distance between selected amenity and track and selected amenities.

**Find tracks which have best length for user**
Query:
```
explain (verbose true, format YAML)
SELECT ST_AsGeoJSON(ST_Transform(roads.way,4326)) as track_data, roads.name, ST_Length(ST_Transform(roads.way, 4326)::geography)/1000 AS roads_length, roads.crossing_area
from planet_osm_line as roads
where(roads.route='bicycle' or roads.bicycle='yes')  and (ST_Length(ST_Transform(roads.way, 4326)::geography) < 10000)
Order by roads_length desc 
Limit 10;
```
Explain:
```
- Plan: 
    Node Type: "Limit"
    Startup Cost: 23437.69
    Total Cost: 23437.71
    Plan Rows: 10
    Plan Width: 412
    Output: 
      - "(st_asgeojson(st_transform(way, 4326), 15, 0))"
      - "name"
      - "((st_length((st_transform(way, 4326))::geography, true) / '1000'::double precision))"
      - "crossing_area"
    Plans: 
      - Node Type: "Sort"
        Parent Relationship: "Outer"
        Startup Cost: 23437.69
        Total Cost: 23438.31
        Plan Rows: 248
        Plan Width: 412
        Output: 
          - "(st_asgeojson(st_transform(way, 4326), 15, 0))"
          - "name"
          - "((st_length((st_transform(way, 4326))::geography, true) / '1000'::double precision))"
          - "crossing_area"
        Sort Key: 
          - "((st_length((st_transform(roads.way, 4326))::geography, true) / '1000'::double precision)) DESC"
        Plans: 
          - Node Type: "Index Scan"
            Parent Relationship: "Outer"
            Scan Direction: "Forward"
            Index Name: "index_line_length"
            Relation Name: "planet_osm_line"
            Schema: "public"
            Alias: "roads"
            Startup Cost: 0.29
            Total Cost: 23432.33
            Plan Rows: 248
            Plan Width: 412
            Output: 
              - "st_asgeojson(st_transform(way, 4326), 15, 0)"
              - "name"
              - "(st_length((st_transform(way, 4326))::geography, true) / '1000'::double precision)"
              - "crossing_area"
            Index Cond: "(st_length((st_transform(roads.way, 4326))::geography, true) < '10000'::double precision)"
            Filter: "((roads.route = 'bicycle'::text) OR (roads.bicycle = 'yes'::text))"
```
`POST /clicked_length`

In body part of request is the information about the maximal length of the track and the number of desired tracks.

***Show nature reserves***
Query:
```
SELECT ST_AsGeoJSON(ST_Transform(parks.way,4326)) as park_data, parks.name as park_name, st_area(parks.way) as park_zone  
from planet_osm_polygon as parks where parks.boundary='national_park'
```
Explain:
```
- Plan: 
    Node Type: "Seq Scan"
    Relation Name: "planet_osm_polygon"
    Alias: "parks"
    Startup Cost: 0.00
    Total Cost: 11836.72
    Plan Rows: 19
    Plan Width: 221
    Filter: "(boundary = 'national_park'::text)"
```
`GET /clicked_zones`

## Response

API calls return json responses in a array. One record consists of , `tracks` and in query 2 `amenity points`. `Tracks` contains an information about its coordinates and crossing_area. Track attributes are:
```
{
  "track": geojson coordinates,
  "crossing_area": float number, # like 70%
}
```
## Indexes

Before adding indexes I searched for the cost of each query.

***Initial Costs***

- Query1: 23,425.27
- Query2: 15,143.22	
- Query3: 23,437.71
- Query4: 11,836.72

***Adding index on route type***

- Query1: 21,717
- Query2: 15,026.47	
- Query3: 21,913
- Query4: 11,836.72

***Adding index on points amenity***

- Query1: 21,717
- Query2: 13,703.66	
- Query3: 21,913
- Query4: 11,836.72

***Adding index on route length***

- Query1: 21,717
- Query2: 13,703.66	
- Query3: 10,179.19 
- Query4: 11,836.72

***Adding index on polygon boundary***

- Query1: 21,717
- Query2: 13,703.66	
- Query3: 10,179.19 
- Query4: 81.91
