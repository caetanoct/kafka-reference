# Kafka

## Messages and Batches

### Messages

- Smallest unit of data (similar to row or a record) is a *message*.
- Message is an array of bytes as far as Kafka is concerned.
    - Can contain optional metadata (*key*)
    - Keys are used to ensure messages with the same key are written to the same partition (it calculates the hash of the key and take modulo). If number of partition changes, then it alters the result of the modulo.

### Batches

- Batch is a collection of messages to the same topic and partition.
- Batch is a tradeoff between latency and throughput: larger batches, more messages can be handled per unit of time, but it takes longer to propagate an individual message.
- Usually compresses, to provide more efficient data transfer and storage (at cost of processing).

## Schemas

Consistent data format allows reading and writing to be decoupled. If it is coupled, when an application subscribes to a topic and the messages change, it need to be updated to handle the new data format.

## Topics and Partitions

- Topic closest analogy is a folder in filesystem or database table.
- A partition is a single *commit log*.
    - Messages are written to it in order (append only)
    - Messages are read in order
- Partitions is the way Kafka provides redundancy and scalability;
- Each partition can be hosted on a different server. Meaning a topic can be scaled horizontally across multiple servers to provide performance.
- A topic is a stream, regardless the number of partitions.
- As messages are produced to the kafka broker, they are appendend to the current log segment for the partition until log.segment.bytes is reached and it is closed.
- When you create a topic, kafka decides how to allocate partitions between brokers
    - For example, you have 6 brokers and you create a topic with 10 partitions and replication factor of 3, you will have 30 partition replicas to split between 6 brokers.
- Segments are closed by default when it reaches 1Gb or 1 week of data.
- You can't delete partitions (reduce the number of partitions).
- You can increase the number of partition, but partition/key hashing logic is changes and alter order of messages.


```txt
+---------------------------------------------------------------------+
|                                                                     |
|   Commit Log / Partition                                            |
|                                                                     |
|                                                                     |
|                                                                     |
|                                                                     |
|                1Gb                            1Gb                   |
|        -------------------            -------------------           |
|                                                                     |
|          +-------------+                +-------------+             |
|          |             |                |             |             |
|          |   Segment1  |                |   Segment2  |             |
|          |             |                |             |             |
|          +-------------+                +-------------+             |
|     offset 0        offset 132     offset 133      offset 265       |
|                                                                     |
|                                                                     |
+---------------------------------------------------------------------+
```

### Choosing a partition count

Several factors to consider:
- Expected throughput to write to the topic?
- Maximum throughput expected to achieve when consuming, if you know the database does not handle more than 50MB per second from each thread writing to it, it means you are limited to 60MB throughput when consuming from a partition.
- If you are sending messages based on keys, adding partitions later can be challenging, calculate throughput based on expected future usage.
- Consider number of partition you will place on each broker and available diskspace/network bandwidth per broker.
- Avoid overestimating (each partition uses memory and other resources, increases leader election times).

For example, if i want to be able to write and read 1Gb/s from a topic and I know each consumer can process 50MB/s, then I know I need at least 20 partitions (20 consumers).

## Producers and Consumers

- Producer, by default, does not care which partition a message is written to
- Producer, by default, will balance messages over all partitions evenly.
- Producer can produce messages to specific partitions. Usually using message key. Or use a custom partitioner that follows other business rules.
- Consumer keep track of which messages it has already consumed by keeping track of the offset of the messages they consumer.
- The offset is a metadata on the message.
- Kafka adds the offset to each message as it is produced.
- By storing the offset value, a consumer can stop and restart at any time.
- Consumer are always inside a consumer group.
- The consumer group assures that each partition is only consumed by one member.
- The consumer -> partition mapping is called *ownership* of the partition by the consumer.
- Consumer can horizontally scale. If one fails, remaining members can rebalance the partitions to take over the missing member.

## Brokers and Clusters

- Broker = Kafka Server.
- Broker receives messages from producers -> assigns an offset -> commit the message to storage on disk.
- Serves consumers. Consumers make a fetch request for partitions and it replies for the messsage commited to disk.
- A single broker can handle thousands of partitions and millions of messages per second.
- Brokers are part of a cluster.
- Within a cluster of brokers, one broker is also the **cluster controller** (elected automatically from the members of the cluster).
- The controller makes administrative operations:
    - Assigning partitions to brokers.
    - Monitor broker failures.
- Every partition is owned by a single broker in the cluster (and it is called the leader of the partition).
- Partition may be assigned to multiple brokers, which will result in replication. The other broker can take over leadership if there is a broker failure. But all consumers/producers must connect to the new broker.
- Kafka brokers are configured with default retention:
    - 7 days of retaining a message; or
    - Until topic reaches certain size in bytes (1 GB);
    - After these limits are reached, messages are deleted.
    - Individual topics can have different retention settings.
    - Topics can be *log compacted*: kafka will retain only the last message produced with a specific key (this can be useful for changelog data), where only the last update is interesting.

## Multiple Clusters

Having multiple clusters benefits: multiple datacenters, isolation for security requirements, segregation of types of data.

## Why Kafka

- Multiple consumers for a single topic
- Disk-based retention(you can freely restart, stop and perform maintanence on consumers without risking data-loss)
- Scalable

## Installing Kafka

### Java

It needs java runtime, but java SDK is recommendend.

### Zookeper

Zoopeker is used to store metadata about the kafka cluster. You can verify if a zookeper server is running locally using `telnet localhost 2181`. you can type command *srvr*.

To configure Zookeeper servers in an ensemble, they must have a common configuration that lists all servers.

Each server needs a myid file in the datadir containing the ID number of the server (must match configuration file).

Example:


```ini
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
initLimit=20
syncLimit=5
server.1=zoo1.example.com:2888:3888
server.2=zoo2.example.com:2888:3888
server.3=zoo3.example.com:2888:3888
```

The servers are specified in the format server.X=hostname:peerPort:leaderPort

**X**
The ID number of the server. This must be an integer, but it does not need to be zero-based or sequential.

**hostname**
The hostname or IP address of the server.

**peerPort**
The TCP port over which servers in the ensemble communicate with each other.

**leaderPort**
The TCP port over which leader election is performed.

### Kafka Broker

After configuring Java and Zookeper you can install kafka.

The following example installs Kafka in /usr/local/kafka and store message log segments stored in /tmp/kafka-logs.

```bash
tar -zxf kafka_2.11-0.9.0.1.tgz
mv kafka_2.11-0.9.0.1 /usr/local/kafka
mkdir /tmp/kafka-logs
export JAVA_HOME=/usr/java/jdk1.8.0_51
/usr/local/kafka/bin/kafka-server-start.sh -daemon
/usr/local/kafka/config/server.properties
```

## Broker Configuration

Default configuratin can be used for POC's, but will not be sufficient for most installations.

### General Broker

- broker.id
    - integer identifier. default value is 0. Good practice is to set this to something intrinsic to the host.
- port
    - default is 9092, if a port is lower than 1024 kafka must be run as root (not recommended)
- zookeper.connect
    - location of zookeper to store broker metada. semicolon-separated list of `hostname:port/path`. where path is an zookeper path to use as chroot env for kafka cluster, if ommited, root path is used. good to use chroot for isolation
- log.dirs
    - kafka persists all messages to disk. log segments are stored in log.dirs specified directories. If more than one path is specified, the broker store partitions in a least-used fashion with one partition log segments store within the same path. The broker will place a new partition in the path that has the least number of partitions currently stored in it, not the least amount of disk space used in the following situations.
        - num.recovery.threads.per.data.dir
            - kafka uses a pool of threads for handling log segments. Currently, this thread pool is used when starting normally (to open each partition log segments), when starting after a failure (to check and truncate each partition log segmente), when shutting down (to cleanly close log segments)
            - by default, only one thread per log directory is used.
        - auto.create.topics.enable
            - default configs state that a broker should automatically create a topic if:
                - a producer starts writing messages to the topic
                - a consumer starts reading messages from the topic
                - any clients requests metadata for the topic
            - if you are managing topic creation you need to set (auto.create.topics.enable=false)

