# IoT-Workshop-Master
## Introduction
This repository contains supporting assets for Cloudera's hands-on workshop for working with data generated by an Eclipse Kura/ESF gateway (specifically the example ["Heater Demo" application](https://marketplace.eclipse.org/node/3452467) available on the Eclipse Marketplace).

Exercises will walk the participant through the process of:
1. Creating a Kafka topic, where raw telemetry data can be landed
2. Creating a Kudu table via Impala, where processed telemetry data can be stored
3. Developing, compiling, and deploying a Spark Streaming application (to consume telemetry data from Kafka, process that data, and store it in Kudu)
4. Generating data from Kura, bridging MQTT to Kafka, and accessing telemetry data stored in Kudu via Impala

### Requirements

Exercises require a Cloudera EDH cluster (version 6.0 or higher) configured with the following services (including all dependencies):
* Kafka
* Kudu
* Impala
* Spark
* Hue

Java 8, Maven, and sbt are required for building modules.

Docker will be used to deploy a Kura emulator and generate data.


## Hands-On Exercise 1: Create a Kafka Topic
The goal of this exercise is to create a Kafka topic. This Kafka topic will serve as the initial landing zone for telemetry data within the Cloudera environment. Ultimately, data generated by the "Heater Demo" application will be streamed to an MQTT broker and bridged into this Kafka topic.

