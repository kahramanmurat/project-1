# Final Project

![alt text](https://weclouddata.s3.amazonaws.com/images/data_engineer/final-project-0.png)

The main objective of the final project is to create a live application that captures GPS data transmitted by IoT devices installed on buses. Our project employs a range of technologies, including Apache NiFi, CDC (Debezium), Kafka (MSK), Spark Structural Streaming, Docker, MySQL, and Superset for data visualization. The project's data source is the TTC bus data provided by the open API named Rest Bus, and a sample of the streaming bus data is available.

``` [{"id":"7","title":"7-Bathurst","_links":{"self":{"href":"http://restbus.info/api/agencies/ttc/routes/7","type":"application/json","rel":"http://restbus.info/_links/rel/full","rt":"route","title":"Full configuration for ttc route 7-Bathurst."},"to":[{"href":"http://restbus.info/api/agencies/ttc/routes/7","type":"application/json","rel":"http://restbus.info/_links/rel/full","rt":"route","title":"Full configuration for ttc route 7-Bathurst."}],"from":[{"href":"http://restbus.info/api/agencies/ttc","type":"application/json","rel":"via","rt":"agency","title":"Transit agency ttc details."}```

http://restbus.info/api/agencies/ttc/routes

By fetching data from the Rest Bus API, we can display it on Google Maps and utilize it to enhance specific routes. 

The retrieved data will be stored in a MySQL database. To achieve this, we will utilize Kafka Connect, Kafka, and Zookeeper applications, which will monitor the MySQL database's Write Ahead Logs. These logs contain data about insertions, updates, and deletions, and they will be written to a MySQL table.