### Topic Defaults

- num.partitions
    - how many partitions a new topic is created with. usually defaults to one partition. can always be increased, but **never decreased**. partitions are the way a topic is scaled within a kafka cluster. it is important to use partition counts that will balance the message load across the entire cluster as brokers are added. usually the partition count is equal or a multiple of the number of brokers in the cluster. allows it to be evenly distributed (which in turn, distributes message load). you can also balance message load by having multiple topics.
- log.retention.ms
    - how long kafka will retain message (default is 7 days - 168 hours). smalled unit sizes take precedence from bigger (example, the default is log.retention.hours=168) and defining this would overwrite its behaviour.
    - retention by time is performed by examining the last modified time on each log segment file on disk. usually it is the time that the log sgment was closed and represents the timestamp of the last message in the file.
- log.retention.bytes
    - expire messages based on number of bytes of messages retained. applied per partition, for example, if you have a topic with 8 partitions and it is set to 1Gb, the amount of data retained is 8Gb.
    - if you configure time/size based retention messages may be removed when either criteria is met. 
- log.segment.bytes
    - the setting above are applied on log segments, not individual messages. once the log segment has reached the size specified by this field (default is 1Gb), the log segment is closed and a new one is opened.
    - adjusting is important if topics have low produce rate. Example: if a topic receives 100 Mb per day of messages and the field is set to the 1Gb default, it will take 10 days to fill one segment. As messages cannot be expired until the log segment is clodes, if the log retention is 7 days, there will actually be up to 17 days of messages retained until the closed log segment expires, since the log segment must be retained for 7 days and it takes 10 days to close a segment.
    - the size of the log segment also affects the behaviour of fetching off-sets by timestamp. when requesting offsets for a partition at a specific timestamp, kafka finds the log segment file that was being written at that time, by using the creation and last modified time of the file, and looking for a file that was created before the timestamp and last modified after the timestamp. The offset at the beginning of that log segment is returned in response.
- log.segment.ms
    - another way to control how log segments are closed is by using log.segment.ms parameter. It is the amount of time after which a log segment should be closed. it is not mutually exclusive with the above one and if both are defined kafka will close the segment based on one of the rules, whichever comes first. there is no default setting.
    - disk performance in time-based segments: there is an impact on disk performance when multiple log segments are closed simultaneously. Can happen when there are many partitions that never reach the size limit for log segments.
- message.max.bytes
    - default value is 1Mb. If a producer tries to send a message larger than this limit it will receive an error from the broker. This configuration deals with compressed message size.
    - Larger messages mean that the broker threads that deal with processing network connections and requests will be working longer on each request.
    - Larger messages increase the size of disk writes, impacs I/O throughput.
    - Message size configured on the broker must be coordinated with fetch.message.max.bytes. If the value is smaller than message.max.bytes, then consumers that encounter larger messages will fail to fetch those messages: this is a situation where the consumer gets stuck and can't proceed. it also applies to replica.fetch.max.bytes configuration.

### Hardware Selection

#### Disk Throughput

Kafka messages must be commited to local physical storage when they are produced. Most clients will wait until at least one broked has confirmed that messages have been commited before considering the send successful.
Faster Disk Writes = Less latency on produce.

When using HDDs, you can improve its performance by using more of them in a broker (by having mutiple data directories or by setting up the drives in a redundant array of independent disks (RAID)).

#### Disk Capacity

Increases based on how many messages need to be retained. If the broker is expected to receive 1Tb of traffic each day and log retention is 7 days, it need a minimum of 7Tb useable storage for log segments.

The total traffic for a cluster can be balanced across it by having multiple partitions per topic, allowing additional brokers to augment the available capacity if the density on a single broker will not suffice.

#### Memory

Normal mode is reading from the end of partitions, where the consumer is caught up and lagging (if lagging) very little. In this situation, the messages the consumer is reading are optimally stored in system page cache.

Having more moemory available to the system for page cache will improve consumer clients performance.

Kafka itself does not need much heap memory configure for the JVM.

Even a broker that is handling X messages/sec and a data rate of X mb/s can run with a 5Gb heap. The rest of the memory will be used by the page cache and will benefit kafka by allowing the system to cache **log segments in use**.

#### Networking

The available network throughput is related to the maximum amount of traffic kafka can handle. Governing factor combined with disk storage and cluster sizing.

Kafka has an inherent imbalance between inbound/outbound traffic, because it supports multiple consumers.

Cluster replication and mirroring will also increase network requirements.

If the network interface is saturated, it is not uncommon for the cluster replication to fall behind, leaving it to a vulnerable state.

#### CPU

Majority of CPU requirements come from compressing/decompressing messages.

Kafka must decompress all message batches, to validate the checksum of individual messages.
It need to recompress message match in order to store to disk.

#### Kafka in the Cloud

A good place to start is with the amount of data retention required, also by the performance needed from the producers.

If very low latency is necessary, I/O optimized instances that have local SSD attached might be required. Otherwise, ephemeral storage (like aws EBS elastic block store) might be sufficient.

In AWS it would usually be m4 or r3 instance types.

m4 instance will allow greater retention periods (but disk throughput is less on EBS).
r3 instance will have better throughput with local SSD but it limits the amount of data to be retained.

For the best of both words it would be necessary to move up to either the i2 or d2 instance types.

### Kafka Clusters

Using kafka clusters is good for replication and load balancing.

#### How many brokers?

You can start by determining how much storage a single broker can store and how much disk is required for retaining message, for example: if the cluster needs to store 10Tb and each broker can store 2TB, you will need at least 5 brokers.

Replication will increase storage requirements by ate least 100%.

For example, for the example above if we replicate the cluster, it would need at least 10 brokers.

Network traffic is also a good metric, for example: if the NIC on a broker is used to 80% capacity at peak, and there are two consumers of that data, the consumers will not be able to keep up with peak traffic unless there are two brokers.

#### Broker Configuration

There are only two requirement in the broker config to allow joining a kafka cluster.
1. All brokers must have the same configuration for the zookeper.connect parameter (zookeeper ensenble and path where metadata is stored)
2. All brokers must have unique broker.id parameter (if two try to join with the same id the second one will fail)

##### OS Tuning

Kafka can benefit from virtual memory/network subsystems tweaks, as well as specific concerns for the disk mount point used for storing log segments.
Parameters configured on `/etc/sysctl.conf`.

###### Virtual Memory

We can make some adjustments to how swap space is handles and dirty memory pages, tuning it for kafka.

The cost of swapping (having pages of memory swapped to disk) will have a noticeable impact in kafka. Also, if kafka is swapping thismeans there is not enough memory being allocated for the system page cache.

Having swap prevent abrubts process killing due to OOM. Therefore, it is recommended to use (**vm.swappiness=1**). This parameter is a percentage of how likely a VM is to use swap space rather than dropping the pache.

It is better to reduce the size of the page cache than swapping.

You can also adjust how the kernel handle dirty pages that must be flushed to disk. 

Since log segments are usually using SSD or a system with NVRAM for caching (RAID). The number of dirty pages allowed can be reduced. Use **vm.dirty_background_ratio** lower than the default 10. this is a percentage of the total amount of system memory, setting it to 5 is appropriate in many situations.

The total number of dirty pages that are allowed before syncrhornous operations to flush them to disk can be increase with **vm.dirty_ratio**, the default is 20. between 60 and 80 is a reasonable number.

The current number of dirty pages can be determining by checking `/proc/vmstat` with dity or writeback filters.

###### Disk

Besides choosing SSD or RAID system.

XFS has better performance than EXT4 out of the box, and would be better for Kafka.

It is recommended to mount log segments disks with the `noatime` mount option. Because, file metadata contains creation time (ctime), last modified time (mtime) and last access time (atime). By default, atime is updated every time a file is read. This generates a lot of disk writes.

###### Networking

Linux default kernel network stack is not tuned for large, high-speed data transfers.

The recommended changes for Kafka are the same for most web servers and other networking applications.

