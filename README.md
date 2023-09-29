# Ros_Python_Kafka_Spark

## My command list to Integrate Ros Python Kafka and Spark for Streaming Data analytics

### Below are the Steps

1. Create a Google Cloud VM Instance with machine type e2-medium or higher.
2. Intall jdk and wget
```
sudo apt install default-jdk
sudo apt install wget
```

3. Install Kafka
```
wget https://archive.apache.org/dist/kafka/3.5.0/kafka_2.12-3.5.0.tgz
tar -xvzf kafka_2.12-3.5.0.tgz
```
4. Export Kafka_home
```
export KAFKA_HOME=/home/mani_dataops/youtubekafka/kafka_2.12-3.5.0
```
5. Start Zookeeper
```
sudo netstat -tulpn
sudo ${KAFKA_HOME}/bin/zookeeper-server-start.sh ${KAFKA_HOME}/config/zookeeper.properties > ${KAFKA_HOME}/logs/zookeeper.log 2>&1 &
```
6. Start Kafka
```
sudo ${KAFKA_HOME}/bin/kafka-server-start.sh ${KAFKA_HOME}/config/server.properties > ${KAFKA_HOME}/logs/broker1.log 2>&1 &
sudo ${KAFKA_HOME}/bin/connect-distributed.sh ${KAFKA_HOME}/config/connect-distributed.properties > ${KAFKA_HOME}/logs/connect.log 2>&1 &
```
7. Create Topic in KAFKA
```
sudo ${KAFKA_HOME}/bin/kafka-topics.sh   --create   --topic FirstTopic   --bootstrap-server localhost:9092   --partitions 1   --replication-factor 1   --config "cleanup.policy=compact"   --config "retention.ms=604800000"   --config "segment.bytes=1073741824"
```
8. Test Kafka.
    
Produce something
```
sudo ${KAFKA_HOME}/bin/kafka-console-producer.sh   --topic FirstTopic   --bootstrap-server localhost:9092   --property "acks=all"   --property "compression.type=gzip"   --property "batch.size=16384"   --property "parse.key=true"   --property "key.separator=:"
```
You can enter {'hi':"kafka"}

Subscribe and Check message
```
sudo ${KAFKA_HOME}/bin/kafka-topics.sh --list --bootstrap-server localhost:9092
sudo ${KAFKA_HOME}/bin/kafka-console-consumer.sh   --topic FirstTopic   --bootstrap-server localhost:9092   --from-beginning   --max-messages 100   --property "print.key=true"   --property "print.value=true"
```

### Install Spark 

1. Install jdk
```
sudo apt-get update -y;
sudo apt-get install openjdk-8-jdk -y;
```

2. Download and Extract Spark files
```
wget https://archive.apache.org/dist/spark/spark-3.0.1/spark-3.0.1-bin-hadoop2.7.tgz;
tar -xvzf spark-3.0.1-bin-hadoop2.7.tgz;
```

3. Move files to /etc
```
sudo mkdir /etc/spark;
sudo chown -R ubuntu /etc/spark;
sudo cp -r spark-3.0.1-bin-hadoop2.7/* /etc/spark/;
```
 
4. Update Spark env file
```
sudo cp /etc/spark/conf/spark-env.sh.template /etc/spark/conf/spark-env.sh;
sudo nano /etc/spark/conf/spark-env.sh;
```

with following lines at the end of spark-env.sh
```
PYSPARK_PYTHON=/usr/bin/python3
PYSPARK_DRIVER_PYTHON=/usr/bin/python3
```

### Test Integration with KAFKA
1. Start Kafka
```
sudo ${KAFKA_HOME}/bin/zookeeper-server-start.sh ${KAFKA_HOME}/config/zookeeper.properties > ${KAFKA_HOME}/logs/zookeeper.log 2>&1 &
sudo ${KAFKA_HOME}/bin/kafka-server-start.sh ${KAFKA_HOME}/config/server.properties > ${KAFKA_HOME}/logs/broker1.log 2>&1 &
sudo ${KAFKA_HOME}/bin/connect-distributed.sh ${KAFKA_HOME}/config/connect-distributed.properties > ${KAFKA_HOME}/logs/connect.log 2>&1 &
sudo netstat -tulpn
sudo ${KAFKA_HOME}/bin/kafka-topics.sh --list --bootstrap-server localhost:9092
sudo ${KAFKA_HOME}/bin/kafka-topics.sh   --create   --topic FirstTopic   --bootstrap-server localhost:9092   --partitions 1   --replication-factor 1
sudo ${KAFKA_HOME}/bin/kafka-console-producer.sh   --topic FirstTopic   --bootstrap-server localhost:9092   --property "acks=all"   --property "compression.type=gzip"   --property "batch.size=16384"   --property "parse.key=true"   --property "key.separator=:"
```
2. Send some message
```
{'hi Spark': 'from kafka'}
```

