# ARLAS-stack-openAQ-tutorial

## About this tutorial

### What will you learn ?

With this tutorial, you'll be able to:

- start an ARLAS-Exploration stack
- Index some air pollution data in [Elasticsearch](https://www.elastic.co/elasticsearch/)
- Reference the indexed data in ARLAS
- Create a view of ARLAS-wui (a dashboard) to explore the air pollution data using ARLAS-wui-hub and ARLAS-wui-builder

### What will you need ?

You will need :

- docker & docker-compose
- curl

## Air pollution data

Let's explore some air pollution data, provided by [__OpenAq__](https://openaq.org/#/?_k=etcbz8) a non-profit organization that share open air quality data around the world. 

The provided data are json-line files where each line corresponds to an emitted measurement of a pollutant by a given station.

Stations are available in many countries around the world and they emit measurements several times per day for one or multiple pollutants such as :

- NO2
- SO2
- CO
- PM10
- PM25
- ...

We fetched a subset of the dataset by downloading data of the 2 first weeks of october 2020 from a S3 [object store](https://openaq-fetches.s3.amazonaws.com/index.html). We took data hosted in `realtime-gzipped` object.


For this tutorial, we simplified the dataset by aggregating all the station measurements daily. Thus a station emits one measurement per day for each pollutant. We stored this data in a line-json file named `openaq_data.ndjson`. 

It contains around 313 000 measurements emitted by different stations around the world during the 2 first weeks of october 2020.

### Data model

Each line of the `openaq_data.ndjson` file is a json object that contains the following attributes

- parameter: the pollutant (pm25, pm10, no2, ....)
- value: The pollutant measurement value
- unit: unit of the measurement value (ppm, µg/m³, ...)
- timestamp: the day the measurement was emitted
- sourceName: name of the station that emitted the measure
- sourceType: the entity responsible of the station. E.g: government
- country: the country where there is the station
- city: the city where there is the station
- location: the location in the city where there is the station
- coordinates.longitude: station's longitude coordinate
- coordinates.latitude: station's latitude coordinate.

For example, a line of the line-json file looks like:

```json
{
    "parameter": "pm10",
    "value": 6.91,
    "unit": "µg/m³",
    "timestamp": "2020-09-30",
    "sourceName": "Australia - ACT",
    "sourceType": "government",
    "country": "AU",
    "city": "Canberra",
    "location": "Civic",
    "coordinates": {
        "longitude":149.131579,
        "latitude": -35.285307
    }
}

```

## Exploring OpenAq data

Let's explore where these stations are located and what are the pollution levels in different parts of the world. For this we will use ARLAS.

__0. Setup__

- Create a repository dedicated to this tutorial

```shell
mkdir ARLAS-stack-openaq-tutorial
cd ARLAS-stack-openaq-tutorial

```

- Download the openAq data

```shell
curl -L -O "https://raw.githubusercontent.com/gisaia/ARLAS-stack-openAQ-tutorial/master/data/openaq_data.ndjson"

```

Check that `openaq_data.ndjson` file is downloaded

```shell
ls -l openaq_data.ndjson

```

- Download the ARLAS-Exploration-stack project and unzip it

```shell
(curl -L -O "https://github.com/gisaia/ARLAS-Exploration-stack/archive/develop.zip"; unzip develop.zip)

```

Check that the `ARLAS-Exploration-stack-develop` stack is downloaded

```shell
ls -l ARLAS-Exploration-stack-develop

```

Now our tutorial environment is set up.


__1. Starting ARLAS Exploration Stack__

```shell
./ARLAS-Exploration-stack-develop/start.sh

```

__2. Indexing opanAQ data in Elasticsearch__

Before indexing the data, we need to apply some adjustments to the data model in order to get the maximum information of it.

### Adapting the initial data structure

We will restructer the data model to give more clarity

The following model

```json
{
    "parameter": "pm10",
    "value": 6.91,
    "unit": "µg/m³",
    "timestamp": "2020-09-30",
    "sourceName": "Australia - ACT",
    "sourceType": "government",
    "country": "AU",
    "city": "Canberra",
    "location": "Civic",
    "coordinates": {
        "longitude":149.131579,
        "latitude": -35.285307
    }
}

```

will be transformed to : 

```json
{
    "datapoint" : {
        "type" : "pm10",
        "value" : 6.91,
        "unit" : "µg/m³",
        "pm10" : 6.91,
        "timestamp" : "2020-09-30"
    },
    "station" : {
        "name" : "Australia - ACT",
        "type" : "government",
        "country" : "AU",
        "city" : "Canberra",
        "location" : "Civic",
        "geometry" : {
            "lon" : 149.131579,
            "lat" : -35.285307
        }
    }
}

```

You notice that we split the document into two objects.

- `datapoint` object contains information about the pollutant measurement: which pollutant, what's the value of the measurement, what is the unit and when was it emitted. Also, notice that we added a key (`pm10` in this example) representing the pollutant, the value of this key is the measurement's value. This restructuring of the data will allow us to obtain analytical views for each pollutant independently. For instance we will be able to analyse the average measures of `pm10` over time using a histogram. Without this adaptation, this representation wouldn't be possible.
- `station` object contains information about the station itself: name, location, coordinates...

### Indexing openAq data with the new data model

Now we will create an index in Elasticsearch that will host our downloaded pollution data. 

- Create `openaq_index` index  with `openaq.mapping.json` mapping file

```shell
curl "https://raw.githubusercontent.com/gisaia/ARLAS-stack-openAQ-tutorial/master/configs/openaq.mapping.json" | \
curl -XPUT "http://localhost:9200/openaq_index/?pretty" \
    -d @- \
    -H 'Content-Type: application/json'

```

The `openaq.mapping.json` mapping file declares to Elasticsearch our data model.

You can check that the index is successfuly created by running the following command

```shell
curl -XGET http://localhost:9200/openaq_index/_mapping?pretty

```

- Index data that is in `openaq_data.ndjson` file in Elasticsearch. For that, we need Logstash as a data processing pipeline that ingests data in Elasticsearch. Logstash needs a configuration file (`openaq2es.logstash.conf`) that indicates how to to apply the data model transformation we described earlier on the `openaq_data.ndjson` file and to index data in Elasticsearch.

```shell
curl "https://raw.githubusercontent.com/gisaia/ARLAS-stack-openAQ-tutorial/master/configs/openaq2es.logstash.conf" \
    -o openaq2es.logstash.conf
```

- Now we will use Logstash in order to apply the data model transformation and to index data in Elasticsearch given the `openaq2es.logstash.conf` configuration file with the docker image `docker.elastic.co/logstash/logstash` :

```shell
network=$(docker network ls --format "table {{.Name}}" | grep arlas)

cat openaq_data.ndjson | docker run -e XPACK_MONITORING_ENABLED=false \
    --net ${network} \
    --env ELASTICSEARCH=elasticsearch:9200  \
    --env INDEXNAME=openaq_index --rm -i \
    -v ${PWD}/openaq2es.logstash.conf:/usr/share/logstash/pipeline/logstash.conf docker.elastic.co/logstash/logstash:7.11.2
```

- Check if __313 291__ pollutant measurementsare indexed:

```shell
curl -XGET http://localhost:9200/openaq_index/_count?pretty

```

__3. Declaring `openaq_index` in ARLAS__

ARLAS-server interfaces with data indexed in Elasticsearch via a collection reference.

The collection references an identifier, a timestamp, and geographical fields which allows ARLAS-server to perform a spatial-temporal data analysis


!!! info "Information"
    Get more details about the [collection model](#arlas-collection-model.md) and [how to manage collections with ARLAS-server](#arlas-api-collection.md)


- Create a openaq collection in ARLAS

```shell
curl "https://raw.githubusercontent.com/gisaia/ARLAS-stack-openAQ-tutorial/master/openaq_collection.json" | \
curl -X PUT \
    --header 'Content-Type: application/json;charset=utf-8' \
    --header 'Accept: application/json' \
    "http://localhost:81/server/collections/openaq_collection?pretty=true" \
    --data @-

```

Check that the collection is created using the ARLAS-server `collections/{collection}`

```shell
curl -X GET "http://localhost:81/server/collections/openaq_collection?pretty=true"

```