1. Change the default maximum amount of memory allocated for the send and receive buffers for each socket. The settings `net.core.wmem_default` and `net.core.rmem_default` are the default size per socket. A reasonable setting would be 131072 (128 KiB). The `net.core.wmem_max` and `net.core.rmem_max` parameters are for the maximum send/receive buffer sizes. A reasonable setting would be 2097152 (2 MiB).
2. The send/receive buffer sizes for TCP Sockers must be set using `net.ipv4.tcp_wmem` and `net.ipv4.tcp_rmem`. These are set using 3 number, it specifies the minimum, default and maximum size. The maximum size can't be above the specified for all sockets (defined above), an example setting would be `4096 65536 2048000` which is 4KiB minimum, 64KiB default and 2MiB maximum buffer. Based on the worload you may want to increase, for greater buffering of the network connections.

> Changing socket buffer size increases performance for large transfers.
> Setting the max size for a socket does not mean it will always be allocated, it means it allow up to that amount.

You can enable TCP Window Scaling by setting `net.ipv4.tcp_window_scaling` to 1. This allows data to be buffered on the broker side.
You can increate the value of `net.ipv4.tcp_max_syn_backlog` above 1024. will allow a greater number of simultaneous connections to be accepted.
You can increate the value of `net.core.netdev_max_backlog` to greater than 1000, can assist in burst of network traffic, specifically when using multigigabit network speeds, allows more packets to be queued for the kernel to process them.

> In TCP communication, the "window size" determines how much data can be in transit before an acknowledgment (ACK) is required. If the window size is small, then for long-distance connections (high latency), we aren't making efficient use of the available bandwidth because we keep waiting for ACKs.

#### Production Concerns

##### Garbage Collector Options

G1 (Garbage collector first), came with Java 7 and is designed to automatically adjust to different workloads, provides different pause times for garbage collection. Handles large heap sizes by segmenting the heap into smaller zones. Does not collect the entire heap in each pause.

There are two configuration options for G1 performance:

- `MaxGCPauseMillis`
    - max pause time for garbage collection cycle (it is not a fixed maximum, G1 can and will exceed this time if required). Default is 200ms. It will attempt to schedule the frequency of GC cycles so that each cycle will take approximately 200ms
- `InitiatingHeapOccupancyPercent`
    - percentage of total heap may be used before G1 will start a collection cycle. Default is 45. It means G1 iwll not start a collection cycle until 45% of the heap is in use.

The GC tuning options provided have been found to be appropriate for a server with 64GB of memory running kafka in a 5Gb heap. you can configure the max pause to 20ms and the percentage to 35.

The start script for kafka does not use G1 collector, it uses a mark and sweep garbage collection.

you can modify the behaviour using:

```bash
export JAVA_HOME=/usr/java/jdk1.8.0_51
export KAFKA_JVM_PERFORMANCE_OPTS="-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+DisableExplicitGC -Djava.awt.headless=true"
/usr/local/kafka/bin/kafka-server-start.sh -daemon
```

Best practice is to have each Kafka Broker in a cluster installed in a place without the same Single Point of Failure (power/network).

## Kafka Producers

Kafka is a binary wire protocol, therefore there are multiple clients in multiple languages, like C++, Python, Go, etc..

The high level flow can be described as follows:

```txt
         return metadata
+----------------------------------------+
|                                        |
|                                        v
|                                  +------------+
|             +------------------->|  Producer  |
|             |                    +-----+------+
|             |                          |
|             |                          |
|             |                          |
|             |                    +-----v------+
|             |                    | Serializer |
|             |                    +-----+------+
|             |                          |
|             |                          |
|             |  can't retry             |
|             |                   +------v------+
|             |                 +-+ Partitioner +------+
|             |                 | +-------------+      |
|             |                 |                      |
|         +---+---+             v                      v
|         | Retry +----->+---------------+    +-----------------+
|         +-------+      |  Topic A      |    |   Topic B       |
|             ^          |  Partition 0  |    |   Partition 1   |
|         yes |          +------+--------+    +----------+------+
|             |                 |                        |
|        +----+---+             |                        |
+--------+  Fail  |             |                        |
         +--------+             |                        |
              ^                 |                        |
              |                 |                        |
              |                 |  +-------------+       |
              |                 +->| Kafka Broker|<------+
              |                    +------+------+
              |                           |
              |                           |
              +---------------------------+
```

The producer creates a ProducerRecord, it must include the topic and a value. Optionally you can specify a key and/or partition.
The producer serialize the key and value to bytearrays so they can be sent over the network.

Next, data is sent to a partition. If partition was specified the partitioner just use the partition specified. If not, partitioner will choose a partition for user, usually based on the ProducerRecord key.
After knowing a partitiOn, it adds the **record to a batch of records that will be sent to the same topic/partition**.
A separate thread is responsible for sending those batches to the appropriate kafka brokers.

When the broker receives the messages, it sends a response. If the message were succesfully written to Kafka, it will return a RecordMetadata object with topic/partition and offset of the record within the partition.
If the broker failed to write, it returns an error. When the producer receives an error it may retry before giving up and returning and error.

If the producer is sending compressed messages, all the messages in a batch are compressed together and sent as the value of a *wrapper message*. When the consumer decompresses the message value, it will have all messages contained in the batch with their own timestamps and offsets.

### Constructing a Kafka Producer

A kafka producer has 3 mandatory fiels:
- `bootstrap.servers`
    - list of brokers to connect. it does not include all brokers because the producer gets this information on the initial connection. It is recommended to have two, though, in case on goes down, it may still be able to connect to the cluster.
- `key.serializer`
    - class that will be used to serialize the keys. the class must implement `org.apache.kafka.common.serialization.Serializer`. It serielizes to byte-array
- `value.serializer`
    - class that will be used to serialize values. It must serialize the object to a byte array.

### Sending a Message to Kafka

There are three primary methods of sending messages:

- fire-and-forget
- synchronous send: wait to see if send was successful or not
- asynchronous send: the callback function gets triggered when it receives a response from the broker

A producer object can be used by multiple threads to send messages.

### Configuring Producers

- `acks`
    - controls how many partitions replicas must receive the record before the producer can consider the write successful.
    - if acks=0: the producer will not wait for a reply from the broker
    - if acks=1: the producer will receive a success response the moment the leader replica receive the message. if leader has not been elected or crashed, the producer receives an error.
    - if acks=all: the producer will receive a success response once all in-sync replicas received the message.
- `buffer.memory`
    - amount of memory to buffer messages waiting to be sent to brokers.
- `compressiont.type`
    - can be set to snappy, gzip or lz4. default is uncompressed. snappy has low cpu overhead, gzip has more cpu overhead but better compression (recommendend if network bandwidth is more restricted)
- `retries`
    - if the error message from the server is transient (e.g., lack of leader for a partition), the producer will retry `retries` times. it waits 100ms between retries. you can adjust retry.backoff.ms parameter.
- `batch.size`
    - multiple records to the same partition are batched. this is the amount of memory in bytes used for each batch. when batch is full the batch is sent. kafka can send half-way batches.
- `linger.ms`
    - amount of time to wait for new messages before sending the batch. if the ms is reached, the messag ewill be sent. it increases latency bbut increases throughput.
- `client.id`
    - identify messages sent from the client
- `max.in.flight.request.per.connection`
    - how many messages send to the server without receiving responses. higher values = higher memory usage = higher throughput, too high => reduces throughput as batching becomes less efficient. setting it to 1 will guarantee messages are written to the broker in order which they are sent, even with retries.
- `timeout.ms,request.timeout.ms, metadata.fetch.timeout.ms`
    - how log a producer will wait for a reply from the server (request.timeout.ms)
    - how long a producer will wait when requesting metadata such as current leaders for the partitions (metadata.fetch.timeout.ms)
    - time broker will wait for in-sync replicat to ack the message in order to meet `acks` config (timeout.ms)
- `max.block.ms`
    - how long the producer will block when calling send() and requesting metadata with partitionsFor().
- `max.request.size`
    - the size of a produce request. example: max request size of 1MB, the largest message you can send is 1Mb or the producer can batch 1000 messages of 1K each into one request. in addition, the broker has its own limit on the size of the largest message it will accept (message.max.bytes).