3. Open New terminal/VM Instance. 
   You can now also check to consume the message at kafka.
```
sudo ${KAFKA_HOME}/bin/kafka-console-consumer.sh   --topic FirstTopic   --bootstrap-server localhost:9092   --from-beginning   --max-messages 100   --property "print.key=true"   --property "print.value=true"
```
5. Prepare your spark job submit file to consume the message in a spark dataframe
```
sudo nano sparkjob.py
```

with below lines
```
import sys
import json
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
if __name__ == "__main__":
    # Checking validity of Spark submission command
    if len(sys.argv) != 4:
        print("Wrong number of args.", file=sys.stderr)
        sys.exit(-1)
    # Initializing Spark session
    spark = SparkSession\
        .builder\
        .appName("MySparkSession")\
        .getOrCreate()
   # Setting parameters for the Spark session to read from Kafka
    bootstrapServers = sys.argv[1]
    subscribeType = sys.argv[2]
    topics = sys.argv[3]
    # Streaming data from Kafka topic as a dataframe
    lines = spark\
        .readStream\
        .format("kafka")\
        .option("kafka.bootstrap.servers", bootstrapServers)\
        .option(subscribeType, topics)\
        .load()
    # Expression that reads in raw data from dataframe as a string
    # and names the column "json"
    lines = lines\
        .selectExpr("CAST(value AS STRING) as json")
    # Writing dataframe to console in append mode
    query = lines\
        .writeStream\
        .outputMode("append")\
        .format("console")\
        .start()
    # Terminates the stream on abort
    query.awaitTermination()
```
 6. Submit Spark job and see the sent messages in console.
 ```
sudo /etc/spark/bin/spark-submit --packages org.apache.spark:spark-streaming-kafka-0-10_2.12:3.0.1,org.apache.spark:spark-sql-kafka-0-10_2.12:3.0.1 sparkjob.py localhost:9092 subscribe FirstTopic
```

### Integration of ROS Python - Kafka - Pypark 

1. Start ROS Terminal
```
c:\opt\ros\noetic\x64\setup.bat
c:\ws\turtlebot3\devel\setup.bat
set TURTLEBOT3_MODEL=waffle
```
2. Launch ros demo
```
roslaunch turtlebot3_demo.launch
```
4. Subscribe to ros message and publish to Kafka
```
import rospy
from nav_msgs.msg import Odometry
import json
from datetime import datetime
from kafka import KafkaProducer

count = 0
def callback(msg):
    global count
    messages={
        "id":count,
        "posex":float("{0:.5f}".format(msg.pose.pose.position.x)),
        "posey":float("{0:.5f}".format(msg.pose.pose.position.y)),
        "posez":float("{0:.5f}".format(msg.pose.pose.position.z)),
        "orientx":float("{0:.5f}".format(msg.pose.pose.orientation.x)),
        "orienty":float("{0:.5f}".format(msg.pose.pose.orientation.y)),
        "orientz":float("{0:.5f}".format(msg.pose.pose.orientation.z)),
        "orientw":float("{0:.5f}".format(msg.pose.pose.orientation.w))
        }

    print(f"Producing message {datetime.now()} Message :\n {str(messages)}")
    producer.send("FirstTopic",messages)
    count+=1

    producer = KafkaProducer(
    bootstrap_servers=["Your.VM.External.IP:9092"],
    value_serializer=lambda message: json.dumps(message).encode('utf-8')
)
if __name__=="__main__":

    rospy.init_node('odomSubscriber', anonymous=True)
    rospy.Subscriber('odom',Odometry,callback)
    rospy.spin()
```

4. Open New terminal/VM Instance.
   You can now also consume the message at kafka consumer
```
sudo ${KAFKA_HOME}/bin/kafka-console-consumer.sh   --topic FirstTopic   --bootstrap-server localhost:9092   --from-beginning   --max-messages 100   --property "print.key=true"   --property "print.value=true"
```

5. Submit your spark job to consume the message in a spark dataframe
```
sudo /etc/spark/bin/spark-submit --packages org.apache.spark:spark-streaming-kafka-0-10_2.12:3.0.1,org.apache.spark:spark-sql-kafka-0-10_2.12:3.0.1 sparkjob.py localhost:9092 subscribe FirstTopic
```
  References: 
  <prev>
  1. https://sandeepkattepogu.medium.com/python-spark-transformations-on-kafka-data-8a19b498b32c
  2. https://github.com/zekeriyyaa/PySpark-Structured-Streaming-ROS-Kafka-ApacheSpark-Cassandra?ref=pythonrepo.com
  <prev>
