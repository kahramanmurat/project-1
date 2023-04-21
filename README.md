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














