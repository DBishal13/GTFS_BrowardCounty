# Managing Public Transportation Data and Visualization for Broward County, Florida

[![Final Map](https://github.com/DBishal13/GTFS_BrowardCounty/blob/main/FinalMap.png)](https://github.com/DBishal13/GTFS_BrowardCounty/blob/main/FinalMap.png)

## 1. Introduction
Broward County, located in the southeastern part of the state of Florida, is known for its vibrant communities, diverse population, and rich cultural heritage. As one of the most populous counties in Florida, Broward County encompasses a wide range of urban, suburban, and coastal areas, including the city of Fort Lauderdale, which serves as the county seat. The county is strategically positioned along the Atlantic Coast, offering access to beautiful beaches, recreational activities, and a thriving tourism industry. Beyond its natural attractions, Broward County is also a major economic hub, with a robust business environment, diverse industries, and a well-developed infrastructure network.

Given its significant population density and economic activity, efficient public transportation systems are vital for Broward County to address mobility needs, reduce traffic congestion, and promote sustainable urban development. Analyzing transit data and optimizing transit networks are essential tasks for transit authorities and urban planners to enhance the quality and accessibility of public transportation services. By conducting spatial analysis and visualization of transit data, stakeholders in Broward County can gain valuable insights into transit patterns, demand hotspots, connectivity gaps, and service efficiency. These analyses can inform decision-making processes related to route planning, service optimization, infrastructure investments, and transit policy development. Efficient public transportation systems are essential for the smooth functioning of urban areas, providing residents with convenient and sustainable mobility options. Understanding the structure and dynamics of transit networks is crucial for urban planners, policymakers, and transit authorities to make informed decisions regarding infrastructure development, service optimization, and resource allocation. In this project, we focus on visualizing the Broward County Transit network in Fort Lauderdale, Florida, using PostgreSQL, PostGIS, and QGIS.

The primary objective of this project is to demonstrate the application of PostgreSQL, PostGIS, and QGIS in processing and visualizing transit data. We will utilize General Transit Feed Specification (GTFS) data obtained from Broward County Transit, which provides information about transit routes, stops, schedules, and geographic coordinates. The project will involve importing GTFS data into PostgreSQL, converting it into a spatial format using PostGIS, and visualizing the transit network in QGIS.

## 2. Data and Methods

### 2.1 General Transit Feed Specification (GTFS) Data
The General Transit Feed Specification (GTFS) is a widely used format for public transportation schedules and associated geographic information. Many transportation agencies around the world publish their data in GTFS format, making it accessible for analysis and visualization. Broward County Transit provides GTFS data containing information about transit routes, stops, schedules, and geographic coordinates. This GTFS data will serve as the basis for our analysis and visualization in this project. The dataset was downloaded from [Broward County Transit](https://www.broward.org/BCT/Pages/Default.aspx). The ZIP file with the data for this project contains a set of CSV files (with extension .txt) as follows:

- `agency.txt` contains the description of the transportation agencies providing the services (one in our case).
- `calendar.txt` contains service patterns that operate recurrently such as, for example, every weekday.
- `calendar_dates.txt` defines exceptions to the default service patterns defined in `calendar.txt`. There are two types of exceptions: 1 means that the service has been added for the specified date, and 2 means that the service has been removed for the specified date.
- `route_types.txt` contains transportation types used on routes, such as bus, metro, tramway, etc.
- `routes.txt` contains transit routes. A route is a group of trips that are displayed to riders as a single service.
- `shapes.txt` contains the vehicle travel paths, which are used to generate the corresponding geometry.
- `stop_times.txt` contains times at which a vehicle arrives at and departs from stops for each trip.
- `translations.txt` contains the translation of the route information in French and Dutch. This file is not used in this tutorial.
- `trips.txt` contains trips for each route. A trip is a sequence of two or more stops that occur during a specific time period.

### 2.2 Loading GTFS Data in PostgreSQL
PostgreSQL is a powerful open-source relational database management system known for its robustness, extensibility, and support for advanced data types and functions. A new database named project was created in pgAdmin 4, and PostGIS extension was added. The following tables were then created which were loaded with the data in the CSV files later.

```sql
CREATE TABLE agency (
  agency_id text DEFAULT '',
  agency_name text DEFAULT NULL,
  agency_url text DEFAULT NULL,
  agency_timezone text DEFAULT NULL,
  agency_lang text DEFAULT NULL,
  agency_phone text DEFAULT NULL,
  CONSTRAINT agency_pkey PRIMARY KEY (agency_id)
);

CREATE TABLE calendar (
  service_id text,
  monday int NOT NULL,
  tuesday int NOT NULL,
  wednesday int NOT NULL,
  thursday int NOT NULL,
  friday int NOT NULL,
  saturday int NOT NULL,
  sunday int NOT NULL,
  start_date date NOT NULL,
  end_date date NOT NULL,
  CONSTRAINT calendar_pkey PRIMARY KEY (service_id)
);
CREATE INDEX calendar_service_id ON calendar (service_id);

CREATE TABLE exception_types (
  exception_type int PRIMARY KEY,
  description text
);

CREATE TABLE calendar_dates (
  service_id text,
  date date NOT NULL,
  exception_type int REFERENCES exception_types(exception_type)
);
CREATE INDEX calendar_dates_dateidx ON calendar_dates (date);

CREATE TABLE route_types (
  route_type int PRIMARY KEY,
  description text
);

CREATE TABLE routes (
  route_id text,
  route_short_name text DEFAULT '',
  route_long_name text DEFAULT '',
  route_desc text DEFAULT '',
  route_type int REFERENCES route_types(route_type),
  route_url text,
  route_color text,
  route_text_color text,
  CONSTRAINT routes_pkey PRIMARY KEY (route_id)
);

CREATE TABLE shapes (
  shape_id text NOT NULL,
  shape_pt_lat double precision NOT NULL,
  shape_pt_lon double precision NOT NULL,
  shape_pt_sequence int NOT NULL
);
CREATE INDEX shapes_shape_key ON shapes (shape_id);

-- Create a table to store the shape geometries
CREATE TABLE shape_geoms (
  shape_id text NOT NULL,
  shape_geom geometry ('LINESTRING', 4326),
  CONSTRAINT shape_geom_pkey PRIMARY KEY (shape_id)
);
CREATE INDEX shape_geoms_key ON shapes (shape_id);

CREATE TABLE location_types (
  location_type int PRIMARY KEY,
  description text
);

CREATE TABLE stops (
  stop_id text,
  stop_code text,
  stop_name text DEFAULT NULL,
  stop_desc text DEFAULT NULL,
  stop_lat double precision,
  stop_lon double precision,
  zone_id text,
  stop_url text,
  location_type integer REFERENCES location_types(location_type),
  parent_station integer,
  stop_geom geometry ('POINT', 4326),
  platform_code text DEFAULT NULL,
  CONSTRAINT stops_pkey PRIMARY KEY (stop_id)
);

CREATE TABLE pickup_dropoff_types (
  type_id int PRIMARY KEY,
  description text
);

CREATE TABLE stop_times (
  trip_id text NOT NULL,
  -- Check that casting to time interval works.
  arrival_time interval CHECK (arrival_time::interval = arrival_time::interval),
  departure_time interval CHECK (departure_time::interval = departure_time::interval),
  stop_id text,
  stop_sequence int NOT NULL,
  pickup_type int REFERENCES pickup_dropoff_types(type_id),
  drop_off_type int REFERENCES pickup_dropoff_types(type_id),
  CONSTRAINT stop_times_pkey PRIMARY KEY (trip_id, stop_sequence)
);
CREATE INDEX stop_times_key ON stop_times (trip_id, stop_id);
CREATE INDEX arr_time_index ON stop_times (arrival_time);
CREATE INDEX dep_time_index ON stop_times (departure_time);

CREATE TABLE trips (
  route_id text NOT NULL,
  service_id text NOT NULL,
  trip_id text NOT NULL,
  trip_headsign text,
  direction_id int,
  block_id text,
  shape_id text,
  CONSTRAINT trips_pkey PRIMARY KEY (trip_id)
);
CREATE INDEX trips_trip_id ON trips (trip_id);

INSERT INTO exception_types (exception_type, description) VALUES
(1, 'service has been added'),
(2, 'service has been removed');

INSERT INTO location_types(location_type, description) VALUES
(0,'stop'),
(1,'station'),
(2,'station entrance');

INSERT INTO pickup_dropoff_types (type_id, description) VALUES
(0,'Regularly Scheduled'),
(1,'Not available'),
(2,'Phone arrangement only'),
(3,'Driver arrangement only');

In this script, several indexes were created on different tables to enhance query performance and optimize data retrieval. For example, in the `calendar` table, an index named `calendar_service_id` was created on the `service_id` column, which likely serves as a primary identifier for service schedules. This index speeds up queries involving filtering or joining based on `service_id`, improving the efficiency of operations related to service schedules. Similarly, indexes such as `calendar_dates_dateidx`, `shapes_shape_key`, `shape_geoms_key`, and `trips_trip_id` are created on relevant columns in their respective tables to facilitate faster data access and retrieval. Additionally, indexes like `arr_time_index` and `dep_time_index` were created on the `arrival_time` and `departure_time` columns in the `stop_times` table to optimize queries involving sorting or filtering based on these time attributes. These indexes contribute to better database performance by reducing query execution times and enhancing the overall efficiency of data operations.

## 2.3 Populating the tables with CSV files
I organized the data by creating individual tables for each CSV file. Additionally, I established a table called "shape_geoms" to consolidate all segments forming a route into a unified geometry. Moreover, I set up auxiliary tables named "exception_types," "location_types," and "pickup_dropoff_types" to accommodate acceptable values for certain columns found in the CSV files. To import the CSV files into their respective tables, the following steps can be taken. Because of permission denied error during the process, the \copy command within the psql shell was utilized as an alternative to the COPY command.

## 2.4 Create Geometries
I utilized PostGIS, an extension of PostgreSQL for handling spatial data, to create geometries for routes and stops. The `shape_geoms` table was populated with line string geometries representing routes. These geometries were generated using latitude and longitude coordinates stored in the `shapes` table. The `stops` table was updated with point geometries, each corresponding to a stop location. The `ST_MakeLine` function constructs linestring geometries by connecting points, while `ST_SetSRID` sets the spatial reference system for point geometries. This process ensures that spatial data is accurately represented and can be effectively queried and analyzed.

## 2.5 Visualization
I created two views: "stops_view" and "shape_geoms_view". These views are designed to be visualized in QGIS, a powerful open-source Geographic Information System (GIS) software. QGIS allows users to visualize, analyze, and interpret spatial data in various formats. By creating these views, we can easily import and display the data in QGIS, enabling us to explore further and analyze the geographic information the views represent.

## 3. Results
The final map showcasing the stops and routes is depicted in the figure below. This visualization offers crucial insights into the transportation network, enabling a comprehensive understanding of the infrastructure and its operations. Understanding this data is paramount for effective traffic management and urban planning. By analyzing the distribution of stops and routes, authorities can optimize public transportation systems, alleviate congestion, and enhance overall mobility within the city. Moreover, it empowers the public to plan their trips efficiently, choose the best routes, and make informed decisions about their daily commute.

Additionally, this project serves as a foundation for further analysis and enhancements. The tables provided a solid framework for conducting in-depth studies such as demand forecasting, service optimization, and route efficiency analysis. By leveraging this data, future projects can explore advanced analytics techniques to unlock valuable insights and further improve transportation systems.