Create this Kafka topic [using Apache Kafka command-line tools](https://www.cloudera.com/documentation/kafka/latest/topics/kafka_command_line.html).

The command `$ kafka-topics --create ...` should be used to create a new topic.

The command `$ kafka-topics --list ...` should be used to verify creation.

The topic name is arbitrary and may be freely chosen, but note that it will be used in other exerises.


## Hands-On Exercise 2: Create a Kudu Table (via Impala) using Hue Impala Query Editor
The goal of this exercise is to create a Kudu table. This Kudu table will serve as the persistent storage layer for processed telemetry data within the Cloudera cluster. Ultimately, data generated by the "Heater Demo" application will be streamed to an MQTT broker, bridged into the Kafka topic created in exercise 1, processed by a Spark Streaming application, and stored in this table.

Create this Kudu table via Impala using Hue Impala Query Editor, taking advantage of [Kudu and Impala's tight integration](http://www.cloudera.com/documentation/enterprise/latest/topics/kudu_impala.html).

First, use a browser to login to Hue and navigate to Impala Query Editor.

Within Hue Impala Query Editor:
1. Create a new database using Impala's `CREATE DATABASE` statement. The name of this database is arbitrary, but note that it will be used in other exercises.
2. Using the newly created database, use Impala's `CREATE TABLE` statement to define a new Kudu table (by specifying the table as `STORED AS KUDU`). This exercise is to create a "tall" table, containing columns named "millis" (type: BIGINT), "id" (type: STRING), "metric" (type: STRING), and "value" (type: STRING). The primary key for this table should be compound, comprised of "millis", "id", and "metric". It should be partitioned by a hash of "id" and "metric" values. The number of partitions should be a small multiple of the number of Kudu Tablet Servers.
3. Verify proper design of the Kudu schema using Impala's `DESCRIBE` statement.

![alt tag](http://raw.githubusercontent.com/jcoopere/IoT-Workshop/blob/master/screenshots/describeTable.png)

Also verify successful creation of the Kudu table by navigating to the Kudu Master web interface and clicking on "Tables". Note that the "Table Name" here should be of the form "impala::<your_database>.<your_table>", since the table was created using Impala.


## Hands-On Exercise 3: Deploy a Spark Streaming Application to Process and Store Telemetry Data
The goal of this exercise is to complete, compile, and deploy a Spark Streaming application. This Spark Streaming application will consume raw telemetry data from the Kafka topic created in exercise 1, deserialize the KuraPayload Protobuf encoded messages into Scala objects, unpack the metrics from those objects, and store them in the Kudu table created in exercise 2.

Most of the work for this has already been done and exists within the `streaming/` subdirectory. Clone this repository on a Spark gateway node where the completed application can be submitted from.

The Spark Streaming application file **which must be completed before compiling** can be found at `streaming/src/main/scala/com/cloudera/demo/iiot/IIoTDemoStreaming.scala`. See comments in that file for instructions and hints. Code requiring completion is indicated by `TODO:`.

Compiling this application requires [sbt](https://www.scala-sbt.org/), which can be installed via `yum` on a RedHat Package Manager (RPM) based Linux host using the following commands:
```
$ curl https://bintray.com/sbt/rpm/rpm | sudo tee /etc/yum.repos.d/bintray-sbt-rpm.repo

$ sudo yum install sbt
```

To build the application, with sbt installed navigate to the `streaming/` subdirectory and execute the command `$ sbt clean assembly`. 

Successful compilation should yield an uber jar (found in `streaming/target/scala-2.11/iiot-demo-assembly-1.0.jar`) which can be submitted to the Cloudera cluster from a gateway node as follows (relative to the `streaming/` subdirectory, and replacing with your hosts and names):
```
$ spark-submit \
  --master yarn \
  --deploy-mode client \
  --class com.cloudera.demo.iiot.IIoTDemoStreaming \
  target/scala-2.11/iiot-demo-assembly-1.0.jar \
  <your_kafka_broker_1>:9092,<your_kafka_broker_2>:9092,<your_kafka_broker_3>:9092 \
  <your_kudu_master>:7051 \
  <your_kafka_topic_name> \
  <your_kudu_table_name>
```
**Note:** Kudu tables created via Impala should be prepended by "impala::", as shown in the Kudu Master UI list of tables. `<your_kudu_table_name>` should match this.

Verify that the Spark application is running by using Hue Job Browser, and/or YARN.

Note that no data will actually start flowing until exercise 4.

Also note that when using `--deploy-mode client` the application will terminate when the shell exits, unless `nohup` is used. Alternatively, submit the job to the cluster using `--deploy-mode cluster`.


## Hands-On Exercise 4: Start Data Flowing and Access Data with Impala
The goal of this exercise is to get "Heater Demo" telemetry data flowing through the pipeline (so that it is landing in Kafka, being consumed and processed by Spark Streaming, and persisted in Kudu), and exploring that stored data using Impala.

### Hands-On Exercise 4.1: Start Data Flowing
In a real-world solution data ingestion from the edge would be handled by ESF and Everyware Cloud, and Everyware Cloud's built-in capabilities for bridging MQTT messages to Kafka topics would be leveraged. 

In this workshop, we will simulate these capabilities using a "Heater Demo" application deployed on Kura/ESF. The "Heater Demo" application will publish properly formatted KuraPayload Protobuf telemetry data to an MQTT broker, and a standalone Java application will be used to simulate the protocol bridging capabilities of Everyware Cloud, consuming messages from the MQTT broker and publishing them to a Kafka topic. Successful completion of *Hands-On Exercise 4.1* will result in Kura/ESF-generated data landing in the Kafka topic created in *Hands-On Exercise 1*. That data will be consumed by the Spark Streaming application deployed in *Hands-On Exercise 3*, and persisted in the Kudu table created in *Hands-On Exercise 2*.

#### Hands-On Exercise 4.1.1: Deploy Kura, Configure MqttDataTransport, and Install the "Heater Demo" application
A Kura emulator can easily be deployed within a Docker container. To install Docker using `yum` and start a Kura within a Docker container, execute the following commands on a RedHat Package Manager (RPM) based Linux host (alternatively, if you wish to install Docker and deploy Kura on a different operating system, find instuctions for installing Docker on the desired OS and only use the final command below for fetching and starting the Kura emulator in Docker):
```
# Install Docker
yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

yum-config-manager --disable docker-ce-edge

yum install -y docker-ce

systemctl start docker

# Start Kura in Docker 
#(do not exit the terminal session or close the OSGi console or it will kill the container)
docker run -ti -p 8080:8080 ctron/kura-emulator:3.0.0
```

Note that exiting the terminal session or closing the OSGi console will kill the Docker container, causing Kura to terminate.

Once Kura has started, access the console in a browser window by navigating to `http://<your_kura_host>:8080/kura` and authenticating with `admin/admin` credentials.

Once authenticated, follow the steps in the following screenshots to configure Kura to publish to the Eclipse-managed public MQTT broker and install the "Heater Demo" application to begin sending sample data:

##### Configure Kura MqttDataTransport
![alt tag](http://github.com/jcoopere/IoT-Workshop/blob/master/screenshots/kura1.png)
![alt tag](http://github.com/jcoopere/IoT-Workshop/blob/master/screenshots/kura2.png)
![alt tag](http://github.com/jcoopere/IoT-Workshop/blob/master/screenshots/kura3.png)

##### Install "Heater Demo" on Kura
**Note:** This will require opening a new browser window and navigating to [Eclipse Marketplace "Heater Demo"](https://marketplace.eclipse.org/node/3452467)

![alt tag](http://github.com/jcoopere/IoT-Workshop/blob/master/screenshots/heater-1.png)
![alt tag](http://github.com/jcoopere/IoT-Workshop/blob/master/screenshots/heater-2.png)
![alt tag](http://github.com/jcoopere/IoT-Workshop/blob/master/screenshots/heater-3.png)
![alt tag](http://github.com/jcoopere/IoT-Workshop/blob/master/screenshots/heater-4.png)

#### Hands-On Exercise 4.1.2: Build and Deploy the Java Protocol Bridge Application
For this workshop, a simple, standalone Java application can be used in order to simulate Everyware Cloud protocol bridging and message routing capabilities.

The source code for this application lives within the `bridge/` subdirectory of this repository, and requires Java 8 and [Maven](https://maven.apache.org/install.html) to build.

Clone this repository on a cluster node, navigate into the `bridge/` subdirectory, and run the following command to build the application: `$ mvn clean package`

Successful compilation will yield a jar file at the following location relative to the `bridge/` subdirectory: `target/MqttKafkaBridge.jar`

Start the MQTT-to-Kafka bridge application with a command similar to the following, replacing your specific parameters:
```
java -cp target/MqttKafkaBridge.jar com.cloudera.demo.mqtt_kafka_bridge.SimpleMqttKafkaBridge \
	--mqtt-broker tcp://<your_mqtt_broker>:1883 \
	--kafka-broker-list <your_kafka_broker_1>:9092,<your_kafka_broker_2>:9092,<your_kafka_broker_3>:9092 \
	--mqtt-topic <your_kura_topic.context.account-name>/# \
	--kafka-topic <your_kafka_topic>
```
**Notes:**
* `--mqtt-broker` requires to the host/port for your MQTT broker, which was configured in Kura (Eclipse sandbox broker address is "tcp://iot.eclipse.org:1883")
* `--kafka-broker-list` requires a CSV list of Kafka host/port pairs
* `--mqtt-topic` requires the MQTT topic string that the application should subscribe to and bridge to Kafka (the first segment of the topic string was configured in Kura as `topic.context.account-name`)
* `--kafka-topic` requires the Kafka topic name string that the application should publish Kafka messages to, which should be the name of the Kafka topic created in *Hands-On Exercise 1*

When this application is running properly, it will log to console each time a message is bridged. If Kura has been properly configured with the "Heater Demo" installed, this application should be logging bridged messages regularly and data should be landing in Kafka, where it can be consumed by Spark Streaming and ultimately persisted in Kudu.


### Hands-On Exercise 4.2: Access Data with Impala
Upon completion of *Hands-On Exercise 4.1*, assuming assets from all other exercises have been properly deployed and are active, data should be flowing end-to-end, and processed telemetry data should now be available in Kudu.

The goal of this final exercise will be to access and explore data in Kudu using Impala.

Consulting the [Impala SQL Language Reference](https://www.cloudera.com/documentation/enterprise/latest/topics/impala_langref.html#langref) may be useful.

Similar to *Hands-On Exercise 2*, use a browser to login to Hue and navigate to Impala Query Editor.

Within Hue Impala Query Editor, develop queries which accomplish the following:
1. Display a count of all rows within a table
2. Display all distinct metrics names from a table
3. Display the 100 most recent rows from a table, displaying time as a human-readable string
4. Display most recent value for each metric in a table
![alt tag](http://github.com/jcoopere/IoT-Workshop/blob/master/screenshots/mostRecentMetrics.png)
5. Display daily maximum, minimum and average values, as well as daily total record count, for metric "temperatureInternal"
![alt tag](http://github.com/jcoopere/IoT-Workshop/blob/master/screenshots/summaryByDay.png)
6. Use Hue to create a graph of the 100 most recent values for metric "temperatureInternal" (using millis on the x-axis and "temperatureInternal" on the y-axis)
![alt tag](http://github.com/jcoopere/IoT-Workshop/blob/master/screenshots/internalTemperatureGraph.png)
7. Display the 100 most recent rows, with distinct metrics 'pivoted' into their own columns.
![alt tag](http://github.com/jcoopere/IoT-Workshop/blob/master/screenshots/tallToWideTableExample.png)