- `receive.buffer.bytes` and `send.buffer.bytes`
    - sizes of the TCP send/receive buffers used by the sockets. if set to -1 the os defauolt will be used. good idea to increase those when producers of consumer communicate with brokers in different datacenter because those network links tipically have higher latency and lower bandwidth.

> Ordering Guarantees: kafka preserves the order of messages within a partition. this means if messages were produces in a order all consumer will read in that order. setting the retries parameter to nonzero and max.in.flights.requests.per.session to more than one, means that is possible that the broker will fail to write the first batch, suceed to write the second (which was already in flight) and retry the first and sucess, reversing the order. Usually setting the number of retries to zero is not an option, so if guaranteeing order is critical, it is recommended to use in.flight.request.per.session=1 to make sure while a batch is retrying additional message can't be sent. it severely limit the throughput of the producer.

### Partitioning

Example of custom partitioner:

```java
import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;
import org.apache.kafka.common.PartitionInfo;
import org.apache.kafka.common.record.InvalidRecordException;
import org.apache.kafka.common.utils.Utils;
public class BananaPartitioner implements Partitioner {
    public void configure(Map<String, ?> configs) {}
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes,
                        Cluster cluster) {
        List<PartitionInfo> partitions =
        cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        if ((keyBytes == null) || (!(key instanceOf String)))
            throw new InvalidRecordException("We expect all messages
            to have customer name as key")
        if (((String) key).equals("Banana"))
            return numPartitions; // Banana will always go to last partition
        // Other records will get hashed to the rest of the partitions
        return (Math.abs(Utils.murmur2(keyBytes)) % (numPartitions - 1))
        }
    public void close() {}
}
```

## Consumers

Application subscribe to topics and receive messages from it.

### Kafka Consumer Groups

Kafka allows multiple consumer to read from a topic. Splitting data between them.

Whenever multiple consumer groups are subscribed to a topic and belong to the same group, each consumer in the group will have a subset of partitions that they are reading from.

On older versions consumer groups information are maintaned in Zookeeper, in newer version it is maintaned within Kafka Brokers.


```txt
+---------------+      +------------------+
|  Topic 1      |      |  Consumer Group  |
++-------------++      | +-----------+    |
|| Partition 1 |+------+-|Consumer 1 |    |
|+-------------+|      | +-----------+    |
|| Partition 2 |+---+  | |Consumer 2 |    |
++-------------++   |  | +-----------+    |
|               |   +--+-|Consumer 3 |    |
|               |      | +-----------+    |
|               |      |                  |
|               |      |                  |
|               |      |                  |
+---------------+      +------------------+
```

> if you add more consumers than partitions, some consumer will be idle!

Adding consumers is a way to scale the application. To make sure an application gets all the messages in a topic, esure the application has its own consumer group.

### Consumer Groups and Partition Rebalance

#### Rebalance

- When we add a new consumer to the group, it starts consuming messages from partitions previously consume by another.
- When a consumer shutdowns or crashes, it leaves the group, the partitions he was consuming will be consumed by another.
- Reassignments also occur when the topic is modified (adding new partition, for example).

During a rebalance, consumers can't consume. It is a short window of unavailability.

Consumers maintain their membership and ownership by sending *heartbeats* to the kafka broker that is the *group coordinator*.
Heartbeats are sent when the consumer polls and when it commits records it has consumed.

If heartbeats stops, consumer is declared dead and a rebalance will be triggered. When closing a consumer cleanly, the rebalance is immediate. When a consumer crashed it will take a few seconds to rebalance, and in those seconds the partition the consumer owned will not be consumed.

> In recent kafka versions, you can have a separate thread that will send heartbeats between polls (separate hearbeat frequency). you can configure how long the application can go without polling before it leaves the group. This configuration prevents a livelock (application did not crash but does not make progress). it is differente from session.timeout.ms. max.poll.interval.ms will tune delays between polling.

The first consumer to join the group is the *group leader*. The leader receives a list of all consumer in the group by the group coordinator (all members that sent a heartbeat recently and are alive), it then assigns a subset of partitions to each consumer. after deciding the partition assignment it sends the list to the group coordinator, which then sends the information to all the consumers. the leader is the only process that know which partitions other are consuming, the other only see their own assignment. This occurs in every rebalance.

### Creating Kafka Consumer

It also has the three mandatory properties: bootstrap.servers ; key.deserializer, value.deserializer

There is a fourth property, `group.id`, it specified which group the consumer belongs.

### Subscribing to Topics

After creating the consumer you need to subscribe to one or more topics.

You can also use subscribe with a regex and whenever a new topic matches, a rebalance will happen.

### The Poll Loop

Once the consumer subscribes, the poll loop handles all details of coordination, partition rebalances, heartbeats, data fetching, leaving a clean api that simply returns available data from the assigned partitions.

The first time you call poll() it is resonsible for finding the group coordinator, joining the group, receiving a partition. the heartbeats are sent within the poll loop and rebalance is handled inside the poll loop.

If a consumer is not polling it is considered dead.

By default, poll will start consuming from the last commited offset.

> You can't have multiple consumer on the same group inside one thread and you can't have multiple threads use the same consumer. One consumer per thread is the rule. To run multiple consumer in the same group in one application, you need to run each on its own thread. Wrap the consumer login in its own object and use Java ExecutorService to start multiple threads each with its own consumer.

### Configuring Consumers

- `fetch.min.bytes`
    - minimum amount of data it wants to receive from the broker when fetching. If the broker receives a request for records from a consumer but the amount is fewer than min.fetch.bytes, the broker will wait until more messages are available before sending to the consumer. It reduces load on consumer/broker as they have to handle fewer back-and-forth messages, in chases where the topics don't have much new activity (or lower activity hours of the day). You wan't to set this parameter higher if consumer is using too much CPU when there is not much data available, or reduce load on brokers when they have a large number of consumers.
- `fetch.max.wait.ms`
    - tell kafka how long to wait until it has enough data (by default up to 500ms). Extra 500ms latency in case there is not enough data flowing to the kafka topic to satisfy the minimum amount of data. if fetch.max.wait.ms is 100 and fecth.min.byts is 1mb kafka will receive a fetch request from the cnsumer and will resply with data either in 100ms or when 1Mb of data to return.
- `max.partition.fetch.bytes`
    - maximum amount of bytes the server will return per pertition. default is 1Mb. If a topic has 20 partitions and you have 5 consumer, each consumer will need 4Mb of memory available for consumer records. In practice, you will want to allocate more memory as each consumer will need to handle more partitions if other consumer in the group fail. this number has to be larger than the largest message a broker will accept (max.message.size property in broker config) or the broker may have messages unable to be consumer, in which case the consumer will hang trying to read them.
- `session.timeout.ms`
    - Amount of time a consumer can be considered alive without sending heartbeat (3 seconds default). closely related to heartbeat.interval.ms. this other property control how frequently the KafkaConsumer poll() will send a heartbeat to the group coordinator, whereas session.timeout.ms constrols how long a consumer can go wihtout sending a heartbeat. If you set this to to low then it may trigger unwanted rebalances.
- `auto.offset.reset`
    - Controls the behaviour of the consumer when it starts reading a partition for which it has not commited offset or the offset it has is invalid (usually because the consumer was down for so long that the record with that offset was already aged out of the broker). default is "latest", which means that lacking a valid offset, the consumer will start reading from the newest records. Alternative is earliest, which means that lacking a valid offset, the consumer will read all data in the partition from the beginning.
- `enable.auto.commit`
    - Controls whether the consumer will commit offsets automatically (default is true). Set to false to control when offsets are commited, necessary to minimize duplicates and avoid missing data. If you set enable.auto.commit to true, then you might also want to control how frequently offsets will be commited using auto.commit.interval.ms.
- `partition.assignment.strategy`
    - By default kafka has two assignments strategies:
        - range
            - assigns each consumer a consecutive subset of partitions from each topic it subscribes to. Example: C1 and C2, topic T1 and T2 with 3 partitions, C1 will be assigned part 0 and 1 and C2 will be assigned part 2.
        - roundrobin
            - takes all the partitions from all subscribed topics and assigns them to consumer sequentially, one by one. C1 would have part 0 and 2 from T1 and 1 from T2. C2 would have part 1 from T1 and 0 and 2 from T2.
    - the default is org.apache.kafka.clients.consumer.RangeAssignor. if you use a custom partitioner this should poin to the name of your class.
