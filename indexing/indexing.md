
# Indexing

There are many things can be improved in the indexing stage. A regular setup looks like this:

```
log file => local log harvester => remote log parser => kafka
kafka => log indexer => elasticsearch indexing node => elasticsearch shards => lucene docvalues
```

It is a terribly long process. It might be fragile, but that is not the main issue. We can always use excessive monitoring to fix any broken process. The main issue I can not bear with is the huge amount of  data copying that can be saved. The raw indexing performance should be way more efficient if we can do it right. A lot of hardware and bandwidth cost can be saved. But how?

If we look at the final destination of the pipeline, it is the lucene doc-values setting in the file system. The file format is very compact, if you have a column of long values, it is just a bunch of numbers one by one, packed together. Essentially it is a file representation of ```long[]```. So the indexing process is all about filling a bunch of ```long[]```, each column will have a separate array to fill. For example:

```
String[] symbolColumn = new String[1024 * 1024];
long[] marketCapColumn = new long[1024 * 1024];
long[] lastSaleColumn = new long[1024 * 1024];
symbolColumn[0] = 'AAPL';
marketColumn[0] = 52269;
lastSaleColumn[0] = 94;
symbolColumn[1] = 'GOOG';
marketColumn[1] = 43234;
lastSaleColumn[1] = 66;
```
When we ship log, it is very natural to accumulate a block of rows in memory and ship together. If we can organize the memory block in column-stride fashion, then the bytes can be directly dumped into the final destination with maximum speed.

