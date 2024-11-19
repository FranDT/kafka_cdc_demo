# 0. Prerequisites

The demo has been run on WSL 2 (Ubuntu 20.04) hosted on Windows 10.
Docker is necessary to run this demo.



# 1. Create the dockers for the environmet

After cloning the repository, from the root directory run the following command to give execution permission to a file we will have to run later:

chmod +x data/02_populate_more_orders.sh

Now let's start our environment with:

docker-compose up -d

This command will take the docker-compose.yml file and generate the specified docker containers. Since all the images must be downloaded, it will take a while to complete.
After this, we need to wait for Kafka Connect to be started, which can be done via the following command:

bash -c ' \
echo -e "\n\n=============\nWaiting for Kafka Connect to start listening on localhost ⏳\n=============\n"
while [ $(curl -s -o /dev/null -w %{http_code} http://localhost:8083/connectors) -ne 200 ] ; do
  echo -e "\t" $(date) " Kafka Connect listener HTTP state: " $(curl -s -o /dev/null -w %{http_code} http://localhost:8083/connectors) " (waiting for 200)"
  sleep 5
done
echo -e $(date) "\n\n--------------\n\o/ Kafka Connect is ready! Listener HTTP state: " $(curl -s -o /dev/null -w %{http_code} http://localhost:8083/connectors) "\n--------------\n"
'

After two or three minutes, it should be possible to visit the Confluent Control Center, which is the central hub to inspect informations about your cluster. 
The Control Center is created via docker-compose.yml at line 80 and provides a useful UI to understand the status of the cluster, along with connectors and more.
To access the Control Center, you can connect to http://localhost:9021/clusters (the port is specified in the docker-compose file): it takes at least two minutes to be available, since it wait for the Kafka brokers and the cluster to be on before start checking their status.

We can also check the list of available Connectors using the command:

curl -s localhost:8083/connector-plugins|jq '.[].class'|egrep 'MySqlConnector|ElasticsearchSinkConnector'

These connectors are installed via docker-compose.yml at line 72 and 73.
We can also check that the MySQL container has been started properly by running the following command to open a MySQL shell:

docker exec -it mysql bash -c 'mysql -u root -p$MYSQL_ROOT_PASSWORD demo'

We can see that there's an ORDERS table in the database we will use for our demo:

SELECT * FROM ORDERS ORDER BY CREATE_TS DESC LIMIT 1;


# 2. Generate data in real time and send them to Kafka

Let's run a script that ingests data continuously in MySQL, inside the orders table. The script is composed by a simple set of inserts:

docker exec mysql /data/02_populate_more_orders.sh

The data are generated right now but are not inserted into Kafka since no connector is set up between Kafka and MySQL. To create a Kafka Connector (through Debezium) we need to run the following command:

curl -i -X PUT -H  "Content-Type:application/json" \
    http://localhost:8083/connectors/source-debezium-orders-00/config \
    -d '{
            "connector.class": "io.debezium.connector.mysql.MySqlConnector",
            "database.hostname": "mysql",
            "database.port": "3306",
            "database.user": "debezium",
            "database.password": "dbz",
            "database.server.id": "42",
            "database.server.name": "asgard",
            "table.whitelist": "demo.orders",
            "database.history.kafka.bootstrap.servers": "broker:29092",
            "database.history.kafka.topic": "dbhistory.demo" ,
            "decimal.handling.mode": "double",
            "include.schema.changes": "true",
            "transforms": "unwrap,addTopicPrefix",
            "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
            "transforms.addTopicPrefix.type":"org.apache.kafka.connect.transforms.RegexRouter",
            "transforms.addTopicPrefix.regex":"(.*)",
            "transforms.addTopicPrefix.replacement":"mysql-debezium-$1"
    }'

The response to this command should show a 201 Created line, which says that the connector has been correctly created.
At this point, we can either inspect the topic via CLI using the following command:

docker exec kafkacat kafkacat \
        -b broker:29092 \
        -r http://schema-registry:8081 \
        -s avro \
        -t mysql-debezium-asgard.demo.ORDERS \
        -C -o -10 -q | jq '.id, .CREATE_TS.string'

Or we can see the topic via the Control Center UI. To do so, just enter the cluster, then enter Topics -> mysql-debezium-asgard.demo.ORDERS. In here, we can see the retention time of the topic, the number of partitions (set to 1 for this demo), and under messages we can see in real-time the arrival of new messages.
In Control Center we can also check the status of the connector, by entering the Connect -> cluster_name tab: our connector will be named source-debezium-orders-00, as we choose in the command to generate the connector at line 2.
In this page we can see that the connector is Running, it is a source connector and it is a MySQLConnector plugin.

# 3. Stream data into Elasticsearch

At this point we will have our data on Kafka ready to be sent to some consumers, so we can start by sending them to Elasticsearch, which we have deployed in a demo version in the docker-compose file at line 126.
Elasticsearch need to be defined as a consumer, and in order to do so we just have to create a new connector from Kafka to Elasticsearch via the following command:

curl -i -X PUT -H  "Content-Type:application/json" \
    http://localhost:8083/connectors/sink-elastic-orders-00/config \
    -d '{
        "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
        "topics": "mysql-debezium-asgard.demo.ORDERS",
        "connection.url": "http://elasticsearch:9200",
        "type.name": "type.name=kafkaconnect",
        "key.ignore": "true",
        "schema.ignore": "true"
    }'

We can check that the Connector has been created again via Control Center.

# 4. Visualize data in Kibana

Since we deployed Kibana connected to our Elasticsearch instance (line 137), we can use either the UI or a command in the CLI to check the status of the data arriving on ES.
As for the command, we can run:

curl -s http://localhost:9200/mysql-debezium-asgard.demo.orders/_search \
    -H 'content-type: application/json' \
    -d '{ "size": 5, "sort": [ { "CREATE_TS": { "order": "desc" } } ] }' |\
    jq '.hits.hits[]._source | .id, .CREATE_TS'

This command will show the 5 latest objects arrived in ES: by running multiple times this command, it will be clear that data are arriving in a stream.
To inspect Kibana via UI, we can open http://localhost:5601/app/home#/ which will redirect us to the Elastic homepage, then we can travel on Kibana -> Discover which will automatically show us the data arriving from Kafka (must be refreshed to see that new data are arriving).

