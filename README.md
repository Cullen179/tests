Add link video, attach link to demonstration link
# Assignment 1: Data Pipeline with Docker

Student Name: Do Tung Lam  \
SID: s3963286 \
Tutorial Lecturer: Dr. Arthur Tang \

## Demonstration Video



# Step-by-step Instruction

## API used for Task 3: Gemini API
Website: https://docs.gemini.com/rest-api/#ticker \
The Gemini REST API provides both public and private endpoints to interact with the Gemini Exchange. Public endpoints offer access to market data, while private endpoints enable account management and order execution.

**Public REST APIs Feature**: 
- Accessing the current order book, which provides a detailed view of market depth and liquidity.
- Retrieving recent trading activity, enabling users to analyze the most up-to-date market movements.
- Viewing the trade history, which offers a comprehensive record of past transactions for deeper market analysis.

**Private REST APIs Feature**: 
- The ability to place and cancel orders, ensuring precise control over trading activity.
- Viewing a list of active orders, allowing users to track their open positions and manage risk effectively.
- Accessing a detailed record of trading history and trade volume, which is critical for performance analysis and compliance reporting.
- Retrieving available balances, enabling users to monitor account funds and make informed decisions about trading and fund transfers.

For **Task 3**, to fetch the real, I utilized the feature **Public APIs - Ticker** which retrieves real-time data about recent trading activity for the symbol.

**Examle Output of Public APIs - Ticker**:
```bash
{
    "ask": "977.59",
    "bid": "977.35",
    "last": "977.65",
    "volume": {
        "BTC": "2210.505328803",
        "USD": "2135477.463379586263",
        "timestamp": 1483018200000
    }
}
```
- **ask**: The lowest ask currently available.
- **bid**: The highest bid currently available.
- **last**: The price of the last executed trade.
- **volume**: Information about the 24 hour volume on the exchange. See below.

## API Set up
Currently, in this file there has already been sample APIs that I use during my demonstration. 
However, in case the API's limits are exceeded, you should create your own API from below links with its publicity and free access.

- OpenWeatherMap API: https://home.openweathermap.org/api_keys \
- Gemini API (don't require API keys to use feature ticker): https://docs.gemini.com/rest-api/#ticker

After obtaining the API keys, please update the files "owm-producer/openweathermap_service.cfg" accordingly.

> **Note:** 
For dock-compose up command, I will add option --build to re-create the container in case it cached the old data. Remove it won't affect your results if you haven't had any similar container before.

## Create docker networks
```bash
$ docker network create kafka-network                         # create a new docker network for kafka cluster (zookeeper, broker, kafka-manager services, and kafka connect sink services)
$ docker network create cassandra-network                     # create a new docker network for cassandra. (kafka connect will exist on this network as well in addition to kafka-network)
```
## Starting Cassandra

Cassandra is setup so it runs keyspace and schema creation scripts at first setup so it is ready to use.
```bash
$ docker-compose -f cassandra/docker-compose.yml up -d --build
```
Check if cassandra is connected to the server
```bash
$ cqlsh --cqlversion=3.4.4 127.0.0.1
#make sure you use the correct cqlversion
```

## Starting Kafka
```bash
$ docker-compose -f kafka/docker-compose.yml up -d --build
# start single zookeeper, broker, kafka-manager and kafka-connect services
```

> **Note:** 
Kafka-Manager front end is available at http://localhost:9000

You can use it to create cluster to view the topics streaming in Kafka (for better understanding, please check the above demonstration link).

- Click Add Cluster
- Name your cluster of your preference: ex. Cluster
- Fill "zookeeper:2181" for field Cluster Zookeeper Hosts
- Check box Enable JMX Polling (Set JMX_PORT env variable before starting kafka server)
- Check box Poll consumer information (Not recommended for large # of consumers if ZK is used for offsets tracking on older Kafka versions)
- Check box Enable Active OffsetCache (Not recommended for large # of consumers)
- Click Save. After this, you should see 7 topics have been connected.

Note: You can also check the sinks have been created by running:
```bash
$ curl -X GET http://localhost:8083/connectors
```
If the sinks is empty, try running this command within kafka-connect Docker CLI
```bash
$ .\start-and-wait.sh
```

## Starting Producers
```bash
$ docker-compose -f owm-producer/docker-compose.yml up -d --build    # start the producer that retrieves open weather map
$ docker-compose -f faker-producer/docker-compose.yml up -d --build     # start the producer for faker data
$ docker-compose -f gemini-producer/docker-compose.yml up -d --build    # start the producer for gemini data
```

## Check that data is arriving to Cassandra

First login into Cassandra's container with the following command or open a new CLI from Docker Desktop if you use that.
```bash
$ docker exec -it cassandra bash
```
Once loged in, bring up cqlsh with this command and query twitterdata and weatherreport tables like this:
```bash
$ cqlsh --cqlversion=3.4.4 127.0.0.1 #make sure you use the correct cqlversion

cqlsh> use kafkapipeline; #keyspace name

cqlsh:kafkapipeline> select * from weatherreport;

cqlsh:kafkapipeline> select * from fakerdata;

cqlsh:kafkapipeline> select * from alphavdata;
```

And that's it! you should be seeing records coming in to Cassandra.


## Visualization

Run the following command the go to http://localhost:8888 and run the visualization notebook accordingly

```
docker-compose -f data-vis/docker-compose.yml up -d --build
```



