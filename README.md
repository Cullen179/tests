Add link video, attach link to demonstration link
# Assignment 1: Data Pipeline with Docker

Student Name: Do Tung Lam  \
SID: s3963286 \
Tutorial Lecturer: Dr. Arthur Tang \

## Demonstration Video



# Step-by-step Instruction

## API used for Task 3: Alpha Vantage API
Website: https://www.alphavantage.co/support/#api-key \
Alpha Vantage is a leading provider of free APIs offering real-time and historical financial market data across various asset classes, including stocks, forex, and cryptocurrencies. Their APIs are designed to be developer-friendly, providing data in JSON and CSV formats.

**Feature**: 
- **Stock Time Series Data**: Access global equity data with multiple temporal resolutions—intraday, daily, weekly, and monthly—spanning over 20 years of historical data.

- **Foreign Exchange (Forex) Data**: Retrieve real-time and historical forex rates for major currency pairs.

- **Cryptocurrency Data**: Obtain data on various digital currencies, including Bitcoin, Ethereum, and others.

- **Technical Indicators**: Utilize over 50 technical indicators for in-depth market analysis.

- **Fundamental Data**: Access detailed company information, including earnings, balance sheets, and cash flow statements.

For **Task 3**, to fetch the stock price data, I utilized the feature **Stock Time Series Data** for intraday period to get the lastest stock price data. The intraday data (including 20+ years of historical data) is updated at the end of each trading day for all users by default.


**Examle Output**:
```bash
{
    "Time Series (5min)": {
        "2024-11-27 18:50:00": {
            "1. open": "226.9200",
            "2. high": "226.9200",
            "3. low": "226.9200",
            "4. close": "226.9200",
            "5. volume": "25"
        },
        "2024-11-27 18:40:00": {
            "1. open": "227.2750",
            "2. high": "227.2750",
            "3. low": "227.2750",
            "4. close": "227.2750",
            "5. volume": "4"
        },
    }
}
```
- **Time Series (5min)**: The main dataset containing stock price and volume data for each interval.
- **1. open**: Opening price of the stock at the given interval.
- **2. high**: Highest price of the stock during the interval.
- **3. low**: Lowest price of the stock during the interval.
- **4. close**: Closing price of the stock at the end of the interval.
- **5. volume**: Number of shares traded during the interval.

## API Set up
Currently, in this file there has already been sample APIs that I use during my demonstration. 
However, in case the API's limits are exceeded, you should create your own API from below links with its publicity and free access.

OpenWeatherMap API: https://home.openweathermap.org/api_keys \
Alpha Vantage API: https://www.alphavantage.co/support/#api-key

After obtaining the API keys, please update the files "alphav-producer/alphav_service.cfg" and "owm-producer/openweathermap_service.cfg" accordingly.

> **Note:** 
For dock-compose up command, I will add option --build to re-create the container in case it cached the old data. Remove it won't affect your results if you haven't had any similar container before.

# Create docker networks
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


## Starting Producers
```bash
$ docker-compose -f owm-producer/docker-compose.yml up -d --build    # start the producer that retrieves open weather map
$ docker-compose -f faker-producer/docker-compose.yml up -d --build     # start the producer for faker data
$ docker-compose -f alphav-producer/docker-compose.yml up -d --build    # start the producer for faker data
```
> **Note:** 
As Alpha Vantage detect fetch rate based on the IP Address where you call, which means that no matter the API key you use, if you still at the same IP Address that exceed the limit, you won't be able to fetch new data. For this, you can change your IP Address using VPN (ex. 1.1.1.1).

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

And that's it! you should be seeing records coming in to Cassandra


## Visualization

Run the following command the go to http://localhost:8888 and run the visualization notebook accordingly

```
docker-compose -f data-vis/docker-compose.yml up -d
```

## Pipeline Teardown

To stop all running kakfa cluster services

```bash
$ docker-compose -f data-vis/docker-compose.yml down # stop visualization node

$ docker-compose -f owm-producer/docker-compose.yml down       # stop open weather map producer

$ docker-compose -f faker-producer/docker-compose.yml down       # stop open faker producer
$ docker-compose -f alphav-producer/docker-compose.yml down       # stop open alpha vantage producer

$ docker-compose -f kafka/docker-compose.yml down              # stop zookeeper, broker, kafka-manager and kafka-connect services

$ docker-compose -f cassandra/docker-compose.yml down          # stop Cassandra
```

To remove the kafka-network network:

```bash
$ docker network rm kafka-network
$ docker network rm cassandra-network
```

To remove resources in Docker

```bash
$ docker container prune # remove stopped containers, done with the docker-compose down
$ docker volume prune # remove all dangling volumes (delete all data from your Kafka and Cassandra)
$ docker image prune -a # remove all images (help with rebuild images)
$ docker builder prune # remove all build cache (you have to pull data again in the next build)
$ docker system prune -a # basically remove everything
```


