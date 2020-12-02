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

Let's explore where these stations are located and what the pollution levels are in different parts of the world. For this we will use ARLAS.

__0. Download the tutorial__

```shell
git clone https://github.com/gisaia/ARLAS-stack-openAQ-tutorial.git

```

__1. Starting ARLAS Exploration Stack__

- Get the docker-compose file from [ARLAS-Exploration-stack](https://github.com/gisaia/ARLAS-Exploration-stack.git) that will allow us to start the ARLAS stack

```shell
curl -XGET \
    "https://raw.githubusercontent.com/gisaia/ARLAS-Exploration-stack/develop/docker-compose-withoutnginx.yaml" \
    -o docker-compose.yaml

```

- Start the ARLAS stack 

```shell
docker-compose up -d \
    arlas-wui \
    arlas-hub \
    arlas-builder \
    arlas-server \
    arlas-persistence-server \
    elasticsearch

```

6 services are started:

- ARLAS-wui at http://localhost:8096
- ARLAS-wui-builder at http://localhost:8095
- ARLAS-wui-hub at http://localhost:8094
- ARLAS-server at http://localhost:19999/arlas/swagger
- ARLAS-persistence at http://localhost:19997/arlas-persistence-server/swagger
- Elasticsearch at http://localhost:9200

Check that the 6 services are up and running using the following command: 

```shell
docker ps

```
ARLAS stack is up and running, we need now to index openAq data in Elasticsearch

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

- `datapoint` object contains information about the pollutant measurement: which pollutant, what's the value of the measurement, what is the unit and when was it emitted. Also, notice that we added a key (`pm10` in this example) representing the pollutant, the value of this key is the measurement's value.
- `station` object contains information about the station itself: name, location, coordinates...

### Indexing openAq data with the new data model

Now we will create an index in Elasticsearch that will host our downloaded pollution data. 

- Create `openaq_index` index  with `configs/openaq.mapping.json` mapping file

```shell
curl -XPUT http://localhost:9200/openaq_index/?pretty \
-d @configs/openaq.mapping.json \
-H 'Content-Type: application/json'

```

The `configs/openaq.mapping.json` mapping file declares to Elasticsearch our data model.

You can check that the index is successfuly created by running the following command

```shell
curl -XGET http://localhost:9200/openaq_index/_mapping?pretty

```

- Index data that is in `openaq_data.ndjson` file in Elasticsearch

    - We need Logstash as a data processing pipeline that ingests data in Elasticsearch. So we will download it and untar it:

        ```shell
        ( wget https://artifacts.elastic.co/downloads/logstash/logstash-7.4.2.tar.gz ; tar -xzf logstash-7.4.2.tar.gz )

        ```
    - Now we will use Logstash in order to apply the data model transformation we described earlier and to index data in Elasticsearch:

        ```shell
        cat data/openaq_data.ndjson \
        | ./logstash-7.4.2/bin/logstash \
        -f configs/openaq2es.logstash.conf

        ```
        `configs/openaq2es.logstash.conf` is the file used to describe the data model transformation

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
    curl -X PUT \
    --header 'Content-Type: application/json;charset=utf-8' \
    --header 'Accept: application/json' \
    "http://localhost:19999/arlas/collections/openaq_collection?pretty=true" \
    --data @openaq_collection.json
    ```

    Check that the collection is created using the ARLAS-server `collections/{collection}`

    ```
    curl -X GET "http://localhost:19999/arlas/collections/openaq_collection?pretty=true"
    ```