- `client.id`
    - can be any string to identify messages sent from this client.
- `max.poll.records`
    - maximum number of records that a single call to poll() will return. useful to help control the amount of data you application will need to process in the polling loop.
- `receive.buffer.bytes` and `send.buffer.bytes`
    - is set to -1 use OS defaults. Sizes of TCP send/receive buffers. Can be good to increase when network links have higher latency and lower bandwidth.

### Commits and Offsets

Whenever we call poll(), it returns records written to kafka that consumer in our group have not read yet. Kafka allow consumer to track their own position (offset) in each partition.

The action of updating the current position is a **commit**.

How does a consumer commit offset? it produces a message to kafka to a special topic called **__consumer_offsets** with the commited offset for each partition.

If a cosumer crashes or a new consumer joins the groups, this will trigger a rebalance. After a rebalance, each consumer may be assigned a new set of partitions than the one it processed before. In order to know where to pick up the work, the consumer will read the latest commited offset of each partition and continue from there.

If the committed offset is smaller than the offset of the last message the client processed, the messages between the last processed offset and the commited offset will be processed twice.

### Auto Commit

If you enable `enable.auto.commit=true`, then every five seconds the consumer will commit the largest offset your client received from poll(). The automatic commits are driven by the poll loop. Whenever you poll the consumer checks if it is time to commit. if so, it will commit the offsets it returned in the last poll.

If we are 3 seconds consuming in the window and a reblaance is trigerred, this 3 seconds that were not committed will be processed twice.

The call to poll() will always commit the last offset returned by the previous poll, this means that you need to ensure all events returned by the poll() are processed before calling poll() again.

### Commit Current Offset

By setting auto.commit.offset=false, offsets will only be committed when the application explicitly chooses to do so.

commitSync() will commit the latest offset returned by poll() and return once the offset is commited, throwing an exception if the commit fails.

Remember that it will commit the latest offset returned by poll, therefore you need to ensure you are done processing all the records in the collection, or you risk missing messaged.

### Asynchronous Commit

The application is locked until the broker responds the commit request. If you use async commit it bypasses this.

The drawback is that while commitSync() will retry the commit until it either suceeds or encounters a nonretriable failure, commitAsync() will not retry.

For example: if we send a request to commit offset to 2000, there is atemporary comunnication problem, so the broker never gets this request and don't respont. Meanwhile, we processed another batch and committed to 3000. if commitAsyc() retries, it might succeed commiting offset 2000 after 3000. this will cause more duplicates.

You can define a callback for when the broker responds.

> A simple pattern to get commit order right for asynchronous retries is to use a monotonically increasing sequence number. Increase the sequence number every time you commit and add the sequence number at the time of the commit to the commitAsync callback. When you’re getting ready to send a retry, check if the commit sequence number the callback got is equal to the instance variable; if it is, there was no newer commit and it is safe to retry. If the instance sequence number is higher, don’t retry because a newer commit was already sent.

### Combine Sync and Async Commits

Occasional failues to commit without retrying are not a huge problem, because if the problem is temporary, the next commit will be successful. But if we know that this is the last commit before we close the consumer, or before a rebalance, we want to make extra sure that the commit succeeds.

A common pattern is to combine commitAsyc with commitSync just before shutdown.

```java
try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record : records) {
            System.out.printf("topic = %s, partition = %s, offset = %d,
            customer = %s, country = %s\n",
            record.topic(), record.partition(),
            record.offset(), record.key(), record.value());
        }
        consumer.commitAsync();
    }
} catch (Exception e) {
    log.error("Unexpected error", e);
} finally {
    try {
       consumer.commitSync();
    } finally {
        consumer.close();
    }
}
```

### Commit Specified Offset

- Commiting the latest offset allows you to commit the offset as often as you finish processing batches.
- If you want to commit more frquently, or commit during a huge batch to avoid reprocess on rebalances, you can't call commitSync/commitAsync because it commit the last offset returned.
- You can call commitSync/commitAsync and pass a map of partitions/offsets that you wish to commit.

### Rebalance Listeners

- A consumer wants to do cleanup work before exiting and before partition rebalancing
- If you know a consumer will lose ownership of a partition, you will want to finish your works and commit offset. Perhaps you want to close database connections and file handlers.
- You can configure a ConsumerRebalanceListeners and there are two methods you can implement:
    - `onPartitionsRevoked(partitions)`
        - Called before rebalancing and after consumer stopped consuming messages. This is where you would comit offsets. So whoever gets this partition next will know where to start.
    - `onPartitionsAssigned(partitions)`
        - Called after partitions have been reassigned to the broker, but before the consumer starts consuming messages.

### Consuming Records with Specific Offsets

- poll will start consuming from the last commited offset in each partition.
- you can start cnosuming new messages only (seekToEnd) and start from the beginning of a partition (seekToBeggining) or from a specific offset.
- there is a way to atomic store the record and offset in one action. you can store offsets externally and only commit DB transaction after storing the offset.
- For exit use shutdown hook + wakeup exception.

### Consuming Alone

You can consume without a consumer group by assigning yourself partitions instead of subscribing to a topic.

## Kafka Internals

### Cluster Membership

- Kafka uses Zookeeper to maintain a list of brokers that are part of a cluster.
- Different kafka components subscribe to /brokers/ids path in zookeeper where brokers are notified when brokers are added/removed.
- When a broker loses conectivity to Zookeeper, the ephemeral node created when starting the broker will be removed from zookeeper. Other kafka components watching this list will be notified.
- If you join the cluster with the same ID of the old broker, you will be assigned the same partitions/topics.

### Controller

- The controller is a kafka broker that elects partition leaders.
- The first broker that starts in the cluster becomes the controller by creating an ephemeral node in ZooKeeper called /controller.
- When other brokers start they also try to create this node but receive and exception (node already exists).
- Brokers create a zookeeper watch on the contrller node so they get notified of changes to this node.
- When the controller stops/loses connectivity to zookeeper, the ephemeral node disappears. Other brokers will be notified and will try to create the node again. When one creates the others will receive an exception and create a new watch on the new controller node.
- Each time a controller is elected, it receives a new, higher controller *epoch* number. The brokers know the current controller epoch, if they receive an message with an older epoch, they will ignore it.
- When the controller notices a broker left the cluster (by watching the relevant zookeeper path), it knows that all the partitions that had a leader on that broker will need a new leader.
    - It then goes over all partitions that need a new leader and determines who the leader whould be (next replica in replica list of that partition).
        - It will then send a request to all brokers that contain either the new leaders or the existing followers for those partitions. Each leader knows tthat it needs to start serving producers and consumer requests from clients while the followers know that they need to start replicating messages from the new leader.
- When the controller notices a broker joined, it uses the broker id to check if there are replicas that exist on this broker. If so, the controller notifies both new and existing brokers of the change, and the replicas on the new broker start replicating messages from existing leaders.

Summary:

1. Kafka uses zookeeper ephemeral node to elect a controller and notify joins/leaves on the cluster.
2. The controller elects leaders among the partitions and replicas whenever it notices nodes join and leave the cluster.
3. The controller uses epoch number to prevent two nodes believe in different controllers.

### Replication

- Replication is how kafka guarantees availability and durability.
- Partitions can have multiple replicas.
- Replicas are stored in brokers.
- Each broker stores hundreds or thousands of replicas.
- There are two types of replicas:
    - *Leader Replica*: each partition has a single replica designates as the leader. All produce and consume requests goes through the leader.
    - *Follower Replica*: followers don't serve client requests. They replicate messages from the leader and stay up-to-date with most recent messages. If the leader of a partition crashes, one of the follower will be the new leader.
