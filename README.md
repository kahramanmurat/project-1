# Final Project

![alt text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-0.png)

The main objective of the final project is to create a live application that captures GPS data transmitted by IoT devices installed on buses. Our project employs a range of technologies, including Apache NiFi, CDC (Debezium), Kafka (MSK), Spark Structural Streaming, Docker, MySQL, and Superset for data visualization. The project's data source is the TTC bus data provided by the open API named Rest Bus, and a sample of the streaming bus data is available.

``` [{"id":"7","title":"7-Bathurst","_links":{"self":{"href":"http://restbus.info/api/agencies/ttc/routes/7","type":"application/json","rel":"http://restbus.info/_links/rel/full","rt":"route","title":"Full configuration for ttc route 7-Bathurst."},"to":[{"href":"http://restbus.info/api/agencies/ttc/routes/7","type":"application/json","rel":"http://restbus.info/_links/rel/full","rt":"route","title":"Full configuration for ttc route 7-Bathurst."}],"from":[{"href":"http://restbus.info/api/agencies/ttc","type":"application/json","rel":"via","rt":"agency","title":"Transit agency ttc details."}```

http://restbus.info/api/agencies/ttc/routes

By fetching data from the Rest Bus API, we can display it on Google Maps and utilize it to enhance specific routes. 

The retrieved data will be stored in a MySQL database. To achieve this, we will utilize Kafka Connect, Kafka, and Zookeeper applications, which will monitor the MySQL database's Write Ahead Logs. These logs contain data about insertions, updates, and deletions, and they will be written to a MySQL table.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-1.png)

Change data capture (CDC) is a technique that captures only newly modified data from the target database and transmits it to downstream processes. Debezium is a reliable, high-speed, and user-friendly open-source platform for implementing CDC. By simply pointing it to your database, it begins transmitting CRUD operations downstream and has numerous connections to popular databases. In our project, the Debezium plugin will extract data from MySQL and send it to a Kafka topic via Kafka Connect.

## Setting up Apache NiFi
Our plan is to utilize Apache NiFi to extract data from the bus API and transmit it to the MySQL database, making the data available for further consumption. Apache NiFi is a versatile, drag-and-drop data engineering tool that is highly scalable and can be likened to a Swiss Army knife for big data integration. It is capable of bidirectional data transmission in batches or streams to numerous data sources.

To implement NiFi, we will utilize Docker, which will run on an EC2 instance. For optimal performance, we recommend using a t2.xlarge or t2.2xlarge instance. So, let's begin by creating a new EC2 instance.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-2.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-3.png)

Select instance type and key pair. You can create a new key pair or use an existing one.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-4.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-5.png)

You can create a new security group or use an existing one.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-6.png)

For now, we can use a security group with All Traffic rule.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-7.png)

Since we are going to use this instance to collect data from the API we should use a larger storage.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-8.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-9.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-10.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-11.png)

After a few moments, our EC2 should have running status and we can ssh into it.

`ssh -i ~/.ssh/final-project-DE.pem ec2-user@'<your_EC2_external_IP>'

**Now we need to install Docker:**

`sudo yum update -y 
sudo yum install docker -y`

**Start Docker services:**

`sudo service docker start
sudo systemctl enable docker
sudo usermod -a -G docker ec2-user `

Exit and reconnect to EC2

## Create MySQL - Debezium

Before we deploy Apache NiFi we need to set up a MySQL database with an extra layer called Debezium for CDC.

`docker run -dit --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=debezium -e MYSQL_USER=mysqluser -e MYSQL_PASSWORD=mysqlpw debezium/example-mysql:1.6`

After MySQL is up and running, we will need to access the MySQL container and create a new database with a corresponding table. Nifi will be used to store the data retrieved from the Bus API into the MySQL table. This process imitates a typical business scenario where SQL databases are constantly updated with new data that needs to be captured for downstream processes. To log into MySQL, you will need to enter the password that was defined in the docker run command (debezium).

`docker exec -it '<container_id>' bash
mysql -u root -p 
debezium # root password for MySWQL
CREATE DATABASE demo;`

Now we can create the table with pre-defined schema. We are going to use it to collect data from the bus API.

```
use demo;

