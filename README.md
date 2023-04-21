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