- The leader is responsible for knowing which of the followers are up-to-date with the leader.
- Follower can fail to stay in sync, for network congestion, for example, or when a broker crashes and all replicas on that broker start falling behind until we start the broker and they can replicate again.
- In order to stay in sync, replicas send the leader FETCH requests (same type of requests  that consumers send).
- Requests are always made in order, so if message 4 is requested, it means that all messages before were received.
- If a replica hasn't requestes a message in more than 10 secs or if the replica hasn't caught up with the most recent message in more than 10 secs the replica is considered *out-of-sync*. Replicas out-of-sync can't become leaders in event of failures.
- In-sync replicas can become leaders.
- `replica.lag.time.max.ms` is the amount of time a follower can be inactive to be considered out of sync.
- The preferred leader is the replica that was the leader when the topic was created. `auto.leader.rebalance.enable=true` by default will cause in-sync preferred leaders to become leader for the partition.

> When using `kafka-topics.sh` you can find the preferred leader, it is the first one in the list.

### Request processing

Most common requests: produce, fetch and metadata.

Requests are processed as they are received.

All request have standard headers:
- Request Type (also called API Key)
- Request Version (so Brokers can Handle different versions)
- Correlation ID (Unique identifier of the request)
- Client ID (Identifies application that sent the request)

For each port a broker is listening on, the broker runs an *acceptor* thread that creates a connection and hands it over to a *processor* thread for handling (also called network threads). This amount can be configured. Network threads are responsible for taking requests from client connections, placing it on a queue and picking up responses from a response queue and sending them back to clients.

Once requests are placed on request queue, *IO threads* are responsible for picking them and processing, the most common type of requests are:
- *Produce Requests* - sent by producers, contain messages to write
- *Fetch requests* - sent by consumer and follower replicas when they read messages from kafka brokers.

Requests (fetch/produce) not sent to the leader of the partition receive an error "Not a Leader for Partition".

- They know who is the leader by using *metadata request*, which includes a list of topics the client is interested in.
- The server shows which partitions exist in topics, the replicas for them, and which broker is the leader.
- Metadata requests can be sent to any broker (brokers have a metadata cache that is updated periodically).
- Clients refresh intervals are controlled by (`meta.data.max.age.ms`)

#### Produce Requests Processing

- Leader broker when receives a produce request will do some validations:
    - Does the user sending have write privileges on the topic?
    - Is the number of `acks` specified in the request valid?
    - If `acks=all`, are there enough in-sync replicas for safely writing this message?
- After making some validations, it will write messages to the local disk. On linux, messages are written to the filesystem cache (marked as dirty pages and will eventually be flushed to disk). Kafka does not wait till data get persisted to disk - it relis on replication for message durability.
- After writing on leader, the broker will look at `acks` and if it is `0` or `1` it will reply immediately; if set to `all`, the request is stored in a buffer called *purgatory* until the leader observes that the follower replicas replicated the message, and then replies to the client.

#### Fetch Requests Processing

- Clients sends requests specifying which topic/partitions/offsets they want messages from, like "send me messages starting at offset 13 in partition 0 of topic test and messages starting at offset 54 in partition 3 of topic test".
- Clients can specify the maximum amount of data the broker can return for each partition (clients need to allocate memory that will hold the response sent back from the broker). Without limit, this could cause large replies to make the client run OOM.
- The request goes to the leader, if then checks if it is valid: does this offset even exist? if it is so old and was deleted, it will reply with an error.
- If the offset exists, the broker will read messages from the partition, up to the limit set by the client (in the request).
- After reading the partition, it sends the messages to the client.
- Kafka uses `zero-copy` method to send messages to the client (kafka sends messages from the file (linux filsystem cache)) directly to the network channel without any intermediate buffers. it is different from most databases, where data is stored in a local cache before being sent to clients. this removes overhead of copying bytes and managing buffers in memory.
- In addition to upper boundary limites, clients can set a lower boundary
    - for example, only return resultes once you have at least 10K bytes to send me. This reduces CPU and network utilization when clients are reading from topics that are not seeing much traffic.
    - Intead of sending requests to the brokers every few ms and asking for data and receiving nothing in return, the client sends a request, the broker waits until there is a certain amount of data and returns the data.
    - after a while, a timeout is triggered, and the broker sends what it gots, which might be nothing.
- Some data written to the leader might not be available for clients to read, since most clients can only read messages that were written to all in-sync replicas. attemps to fetch those messages will result in empty response rather than an error.
    - Because if messages are not replicated to enough replicas they are considered "unsafe", these message might no longer exist in kafka in case of crashes or other replicas take its place. Allowing clients to read messages that only exist in the leade can cause inconsistency. Example: consumer reads message from leader, leader crashes and message no longer exists.
    - If we wait till all in-sync replicas, it means that if the replication is slow, the messages new messages will take longer to arrive to consumers; This delay is mited to `replica.lag.time.max.ms`, the amount of time a replica can be delayed in replicating new messages while still being considered in-sync.
    - Consumer only see messages that were replicated to all replicas!

#### Other Requests

All kafka clients communicate with kafka brokers using kafka binary protocol over the network.

It is recommended updating brokers before clients, because brokers can handle multiple versions of requests.

### Physical Storage

- Basic storage unit of kafka is a partition replica.
- Partitions can't be split between multiple brokers and not even between multiple disks on the same broker.
- The size of the partition is limited by the space available on a single mount point.
    - A mount point will be either a single disk (using JBOD) or multiple disks if RAID is configured.
- the `logs.dirs` parameter is where partitions will be stored. (not to be confused with error log, configured in `log4j.properties` file).
- the usual configuration includes a directory for each mount point that Kafka will use.

#### Partition Allocation

Goals on allocation:
- Spread replicas evenly among brokers.
- Make sure that for each partition, each replica is on a different broker.
    - Example: if partition 0 has the leader on broker 2, we cannot place followers on 2 and not both on 3, but place on 3 and 4.
- If the brokers have rack informaion, assign replicas for each partition to different racks, if possible.

To do this, start with a random broker and start assigning partitions to each broker in round-robin.
Example: broker 4 is the random broker selected, it wil be partition leader 0, partition 1 leader will be on broker 5 and partition 2 leader will be on broker 0 (if you have 6 brokers enumerated from 0 to 5).

Then, for each partition, replicas are places at increasing offsets from the leader.
Example: if the leader for partition 0 is on broker 4, the first follower will be on broker 5 and the second follower on broker 0. The leader for partition 1 is on broker 5, so the first replica is on broker 0 and the second on broker 1.

If you have rack awareness: instead of picking brokers in numerical order, a rack-alternating broker list is created.
Example: if you have broker 0, 1 and 2 on the same rack and 3,4 and 5 on separate rack. Instead of picking brokers in order 0 to 5 the order is : 0,3,1,4,2,5.

After choosing brokers for each partition/replica. it is time to decide which directory to use for new parittions.
- We count the number of partitions on each directory and add the new partition to the directory with fewest partitions. This means, if you add a new disk, all new partitions will be created on that disk.Because, until things balance out, the new isk will always have the fewest partitions.

> Note that disk size/available space is not taken into account, this means that you can have some parititons abnormally large.

#### File Management

- Partitions are splitted into *segments*. By default, each segment contains 1GB of data or a week of data.
- The segment kafka is writing to currently is called *active segment*. The *active segment* is **NEVER** deleted.
- Log retention is applied to closed segments, this means if you delete segments every day but each segment contains five days of data, you will wait 5 days because active segment cannot be deleted.
- If you have 7 days retention and you close a new segment every day, you will see that every day the oldes segment is deleted, most of the time the partition will have 7 segments.
- Kafka has one file handle open to every segment in every partition. This leads to a unuasually high number of open file handles.

#### File Format

- Each segment is stored in a single data file.
- Inside the file we store kafka messages and their offsets.
- The format of the data on the disk is identical to the format of the messages we send from the producer to the broker. and from the broker to the consumers.
    - This allows zero-copy optimization when sending messages, also avoid decompressing/recompressing messages.
- Each message contains:
    - (key,value,offset)
    - (message size, checksum code, magic byte (version of the message format), compression codec, timestamp). Timestamp is given either by the producer or by the broker when the message arrived - depending on configuration.
