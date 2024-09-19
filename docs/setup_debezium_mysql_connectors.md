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

Sau khi thiết lập xong phần Kafka Connect và MySQL, ta thực hiện tạo Debezium Connector nhằm mục đích chọc vào 1 table trên database, lấy sự thay đổi data, structure... trong table đó thông qua `binlog` MySQL và sau đó bắn message lên Kafka Topic thông Kafka Connect

- Cài đặt package `jq` phục vụ cho mục đích query json

```sh
sudo apt install jq -y
```

- **Thực hiện tạo Connectors bằng API của Debezium**

```sh
curl -XPOST --header "Content-Type: application/json" localhost:8083/connectors -d '
{
  "name": "mysql-connector-debezium-test", 
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
      "topic.prefix": "tubt-test",  
      "schema.history.internal.kafka.bootstrap.servers": "10.1.20.12:9092", 
      "schema.history.internal.kafka.topic": "schemahistory.tubt-test", 
      "include.schema.changes": "true",
      "snapshot.isolation.mode": "read_committed"
  }
}'
```

- **Show các Connector sau khi tạo**

```sh
curl -s localhost:8083/connectors | jq
```

- **Kiểm tra trạng thái của connector `mysql-connector-debezium-test` sau khi tạo**

```sh
curl -s localhost:8083/connectors/mysql-connector-debezium-test/status | jq
```

- **Output:**

```sh
root@tubt16-computer:/home# curl -s localhost:8083/connectors/mysql-connector-debezium-test/status | jq
{
  "name": "mysql-connector-debezium-test",
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

### Consume Message Topic Kafka