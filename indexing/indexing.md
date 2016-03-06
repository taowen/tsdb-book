
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


## Log file => Local log harvester



