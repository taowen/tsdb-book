In this chapter, we explore the features provided by Elasticsearch that can be used to build a agile time series database. We can see how good experience from existing solutions can be applied to Elasticsearch, also how to mitigate the problems of existing solutions.

# Architecture

To answer query like

```
SELECT AVG(age) FROM login WHERE timestamp > xxx AND timestamp < yyy AND site = 'zzz'
```

The query need to go through three things (in RDBMS terms):

![](three-steps.png)

In Mysql, the three steps are:

1. look up the filters in its b-tree index, and found a list of row ids
2. use the row ids to retrieve rows from the primary row store
3. compute the group by and projections in memory from the loaded rows

Using Elasticsearch as a database is very similar:

1. look up the filters in its inverted index, found a list of document ids
2. use the document ids to retrieve the values of columns mentioned in projections
3. compute the group by and projections in memory from the loaded documents

Conceptually despite Elasticsearch is a full-text index not a relational database, the process are nearly identical. The differences are:

* Elasticsearch is using inverted index instead of b-tree index to process filters
* Mysql row is called document in Elaticsearch
* Mysql has only one way to load the whole row from row store using row ids, Elaticsearch has 4 different ways to load the values using document ids

Elaticsearch is faster and more scalable than mysql for time series data because:

* Inverted index is more efficient to filter out large number of documents than b-tree index
* Elasticsearch has a way to load large number of values using the document ids efficiently, mysql does not which makes its b-tree index useless in this context
* Elasticsearch has a fast and sharded in memory computation layer to scale out, and there are optimization to bring in SIMD to optimize the speed further

Let's go through them one by one, I am sure you will love Elasticsearch.

# Finite State Transducers (FST)

There is one article that explains inverted index very well: http://blog.parsely.com/post/1691/lucene/

The example from the article illustrated what is inverted index very well:

```
doc1={"tag": "big data"}
doc2={"tag": "big data"}
doc3={"tag": "small data"}
```

The inverted index for the documents looks like:

```
big=[doc1,doc2]
data=[doc1,doc2,doc3]
small=[doc3]
```
b-tree index is not that different from end-user perspective, b-tree also invert the rowid=>value to value=>rowid, so that find rows contains certain value can be very fast.

There is a great article on how inverted is stored and queried using Finite State Transducers
http://www.cnblogs.com/forfuture1978/p/3945755.html

![](fst.png)

For people do not have time to read the article or lucene source code. It works like English dictionary. You look up the section, find the chapter, than find the material you are looking for.

# Bitset

In mysql, if you have two columns and you have query like "plat='wx' AND os='android'" then using b-tree index to index plat and os is not enough. In the runtime, if you have index for column plat and another index for column os, mysql have to pick one of most selective index to use, leave another index not used at all. There is a great presentation on how index work in mysql: http://www.slideshare.net/vividcortex/optimizing-mysql-queries-with-indexes

![](multi-column-index.jpg)

To index both column plat and os, then you have to create a multi column index to include both and even the ordering of the columns matters, index for plat,os and os,plat are different.

In elasticsearch, the inverted index are composable. You only need to index os, and plat separately. In the runtime, the both index can be used and combined to speed up the query. The filter plat='wx, os='android' can event be cached separately to speed up future querys.