CREATE TABLE bus_status (
    record_id INT NOT NULL AUTO_INCREMENT,
    id INT NOT NULL,
    routeId INT NOT NULL,
    directionId VARCHAR(40),
    predictable BOOLEAN,
    secsSinceReport INT NOT NULL,
    kph INT NOT NULL,
    heading INT,
    lat REAL NOT NULL, 
    lon REAL NOT NULL,
    leadingVehicleId INT,
    event_time DATETIME DEFAULT NOW(),
    PRIMARY KEY (record_id)
);
```
![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-12.png)

Next, we need to set up Apache Nifi, for the final project we are going to use 1.12.0 version since it is more stable compare to more newer versions.

```
docker run --name nifi -p 8080:8080 -p 8443:8443 --link mysql:mysql -d apache/nifi:1.12.0
```

Now if you visit http://'':8080/nifi/ page, you should see the NiFi interface. So we can start working on our ingestion logic.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-13.png)

For our final project, we are going to use Processor, Processor groups and Template. First step, we are going to create a processor.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-14.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-16.png)

The processor is designed to connect to the Rest Bus API endpoint and gather JSON blobs. To configure the InvokeHTTP processor, double-click on it and access the settings. Change the log level from WARN to INFO. Then, select the scheduling option and adjust the "Run Schedule" to 30 seconds. This setting will prompt the processor to retrieve API information every 30 seconds, which is a reasonable timeframe for the project and will safeguard our IP from being blacklisted.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-17.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-18.png)

Under Properties add the Rest Bus API endpoint under Remote URL and click apply.

```
http://restbus.info/api/agencies/ttc/routes/7/vehicles
```

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-19.png)

Next we will add the LogAttribute Processor to handle edge cases.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-20.png)

Now we can connect two processors by drawing this arrow and pointing it to the LogAttribute Processor.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-21.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-22.png)

Alter connecting them, select Failure, No Retry, Retry in the LogAttribute Processor and click add.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-23.png)

Next, we need to add another processor that will convert JSON blobs into SQL inserts. We will be using it to insert new records into our target MySQL table. For this purpose, we are going to use ConverterJSONToSQL Processor.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-24.png)

Let’s connect the InvokeHTTP processor with ConverterJSONToSQL Processor and select Original and Response.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-25.png)

So our pipeline should look as following. Next double click on the “ConverterJSONToSQL” processor to configure it.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-26.png)

We don’t need to change anything in the settings and schedule. However, we need to set up a connection JDBC pool under properties.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-27.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-28.png)

Next we need to change “Statement Type” to INSERT and add a Table Name “bus_status”. You also can change “Unmatched Column Behavior” to Ignore Unmatched Columns, meaning if there are any unmatched columns, they will be ignored by the insert operation.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-29.png)

Now we need to configure JDBC connection. You need to click on this arrow icon.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-30.png)

After a new window loads you need to click on a gear icon.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-31.png)

A new window will pop up and you would need to click on the properties tab and start configuring the connection.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-32.png)

First, we need to click on Database Connection URL to add our MySQL connection URL.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-33.png)

```
jdbc:mysql://mysql:3306/demo
```

Second, we need to specify JDBC class name.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-34.png)

```
com.mysql.jdbc.Driver
```

Next we would need to add JDBC jar since Apache NiFi didn’t come repackage with it. We need to exec into Docker NiFi container and download JDBC connector libraries, but first let’s create a new folder custom-jars and download the JDBC jar.

```
docker exec -it nifi bash
mkdir custom-jars
cd custom-jars
wget http://java2s.com/Code/JarDownload/mysql/mysql-connector-java-5.1.17-bin.jar.zip
unzip mysql-connector-java-5.1.17-bin.jar.zip
```

So the final location of your jar file will be…

```
/opt/nifi/nifi-current/custom-jars/mysql-connector-java-5.1.17-bin.jar
```
![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-35.png)

Now, we need to define our Database user name (root) and password (debezium).

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-36.png)

Click apply and enable the service by clicking on “light bolt” icon.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-37.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-38.png)

Now we need to add an execution engine to execute those SQL statements that are generated by the ConverterJSONToSQL processor. We are going to use PutSQL processor.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-39.png)

Connect the the ConvertJSONToSQL processor with the PutSQL processor and select SQL.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-40.png)

Double click on the PutSQL processor and go to “Properties” to specify JDBC connection pool. We can also link the PutSQL processor to itself to capture and reprocessing any retries.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-41.png)

Now we can add another LogAttribute Processor to log any errors. Create a connection from the ConvertJSONToSQL processor to the new LogAttribute Processor and select failure and original.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-42.png)

Optionally, we can add two more LogAttribute Processors to log failure and success for the PutSQL processor step. So our pipeline will look something like this.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-43.png)

To sart all processors you can click right click and select Start. To see if any data is getting processed you can click on the first queue and and list the queue.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-44.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-45.png)

You should see incoming messages

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-46.png)

```
[
   {
      "id":"8189",
      "routeId":"7",
      "directionId":"7_0_7",
      "predictable":true,
      "secsSinceReport":3,
      "kph":5,
      "heading":171,
      "lat":43.7837776,
      "lon":-79.4455647,
      "leadingVehicleId":null,
      "_links":{
         "self":{
            "href":"http://restbus.info/api/agencies/ttc/vehicles/8189",
            "type":"application/json",
            "rel":"self",
            "rt":"vehicle",
            "title":"Transit agency ttc vehicle 8189."
         },
         "to":[

         ],
         "from":[
            {
               "href":"http://restbus.info/api/agencies/ttc/vehicles",
               "type":"application/json",
               "rel":"section",
               "rt":"vehicle",
               "title":"A collection of vehicles for agency ttc."
            },
            {
               "href":"http://restbus.info/api/agencies/ttc/routes/7/vehicles",
               "type":"application/json",
               "rel":"section",
               "rt":"vehicle",
               "title":"A collection of vehicles currently on route 7 for agency ttc."
            }
         ]
      }
   }
]
```

We can also check sql queue by listing it.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-47.png)

If you see some question marks in values, those are used as placeholders for attributes taken from FlowFiles.

```
INSERT INTO bus_status 
(id, routeId, directionId, predictable, secsSinceReport, kph, heading, lat, lon, leadingVehicleId)
VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
```

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-48.png)

Now if we exec into MySQL container and log into MySQL we can see that the bus_status table is populated with data.

```
docker exec -it mysql bash
mysql -u root -p
```

```
use demo;
select * from bus_status limit 10;
```

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-49.png)

Using Processor group, we can create a multiple pipelines processing different bus routes. We can template our current pipeline and add it to a new processor group. Processor groups allow better organization of different data processing pipelines. Templates can be dowloaded as XML files and can be version controlled by Git.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-50.png)

## Debezium and Kafka Setup

Running Debezium involves Zookeeper, Kafka, and services that run Debezium's connectors. Debezium is a distributed platform that turns your existing databases into event streams, so applications can quickly react to each row-level change in the databases. Debezium is built on top of Kafka and provides Kafka Connect compatible connectors that monitor specific database management systems. Debezium records the history of data changes in Kafka topics.

First step we are going to set up Zookeeper

```
docker run -dit --name zookeeper -p 2181:2181 -p 2888:2888 -p 3888:3888 debezium/zookeeper:1.6
```
Next we need to create a Kafka container

```
docker run -dit --name kafka -p 9092:9092 --link zookeeper:zookeeper debezium/kafka:1.6
```

Now we can set up a Debezium connect container which will create a connection with Kafka, Zookeeper and MySQL containers.

```
docker run -dit --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my-connect-configs -e OFFSET_STORAGE_TOPIC=my-connect-offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link zookeeper:zookeeper --link kafka:kafka --link mysql:mysql debezium/connect:1.6
```

Now we should see all five containers running.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-51.png)

We can test if the Kafka connect application up and running by calling the following command.

```
curl -H "Accept:application/json" localhost:8083
```

You should see a similar response.

```
{"version":"2.7.1","commit":"61dbce85d0d41457","kafka_cluster_id":"O5xBxRkRRFS3hX194Edn2A"}
```

To check connectors that are currently running use the following command

```
curl -H "Accept:application/json" localhost:8083/connectors/
```

Now we are going to enable MySQL debezium connector for Kafka Connect, so we can start monitoring changes in our MySQL target table. We need to run the following command.

```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "database.server.name": "dbserver1", "database.include.list": "demo", "database.history.kafka.bootstrap.servers": "kafka:9092", "database.history.kafka.topic": "dbhistory.demo" } }'
```

To check if the Kafka topic was created we can exec into kafka docker container and run the following command.

```
docker exec -it kafka bash
bin/kafka-topics.sh --list --zookeeper zookeeper:2181
```

You should see that the Kafka topic was created.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-52.png)

Now we can check if our Kafka topic is receiving data by running the following command.

```
bin/kafka-console-consumer.sh --topic dbserver1.demo.bus_status --bootstrap-server '<container_id>':9092
```
You should see incoming messages in our Kafka topic.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-53.png)

```
{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"record_id"},{"type":"int32","optional":false,"field":"id"},{"type":"int32","optional":false,"field":"routeId"},{"type":"string","optional":true,"field":"directionId"},{"type":"int16","optional":true,"field":"predictable"},{"type":"int32","optional":false,"field":"secsSinceReport"},{"type":"int32","optional":false,"field":"kph"},{"type":"int32","optional":true,"field":"heading"},{"type":"double","optional":false,"field":"lat"},{"type":"double","optional":false,"field":"lon"},{"type":"int32","optional":true,"field":"leadingVehicleId"},{"type":"int64","optional":true,"name":"io.debezium.time.Timestamp","version":1,"default":0,"field":"event_time"}],"optional":true,"name":"dbserver1.demo.bus_status.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"record_id"},{"type":"int32","optional":false,"field":"id"},{"type":"int32","optional":false,"field":"routeId"},{"type":"string","optional":true,"field":"directionId"},{"type":"int16","optional":true,"field":"predictable"},{"type":"int32","optional":false,"field":"secsSinceReport"},{"type":"int32","optional":false,"field":"kph"},{"type":"int32","optional":true,"field":"heading"},{"type":"double","optional":false,"field":"lat"},{"type":"double","optional":false,"field":"lon"},{"type":"int32","optional":true,"field":"leadingVehicleId"},{"type":"int64","optional":true,"name":"io.debezium.time.Timestamp","version":1,"default":0,"field":"event_time"}],"optional":true,"name":"dbserver1.demo.bus_status.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"version"},{"type":"string","optional":false,"field":"connector"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"ts_ms"},{"type":"string","optional":true,"name":"io.debezium.data.Enum","version":1,"parameters":{"allowed":"true,last,false,incremental"},"default":"false","field":"snapshot"},{"type":"string","optional":false,"field":"db"},{"type":"string","optional":true,"field":"sequence"},{"type":"string","optional":true,"field":"table"},{"type":"int64","optional":false,"field":"server_id"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"query"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"},{"type":"struct","fields":[{"type":"string","optional":false,"field":"id"},{"type":"int64","optional":false,"field":"total_order"},{"type":"int64","optional":false,"field":"data_collection_order"}],"optional":true,"field":"transaction"}],"optional":false,"name":"dbserver1.demo.bus_status.Envelope"},"payload":{"before":null,"after":{"record_id":487,"id":8326,"routeId":7,"directionId":"7_0_7","predictable":1,"secsSinceReport":7,"kph":0,"heading":166,"lat":43.666602,"lon":-79.4111855,"leadingVehicleId":null,"event_time":1656980233000},"source":{"version":"1.8.0.Final","connector":"mysql","name":"dbserver1","ts_ms":1656980233000,"snapshot":"false","db":"demo","sequence":null,"table":"bus_status","server_id":223344,"gtid":null,"file":"mysql-bin.000011","pos":1532138,"row":0,"thread":null,"query":null},"op":"c","ts_ms":1656980233228,"transaction":null}
```

## EMR Setup

Now, let’s create an EMR cluster.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/msk23.jpg)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/msk24.jpg)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/msk25.jpg)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/msk26.jpg)

Keep note of the IAM Role associated with EC2 Instance Profile. You need to make sure this role also has AmazonS3FullAccess. This will allow the EC2 instance to load the python script from S3.

Use the same EC2 key pair as your VM.

When EMR is up and running, ssh into it and submit our streaming application by using the following command. When you can’t connect, check EMR Master security group and add SSH rule. Run this from your computer.

```
ssh -i ~/Location_of_your_pem_key/final-project-DE.pem hadoop@ec2-3-128-190-110.us-east-2.compute.amazonaws.com
```

This command can be found under Summary tab.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/msk27.jpg)

Here is our python script. It is available in the zip folder.

```
from pyspark.sql import SparkSession
from pyspark.sql.types import *
from pyspark.sql.functions import *

# NOTE: This variable needs to be reviewed if we are working with a new MSK
BOOTSTRAP_SERVERS='b-1.finalproject.m5cy8k.c3.kafka.ca-central-1.amazonaws.com:9092,b-3.finalproject.m5cy8k.c3.kafka.ca-central-1.amazonaws.com:9092,b-2.finalproject.m5cy8k.c3.kafka.ca-central-1.amazonaws.com:9092'

if __name__ == "__main__":
   spark = SparkSession.builder.getOrCreate()

   # NOTE: we cant load the schema file from the local machine anymore, so we have to pull it from s3
   schema = spark.read.json('s3://bus-service-wcd-eu-west-1/msk/bus_status_schema.json').schema

 # We have to connect to the bootstrap servers, instead of kafka:9092
   df = spark \
       .readStream \
       .format("kafka") \
       .option("kafka.bootstrap.servers", BOOTSTRAP_SERVERS) \
       .option("subscribe", "dbserver1.demo.bus_status") \
       .option("startingOffsets", "latest") \
       .load()

   transform_df = df.select(col("value").cast("string")).alias("value").withColumn("jsonData",from_json(col("value"),schema)).select("jsonData.payload.after.*")

   # NOTE: We cannot checkpoint to a local machine because we are working on the cloud. S3 is a reliable location for the cluster
   checkpoint_location = "s3://bus-service-wcd-eu-west-1/msk/checkpoint/sparkjob"

   table_name = 'bus_status'
   hudi_options = {
       'hoodie.table.name': table_name,
       "hoodie.datasource.write.table.type": "COPY_ON_WRITE",
       'hoodie.datasource.write.recordkey.field': 'record_id',
       'hoodie.datasource.write.partitionpath.field': 'routeId',
       'hoodie.datasource.write.table.name': table_name,
       'hoodie.datasource.write.operation': 'upsert',
       'hoodie.datasource.write.precombine.field': 'event_time',
      'hoodie.upsert.shuffle.parallelism': 100,
       'hoodie.insert.shuffle.parallelism': 100
   }

   s3_path = "s3://bus-service-wcd-eu-west-1/msk/routes"

   def write_batch(batch_df, batch_id):
       batch_df.write.format("org.apache.hudi") \
       .options(**hudi_options) \
       .mode("append") \
       .save(s3_path)

   transform_df.writeStream.option("checkpointLocation", checkpoint_location).queryName("wcd-bus-streaming").foreachBatch(write_batch).start().awaitTermination()
```

We need to prep the python code, see the Notes in the pyspark_job.py file for the changes.

copy the python file over to s3 bucket: s3://''/jars/pyspark_job.py

```
aws s3 cp EMR-setup/pyspark_job.py 
s3://bus-service-wcd-eu-west-1/msk/jars/pyspark_job.py
```

## Spark-submit

```
# SSH into the emr-master from your computer
ssh -i 01-setup-ec-vm/nifi-ec-vm.pem hadoop@ec2-3-97-52-3.ca-central-1.compute.amazonaws.com

#Submitting the spark streaming app using  client mode.
spark-submit --master yarn --deploy-mode client --name wcd-streaming-app --jars /usr/lib/hudi/hudi-spark-bundle.jar,/usr/lib/spark/external/lib/spark-avro.jar --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.3.1 --conf "spark.serializer=org.apache.spark.serializer.KryoSerializer" --conf "spark.sql.hive.convertMetastoreParquet=false" s3://bus-service-wcd-eu-west-1/msk/jars/pyspark_job.py

#Submitting the spark streaming app using  cluster mode.
spark-submit --master yarn --deploy-mode cluster --name wcd-stremaing-app --jars /usr/lib/hudi/hudi-spark-bundle.jar,/usr/lib/spark/external/lib/spark-avro.jar --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.3.1 --conf "spark.serializer=org.apache.spark.serializer.KryoSerializer" --conf "spark.sql.hive.convertMetastoreParquet=false" s3://bus-service-wcd-eu-west-1/msk/jars/pyspark_job.py
```

Shortly after submitting your spark streaming app, you will be able to see the data in Athena table.

## Athena

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/Athena-1.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/Athena-2.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/Athena-3.png)

Bulk add columns for your table:

```
record_id int, id int, routeId int, directionId string, kph int, predictable int, secsSinceReport int, heading int, lat double, lon double, leadingVehicleId int, event_time date
```

Make sure the data types are as below:

```
record_id INT NOT NULL AUTO_INCREMENT,
id INT NOT NULL,
routeId INT NOT NULL,
directionId VARCHAR(40),
kph INT NOT NULL,
predictable BOOLEAN, ====> int
secsSinceReport INT NOT NULL,
heading INT,
lat REAL NOT NULL ====>  double
lon REAL NOT NULL ====> double
leadingVehicleId INT,
event_time DATETIME DEFAULT NOW()
```

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/Athena-4.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/Athena-5.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/Athena-6.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/Athena-7.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/Athena-8.png)

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/Athena-11.png)

Now check the results:

```
select * from "bus-service-wcd-eu-west-1"."routes" limit 100
```

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/Athena-12.png)

You will get the error:

```
No output location provided. An output location is required either through the Workgroup result configuration setting or as an API input.
```

To fix it:

You have to go to Athena settings and specify an output path as shown above.

![alt_text](https://weclouddata.s3.amazonaws.com/images/data_engineer/Athena-last.jpg)



 


