- `DumpLogSegment` allows you to look at a partition segment in the filesystem and examine its contents. It will show you the offset, checksum, magic byte, size and compression codec for each message:
```bash
bin/kafka-run-class.sh kafka.tools.DumpLogSegments
```
if you use `--deep-iteration` parameter it will show you information about messages compressed inside the wrapper messages.

#### Indexes

- Since kafka allows reading from any offset, it needs to efficiently lookup messages on the offset specifid. It does so by using an index for each partition.
- The index maps offsets to segment files and positions within the file.
- Indexes are also broken into segments, so we can delete old index entries when messages are purged.
- If indexes becomes corrupted, it will get regenerated from the matching log segment by rereading messages and recording the offsets/locations.
- They are generated automatically.

#### Compaction

- There are scenarios where you don't want to keep data for a period of time.
    - Example: you use kafka to store shipping addresses for your constumers.
    - In this case, it is better to store the last address for each consumer rathen than data for just the last week.
    - This way, you don't need to wory about old addresses and you still retain the address for costumers who haven't moved in a while.
    - Can also be used for applications that use kafka to store the state. Every state change is written to kafka.
        - In this case, application can recover from its latest state.
- Kafka support these different scenarios by allowing retention policy to be:
    - `delete`: which deletes events older than retention time
    - `compact`: only stores the most recent value for each key in the topic. Therefore, you can't have "null" keys, compaction would fail in that scenario, you must specify keys when using this.

##### How compation works

Each log is viewed as split into two portions:
- *Clean*
    - Messages that have been compacted before. Contains only one value for each key, which is the latest value at the time of the previous compaction.
- *Dirty*
    - Messages that are written after last compaction

Compaction is enable by using `log.cleaner.enabled`.
If compaction is enable, each broker will start a compaction manager thread and a number of compaction threads.
These thread are responsible for compaction.
Each of these threads chooses the partition with the **highest ratio of dirty messages to partition size** and cleans this partition.

This is what the cleaner thread does:

It reads the dirty section of the partition and creates an in-memory map. each map entry is a 16-byte hash of a message key and the 8-byte offset of the previous message that had this same key. This means each map entry only uses 24 bytes. If we are compacting a 1GB segment that has 1KB messages, the segment will contain 1million messages and we would need 24Mb map to compact the segment.

When configuring kafka, the administrator configure how much memory compaction threads can use for this offset map. Even though each thread has its own map, the configuration is for total memory across all threads.
Kafka does not require that the entire dirty section of the partition has to fit in the size allocated for that map, but at leas one full segment has to fit. If it doesn't kafka will log and error, and the admin would have to allocate more memory or use fewer cleaner threads (memory is divided by cleaner threads),

If only a few segments fit, kafka will start by compacting the oldest segments that fit into the map. The rest will remain dirty and wait for the next compaction.

Once the cleaner thread build the offset map, it will start reading off the clean segments, starting with the oldest, and check their contents against the offset map. For each message it checks, if the key of the message exists in the offset map.
- If the key does not exist in the map: the value of the message we've just read is the latest and we copy over the message to a replacement segment.
- If the key does exist: we omit the message because there is a message with an identical key but newer value later in the partition.

Once we copied all the messages that contain the latest value for their key, we swap the replacement segment for the original, and move on to the next segment. At the end of the process, we are left with one message per key.

##### Deleted Events

- To delete a key from the system completely (not even saving the last message), the application must produce a message that contains that key and a null value.
- When the cleaner thread find this message it will do a normal compaction and retain the message with the null value (called *tombstone*).
- Databases can delete from database when they receive a tombstone message.
- Kafka keeps tombstone message for a configurable amount of time.

##### When are topics compacted

- *current segment* is **NEVER** compacted.
- In version 0.10.0 and older kafka will start compacting when 50% of the topic contains dirty records.

## Administering Kafka

Kafka CLI utilities are implemented in Java classes, and a set of scripts are provided to call those classes properly.

### Topic Operations (`kafka-topics.sh`)

You can create, modify, delete and list information.

Creating a topic: `./kafka-topics.sh --replication-factor INT --bootstrap-server STR --partitions INT --topic STR`, you can use `--disable-rack-aware` if rack aware assignment is not desired.
Adding partitions: `./kafka-topics.sh --topic STR --partitions INT`
Deleting a topic: `./kafka-topics.sh --topic STR --delete`
List topics: `./kafka-topics.sh --topic STR --list`
Describe topics: `./kafka-topics.sh --describe`. Will provide useful information. You can filter out-of-sync replicas using `./kafka-topics.sh --describe --under-replicated-partitions`, this will show all parititons where one or more replicas for the partition are not in-sync with the leader. the `--unavailable-partitions` shows all partitions without a leader (this means that the partition is offline and unavailable for produce/consume clients)

### Consumer Groups Operations

To list consumer groups using the older consumer clients execute with `--zookeper` and `--list parameter`, for the new consumer, use `--bootstrap-server`, `--list` and `--new-consumer`
```bash
kafka-consumer-groups.sh --zookeeper zoo1.example.com:2181/kafka-cluster --list
kafka-consumer-groups.sh --new-consumer --bootstrap-server kafka1.example.com:9092/kafka-cluster --list
```

You can then describe for each group using `--describe` and `--group` params.

```bash
kafka-consumer-groups.sh --zookeeper zoo1.example.com:2181/kafka-cluster --describe --group testgroup
```

The output contains the following fields:
| Field         | Description                                                                                                       |
|---------------|-------------------------------------------------------------------------------------------------------------------|
| GROUP         | The name of the consumer group.                                                                                    |
| TOPIC         | The name of the topic being consumed.                                                                              |
| PARTITION     | The ID number of the partition being consumed.                                                                     |
| CURRENT-OFFSET| The last offset committed by the consumer group for this topic partition. This is the position of the consumer within the partition. |
| LOG-END-OFFSET| The current high-water mark offset from the broker for the topic partition. This is the offset of the last message produced and committed to the cluster. |
| LAG           | The difference between the consumer Current-Offset and the broker Log-End-Offset for this topic partition.       |
| OWNER         | The member of the consumer group that is currently consuming this topic partition. This is an arbitrary         |

You can delete consumer groups only on older clients:
```bash
kafka-consumer-groups.sh --zookeeper zoo1.example.com:2181/kafka-cluster --delete --group testgroup
```
You can also delete the offsets for a single topic: `kafka-consumer-groups.sh --zookeeper zoo1.example.com:2181/kafka-cluster --delete --group testgroup --topic my-topic`

#### Offset Management

You can retrieve the offsets and store new offsets in a batch. Useful for reseting offsets, or advancing offsets past a message that the consumer is having problem with.

Back in the day there was no script to export offsets, but you could achieve it using `kafka-run-class.sh` to execute underlying java class for the tool.

##### Exporting

Exporting offsets will produce a file that contains eachc topic/partition for the group and its offsets in a defined format that the import tool can read. It has the following format: `/consumers/GROUPNAME/offsets/topic/TOPICNAME/PARTITIONID-0:OFFSET`.

```
# kafka-run-class.sh kafka.tools.ExportZkOffsets --zkconnect zoo1.example
# cat offsets
/consumers/testgroup/offsets/my-topic/0:8905
/consumers/testgroup/offsets/my-topic/1:8915
/consumers/testgroup/offsets/my-topic/2:9845
/consumers/testgroup/offsets/my-topic/3:8072
/consumers/testgroup/offsets/my-topic/4:8008
/consumers/testgroup/offsets/my-topic/5:8319
/consumers/testgroup/offsets/my-topic/6:8102
/consumers/testgroup/offsets/my-topic/7:12739
```

##### Importing

Takes the file produced and uses it to set the current offsets for the consumer group.

To import offset for the consumer group named testgroup from the example above, use `kafka-run-class.sh kafka.tools.ImportZkOffsets --zkconnect zoo1.example.com:2181/kafka-cluster --input-file offsets`

> All consumers must be stopped, they will not read new offsets if they are written while the group is active. Consumers would just overwrite the imported offsets.

### Dynamic Configuration Changes