The trick is called "roaring bitmap" (http://roaringbitmap.org/). Druid database has a detailed on article on how the magic works: http://druid.io/blog/2012/09/21/druid-bitmap-compression.html

There are two challenges to use bitset on the filter result. If we have 1 million documents, then for a given filter(for example plat='wx') there will be 1 million 1/0 result. How to represent the 1 million boolean result with minimum size is not a easy job.

This is a active research area to compress the sparse bitset. The filter result is sometime very sparse. For example user="wentao" will likely only have 1 hit out of 1 million documents. Storing nearly 1 million zeros will be such a waste. So the simplest compression is to represent a series of 1 or 0 using a compact form.

for example: ```11111111 10001000 11110001 11100010 11111111 11111111```

the last two words are all 1, so it can be compressed to

```
header section:   00001
content section:  11111111 10001000 11110001 11100010 10000010
```

Another challenge is to do AND/OR/NOT calculation for the compressed bitset. If we have to uncompress to do boolean logic operation, then the compression is not very helpful to save memory. Algorithms are invented to do the AND/OR/NOT on the compressed data.

The actual compression algorithm used in Elasticsearch (lucene is the underlying storage) is very complex and efficient. http://svn.apache.org/repos/asf/lucene/dev/trunk/lucene/core/src/java/org/apache/lucene/util/SparseFixedBitSet.java

The biggest benefit Elasticsearch can give us is to index a lot of columns separately. When doing the query, filters on different columns can be evaluated to bitset and cached efficiently. Then the filter result is AND/OR/NOT of the those individual bitset. There is no composite index in Elasticsearch, unlike mysql.

# DocValues

In mysql, having a bunch of row ids filtered from the b-tree index, then load from the row store are very time consuming "random access" process. Similarly, having a bunch of document ids filtered from the inverted index, how to load other fields of those documents efficiently?

There are 4 ways to load document values that in Elasticsearch, take field "user_age" as example:

* Using _source "stored field", the document is stored as json, need to parse and get the field "user_age"
* Using user_age "stored field" if it is stored
* If user_age is indexed in inverted index, Elasticsearch will un-invert the index to form a document_id=>user_age mapping in memory (called "field cache"), then load from the "field cache"
* If user_age is docvalues, then the field can be loaded from docvalues file

DocValues is a real game changer here. It turns Elasticsearch into a valid player in the analytical database market. DocValues is a column-oriented data store, optimized for batch loading.

![](int-columnar-store.jpg)

Above is how a int typed field gets stored on disk. There will be separate file for each DocValues field, within the file, values of documents are stored continuously. Each document occupy a fixed size to store its value of that field. So the memory mapped file looks like a array in memory, seek to the values of certain documents is a very fast operation. Batching loading a continuous documents is also very fast sequential read operation.

There is a very good video on DocValues:

* video: https://www.youtube.com/watch?v=JLUgEpJcG40
* slides: http://www.slideshare.net/lucenerevolution/willnauer-simon-doc-values-column-stride-fields-in-lucene

Mysql is a row oriented database, which means the rows are store one after another, each row has all fields in it. In row oriented database, there is no good way to load values of just one column out of 100. In column oriented database, it is very easy to just load the needed values. Also the column oriented layout is good for cache of CPU, as the values read is the values to be computed.

![](column-vs-row-oriented-database.png)

# Compression: bit packing

It is worth mentioning, Elasticsearch also compress the stored DocValue. One of the technique is called bit packing. Using bit-packing numeric values can be stored with less disk space. The key observation is despite some int are large, the range of serie of ints are very close to each other. For example:

```101, 102, 105``

If we subtract 100 from the value, it become

```1, 2, 5```

then those values can be stored with less on disk space.

![](bit-packing.jpg)

# Compression: dictionary encoding

Store repeated string values in doc values will not store the same value again and again. Recall there is TSUID in opentsdb, the idea is metric name is very repeative and long (many bytes), instead of store the metric name on each data point, we translate the metric name to a numeric value to save disk space. Elasticsearch DocValues will do the translation internally for us, there is no need to do the optimization trick in application layer.

For example, we have 3 documents

```
doc[0] = "aardvark"
doc[1] = "beaver"
doc[2] = "aardvark"
```

It will be stored as

```
doc[0] = 0
doc[1] = 1
doc[2] = 0

term[0] = "aardvark"
term[1] = "beaver"
```

# Nested Documents

# Distributed Computation

Most of the goodness is built-in the underlying lucene storage engine. But the distributed computation part is authentic part of Elasticsearch. Having elasticsearch built the distributed part for us, so that we do not need to shard our database, we do not need to send queries to "related" nodes and collect the results back for further aggregation. All of these things have been taken care of.

![](es-sharding.png)

The sharding is not perfect. Master is not using zookeeper, makes it prone to split-brain problem. From crate.io blog we can know the aggregation can be improved or we can use crate.io instead (which builds on top of Elasticsearch): https://crate.io/blog/crate_data_elasticsearch/

```
Elasticsearch currently supports the HyperLogLog aggregations, whereas Crate.IO supports accurate aggregations. Also Elasticsearch scatters the queries to all nodes, gathering the responses and doing the aggregations afterwards which results in high memory consumption on the node that is handling the client request (and so doing the aggregation).

Crate distributes the collected results to the whole cluster using a simple modulo based hashing, and as a result uses the complete memory of the cluster for merging. Think of it as some kind of distributed map/reduce.
```

But nevertheless, Elasticsearch has a distributed computation layer that works.

# Map/reduce story

Elasticsearch has very good story on Map/reduce as well. There are two ways to do that

* Use Elasticsearch distributed computation as a map/reduce engine, you plug script into the calculation process
* Use Elasticsearch as a data store with good filtering support, build the distributed computation layer using Spark

This is how Elasticsearch can be used as RDD in spark:

```
// Set our query
jobConf.set("es.query", query)
// Create an RDD of the tweets
val currentTweets = sc.hadoopRDD(jobConf,
 classOf[EsInputFormat[Object, MapWritable]],
 classOf[Object], classOf[MapWritable])
// Convert to a format we can work with
val tweets = currentTweets.map{ case (key, value) =>
 SharedIndex.mapWritableToInput(value) }
// Extract the hashtags
val hashTags = tweets.flatMap{t =>
 t.getOrElse("hashTags", "").split(" ")
}
```

The filter is push down to be executed by Elasticsearch, the aggregation part is done by spark.

# Future: Off-heap 

Elasticsearch is using a lot of Java Heap to cache. One important future direction to move a lot of cache off the heap. Here is video on this effort forked from Solr (Another lucene based search engine):

* video: https://www.youtube.com/watch?v=dDywZDQJQ3o&index=43&list=PLU6n9Voqu_1FM8nmVwiWWDRtsEjlPqhgP
* slides: http://www.slideshare.net/lucidworks/native-code-off-heap-data-structures-for-solr-yonik-seeley

# Future: SIMD

Elsaticsearch is limited by the JVM, so that leveraging SIMD cpu instructions to aggregate data faster is hard to implement. There are smart guys working on bring SIMD acceleration into Elasticsearch, and have very promising results:

* video: https://berlinbuzzwords.de/session/fast-decompression-lucene-codec
* post: http://blog.griddynamics.com/2015/02/proposing-simd-codec-for-lucene.html

# Success Stories

There are two company using Elasticsearch as time series database from my knowledge. There must be many more companies using it, but I was not aware of.

* Bloomberg: http://www.slideshare.net/lucidworks/search-analytics-component-presented-by-steven-bower-bloomberg-lp
* Parse.ly: https://www.elastic.co/blog/pythonic-analytics-with-elasticsearch

Because there are many more people using Elasticsearch as a analytics database instead of search engine, the company behind Elasticsearch is now called "Elastic" without search.


