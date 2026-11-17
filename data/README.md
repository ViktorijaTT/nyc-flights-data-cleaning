# Data dictionary

## Folder contents

| Path | What it is |
|---|---|
| `raw/airlines.csv`, `raw/airports.csv`, `raw/planes.csv`, `raw/weather.csv` | Raw files, exactly as received |
| `raw/flights.zip` | Raw flights table (one file inside: `flights.csv`, 45 MB unzipped) |
| `processed/airlines_clean.csv`, `airports_clean.csv`, `planes_clean.csv`, `weather_clean.csv` | Cleaned tables |
| `processed/flights_clean.zip` | Cleaned flights table (one file inside: `flights_clean.csv`, 62 MB unzipped) |
| `processed/cleaning_log.csv` | The audit table: 47 issues, rows affected and the decision for each |

Source: `nycflights23` data package v0.2.0 (github.com/moderndive/nycflights23), licence CC0. It compiles
US DOT Bureau of Transportation Statistics on-time data, the FAA aircraft registry, airport metadata and
NOAA/ASOS hourly weather. See the project README for details.

## File format of the cleaned tables

- UTF-8, comma separated, one header row, Unix line endings (`\n`), text quoted with `"` only where needed.
- **Missing values are empty fields** (nothing between two commas). When loading into MySQL with
  `LOAD DATA`, convert empty fields to `NULL` explicitly (for example `SET col = NULLIF(@col, '')`), otherwise
  MySQL stores `0` or an empty string instead of `NULL`.
- Dates are `YYYY-MM-DD`, datetimes `YYYY-MM-DD HH:MM:SS`, times `HH:MM:SS`.
- Airport names contain accented characters, so the database should use a UTF-8 character set (`utf8mb4`).

The "MySQL type" column below is a suggestion that fits every value in the file.

## flights_clean (435,352 rows)

One row per scheduled flight departing JFK, LGA or EWR in 2023. Primary key: `flight_id`.

| Column | MySQL type | Missing allowed | Rows missing | Meaning |
|---|---|---|---|---|
| `flight_id` | INT UNSIGNED, primary key | No | 0 | Surrogate key, 1 to 435,352, in the order of the raw file |
| `flight_date` | DATE | No | 0 | Scheduled departure date (local New York time) |
| `carrier` | CHAR(2) | No | 0 | Airline code. Foreign key to `airlines.carrier` |
| `flight_number` | SMALLINT UNSIGNED | No | 0 | Flight number (1 to 1,972). Raw name: `flight` |
| `tailnum` | VARCHAR(6) | Yes | 1,913 | Aircraft registration. Missing only on cancelled flights. Links to `planes.tailnum`, but see "Relationships" |
| `origin` | CHAR(3) | No | 0 | Departure airport: EWR, JFK or LGA. Foreign key to `airports.faa` |
| `dest` | CHAR(3) | No | 0 | Destination airport (118 values). Foreign key to `airports.faa` |
| `flight_status` | VARCHAR(22) | No | 0 | `completed`, `cancelled` or `diverted_or_incomplete` (rules below) |
| `sched_dep_datetime` | DATETIME | No | 0 | Scheduled departure, local New York time |
| `actual_dep_datetime` | DATETIME | Yes | 10,738 | Actual departure, local New York time. Can fall on the day after `flight_date`. Missing for cancelled flights |
| `dep_delay_min` | SMALLINT | Yes | 10,738 | Departure delay in minutes (negative = early; range -50 to 1,813). Missing for cancelled flights |
| `sched_arr_time` | TIME | No | 0 | Scheduled arrival clock time, **local time at the destination**. No date (see note) |
| `actual_arr_time` | TIME | Yes | 11,453 | Actual arrival clock time, local time at the destination. Missing for cancelled flights and 715 diverted/incomplete flights |
| `arr_delay_min` | SMALLINT | Yes | 12,534 | Arrival delay in minutes (negative = early; range -97 to 1,812). Missing unless `flight_status = 'completed'` |
| `air_time_min` | SMALLINT UNSIGNED | Yes | 12,534 | Time in the air in minutes (18 to 701). Missing unless `flight_status = 'completed'` |
| `distance_miles` | SMALLINT UNSIGNED | No | 0 | Distance between origin and destination in miles (80 to 4,983) |
| `time_hour` | DATETIME | No | 0 | `sched_dep_datetime` rounded down to the hour. With `origin`, the join key to `weather` |

