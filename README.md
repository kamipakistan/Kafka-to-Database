# Kafka-to-Database
1. Bring up the stack
```
git clone https://github.com/confluentinc/demo-scene.git
cd kafka-to-database
```
2. Bring the Docker Compose up
```
docker-compose up -d
```
3. Make sure everything is up and running
```
docker-compose ps
```
```text
     Name                    Command                  State                     Ports
---------------------------------------------------------------------------------------------------
broker            /etc/confluent/docker/run        Up             0.0.0.0:9092->9092/tcp
kafka-connect     bash -c cd /usr/share/conf ...   Up (healthy)   0.0.0.0:8083->8083/tcp, 9092/tcp
kafkacat          /bin/sh -c apk add jq;           Up
                  wh ...
ksqldb            /usr/bin/docker/run              Up             0.0.0.0:8088->8088/tcp
mysql             docker-entrypoint.sh mysqld      Up             0.0.0.0:3306->3306/tcp, 33060/tcp
schema-registry   /etc/confluent/docker/run        Up             0.0.0.0:8081->8081/tcp
zookeeper         /etc/confluent/docker/run        Up             2181/tcp, 2888/tcp, 3888/tcp
```
4. Create test topic + data using ksqlDB (but it’s still just a Kafka topic under the covers)
```
docker exec -it ksqldb ksql http://ksqldb:8088
```
```
CREATE STREAM TEST01 (KEY_COL VARCHAR KEY, COL1 INT, COL2 VARCHAR)
  WITH (KAFKA_TOPIC='test01', PARTITIONS=1, VALUE_FORMAT='AVRO');
```
```
INSERT INTO TEST01 (KEY_COL, COL1, COL2) VALUES ('X',1,'FOO');
INSERT INTO TEST01 (KEY_COL, COL1, COL2) VALUES ('Y',2,'BAR');
```
```
SHOW TOPICS;
```
```
PRINT test01 FROM BEGINNING LIMIT 2;
```

5. Create the Sink connector
```
curl -X PUT http://localhost:8083/connectors/sink-jdbc-mysql-01/config \
     -H "Content-Type: application/json" -d '{
    "connector.class"                    : "io.confluent.connect.jdbc.JdbcSinkConnector",
    "connection.url"                     : "jdbc:mysql://mysql:3306/demo",
    "topics"                             : "test01",
    "key.converter"                      : "org.apache.kafka.connect.storage.StringConverter",
    "value.converter"                    : "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",
    "connection.user"                    : "connect_user",
    "connection.password"                : "asgard",
    "auto.create"                        : true,
    "auto.evolve"                        : true,
    "insert.mode"                        : "insert",
    "pk.mode"                            : "record_key",
    "pk.fields"                          : "MESSAGE_KEY"
}'
```
