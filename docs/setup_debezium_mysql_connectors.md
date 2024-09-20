# Setup Debezium_mysql on VM

# Mô hình

|Node name|IP|Installed|
|---|---|---|
|node01|10.1.20.12|Kafka, Zookeeper, Kafka connect, MySQL, Debezium MySQL Connector|

# Version

|Stack|Version|
|---|---|
|Mysql|5.7|
|Kafka|3.4|
|Debizium|2.4|

# Install Debezium

## Setup Kafka Connect

Sau khi cài đặt xong Zookeeper và Kafka theo tài liệu [Install Kafka, Zookeeper](../docs/setup_kafka_zookeeper_standalone.md). Ta cần phải setup Kafka Connect để Debezium Connector có thể kết nối vào lấy dữ liệu từ MySQL và gửi các thay đổi đến Kafka topics

**Bước 1: Thay đổi cấu hình mặc định của Kafka Connect**

Mở file `/opt/kafka/config/connect-distributed.properties` tìm đến dòng `plugin.path` và thay đổi giá trị như bên dưới 

```sh
. . .
plugin.path=/home/kafka/connect
```

**Bước 2: Tạo đường dẫn chứa plugin Debezium vừa khai báo trong Kafka Connect**

```sh
sudo mkdir -p /home/kafka/connect
sudo chown -R kafka. /home/kafka/connect
```

**Bước 3: Download Plugin Debezium MysSQL Connector**

```sh
sudo wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-mysql/2.4.1.Final/debezium-connector-mysql-2.4.1.Final-plugin.tar.gz

sudo tar -xzvf debezium-connector-mysql-2.4.1.Final-plugin.tar.gz

sudo rm debezium-connector-mysql-2.4.1.Final-plugin.tar.gz
```

**Bước 4: Chạy Kafka connect dưới dạng service Linux với systemd**

Tạo file `/etc/systemd/system/kafka-connect-distributed.service` và chèn vào nội dung sau:

```sh
[Unit]
Description=Kafka Connect Distributed
After=network.target

[Service]
Type=simple
User=kafka
Group=kafka
ExecStart=/opt/kafka/bin/connect-distributed.sh /opt/kafka/config/connect-distributed.properties
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
Environment=KAFKA_HEAP_OPTS="-Xms256M -Xmx2G"
WorkingDirectory=/opt/kafka

[Install]
WantedBy=multi-user.target
```

**Bước 5: Start & Enable Kafka Connect**

```sh
sudo systemctl start kafka-connect-distributed
sudo systemctl enable kafka-connect-distributed
```

**Bước 6: Verify**

Kafka Connect chạy trên cổng 8083

```sh
sudo systemctl status kafka-connect-distributed
```

hoặc

```sh
netstat -tupln | grep 8083
```

## Setup Debezium connector for MySQL

### Setup MySQL

