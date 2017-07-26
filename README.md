<h1>ConnectX IoT platform</h1>


<h2>Server setup</h2><br>

<b>AWS EC2 Ubuntu Server 64 bit</b><br>
Launch new instance, set security group at allow TCP and http connections<br>
Dowload the pem file<br>
Use puttygen to convert pem to .ppk<br>
To SSH from putty:<br>
Host name : Replace (PUBLIC IP) by the publis IP<br>
ubuntu@(PUBLIC IP) Port 22 SSH<br>
Connection->SSH->Auth->upload private key file for uthentication (.ppk)<br>

<b>Filezilla:</b><br>
Site manager<br>
Host public ip, port blank<br>
Protocol SFTP<br>
Logon type : Normal<br>
user: ubuntu<br>
password: blank<br>
edit->settings->sftp add the ppk key file<br>
<br>
<b>SSH into instance </b><br>




<h2>Getting server ready</h2>

<b>Install GCC and MAKE</b>

```
sudo apt-get install make
sudo apt-get install gcc
```
<br>
<b>Install Erlang:</b>

```
wget https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb
sudo dpkg -i erlang-solutions_1.0_all.deb
sudo apt-get update
sudo apt-get install erlang
```
<br>
<b>Install JAVA:</b>

```
java -version
sudo apt-get update
sudo apt-get install default-jdk
```

<br>
<b>Install Scala:</b><br>

```
sudo apt-get install scala
```

<br>
<b>Install Spark:</b>

```
sudo apt-get install git
```
Next, go to https://spark.apache.org/downloads.html and download a pre-built for Hadoop 2.7 version of Spark (preferably Spark 2.0 or later).
Then download the .tgz file and remember where you save it on your computer.<br>

```
tar xvf spark-2.0.2-bin-hadoop2.7.tgz
```

Check Spark installation :<br>

```
cd spark-2.0.2-bin-hadoop2.7.tgz
cd bin
./spark-shell
```
<b>Install MongoDB:</b><br>
https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu/<br>
<br>

<b>Install NodeJS, ExpressJS and NPM:</b>

```
Sudo apt-get update
Sudo apt-get install nodejs
Sudo apt-get install npm
Npm install ejs --save
```

<b>Install kafka:</b>

```
wget http://www-eu.apache.org/dist/kafka/0.11.0.0/kafka_2.11-0.11.0.0.tgz
tar -xzf kafka_2.11-0.11.0.0.tgz
cd kafka_2.11-0.11.0.0
```

If there are problems with java heap:<br>

```
vi bin/kafka-server-start.sh
replace -Xmx256M -Xms128M
vi bin/sookeeper-server-start.sh
replace -Xmx256M -Xms128M
```



<h2>Setup Kafka and MQTT</h2><br>

<b>Build emqtt broker</b><br>

1. clone emq-relx project

```
git clone https://github.com/emqtt/emq-relx.git
```

2. Add DEPS of our custom plugin (https://github.com/SkylineLabs/emqttd_kafka_bridge) in the makefile

```
cd emq-relx
vi Makefile
DEPS += emqttd_kafka_bridge
Add line:
dep_emqttd_kafka_bridge = git https://github.com/SkylineLabs/emqttd_kafka_bridge.git master
```

3. Add plugin in relx.config

```
cd emq-relx
vi relx.config
Add the line:
{emqttd_kafka_bridge, load},
```

4. Build

```
cd emq-relx
make
```
Note: You will have to edit the configurations of the bridge to set the kafka IP address and port.<br>

```
cd emq-relx/deps/emqttd_kafka_bridge/etc
vi emqttd_kafka_bridge.config
```
```
[
	{emqttd_kafka_bridge, [{values, [
	%%edit this to address and port on which kafka is running
	{bootstrap_broker, {"1.1.1.1", 9092} },
	%% partition strategies can be strict_round_robin or random
	{partition_strategy, strict_round_robin},
	%% Change the topic to produce to kafka. Default topic is "Kafka". It is on this topic that the messages will be sent from the broker to a kafka consumer
	{kafka_producer_topic, <<"kafka">>}
	]}]}
].
```
<br>

<b>Start the emqttd broker:</b>

```
cd emq-relx/_rel/emqttd
./bin/emqttd start
./bin/emqttd_ctl plugins load emqttd_kafka_bridge
```

<b>Start Kafka Server</b><br>
cd kafka_2.11-0.11.0.0/<br>
1) Start the zookeeper either in the background or in a new terminal<br>
Background
	
```
nohup bin/zookeeper-server-start.sh config/zookeeper.properties &
```
	
New Terminal
	
```
bin/zookeeper-server-start.sh config/zookeeper.properties
```

2) Start the kafka server either in the background or in a new terminal<br>
Background:

```
nohup bin/kafka-server-start.sh config/server.properties &
```
	
New Terminal
	
```
bin/kafka-server-start.sh config/server.properties
```

<b>To test whether the messages coming from MQTT are reaching kafka:</b><br>
1) Use eclipse paho(or any other MQTT client) for connecting to your emqttd broker. <br>
	Broker URI : tcp://ipaddress_of_running_broker:1883<br>
	Connect and subscribe to a topic. Publish a message using that topic and see if it is received. <br>
	If received, the emqttd broker along with the bridge is up and running.<br>

2) Start a kafka consumer:

```
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic kafka
```

Set --topic as "kafka". Note that this should be same as that set in the emqttd_kafka_bridge.config file.<br>

3) First check if kafka has started without issues by producing messages locally<br>
Start a producer in a different terminal from the consumer and produce to topic "kafka":

```
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic kafka
```
send a message from producer and see if received by consumer. If it is, kafka is working properly<br>
	
4) Send a message on a random topic from eclipse Paho which was connected in the first step.<br>
The folling should be received by the kafka consumer :

