There are a lot of existing solutions. They all have some good internal design that makes them particularly good at some aspect, by pushing the underlying storage to its limit. Learning from the good old experience is a necessary to build a better one.


# Opentsdb

Opentsdb is a good presentive of fast k/v based solution. It makes very good use of underlying HBase features to maximize the performance and minimized the cost to store large volume of data.

## TSUID

http://opentsdb.net/docs/build/html/user_guide/uids.html

Opentsdb support metric name, tags. But metric name and tags are not stored in each data point to save space. Instead, each metric name + tags combination will be assigned with a TSUID (time series unique id), then each data point only need to store TSUID with it. When querying the data, first translate the metric name and tags to TSUID, then load the data points using TSUID.

![](opentsdb-tsuid-mapping.jpg)

This is the bi-directional mapping table for TSUID and raw metric name + tags

![](opentsdb-tsuid.jpg)

Metric name + tags will be combined to a long TSUID. take the above example

```
proc.loadavg.1m host=web42 pool=static
metric_name tagk=tagv tagk=tagv ....
052 001 028 047 001
052001028047001 is the final TSUID used
```

By default, UIDs are encoded on 3 bytes in storage, giving a maximum unique ID of 16,777,215 for each UID type. Given tag value is also encoded as TSUID, when the tag value contains something like QQ number, very quickly we can run out of 16 million limit. To change the 3 bytes limit to 4 bytes or more, requires recompile Opentsdb itself, and existing data will not be compatible.

Scanning all data with same metric name but different tags is possible. TSUID is part of the hbase row key, so scan the sequential rows can load metric at one timestamp of all tags. Take the above example, search rows start with 052 can give us all data of metric "proc.loadavg.1m"

The key idea is using table lookup (or dictionary encoding) to compress low cardinal repeated values (the dimensions of time series data). It is a common technique to compress string data in analytical database. Opentsdb TSUID is just a sepcial purpose implementation.

## HBase Scan

HBase can scan a large number of records in very short time. The scan limited to scan along the sort order defined by row key. Defining the right row key to support the query, so that scan can be most optimal is the key to fast HBase usage. Essentially, Rowkey in HBase is the index in database terms, and the only index. Not only for fast single record retrival, but also for large range scan.

This is the rowkey for opentsdb

![](opentsdb-scan.jpg)

Rowkey is in the format: metric_name+timestamp then 8 optional tagk,tagv pair

So query "proc.loadavg.1m" between 12:05 and 13:00 is a very fast range scan to batch load all the data. As the data is first sorted by metric name and then sorted by timestamp. Data of other metric will not affect the query performance. Data at yesterday will not affect the query performance.

But if the query is about "proc.loadavg.1m{host=web1}" then we have to scan all "proc.loadavg.1m" within the time range, then filter out host=web1 in memory. If there are 1 million hosts per minute, then we need to load 1 millions records for 1 minute, but only keep 1 record, which is very slow. So a common techique is to encode the tagk/tagv in the metric itself. Then we will have metric like "proc.loadavg.1m.host.web1".

## Compact

The idea of compact to store 1 minute even 1 hour data per row, instead of 1 second data per row. So we can reduce the number of rows to speed up the query performance. But to keep the insert fast, we are not going to rewrite the row for every insert, instead the insert will be 1 second level, then periodically the 1 second data will be "compacted" to 1 minute or 1 hour.

![](opentsdb-compact.png)

The cost of row is too heavy (for example, the dimensions need to stored again and again), it is better to pay the cost once and get as many as possible out of a single row. Note, even the per column cost is too high, opentsdb is not leveraging the multi-column feature, instead everything is packed into the same column.