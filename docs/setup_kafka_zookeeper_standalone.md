# Setup Kafka, Zookeeper standalone

## Version

|Stack|Version|
|---|---|
|Java|OpenJDK 11|
|Kafka|3.4|

## Install Kafka

**Bước 1: Tạo User `kafka`**

Tạo user `kafka` mới để run service bằng systemd bằng user `kafka` này

```sh
sudo useradd kafka -m -d /home/kafka -s /sbin/nologin
```

**Bước 2: Cài đặt Java**

Kafka được viết bằng Java và Scala, vì thế cần có Java Runtime Environment (JRE) để chạy nó

- Update OS (**Nếu cần**)

```sh
sudo apt update
```

- Cài đặt package OpenJDK 11

```sh
sudo apt install openjdk-11-jdk
```

**Bước 3 Download Kafka**

Tài liệu này sẽ Download, cài đặt Kafka 3.4 và giải nén nó vào thư mục `/opt`

- Di chuyển tới thư mục `/opt` và tải xuống Kafka 3.4

```sh
cd /opt
sudo wget https://archive.apache.org/dist/kafka/3.4.0/kafka_2.12-3.4.0.tgz
```

- Giải nén

```sh
sudo tar -xvzf /opt/kafka_2.12-3.4.0.tgz
```

- Đổi tên thư mục sau khi giải nén và xóa file nén 

```sh
sudo mv /opt/kafka_2.12-3.4.0/ /opt/kafka/
sudo rm /opt/kafka_2.12-3.4.0.tgz
```

**Bước 4: Sửa cấu hình mặc định của Kafka**

Thay đổi `log.dir` và `num.partitions` mặc định của Kafka. 

- Mở file `/opt/kafka/config/server.properties` tìm đến dòng `log.dir` và sửa thành giá trị `/home/kafka/kafka-logs`

```sh
. . .
. . .
log.dirs=/home/kafka/kafka-logs
. . .
```

**Bước 5: Sửa cấu hình mặc định của Zookeeper**

Thay đổi `dataDir` mặc đinh của Zookeeper

Mở file `/opt/kafka/config/zookeeper.properties` tìm đến dòng `dataDir` và sửa giá trị thành

```sh
. . .
. . .
dataDir=/home/kafka/zookeeper
. . .
```

**Bước 6: Chạy Zookeeper dưới dạng service Linux với systemd**

Để làm được điều này ta cần tạo file `/etc/systemd/system/zookeeper.service` và chèn vào nội dung sau:

```sh
[Unit]
Description=Apache Zookeeper Service
Requires=network.target                 
After=network.target                 

[Service]
Type=simple
User=kafka
ExecStart=/opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties        
ExecStop=/opt/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

**Bước 7: Chạy Kafka dưới dạng service Linux với systemd**

Để làm được điều này ta cần tạo file `/etc/systemd/system/kafka.service` và chèn vào nội dung sau:

```sh
[Unit]
Description=Apache Kafka Service that requires zookeeper service
Requires=zookeeper.service
After=zookeeper.service

[Service]
Type=simple
User=kafka
ExecStart= /opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties                            
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

## Bước 8: Reload systemd

```sh
sudo systemctl daemon-reload
```

## Bước 9: Start & Enable Kafka, Zookeeper

```sh
sudo systemctl start kafka
sudo systemctl start zookeeper
sudo systemctl enable kafka
sudo systemctl enable zookeeper
```

## Bước 10: Verify

```sh
sudo systemctl status kafka
sudo systemctl status zookeeper
```

## Bước 11: Tạo topic phục vụ cho mục đích test

```sh
bash /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic tubt-test1
```