```
long[] output = new long[1024 * 1024];
for (int i=0; i<input.length; i++) {
  output[i] = input[i];
}
```
This kind of code can be optimized by the JVM hotspot automatically (http://psy-lob-saw.blogspot.com/2015/04/on-arraysfill-intrinsics-superword-and.html). In modern cpu and JVM, per column process can be very efficient.

If we want to process in column stride fashion, we have to know the schema of data, how many columns and what is the column data type. Current ELK tooling is geared towards schema-less setup, and tend to represent the data as ```Map<String, Object>``` as each row, and ```List<Map<String, Object>>``` as a bunch of rows. This is not the most efficent way of dealing with time series data or OLAP. If we bring back strong schema, and build a new set of tooling leverage the fact we can use ```long[]``` instead of ```Map<String, Object```, it should pump data from data source to Lucene doc-values faster than current setup(without bench-marking to back the theory yet). Elasticsearch is a very good tool to store, and qury from, because in the storage layout it is strong typed and packed the data very compactly for each different data type. But the ingestion process does not have special fast-lane for already structured and typed data. In following sections, I am going to examine the current indexing process and give my 2 cents on how to improve it.


## Log file => Local log harvester => remote log parser

### Packet Encoding

We collect logs from where data is generated. At those places, CPU cycles are much precious, and should be saved to better serve our customer. So it is general best practices to do minimum parsing and just ship the data to remote side for further processing. Some parsing is necessary as we have to send the data in small chunks and shuffle them to different remote servers. If the chunk was not splitted in the right boundary, then the remote server can not handle the data properly. The most commonly used boundary is "\n" to separate log lines. 

Not all log files are not structured. Some log file such as the binlog of mysql server, it is very compact and structured. "\n" is just a less-sophisticated form of packet encoding to convey the meaning of "event". There are much formal ways to encode a "event" into a packet. If the data source can generate stream of events in easy to parse packet form, the performance can be improved. For example:

```
[packe_size][... remaining bytes ...][packet_size][... remaining bytes ...]
```
It is very common to have a header containing the size of packet. So that it is easy to chop out the bytes from the stream without needing to understand the whole structure. "\n" is not the best solution here.

### Inter-process Communication

```
Data source =IPC=> Harvester =RPC=> Remote 
```
Log file is a means of IPC between data source process and harvester process. Use file to pass events from one process to another might not be very efficient, but modern Linux system optimized the pattern efficient enough.
We can implement the IPC side a unix domain socket. Then the harvester can be a TCP proxy to relay events from data source to remote. If one data source correspond to only one remote, then the process can be as fast as direct copy. 

### Shuffling

It is common requirement to shuffle the data in the process. For example, in the log file
```
LOGIN,xxxx
LOGOUT,xxx
LOGIN,xxx
LOGIN,xxx
```
The data of login and logout might need to be send to different kafka topic. So the harvester or the remote parser will need to shuffle the data from same source to different destination. It is better to generate several log files from the upstream. The cost of picking and shuffling is just for programming convenience. After all, the events are normally generated from different site of the code, why not just output to different log files?

### Status

Current agent to collect data for Elasticsearch is assuming the data is unstructured log, and trying to put as many features in it. The result is slow agent with too many CPU used when the volume is high. We need a new agent for structured and typed data to handle vast amount of data for time series and IOT. The agent can be

* a unix domain socket server listen at 127.0.0.1
* binary compact packet encoding, without need to split at "\n"
* allow upstream to specify the destination in the packet header
* or let upstream to specify the destination in the TCP connection level, so the harvester agent can just forward the stream to remote server

## local log harvester => remote log parser => kafka

### Packet Parsing

Regex parsing is slow. JSON parsing is faster. Binary encoding such as msgpack or avro is much much better. If the packet is already in json or even avro, the log parser is optional then. We can store input bytes directly in kafka.

### Shuffling

The cost of remote log parser is not just parsing some string according some complex regex. It also introduce a lot of copying. Think about:

```
data source machines1 => log parser machine
data source machines2 => log parser machine
...
data source machines1000 => log parser machine
```

It is easily to overwhelm the central parsing machine. So we distribute the load
```
data source machines1 => log parser machine1
data source machines2 => log parser machine1
...
data source machines1000 => log parser machine10
```
But the backend kafka server is also distributed.
```
data source[K] => log parser[L] => kafka[M]
```
The data will copy from data source cluster to log parser cluster and then to kafka cluster. It is very hard to co-locate the log parser and kafka in the same machine, as the whole point of log parsing could be shuffle different event to different topic in kafka. Then we are looking at double the bandwidth cost compared to kafka only solution.

### Status

It is best practice to have a central logstash cluster setup in the middle to parse the logs. It has huge cost in terms of parsing and bandwidth. If we have structured and typed data already, we can skip this process altogether, just store the raw bytes in the right kafka topic, from the beginning the data stream. The reason we do not normally write kafka directly, but rather use a log based parsing solution is:

* we do not want to slow down the application server
* file is reliable, in case of network partition, we can hold up

This can be solved by a better local agent discussed above. It can be

* async
* use batching internally
* memory mapped file based queueing

In case of failure, the memory mapped file can provide the safety we need. Essentially, we implement some kind of write-ahead logging using local file system, and store the data async to remote kafka cluster.

## Log file => Kafka

Kafka is just another form of log file. We use a local agent implement write ahead logging to support reliable transfer stream of events from local machine to central kafka. It should be reliable and fast this way.
But why we need to copy log file from one form to another? Well, kafka is in the central server and have well defined topics to consume from. The benefit of having a publicly accessible event stream out-weight the cost of log shipping.

## kafka => log indexer => elasticsearch nodes

### JSON

Elasticsearch only speaks JSON. Any format must be converted to JSON to index. If we want to be fast, we need to modify elasticsearch to let it speak another language.

### Data shuffling

If the log indexer is a node client, then the data is just copied from kafka to indexer, then from indexer to es nodes. If the log indexer is not a node client, then the data will be copied three times.

```
kafka => log indexer => elasticsearching indexing node => elasticsearch shard WAL => lucene
```

Between final lucene file and kafka, there are too many stages. If the data is structured and typed, then what is the difference between kafka and WAL of elasticsearch shard?

### 



