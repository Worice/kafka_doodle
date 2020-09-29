# System
Processor				: Intel(R) Core(TM) i7-10750H CPU @ 2.60GHz
Memory					: 8168MB (2881MB used)
Machine Type			: Virtual (VirtualBox)
Operating System		: LUbuntu 18.04.5 LTS

# Configuration
Install JDK:
```sh 
sudo apt-get install openjdk-8-jdk
```
Install processes sych service Apache Zookeeper (Zk):
``` sh
sudo apt-get install zookeeperd
```
Verify Zk status (active/inactive)
```sh
sudo systemctl status zookeeper`
```
Add kafka user profile
```sh
sudo useradd kafka -m`
```
Assign kafka sudo rights
```sh
sudo adduser kafka sudo
```
Download kafka, set version 
```sh
wget http://www.apache.org/dist/kafka/2.x.0/kafka_2.x-x.x.x.tgz
```
Create ~/kafka and extract there
```sh
sudo tar xvzf kafka_2.x-x.x.x.tgz --strip 1
```
Since an operation such as renaming is not feasible on a topic, you will appreciate being able to eliminate them. Via text editor (i.e. nano), add
`>> delete.topic.enable=true`
in
```sh
sudo nano ~/kafka/config/server.properties
```
Verify kafka status
```sh
sudo journalctl -u kafka
```
# Start up 
Kafka and zookeeper may suffer problems due to data logs from previus sessions. A Kafka start up error where it is unable to access data dir should be addressed by removing both:
```sh
sudo rm -rf /tmp/kafka-logs 
sudo rm -rf /tmp/zookeeper 
```
They will be re-created at the next startup.
## Standard input
Start a kafka server. In `~/kafka` folder, with two different terminals:
```sh
bin/zookeeper-server-start.sh config/zookeeper.properties
bin/kafka-server-start.sh config/server.properties
```
Let's create an input topic. Open a new terminal and:
```sh
bin/kafka-topics.sh --create \
    --bootstrap-server localhost:9092 \
    --replication-factor 1 \
    --partitions 1 \
    --topic streams-plaintext-input
```
and an output topic:
```sh
bin/kafka-topics.sh --create \
    --bootstrap-server localhost:9092 \
    --replication-factor 1 \
    --partitions 1 \
    --topic streams-plaintext-output \
    --config cleanup.policy=compact
```
We can count word occurrences with a build in function:
```sh
bin/kafka-run-class.sh org.apache.kafka.streams.examples.wordcount.WordCountDemo
```
It will allow us to read from the topic `streams-plaintext-input` without  showing any standard output. The results go to the topic `streams-wordcount-output`.
In order to write new data for the input topic, we start the console producer in a separate terminal:
```sh
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic streams-plaintext-input
```
We can inspect the output of wordcount by reading from its output topic in a new terminal:
```sh
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
    --topic streams-wordcount-output \
    --from-beginning \
    --formatter kafka.tools.DefaultMessageFormatter \
    --property print.key=true \
    --property print.value=true \
    --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
    --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
```
## Doodle data
Now we can extract doodle data. We also have to consider that for Kafka each message is array of bytes. How it treats it, depends by client's application (producer, consumer, etc).  Kafka Producer and Consumer uses Deserializer, Serializer to transform from/to Array of bytes to/from business object (String, POJO).

With
```sh
jq -rc . sampledata.json | kafka-console-producer --broker-list localhost:9092 --topic  stream-test-topic
```
data are fed to the topic not as text, but as JSON, holding hence their format:
```sh
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic streams-plaintext-input --from-beginning
```
# My Kafka App
My plan was to use the Pipe app from apache manual to redirect data for analysis, cleaning and performance testing. Neverthless, my actual grasp on serializers and deserializers is still to primitive. Because of that, I have not been able to further proceed.

In any case, I generate a `pom` file as a manifestation of my intentions:
```sh
        mvn archetype:generate \
            -DarchetypeGroupId=org.apache.kafka \
            -DarchetypeArtifactId=streams-quickstart-java \
            -DarchetypeVersion=2.5.0 \
            -DgroupId=streams.examples \
            -DartifactId=streams.examples \
            -Dversion=0.1 \
            -Dpackage=myapps
```
`Pipe` will need an output file. Simply generate one:
```sh
bin/kafka-topics.sh --create \
	--bootstrap-server localhost:9092 \     
	--replication-factor 1     --partitions 1 \     
	--topic streams-plaintext-output \
	--config cleanup.policy=compact
```
Now, `mvn clean package && mvn exec:java -Dexec.mainClass=myapps.Pipe` and the app is compiled and running.
You may have problems with `slf4j`. If so, slap in `pom.xml` this bad boy:
```sh
     <dependency> 
         <groupId>org.slf4j</groupId> 
         <artifactId>slf4j-jdk14</artifactId> 
         <version>1.7.25</version> 
     </dependency> 
```
# Comments