**flight_status rules**

| Value | Rule applied to the raw data | Rows |
|---|---|---|
| `cancelled` | `dep_time` is missing | 10,738 |
| `diverted_or_incomplete` | `dep_time` is present but `arr_delay` is missing | 1,796 |
| `completed` | departure and arrival delay are both recorded | 422,818 |

**Notes**

- Delays of cancelled and diverted flights are deliberately left missing. Do not treat them as 0.
- `actual_dep_datetime - sched_dep_datetime` equals `dep_delay_min` for every departed flight except two that
  were delayed overnight across a daylight-saving clock change (the difference is exactly one hour).
- Arrival columns are clock times only. They are in the destination's time zone and the raw data has no
  arrival date, so subtracting departure from arrival does not give the flight duration. Use `air_time_min`.
- Columns removed from the raw table: `year`, `month`, `day` (now `flight_date`), `dep_time`, `sched_dep_time`
  (now the two datetime columns), `arr_time` (now `actual_arr_time`), `hour`, `minute` (redundant).

## airlines_clean (14 rows)

Primary key: `carrier`.

| Column | MySQL type | Missing allowed | Rows missing | Meaning |
|---|---|---|---|---|
| `carrier` | CHAR(2), primary key | No | 0 | Two-character airline code |
| `airline_name` | VARCHAR(50) | No | 0 | Airline name (longest is 22 characters). Raw name: `name` |

## airports_clean (1,255 rows)

Primary key: `faa`. Only 121 of these airports are used by the flights table (3 origins and 118 destinations);
the rest are kept as a general lookup.

| Column | MySQL type | Missing allowed | Rows missing | Meaning |
|---|---|---|---|---|
| `faa` | CHAR(3), primary key | No | 0 | FAA airport code |
| `airport_name` | VARCHAR(100) | No | 0 | Airport name (longest is 60 characters; not unique; contains accented characters). Raw name: `name` |
| `lat` | DECIMAL(9,6) | No | 0 | Latitude in degrees |
| `lon` | DECIMAL(9,6) | No | 0 | Longitude in degrees (negative = west) |
| `alt_ft` | DECIMAL(6,1) | No | 0 | Altitude in feet (-115 to 9,070). Raw name: `alt` |
| `utc_offset_hours` | TINYINT | Yes | 50 | Standard-time offset from UTC in hours (-10 to 1). Raw name: `tz` |
| `dst` | CHAR(1) | Yes | 71 | Daylight-saving rule: `A` = US rules, `N` = no daylight saving. Unknown is missing |
| `tzone` | VARCHAR(30) | Yes | 119 | IANA time-zone name, e.g. `America/New_York` (longest is 19 characters) |

All missing time-zone values are on airports that no flight in this dataset uses.

## planes_clean (4,840 rows)

Primary key: `tailnum`. Source: FAA aircraft registry.

| Column | MySQL type | Missing allowed | Rows missing | Meaning |
|---|---|---|---|---|
| `tailnum` | VARCHAR(6), primary key | No | 0 | Aircraft registration |
| `year_manufactured` | SMALLINT UNSIGNED | Yes | 89 | Year the aircraft was built (1990 to 2023). Raw name: `year` |
| `aircraft_type` | VARCHAR(30) | No | 0 | `Fixed wing multi engine`, `Fixed wing single engine` or `Rotorcraft`. Raw name: `type` |
| `manufacturer` | VARCHAR(30) | No | 0 | Standardised manufacturer: Airbus, Airbus Canada, Boeing, Bombardier, Diamond, Embraer, Eurocopter |
| `model` | VARCHAR(20) | No | 0 | Model designation (longest is 15 characters) |
| `engines` | TINYINT UNSIGNED | No | 0 | Number of engines (1 to 3) |
| `seats` | SMALLINT UNSIGNED | No | 0 | Registry seat figure (4 to 563). **Not the installed cabin size**; do not use for passenger capacity |
| `engine_type` | VARCHAR(20) | No | 0 | `Turbo-fan`, `Turbo-jet`, `Turbo-shaft` or `Reciprocating`. Raw name: `engine` |

