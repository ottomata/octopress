---
layout: post
title: "Too many Hive JSON SerDes"
date: 2014-01-13 11:32
comments: true
categories: bigdata analytics hadoop hive
---

One of the big advantages of Hive is the ability to query almost any data format.  All one has to do is to provide a '[SerDe](https://cwiki.apache.org/confluence/display/Hive/SerDe)' class so that Hive knows how to serialize and deserialize data.  Hive ships with a few built in SerDes (Avro, ORC, RegEx, Thrift), but not JSON!  I  [found](https://code.google.com/p/hive-json-serde/) [a](https://github.com/rcongiu/Hive-JSON-Serde) [few](https://github.com/cloudera/cdh-twitter-example#setting-up-hive) third party JSON SerDes out there, but most were either incomplete or threw exceptions for simple use cases.  Hive's [SerDe documentation](https://cwiki.apache.org/confluence/display/Hive/SerDe) references [Amazon's JSON SerDe](s3://elasticmapreduce/samples/hive-ads/libs/jsonserde.jar), but I can't seem to find the source code, and I'd rather not use this in production if I don't know what it is doing.

I had mostly been using Cloudera's JSONSerde from their [cdh-twitter-example](https://github.com/cloudera/cdh-twitter-example#setting-up-hive).  My table defines a ```bigint``` sequence number field.  The data in this field is a contiguous increasing sequence number starting at 0.  This JSONSerde uses [Jackson](https://github.com/FasterXML/jackson-core) to parse the JSON into Java objects, and infers the types of the Java objects from the values in the JSON.  If a value looks like a an integer, it will be a Java Integer.  If a value looks like a Long, it will be a Java Long.  The same applies for Java Floats and Doubles as well.  Since sequence numbers start small but eventually get very large, this SerDe will end up throwing ```ClassCastExceptions``` for my sequence field.   Others have had [similar issues](https://groups.google.com/a/cloudera.org/d/msg/cdh-user/4iXuZyIb_d8/ZHx3K0kmW9IJ).

After reading the source of this and the other JSON SerDes, I realized that this could be fixed by casting the Jackson parsed value to whatever the Hive table expects.  I then noticed that [HCatalog](https://hive.apache.org/hcatalog/) has a [JSON SerDe](https://github.com/apache/hcatalog/blob/branch-0.5/core/src/main/java/org/apache/hcatalog/data/JsonSerDe.java) that does exactly this.  This code is specific to HCatalog, so it is possible that writing data using this SerDe may result in something I'm not expecting.  But, this is the most complete JSON SerDe code I've seen so far, and I am able to read my JSON data with no problems using it.  Also, an HCatalog jar ships with CDH4, which makes it easier to use in our production systems.

If you are using CDH4, you add the hcatalog-core to your Hive auxpath and create a Hive table that uses this SerDe like this:

```bash
$ cdh_version=4.3.1
$ hive --auxpath \
/usr/lib/hcatalog/share/hcatalog/hcatalog-core-0.5.0-cdh${cdh_version}.jar

hive (default)>
create table my_table(...)
ROW FORMAT SERDE
  'org.apache.hcatalog.data.JsonSerDe'
...
;
```

### JSON SerDes Overview

* [hive-json-serdes -  code.google.com](https://code.google.com/p/hive-json-serde/source/browse/trunk/src/org/apache/hadoop/hive/contrib/serde2/JsonSerde.java)
  * Hasn't been updated since 2011
  * Doesn't have serialization support
  * Uses built in Java JSON library
* [Hive-JSON-Serde - rcongiu](https://github.com/rcongiu/Hive-JSON-Serde/blob/develop/src/main/java/org/openx/data/jsonserde/JsonSerDe.java)
  * Primitive timestamp support
  * Doesn't cast other primitive values to the expected Hive field types (```ClassCastException```)
  * Uses built in Java JSON library
* [JSONSerde - cdh4-twitter-example](https://github.com/cloudera/cdh-twitter-example/blob/master/hive-serdes/src/main/java/com/cloudera/hive/serde/JSONSerDe.java)
  * Doesn't cast other primitive values to the expected Hive field types (```ClassCastException```)
  * Uses Jackson JSON library
* [jsonserde - Amazon](s3://elasticmapreduce/samples/hive-ads/libs/jsonserde.jar)
  * No source code
* [JsonSerDe - HCatalog](https://github.com/apache/hcatalog/blob/branch-0.5/core/src/main/java/org/apache/hcatalog/data/JsonSerDe.java)
  * Casts primitive values to expected Hive field types
  * Uses Jackson JSON library

## See also:
* [How Do You Make a Hive Table out of Json Data](http://stackoverflow.com/questions/11479247/how-do-you-make-a-hive-table-out-of-json-data)
* [Interesting post on how to parse JSON during query time](http://pkghosh.wordpress.com/2012/05/06/hive-plays-well-with-json/)



