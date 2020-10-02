# Kafaka theory

## Topics, partitions and offsets

### Topics:

+ A particular stream of data
  + Similar to a table in DB
  + Identified by its name

### Partitions

+ Topics are split in partitions
  + Each partition is ordered
  + Each mesage within a partition gets an incremental id, called *offset*

### Offset

+ Offset only have meaning to a specific partition
+ Order is guaranteed only within a partition (not across partitions)
+ Data is kept only for a limited time
+ Data is immuatble once it is written

> Topic
>
> - Partition 0 contains message with id *offset* -  |0|1|2|3|4|5|
> - Partition 1 |0|1|2|3|4|5|6|7|
> - Partition 2 |0|1|2|3|

## Brokers

+ A Kafka cluster is composed of multiple brokers (servers)
+ Each broker is identified with its ID 
+ Each broker contains certain topic partitions
+ Will be connected to the entire cluster once connect to any broker (*bootstrap broker*)

> + Topic A with 3 partitions
> + Topic B with 3 partitions

| Broker 101            | Broker 102            | Broker 103            |
| :-------------------- | --------------------- | --------------------- |
| Topic A - partition 0 | Topic A - partition 1 | Topic A - partition 2 |
| Topic B - partition 0 | Topic B - partition 1 |                       |

## Topic replication factor

+ Topics should have a replication factor > 1 (usually between 2 and 3)
+ This way if a broker is down, another broker can serve the data

>  Example: replication factor of 2.

| Broker 101            | Broker 102                         | Broker 103                         |
| :-------------------- | ---------------------------------- | ---------------------------------- |
| Topic A - partition 0 | Topic A - partition 0 (replicated) | Topic A - partition 1 (replicated) |
|                       | Topic A - partition 1              |                                    |

### Leader for a Partition

+ Only one broker can be a leader for an given partition
+ Only that leader can receive and serve data for a partition
+ The other brokers will synchronized the data
+ Each partition has one leader and multiple ISR (in-sync replica)

| Broker 101                     | Broker 102                     | Broker 103                  |
| :----------------------------- | ------------------------------ | --------------------------- |
| Topic A - partition 0 (leader) | Topic A - partition 0 (ISR)    | Topic A - partition 1 (ISR) |
|                                | Topic A - partition 1 (leader) |                             |

## Producers

+ Write data to topics
+ Automtically know which broker and partition to write to
+ Producers will automatically recover in case of broker failures
+ Can choose to receive acknowledgment of data writes:
  + acks=0: Producer won't wait for acknowledgment (possible data loss)
  + acks=1: will wait for leader acknowledgment (limited data loss)
  + acks=all: Leader + replicas acknowledgment (no data loss)

### Message Keys

+ Can choose to send a key with the message
  + if key == null, data is sent round robin
  + if key is sent, then all messages for that key will always go to the same partition
+ A key is basically sent if you need message ordering for a specific field

## Consumers

+ Read data from a topic
+ Know which broker to read from
+ In case of broker failure, comsumers know how to recover
+ Data is read in order within each partitions

### Consumer groups

+ Consumers read data in comsumer groups
+ Each consumer within a group reads from exclusive partitions
+ If you have more consumers then partitions, some consumers will be inactive

### Consumer Offsets

+ *Kafka* stores the offsets at which a consumer group has been reading
+ The offsets committed live in a Kafka *topic* named *__consumer_offsets* 
+ The offsets will be committed once a consumer in a group has processed data received from Kafka
+ Consumer will be able to read back from where it left off by the commited consumer offsets

### Delivery semantics for consumers

+ Consumers choose when to commit offsets
  + At most once:
    + Offset are committed as soon as the message is received
    + If the processing goes wrong, the message will be lost (won't read again)
  + At least once: (preferred)
    + Offsets are committed after the message is processed
    + The message will be read again if the processing goes wrong
    + THis can result it duplicate processing of messages.
      + Make sure your porcessing is idempotent. (processing duplicate message won't impact system)
  + Exactly once:
    + Kafka streams API
    + Use an idempotent consumer

## ZooKeeper

+ Zookeeper manages brokers (keep a list of them)
+ Helps in performing leader election for partitions
+ Sends notifications to Kafka in case of changes
+ Kafaka dependent on ZooKeeper
+ By design perates with an odd number of servers
+ Zookepper has a leader (handler writes) the rest of the servers are followers (handle reads)



# Kafka  Commands

**Start Server**

1. Start zookeeper server (with conf file)
   
> zookeeper-server-start zookeeper.properties
   
2. Start Kafka server
   
   > kafka-server-start server.properties

**Kafka CLI - Topic**

1. Create topic
   
> kafka-topics --zookeeper 127.0.0.1:2181 --topic first_topic --create --partitions 3 --replication-factor 1
   
2. List all topics
	
> kafka-topics --zookeeper 127.0.0.1:2181 --list
	
3. Kafka describle topics
	
> kafka-topics --zookeeper 127.0.0.1:2181 --topic first_topic --describe
	
4. Mark topic as deleted
	
	> kafka-topics --zookeeper 127.0.0.1:2181 --topic second_topic --delete

**Kafka CLI - Producer**
1. Producing
	
> kafka-console-producer --broker-list 127.0.0.1:9092 --topic first_topic
	
2. Producing with property
	
	> kafka-console-producer --broker-list 127.0.0.1:9092 --topic first_topic --producer-property acks=all

3. Producing with keys
	>. kafka-console-producer --broker-list 127.0.0.1:9092 --topic first_topic --property parse.key=true --property key.separator=,

**Kafka CLI - Consumer**
1. Consuming
	> kafka-console-consumer --bootstrap-server 127.0.0.1:9092 --topic first_topic 
	
2. Consuming from beginning
	> kafka-console-consumer --bootstrap-server 127.0.0.1:9092 --topic first_topic --from-beginning
	
3. Consuming by different group
	> kafka-console-consumer --bootstrap-server 127.0.0.1:9092 --topic first_topic --group my-first-app

4. Consuming with keys
	> kafka-console-consumer --bootstrap-server 127.0.0.1:9092 --topic first_topic --from-beginning --property print.key=true --property key.separator=,

5. Describe groups
	> kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group my-second-app
	> Consumer group 'my-second-app' has no active members.
GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG     CONSUMER-ID     HOST            CLIENT-ID
my-second-app   first_topic     0          1               1               0               -               -               -
my-second-app   first_topic     1          2               2               0               -               -               -
my-second-app   first_topic     2          3               3               0               -               -               -

> kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group my-first-app
Consumer group 'my-first-app' has no active members.
GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
my-first-app    first_topic     0          1               1               0               -               -               -
my-first-app    first_topic     1          1               2               1               -               -               -
my-first-app    first_topic     2          1               3               2               -               -               -

6. Reset offsets
> kafka-consumer-groups --bootstrap-server localhost:9092 --group my-first-app --reset-offsets --to-earliest --execute --topic first_topic
 
