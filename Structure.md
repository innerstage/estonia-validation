## Statistics Estonia Website and API

The Statistics Estonia website offers a platform to interact with historical Estonian data, the one we're focusing on is under ECONOMY -> Foreign trade -> Foreign trade in goods. The website is: http://andmebaas.stat.ee/

The data in the website is available for querying and downloading through the Statistics Estonia API, instructions for the API can be found here: https://www.stat.ee/public/andmebaas/API-instructions.pdf

In the Estonia Validation notebook, testing shows that the information offered by the API is exactly the same as the one in the front end of the website.

## Querying/Downloading a Month of information

There are several cubes of information available in the SE API, we only use three:
* **VK10_1**: Storing trade data from 2004 to 2011.
* **VK10_2**: Storing trade data from 2012 to 2017.
* **VK10_3**: Storing data from 2018 and on.

In order to query only the metadata of the first cube we can do: http://andmebaas.stat.ee/sdmx-json/metadata/VK10_1, the response has this structure:

```
+-- RESPONSE
|   +-- header
|   +-- structure
|       +-- links
|       +-- name
|       +-- description
|       +-- dimensions
|           +-- observation
|               +-- observation
|                   +-- observation[0]["values"][0] <- Indicator values are stored in this list.
|                       +-- {"id" : "TRD_VAL", "name" : "Commodity value"}
|                       +-- {"id" : "NET_UNIT", "name" : "Commodity net weight"}
|                       +-- {"id" : "SUPPL_UNIT", "name" : "Commodity quantity in supplementary units"}
|                   +-- observation[0]["values"][1] <- Flow values
|                   +-- observation[0]["values"][2] <- Product values
|                   +-- observation[0]["values"][2] <- Country values
|                   +-- observation[0]["values"][3] <- Frequency values
|                   +-- observation[0]["values"][2] <- Reference Period values
|       +-- attributes
|       +-- annotations
```

Basically, a request for data to the API has the following structure:
`"http://andmebaas.stat.ee/sdmx-json/data/<Cube>/<Indicator>.<Flow>.<Product>.<Country>.<Frequency>/all?startTime=<Start>&endTime=<End>&dimensionAtObservation=allDimensions"`

If any of the tags is blank, the API will automatically return all the values for that tag, except for "<Cube>". A query for September 2012 would look like this:
`"http://andmebaas.stat.ee/sdmx-json/data/VK10_2/TRD_VAL....M/all?startTime=2012-09&endTime=2012-09&dimensionAtObservation=allDimensions"` and it would have the following structure:

```
+-- RESPONSE
|   +-- header
|   +-- dataSets
|       +-- dataSets[0]["observations"] <- This dictionary contains the dataset for this month.
|           +-- '0:0:0:0:0:0': [1122001797.0, 0, None, 0, 0, None]
|           +-- '0:0:1:0:0:0': [36426892.0, 0, None, 0, 0, None]
|           +-- '0:0:2:0:0:0': [3728110.0, 0, None, 0, 0, None]
|           [...]
|   +-- structure
```

The "structure" part of the cube is the same as the one in the metadata. Each observation's key has six integers corresponding to each dimension, and the first value of the list shows the total export or import in euros (EUR).

## Cleaned Dataset

If we split each number and store it in a DataFrame we will get something like this:

| indicator_id | flow_id | product_id | country_id | frequency_id | total      |
|--------------|---------|------------|------------|--------------|------------|
| 0            | 0       | 0          | 0          | 0            | 1122001797 |
| 0            | 0       | 1          | 0          | 0            | 36426892   |
| 0            | 0       | 2          | 0          | 0            | 3728110    |
| 0            | 0       | 3          | 0          | 0            | 975885     |
| 0            | 0       | 4          | 0          | 0            | 975885     |
| 0            | 0       | 5          | 0          | 0            | 181338     |
| 0            | 0       | 6          | 0          | 0            | 229496     |
| 0            | 0       | 7          | 0          | 0            | 134908     |
| 0            | 0       | 8          | 0          | 0            | 314235     |
| 0            | 0       | 9          | 0          | 0            | 115908     |

Generating dictionaries from the "structure" information, we can map this numeric values to the corresponding values:

