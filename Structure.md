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

The "structure" part of the cube is the same as the one in the metadata. 