```
{"topic":"yourtopic", "message":[yourmessage]}
```
This is the format in which kafka will receive the MQTT messages<br>

<h2>Setup MongoDB</h2><br>
Once Kafka and MQTT is set, the mongoDB schema can be loaded by reloading the MongGo dump provided.<br>
Note the MongoDB version as it will decide the version of MongoDB libraries used in MEAN stack app and Spark application.<br>
Start the Mongo service and check if the Mongo dump has been loaded and accessible.<br>
Tested version of MongoDB 3.4.1 while development.<br>
Mongo dump has been provided in the ‘Mongo DB’ folder submitted<br>

```
sudo service mongod start
```
<h2>e.	Setup security features</h2><br>
<b>Monitoring connection status of devices</b><br>
Add Plugin (emqttd_mongodb_plugin) for monitoring status of IoT devices.<br>

Requirements:<br>
1) All devices connecting to the broker need to have a client id of the form -<br>
ProjectName/ThingType/ThingName<br>
eg:<br>
HomeAutomation/Kitchen/Light<br>

2) Create a mongoDB database "thingsDB"<br>
Insert in collection "HomeAutomation" the following document:

```
{_id:"Kitchen/Light", "connected":0}
```
Insert 0 because initially the device is not connected<br>
Install Plugin:<br>
1. Add DEPS of our custom made plugin (https://github.com/SkylineLabs/emqttd_mongodb_plugin) in the makefile
	
	```
	vi emq-relx/Makefile
	DEPS += emqttd_mongodb_plugin
	dep_emqttd_kafka_bridge = git https://github.com/SkylineLabs/emqttd_mongodb_plugin.git master
	```
3. Add plugin in relx.config

	```
	vi emq-relx/relx.config
	{emqttd_mongodb_plugin, load},
	```
4. Build<br>

	```
	cd emq-relx && make
	```
  
Test the plugin:<br>
When a client (say HomeAutomation/Kitchen/Light) connects via MQTT,<br>
the corresponding document in the "thingsDB" in the corresponding collection "HomeAutomation" will get updated to :

```
{_id:"Kitchen/Light", "connected":1, "timestap":24-07-2017 20:34:37}
```
The purpose of timestamp:<br>
1) If connected is 0, then timestamp tells the last seen of the device<br>
2) If connected is 1, then timestamp tells the time since which the device is online<br>



<b>Authentication, ACL with MongoDB</b>

```
./bin/emqttd_ctl plugins load emq_auth_mongo
```

1) Create database "mqtt" and two collections

```
use mqtt
db.createCollection("mqtt_user")
db.createCollection("mqtt_acl")
```

2) In mqtt_user, store usernames and passwords
  
  ```
  {
      username: "user",
      password: "password hash",
      is_superuser: boolean (true, false)
  }
  ``` 
3) In mqtt_acl, store the restrictions of publish and subscribe
  
  ```
  {
      username: "username",
      publish: ["topic1", "topic2", ...],
      subscribe: ["subtop1", "subtop2", ...],
      pubsub: ["topic/#", "topic1", ...]
  }
  ```
4) create an admin that can pubsub to all topics

```
db.mqtt_user.insert({username: "admin", password: "password hash", is_superuser: true})
db.mqtt_acl.insert({username: "admin", pubsub: ["#"]})
```



<h2>Setting up the web-application</h2><br>
Start new project by

```
express --ejs
```
package.json contains all the dependencies for the project<br>
 
Install the dependencies using command inside the app directory:

```
  npm install
  Download the code using
  git clone https://github.com/SkylineLabs/iot-management-webapp.git
```
<br>
<b>Folders content:</b><br>
bin: Contains www file which runs the server<br>
public: Contains files and folders accessible throughout the project for rendering<br>
_routes: Files which interact with DB and fetches the data for webpages dynamically<br>
_views: The UI part of the webpage<br>
_App.js: main file for the project, which handles all the requests made to the app; and redirects the <br>
<br>
Run the server by executing the following command.<br>

```
node www
```

<h2>Starting the Spark application</h2><br>
Download the necessary libraries to compile the Spark application.<br>
Save the mentioned jar files in the ‘jars’ folder of Apache Spark installation.<br>
  spark-streaming-kafka-0-8-assembly_2.11-2.1.0.jar<br>
  spark-streaming-kafka-0-8-assembly_2.11-2.1.0-sources.jar<br>
  org.eclipse.paho.client.mqttv3-1.0.2.jar<br>
  mongodb-driver-core-3.4.1.jar<br>
  mongodb-driver-3.4.1.jar<br>
  mongo-spark-connector_2.10-2.0.0.jar<br>
  mail-1.4.1.jar<br>
  courier_2.12-0.1.4.jar<br>
  courier_2.12-0.1.4-sources.jar<br>
  bson-3.4.1.jar<br><br>


The jar versions mentioned above have been tested while development.<br>

Clone our Spark Streaming repository <br>
  git clone https://github.com/SkylineLabs/spark-kafka-rules-engine.git<br>

Run the Spark application using the following commands (In the same folder as the Scala file)<br>
The following commands are for Spark installed at location “/home/ubuntu/spark”

```
scalac *.scala -classpath "/home/ubuntu/spark/jars/*”
jar -cvf Iot.jar in/skylinelabs/spark/*.class /home/ubuntu/spark/jars/*
spark-submit --class in.skylinelabs.spark.IoT --master local Iot.jar
```



<br><br>


<h2>Connect-X IoT platform is ready to go!</h2>
Instructions to use the IoT platform are present in the project report.

To test the platform, use Eclipse paho or our Client SDKs available at:
https://github.com/SkylineLabs/connectX-IoT-platform-sdk.git



License
-------

Apache License Version 2.0