Known suspect records, left unchanged: `N135NK` (listed as a helicopter) and `N956JB` (listed as a four-seat
piston plane) each appear on one airline flight; `N852NW`, `N859NW`, `N860NW` (Airbus A330-223) are listed
with 3 engines. Column removed: `speed` (0 on every row).

## weather_clean (26,207 rows)

One row per airport per hour. Primary key: (`origin`, `time_hour`).

| Column | MySQL type | Missing allowed | Rows missing | Meaning |
|---|---|---|---|---|
| `origin` | CHAR(3), part of primary key | No | 0 | Airport: EWR, JFK or LGA. Foreign key to `airports.faa` |
| `time_hour` | DATETIME, part of primary key | No | 0 | Start of the hour the observation belongs to. Time zone unconfirmed (see note) |
| `temp_f` | DECIMAL(4,1) | Yes | 2 | Temperature in degrees Fahrenheit. Raw name: `temp` |
| `dewpoint_f` | DECIMAL(4,1) | Yes | 2 | Dew point in degrees Fahrenheit. Raw name: `dewp` |
| `humidity_pct` | DECIMAL(5,2) | Yes | 2 | Relative humidity in percent. Raw name: `humid` |
| `wind_dir_deg` | SMALLINT UNSIGNED | Yes | 1,225 | Wind direction in degrees (10 to 360). `0` means calm (no wind), not north. Raw name: `wind_dir` |
| `wind_speed_mph` | DECIMAL(5,2) | Yes | 1,041 | Wind speed in miles per hour. Raw name: `wind_speed` |
| `precip_in` | DECIMAL(6,4) | No | 0 | Precipitation in inches. `0.0001` is a code for a trace amount (1,715 rows). Raw name: `precip` |
| `pressure_mb` | DECIMAL(6,1) | Yes | 10 | Sea-level pressure in millibars. Raw name: `pressure` |
| `visibility_miles` | DECIMAL(4,2) | Yes | 23 | Visibility in miles, capped at 10. Raw name: `visib` |

**Notes**

- 73 airport-hours have no row: all of 31 December 2023 at the three airports (72) and 02 May 05:00 at LGA (1).
- The daily temperature cycle in this table peaks around 18:00 to 19:00, three to four hours later than expected
  for New York local time. The timestamps may be in UTC. This could not be confirmed from the files, so the
  values were not shifted. Confirm before using weather to explain individual flights.
- Columns removed: `year`, `month`, `day`, `hour` (redundant with `time_hour`) and `wind_gust` (not a real
  measurement: exactly `wind_speed` x 1.15078 on every row).

## Relationships

| From | To | Status in the cleaned data |
|---|---|---|
| `flights.carrier` | `airlines.carrier` | Valid for all rows. Safe as a foreign key |
| `flights.origin` | `airports.faa` | Valid for all rows. Safe as a foreign key |
| `flights.dest` | `airports.faa` | Valid for all rows. Safe as a foreign key |
| `weather.origin` | `airports.faa` | Valid for all rows. Safe as a foreign key |
| `flights.tailnum` | `planes.tailnum` | **Exception.** 9,371 flights (277 aircraft) have a tail number that is not in planes, and 1,913 have no tail number. A strict foreign key would fail; index the column and use `LEFT JOIN` |
| `flights (origin, time_hour)` | `weather (origin, time_hour)` | **Exception.** 826 flights have no weather row (823 on 31 December, 3 at LGA on 2 May). Use `LEFT JOIN` |

## cleaning_log.csv

| Column | Meaning |
|---|---|
| `issue_id` | Running number, 1 to 47 |
| `table` | Raw table the issue was found in |
| `column` | Raw column name(s) affected |
| `issue` | What was found |
| `rows_affected` | Number of rows affected (0 = a check that found nothing) |
| `decision` | What was done and, briefly, why |