Sau khi cài đặt MySQL 5.7 theo hướng dẫn [Cài đặt MySQL 5.7](https://www.devart.com/dbforge/mysql/how-to-install-mysql-on-ubuntu/) ta cần thực hiện một vài tác vụ thiết lập MySQL trước khi có thể cài đặt và chạy Debezium connector

**Tạo Database, Table phục vụ cho mục đích test**

```sh
mysql> create database 9pay_test character set utf8 collate utf8_unicode_ci;

mysql> CREATE TABLE 9pay_test.person (
    PersonID int,
    LastName varchar(255),
    FirstName varchar(255),
    Address varchar(255),
    City varchar(255)
);

mysql> FLUSH PRIVILEGES;
```

**Tạo User**

Debezium MySQL Connector yêu cầu cần 1 user trên database MySQL. User này phải có các quyền thích hợp trên **Các Database** mà Debezium muốn bắt sự kiện thay đổi dữ liệu trên đó

**Bước 1: Tạo MySQL user**

Tạo user `debezium` trên database MySQL

```sh
mysql> CREATE USER 'debezium'@'%' IDENTIFIED BY 'tubt160999';
```

**Bước 2: Grant các quyền cần thiết cho user `debezium`**

```sh
mysql> GRANT SELECT, LOCK TABLES ON 9pay_test.* TO 'debezium'@'%';


mysql> GRANT RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'debezium'@'%';

mysql> FLUSH PRIVILEGES;
```

> Lệnh 1 sẽ GRANT quyền `SELECT` và `LOCK TABLES` trên database `9pay_test` cho user `debezium`

> Lệnh 2 sẽ GRANT quyền `RELOAD`, `SHOW DATABASES`, `REPLICATION SLAVE`, `REPLICATION CLIENT` trên **TOÀN BỘ DATABASE** cho user `debezium`

**Bước 3: Enable `binlog`, `GTIDs`, `query_log_events` trên MySQL**

- **Trước khi enable `bin_log`, tạo folder chứa log**

```sh
sudo touch /var/log/mysql/mysql-bin.log

sudo chown -R kafka. /var/log/mysql/mysql-bin.log
```

- **Mở file config MySQL `/etc/mysql/mysql.conf.d/mysqld.cnf` và thêm vào các dòng sau:**

```sh
. . .
. . .
gtid_mode 	= ON
enforce_gtid_consistency = ON
log_bin 	= /var/log/mysql/mysql-bin.log
server_id 	= 1
binlog_format 	= row
binlog_row_image  = FULL
binlog_rows_query_log_events = ON
expire_logs_days = 7
```

- **Verify sau khi enable**

```sh
mysql> SELECT variable_value as "BINARY LOGGING STATUS (log-bin) ::"
FROM performance_schema.global_variables WHERE variable_name='log_bin';

mysql> show global variables like '%GTID%';

mysql> show global variables like 'binlog_rows_query_log_events';
```

- **Sample Output**

```sh
mysql> SELECT variable_value as "BINARY LOGGING STATUS (log-bin) ::"
    -> FROM performance_schema.global_variables WHERE variable_name='log_bin';
+------------------------------------+
| BINARY LOGGING STATUS (log-bin) :: |
+------------------------------------+
| ON                                 |
+------------------------------------+
1 row in set (0,00 sec)

mysql> show global variables like '%GTID%';
+----------------------------------+-------------------------------------------+
| Variable_name                    | Value                                     |
+----------------------------------+-------------------------------------------+
| binlog_gtid_simple_recovery      | ON                                        |
| enforce_gtid_consistency         | ON                                        |
| gtid_executed                    | cd677134-75a3-11ef-b969-70b5e857308a:1-34 |
| gtid_executed_compression_period | 1000                                      |
| gtid_mode                        | ON                                        |
| gtid_owned                       |                                           |
| gtid_purged                      |                                           |
| session_track_gtids              | OFF                                       |
+----------------------------------+-------------------------------------------+
8 rows in set (0,00 sec)

mysql> show global variables like 'binlog_rows_query_log_events';
+------------------------------+-------+
| Variable_name                | Value |
+------------------------------+-------+
| binlog_rows_query_log_events | ON    |
+------------------------------+-------+
1 row in set (0,00 sec)

mysql>
```

### Create Debezium Connectors

Sau khi thiết lập xong phần Kafka Connect và MySQL, ta thực hiện tạo Debezium Connector nhằm mục đích theo dõi 1 table trên database, lấy sự thay đổi data, structure... trong table đó thông qua `binlog` MySQL và sau đó push message lên Kafka Topic thông qua Kafka Connect

- **Cài đặt package `jq` phục vụ cho mục đích query Json**

```sh
sudo apt install jq -y
```

- **Thực hiện tạo Connectors bằng API của Debezium**

```sh
curl -XPOST --header "Content-Type: application/json" localhost:8083/connectors -d '
{
  "name": "mysql-connector-debezium-9pay_test.person", 
  "config": {
      "connector.class": "io.debezium.connector.mysql.MySqlConnector", 
      "database.hostname": "10.1.20.12",
      "database.port": "3306", 
      "database.user": "debezium", 
      "database.password": "tubt160999", 
      "database.server.id": "1", 
      "database.server.name": "9paysystem",
      "database.include.list": "9pay_test",
      "table.include.list": "9pay_test.person",
      "topic.prefix": "debezium-data-capture",  
      "schema.history.internal.kafka.bootstrap.servers": "10.1.20.12:9092",
      "schema.history.internal.kafka.recovery.mode": "SCHEMA_ONLY_RECOVERY",
      "schema.history.internal.kafka.topic": "debezium-structure-capture.9pay_test.person", 
      "include.schema.changes": "true",
      "snapshot.isolation.mode": "read_committed"
  }
}'
```

- **Show các Connector sau khi tạo**

```sh
curl -s localhost:8083/connectors | jq
```

- **Kiểm tra trạng thái của connector `mysql-connector-debezium-9pay_test.person` sau khi tạo**

```sh
curl -s localhost:8083/connectors/mysql-connector-debezium-9pay_test.person/status | jq
```

- **Output:**

```sh
root@tubt16-computer:/home# curl -s localhost:8083/connectors/mysql-connector-debezium-9pay_test.person/status | jq
{
  "name": "mysql-connector-debezium-9pay_test.person",
  "connector": {
    "state": "RUNNING",
    "worker_id": "127.0.1.1:8083"
  },
  "tasks": [
    {
      "id": 0,
      "state": "RUNNING",
      "worker_id": "127.0.1.1:8083"
    }
  ],
  "type": "source"
}
```

### Verify quá trình capture change data trong DB

Sau khi tạo Connector `mysql-connector-debezium-9pay_test.person`, Connector này sẽ thực hiện theo dõi sự thay đổi trong table `person` thuộc database `9pay_test`, schema `9pay_test` dựa trên `binlog` của MySQL

- Nếu có sự thay đổi về mặt **data** trong table `9pay_test.person`, nó sẽ thực hiện push message thay đổi lên topic `debezium-data-capture.9pay_test.person` của Kafka thông qua Kafka Connector

- Nếu có sự thay đổi về mặt **structure** trong table `9pay_test.person`, nó sẽ thực hiện push message thay đổi lên topic `debezium-structure-capture.9pay_test.person` của Kafka thông qua Kafka Connector

**Bước 1: Update data trong table `9pay_test.person` sau đó consume message để kiểm tra dữ liệu có được push vào Topic `debezium-data-capture.9pay_test.person` hay không**

- Chạy lệnh dưới trong database `9pay_test`

```sh
mysql> INSERT INTO 9pay_test.person (PersonID, LastName, FirstName, Address, City)
VALUES ('100', 'Bui', 'Tu', 'Dong Anh', 'HN');
```

- Sau đó kiểm tra Topic `debezium-data-capture.9pay_test.person` trên Kafka có nhận được message về sự thay đổi dữ liệu trong table `9pay_test.person` hay không bằng cách sau:

```sh
bash /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic debezium-data-capture.9pay_test.person --from-beginning
```

**Output**

```sh
{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":true,"field":"PersonID"},{"type":"string","optional":true,"field":"LastName"},{"type":"string","optional":true,"field":"FirstName"},{"type":"string","optional":true,"field":"Address"},{"type":"string","optional":true,"field":"City"},{"type":"string","optional":true,"field":"Email"},{"type":"string","optional":true,"field":"Email1"}],"optional":true,"name":"debezium-data-capture.9pay_test.person.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":true,"field":"PersonID"},{"type":"string","optional":true,"field":"LastName"},{"type":"string","optional":true,"field":"FirstName"},{"type":"string","optional":true,"field":"Address"},{"type":"string","optional":true,"field":"City"},{"type":"string","optional":true,"field":"Email"},{"type":"string","optional":true,"field":"Email1"}],"optional":true,"name":"debezium-data-capture.9pay_test.person.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false,incremental"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"sequence"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"name":"event.block","version":1,"field":"transaction"}],"optional":false,"name":"debezium-data-capture.9pay_test.person.Envelope","version":1},"payload":{"before":null,"after":{"PersonID":100,"LastName":"Bui","FirstName":"Tu","Address":"Dong Anh","City":"HN","Email":null,"Email1":null},"source":{"version":"2.4.1.Final","connector":"mysql","name":"debezium-data-capture","ts_ms":1726801953000,"snapshot":"false","db":"9pay_test","sequence":null,"table":"person","server_id":1,"gtid":"cd677134-75a3-11ef-b969-70b5e857308a:57","file":"mysql-bin.000007","pos":11784,"row":0,"thread":29,"query":null},"op":"c","ts_ms":1726801953904,"transaction":null}}
```

> Ta có thể thấy phần thay đổi về mặt **dữ liệu** trong scope `before` và `after` từ đoạn output trên

**Bước 2: Update structure trong table `9pay_test.person` sau đó consume message để kiểm tra dữ liệu có được push vào Topic `debezium-structure-capture.9pay_test.person` hay không**

- Chạy lệnh dưới trong database `9pay_test`

```sh
mysql> alter table 9pay_test.person add column Email varchar(50);
```

- Sau đó kiểm tra Topic `debezium-structure-capture.9pay_test.person` trên Kafka có nhận được message về sự thay đổi structure trong table `9pay_test.person` hay không bằng cách sau:

```sh
bash /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic debezium-structure-capture.9pay_test.person --from-beginning
```

**Output**

```sh
{
  "source" : {
    "server" : "debezium-data-capture"
  },
  "position" : {
    "transaction_id" : null,
    "ts_sec" : 1726802588,
    "file" : "mysql-bin.000007",
    "pos" : 12475,
    "gtids" : "cd677134-75a3-11ef-b969-70b5e857308a:1-59",
    "server_id" : 1
  },
  "ts_ms" : 1726802588449,
  "databaseName" : "9pay_test",
  "ddl" : "alter table 9pay_test.person add column Email varchar(50)",
  "tableChanges" : [ {
    "type" : "ALTER",
    "id" : "\"9pay_test\".\"person\"",
    "table" : {
      "defaultCharsetName" : "utf8",
      "primaryKeyColumnNames" : [ ],
      "columns" : [ {
        "name" : "PersonID",
        "jdbcType" : 4,
        "typeName" : "INT",
        "typeExpression" : "INT",
        "charsetName" : null,
        "length" : 11,
        "position" : 1,
        "optional" : true,
        "autoIncremented" : false,
        "generated" : false,
        "comment" : null,
        "hasDefaultValue" : true,
        "enumValues" : [ ]
      }, {
        "name" : "LastName",
        "jdbcType" : 12,
        "typeName" : "VARCHAR",
        "typeExpression" : "VARCHAR",
        "charsetName" : "utf8",
        "length" : 255,
        "position" : 2,
        "optional" : true,
        "autoIncremented" : false,
        "generated" : false,
        "comment" : null,
        "hasDefaultValue" : true,
        "enumValues" : [ ]
      }, {
        "name" : "FirstName",
        "jdbcType" : 12,
        "typeName" : "VARCHAR",
        "typeExpression" : "VARCHAR",
        "charsetName" : "utf8",
        "length" : 255,
        "position" : 3,
        "optional" : true,
        "autoIncremented" : false,
        "generated" : false,
        "comment" : null,
        "hasDefaultValue" : true,
        "enumValues" : [ ]
      }, {
        "name" : "Address",
        "jdbcType" : 12,
        "typeName" : "VARCHAR",
        "typeExpression" : "VARCHAR",
        "charsetName" : "utf8",
        "length" : 255,
        "position" : 4,
        "optional" : true,
        "autoIncremented" : false,
        "generated" : false,
        "comment" : null,
        "hasDefaultValue" : true,
        "enumValues" : [ ]
      }, {
        "name" : "City",
        "jdbcType" : 12,
        "typeName" : "VARCHAR",
        "typeExpression" : "VARCHAR",
        "charsetName" : "utf8",
        "length" : 255,
        "position" : 5,
        "optional" : true,
        "autoIncremented" : false,
        "generated" : false,
        "comment" : null,
        "hasDefaultValue" : true,
        "enumValues" : [ ]
      }, {
        "name" : "Email",
        "jdbcType" : 12,
        "typeName" : "VARCHAR",
        "typeExpression" : "VARCHAR",
        "charsetName" : "utf8",
        "length" : 50,
        "position" : 6,
        "optional" : true,
        "autoIncremented" : false,
        "generated" : false,
        "comment" : null,
        "hasDefaultValue" : true,
        "enumValues" : [ ]
      } ],
      "attributes" : [ ]
    },
    "comment" : null
  } ]
}
```

## Command

Một số command phục vụ mục đích tạo, xóa, kiểm tra Connector (Lưu ý nếu chưa cài jq thì bỏ phần jq trong câu lệnh)

- Kiểm tra các Connector đã được tạo

```sh
curl -s localhost:8083/connectors | jq
```

- **Tạo mới Connector**

```sh
curl -XPOST --header "Content-Type: application/json" localhost:8083/connectors -d '
{
  "name": "mysql-connector-debezium-9pay_test.person", 
  "config": {
      "connector.class": "io.debezium.connector.mysql.MySqlConnector", 
      "database.hostname": "10.1.20.12",
      "database.port": "3306", 
      "database.user": "debezium", 
      "database.password": "tubt160999", 
      "database.server.id": "1", 
      "database.server.name": "9paysystem",
      "database.include.list": "9pay_test",
      "table.include.list": "9pay_test.person",
      "topic.prefix": "debezium-data-capture",  
      "schema.history.internal.kafka.bootstrap.servers": "10.1.20.12:9092",
      "schema.history.internal.kafka.recovery.mode": "SCHEMA_ONLY_RECOVERY",
      "schema.history.internal.kafka.topic": "debezium-structure-capture.9pay_test.person", 
      "include.schema.changes": "true",
      "snapshot.isolation.mode": "read_committed"
  }
}'
```

> Có thể sửa giá trị của trường `include.schema.changes` thành `false` để chỉ capture thay đổi về mặt dữ liêu trên database (bỏ qua phần capture structure)

- Xóa Connector `mysql-connector-debezium-9pay_test.person`

```sh
curl -X DELETE localhost:8083/connectors/mysql-connector-debezium-9pay_test.person
```

- Kiểm tra trạng thái của Connector `mysql-connector-debezium-9pay_test.person`

```sh
curl -s localhost:8083/connectors/mysql-connector-debezium-9pay_test.person/status | jq
```