| indicator_id | flow_id | product_id | country_id | frequency_id | total      |
|--------------|---------|------------|------------|--------------|------------|
| TRD_VAL      | EXP     | TOTAL      | TOTAL      | M            | 1122001797 |
| TRD_VAL      | EXP     | CNI        | TOTAL      | M            | 36426892   |
| TRD_VAL      | EXP     | CN01       | TOTAL      | M            | 3728110    |
| TRD_VAL      | EXP     | CN0102     | TOTAL      | M            | 975885     |
| TRD_VAL      | EXP     | CN010229   | TOTAL      | M            | 975885     |
| TRD_VAL      | EXP     | CN01022910 | TOTAL      | M            | 181338     |
| TRD_VAL      | EXP     | CN01022941 | TOTAL      | M            | 229496     |
| TRD_VAL      | EXP     | CN01022949 | TOTAL      | M            | 134908     |
| TRD_VAL      | EXP     | CN01022961 | TOTAL      | M            | 314235     |
| TRD_VAL      | EXP     | CN01022991 | TOTAL      | M            | 115908     |

## Issues and Details

* As shown in the validation notebook, the deepest level (HS8) can be rolled up to HS6 and HS4, but there's a gap between HS4 and HS2... although HS2 does roll up to the Chapter. This gaps starts appearing from 2004-04 on.
* The current database is built with HS6 as the deepest level (Per request of the client), and each level is rolled up from it.
* Each one of these monthly files is updated when Statistics Estonia conducts reviews. Each time a new month information is added to their database, they review the data up to two years back.
* A specific month's information is added to the database on the 10th after two months have passed (March should be the last month added on May 10th)
* The client has asked to have matching values between their database totals and the visualisation. It's very difficult to achieve this due to the nature of the data and the reviews.

## Current Design of Tables

In the fact table, all products are filtered and only the ones in the HS6 level are kept, in addition to the other transformations performed on the dataframe, the result looks like this:

| date_id | flow_id | product_id | geography_id | total  |
|---------|---------|------------|--------------|--------|
| 121     | 1       | 1          | 0            | 975885 |

* A *date_id* is created in the ETL to keep track of the time dimension, the index 121 would point to "2012-09" in the date table.
* The *flow_id* points to "EXP".
* The *product_id* points to the HS6 code "010229".
* The *geography_id* points to "TOTAL", but these values are currently dropped from the database, in order to keep only trades pointing to one specific country. We'll keep the value here to illustrate the products behavior rather than the countries one.

The products dimension table has this index, connects the levels and shows the names in English and Estonian, in long and short versions, to improve the visualisations in the Front End. A short version of the table would look like this:

| product_id | hs6_id | hs6_name                                   | hs4_id | hs4_name            | hs2_id | hs2_name                      | chapter_id | chapter_name    |
|------------|--------|--------------------------------------------|--------|---------------------|--------|-------------------------------|------------|-----------------|
| 1          | 010229 | Live cattle (excl. pure-bred for breeding) | 0102   | Live bovine animals | 01     | Live animals; animal products | 01         | Animal products |

## Proposed Design

A possible solution for this issue, would be to keep both HS6 and HS2 in the fact table, and then redesign the product dimension table.

Raw Fact Table:

| indicator_id | flow_id | product_id | country_id | frequency_id | total      |
|--------------|---------|------------|------------|--------------|------------|
| TRD_VAL      | EXP     | CN01       | TOTAL      | M            | 3728110    |
| TRD_VAL      | EXP     | CN010229   | TOTAL      | M            | 975885     |

Transformed Fact Table:

| date_id | flow_id | product_id | geography_id | total  |
|---------|---------|------------|--------------|--------|
| 121     | 1       | 1          | 0            | 3728110|
| 121     | 1       | 2          | 0            | 975885 |

Product Dimension Table:

| product_id | lower_id | lower_name                                 | higher_id | higher_name         |
|------------|----------|--------------------------------------------|-----------|---------------------|
| 1          | 01       | Live animals; animal products              | 01        | Animal products     |
| 2          | 010229   | Live cattle (excl. pure-bred for breeding) | 0102      | Live bovine animals |

This way, **lower_id** would have HS6 and HS2, while **higher_id** would have HS4 and Chapter. This separation of the hierarchy will allow us to perform aggregations with different data, matching the values from the SE Database exactly in the visualisation.
