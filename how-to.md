Knowing all those great features provided by Elasticsearch. Now, we need to build a time series database using Elasticsearch. There are two thoughts:

* Design a Elasticsearch mapping that maximizes the aggregation performance and minimize the storage size.
* Pre-compute the common aggregation, so that for most of query the aggregation is already done

Mapping
-------------

This is what the final mapping looks like:

```
{  
   "login":{  
      "_source":{  
         "enabled":false
      },
      "_all":{  
         "enabled":false
      },
      "properties":{  
         "login_records":{  
            "properties":{  
               "login_os":{  
                  "index":"not_analyzed",
                  "fielddata":{  
                     "format":"doc_values"
                  },
                  "doc_values":true,
                  "type":"string"
               },
               "login_plat":{  
                  "index":"not_analyzed",
                  "fielddata":{  
                     "format":"doc_values"
                  },
                  "doc_values":true,
                  "type":"string"
               },
               "login_timestamp":{  
                  "fielddata":{  
                     "format":"doc_values"
                  },
                  "format":"dateOptionalTime",
                  "doc_values":true,
                  "type":"date"
               },
               "login_cc_set":{  
                  "index":"not_analyzed",
                  "fielddata":{  
                     "format":"doc_values"
                  },
                  "doc_values":true,
                  "type":"string"
               },
               "login_count":{  
                  "index":"no",
                  "fielddata":{  
                     "format":"doc_values"
                  },
                  "doc_values":true,
                  "type":"integer"
               },
               "login_biz_id":{  
                  "index":"not_analyzed",
                  "fielddata":{  
                     "format":"doc_values"
                  },
                  "doc_values":true,
                  "type":"string"
               }
            },
            "type":"nested"
         },
         "login_min_timestamp":{  
            "fielddata":{  
               "format":"disabled"
            },
            "format":"dateOptionalTime",
            "doc_values":true,
            "type":"date"
         },
         "login_max_timestamp":{  
            "fielddata":{  
               "format":"disabled"
            },
            "format":"dateOptionalTime",
            "doc_values":true,
            "type":"date"
         }
      }
   }
}
```

 The fields are:

```
timestamp, biz_id, plat, os, cc_set, count
```

timestamp, biz_id, plat, os, cc_set are the dimensional fields

count is the measurement field or value field.

Here is a explanation on why the default mapping looks like this:

* _source is disabled: avoid duplicated storage, save disk space
* _all is disabled: no query need _all
* index not analyzed: we do not need full text search. not analyzed means the filter need to be speeded up by index
* fielddata docvalues: the column data is stored on disk in docvalues format. the default format is in memory cache, which is consumes too much memory. docvalues can be preceived as something like parquet columnar file format.
* login_records as nested documents: one elasticsearch doc contains several nested document. Each nested document is one record (a.k.a data point). nested document is physically stored together to compact the storage size.
min/max timestamp: used to locate the root level document without un-nest the nested document
* login_xxx prefix: all fields is prefixed by the type name (login). In Elasticsearch, mapping within same index share same lucece segment file. If the field of different mapping has same name but different type or other properties, error will happen.

Clustered Fields
-----------------

One optimization can be used on the default mapping is to "pull up" some of the fields from nested document to parent document. For example we can pull up the biz_id field, so that records for the same biz_id of some time range will be combined into one parent document, but records of different biz_id will always be separated.

```
{  
   "login":{  
      "_source":{  
         "enabled":false
      },
      "_all":{  
         "enabled":false
      },
      "properties":{  
         "login_records":{  
            "properties":{  
               "login_os":{  
                  "index":"not_analyzed",
                  "fielddata":{  
                     "format":"doc_values"
                  },
                  "doc_values":true,
                  "type":"string"
               },
               "login_plat":{  
                  "index":"not_analyzed",
                  "fielddata":{  
                     "format":"doc_values"
                  },
                  "doc_values":true,
                  "type":"string"
               },
               "login_timestamp":{  
                  "fielddata":{  
                     "format":"doc_values"
                  },
                  "format":"dateOptionalTime",
                  "doc_values":true,
                  "type":"date"
               },
               "login_cc_set":{  
                  "index":"not_analyzed",
                  "fielddata":{  
                     "format":"doc_values"
                  },
                  "doc_values":true,
                  "type":"string"
               },
               "login_count":{  
                  "index":"no",
                  "fielddata":{  
                     "format":"doc_values"
                  },
                  "doc_values":true,
                  "type":"integer"
               }
            },
            "type":"nested"
         },
        "login_biz_id":{  
           "index":"not_analyzed",
           "fielddata":{  
              "format":"doc_values"
           },
           "doc_values":true,
           "type":"string"
        }
         "login_min_timestamp":{  
            "fielddata":{  
               "format":"disabled"
            },
            "format":"dateOptionalTime",
            "doc_values":true,
            "type":"date"
         },
         "login_max_timestamp":{  
            "fielddata":{  
               "format":"disabled"
            },
            "format":"dateOptionalTime",
            "doc_values":true,
            "type":"date"
         }
      }
   }
}
```

Pre-computation
-------------------------

We create one index for one day data. The indices will looks like:

* logs-2015-07-05
* logs-2015-07-06
* logs-2015-07-07

We can pre-compute the data to a separate indices:

* logs-precomputed-2015-07-05
* logs-precomputed-2015-07-06
* logs-precomputed-2015-07-07

In above mapping, there are four dimensions:

```
biz_id, cc_set, plat, os
```

In the pre-computed data, we can omit the cc_set, plat, os dimensions, so that query the total login for one biz_id can be faster.

```
{  
      "_source":{  
         "enabled":false
      },
      "properties":{  
         "login_precomputed_biz_id_biz_id":{  
            "index":"not_analyzed",
            "fielddata":{  
               "format":"doc_values"
            },
            "doc_values":true,
            "type":"string"
         },
         "login_precomputed_biz_id_precomputed_max_of_count":{  
            "index":"no",
            "fielddata":{  
               "format":"doc_values"
            },
            "doc_values":true,
            "type":"integer"
         },
         "login_precomputed_biz_id_precomputed_sum_of_count":{  
            "index":"no",
            "fielddata":{  
               "format":"doc_values"
            },
            "doc_values":true,
            "type":"integer"
         },
         "login_precomputed_biz_id_precomputed_count":{  
            "index":"no",
            "fielddata":{  
               "format":"doc_values"
            },
            "doc_values":true,
            "type":"integer"
         },
         "login_precomputed_biz_id_precomputed_min_of_count":{  
            "index":"no",
            "fielddata":{  
               "format":"doc_values"
            },
            "doc_values":true,
            "type":"integer"
         },
         "login_precomputed_biz_id_timestamp":{  
            "fielddata":{  
               "format":"doc_values"
            },
            "format":"dateOptionalTime",
            "doc_values":true,
            "type":"date"
         }
      },
      "_all":{  
         "enabled":false
      }
   }
   ```

Indexing & Querying
--------------------

Using this format, indexing and querying will not be straight forward. It is certainly unlike

```
INSERT INTO login(biz_id, cc_set, plat, os, cunt) VALUES(...)
```

There should be two service wrap Elasticsearch to provide more usable index/query api.