The `kafka-configs.sh` is a CLI tool that allows you to set configurations for topics/client IDs. Once set, these configurations are permanent for the cluster, stored in zookeeper and read by each broker when it starts.

#### Overriding Topic Configuration Defaults

Most configurations have a defualt specified in broker configuration, which will apply unless an override is set. To change topic configuration you can use:

```bash
kafka-configs.sh --zookeeper zoo1.example.com:2181/kafka-cluster --alter --entity-type topics --entity-name <topic name> --add-config <key>=<value>[,<key>=<value>...]
```

Some configurations for the topic:

| Configuration Key                   | Description                                                                                                                                                           |
|------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `cleanup.policy`                   | If set to compact, the messages in this topic will be discarded such that only the most recent message with a given key is retained (log compacted).                  |
| `compression.type`                 | The compression type used by the broker when writing message batches for this topic to disk. Current values are gzip, snappy, and lz4.                                 |
| `delete.retention.ms`              | How long, in milliseconds, deleted tombstones will be retained for this topic. Only valid for log compacted topics.                                                   |
| `file.delete.delay.ms`             | How long, in milliseconds, to wait before deleting log segments and indices for this topic from disk.                                                                 |
| `flush.messages`                   | How many messages are received before forcing a flush of this topic’s messages to disk.                                                                               |
| `flush.ms`                         | How long, in milliseconds, before forcing a flush of this topic’s messages to disk.                                                                                   |
| `index.interval.bytes`             | How many bytes of messages can be produced between entries in the log segment’s index.                                                                                |
| `max.message.bytes`                | The maximum size of a single message for this topic, in bytes.                                                                                                       |
| `message.format.version`           | The message format version that the broker will use when writing messages to disk. Must be a valid API version number (e.g., “0.10.0”).                              |
| `message.timestamp.difference.max.ms` | The maximum allowed difference, in milliseconds, between the message timestamp and the broker timestamp when the message is received. This is only valid if the message.timestamp.type is set to CreateTime. |
| `message.timestamp.type`           | Which timestamp to use when writing messages to disk. Current values are CreateTime for the timestamp specified by the client and LogAppendTime for the time when the message is written to the partition by the broker. |
| `min.cleanable.dirty.ratio`        | How frequently the log compactor will attempt to compact partitions for this topic, expressed as a ratio of the number of uncompacted log segments to the total number of log segments. Only valid for log compacted topics. |
| `min.insync.replicas`              | The minimum number of replicas that must be in-sync for a partition of the topic to be considered available.                                                          |
| `preallocate`                      | If set to true, log segments for this topic should be preallocated when a new segment is rolled.                                                                     |
| `retention.bytes`                  | The amount of messages, in bytes, to retain for this topic.                                                                                                          |
| `retention.ms`                     | How long messages should be retained for this topic, in milliseconds.                                                                                                |
| `segment.bytes`                    | The amount of messages, in bytes, that should be written to a single log segment in a partition.                                                                     |
| `segment.index.bytes`              | The maximum size, in bytes, of a single log segment index.                                                                                                           |
| `segment.jitter.ms`                | A maximum number of milliseconds that is randomized and added to segment.ms when rolling log segments.                                                                |
| `segment.ms`                       | How frequently, in milliseconds, the log segment for each partition should be rotated.                                                                               |
| `unclean.leader.election.enable`   | If set to false, unclean leader elections will not be permitted for this topic.                                                                                      |

#### Overriding Client Configuration Details

You can configure maximum produce/consume rate for a single client id using:

```bash
kafka-configs.sh --zookeeper zoo1.example.com:2181/kafka-cluster --alter --entity-type clients --entity-name <client ID> --add-config <key>=<value>[,<key>=<value>...]
```

Supported keys are: `producer_bytes_rate` and `consumer_bytes_rate`.

#### Describing Configuration Overrides

You can display all configuration overrides using cli tools, for example, to list config overrides for "my-topic": `kafka-configs.sh --zookeeper zoo1.example.com:2181/kafka-cluster --describe --entity-type topics --entity-name my-topic`

> To delete a config override you should use: `kafka-configs.sh --zookeeper zoo1.example.com:2181/kafka-cluster --alter --entity-type topics --entity-name my-topic --delete-config retention.ms`

### Partition Management

There are two script: one for reelection of leader replicas and a low-level utility for assigning partitions to brokers.

#### Preferred Replica Edition

When brokers is stopped and restarted it does not resume leadership of any partitions automatically, you can do this by trigerring a preferred replica election. For example:

```
kafka-preferred-replica-election.sh --zookeeper zoo1.example.com:2181/kafka-cluster
```

> if request is greater than 1MB, you will need to create a file that contains a JSON listing partitions to elect.

#### Changing Partition Replicas Assignments

Some examples where this might be needed:

* If a topic’s partitions are not balanced across the cluster, causing uneven load on
brokers
* If a broker is taken offline and the partition is under-replicated
* If a new broker is added and needs to receive a share of the cluster load

You should use `kafka-reassign-parititons.sh`

#### Changing Replication Factor

Use the reassignment tool but increasing/decreasing replicas.

#### Dumping Log Segments

If you have to go looking for the specific content of a message, there is a helper tool to decode log segments for a parition. The tool takes a comma-separated list of log segment files as an argument and can print out either message summary information or detailed message data.

For example, decode log segment file name *00000000000052368601.log*: `kafka-run-class.sh kafka.tools.DumpLogSegments --files 00000000000052368601.log` you can add `--print-data-log` to print message data.

You can also validate the index file that goes along with a log segment. The option --index-sanity-check will just check that the index is in a useable state, while --verify-index-only will check for mismatches in the index without printing out all the index entries

#### Replica Verification

To validate that the replicas for a topic’s partitions are the same across the cluster, you can use the kafka-replica-verification.sh tool for verification. This tool will fetch messages from all the replicas for a given set of topic partitions and check that all messages exist on all replicas. You must provide the tool with a regular expression that matches the topics you wish to validate. If none is provided, all topics are validated.

### Consuming and Producing

There are two utilities provided to help with this: kafka-console-consumer.sh and kafka-console-producer.sh.

#### Console consumer

Displays raw bytes in the message to stdout using DefaultFormatter.

You can use --consumer.config CONFIGFILE. or -consumer-property KEY=VALUE to configure consumer behaviour.

You can use --from-beginning to consume from the beginning

You can use --max-messages to consume at most MAX messages

You can use --partition NUM to consume only from the specific partition.

##### Formatter Options

Consumes messages but does not output them at all. The kafka.tools.DefaultMessageFormatter also has several useful options that can be passed using the --property command-line option:

print.timestamp

Set to “true” to display the timestamp of each message (if available).

print.key

Set to “true” to display the message key in addition to the value.

##### Consuming The Offset Topics `__consumer_offsets`

You may want to see if a particular group is committing offsets at all, or how often offsets are being committed

```
kafka-console-consumer.sh --zookeeper zoo1.example.com:2181/kafka-cluster --topic __consumer_offsets --formatter 'kafka.coordinator.GroupMetadataManager$OffsetsMessage Formatter' --max-messages 1 [my-group-name,my-topic,0]::[OffsetMetadata[481690879,NO_METADATA] ,CommitTime 1479708539051,ExpirationTime 1480313339051]
```

#### Console Producer

By default, read from stdin where TABs separates key and value.

You can use a config file or specify properties, similar to the consumer.

You can use --sync to produce synchronously.

You can use --compression-codec STRING to produce using a compresion.

You can use custom key/value serializer also.

### Client ACL's

kafka-acls.sh, provided for interacting with access controls for Kafka clients.

### Unsafe Operations

- Moving Cluster Controller.
- Killing partition Move.
- Removing topics to be deleted.
- Deleting Topics manually.

# References

1. Kafka: The Definitive Guide by Neha Narkhede, Gwen Shapira, and Todd Palino (O’Reilly). Copyright 2017 Neha Nar‐khede, Gwen Shapira, and Todd Palino, 978-1-491-93616-0.
2. Apache Kafka Message Compression. (n.d.). Confluent. https://www.confluent.io/blog/apache-kafka-message-compression/
