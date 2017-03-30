---
layout: post
title: "Importing JSON into Hadoop via Kafka"
date: 2017-03-31 11:08
comments: true
categories: kafka hadoop wikimedia json camus
---

_This was originally published on [Wikimedia's blog](https://blog.wikimedia.org/2017/01/13/json-hadoop-kafka/)_

## JSON is…not binary

JSON is awesome.  It is both machine and human readable.  It is concise (at least compared to XML), and is even more concise when represented as YAML. It is well supported in many programming languages.  JSON is text, and works with standard CLI tools.

JSON sucks.  It is verbose.  Every value has a key in every single record.  It is schema-less and fragile. If a JSON producer changes a field name, all downstream consumer code has to be ready.  It is slow.  Languages have to convert JSON strings to binary representations and back too often.
JSON is ubiquitous.  Because it is so easy for developers to work with, it is one of the most common data serialization formats used on the web [citation needed!].  Almost any web based organization out there likely has to work with JSON in some capacity.

[Kafka](https://kafka.apache.org/) was [originally developed by LinkedIn](https://engineering.linkedin.com/kafka/first-apache-release-kafka-out), and is now an open source Apache project with strong support from [Confluent](http://confluent.io/).   Both of these organizations prefer to work with strongly typed and schema-ed data.  Their serialization format of choice is [Avro](https://avro.apache.org/docs/current/).  Organizations like this have tight control over their data formats, as it rarely escapes outside of their internal networks.  There are very good [reasons](https://www.confluent.io/blog/stream-data-platform-2/) Confluent is pushing Avro instead of JSON, but for many, like Wikimedia, it is impractical to transport data in a binary format that is unparseable without extra information (schemas) or special tools.

The Wikimedia Foundation lives openly on the web and has a commitment to work with volunteer open source contributors.  Mediawiki is used by people of varying technical skill levels in different operating environments.  Forcing volunteers and Wikimedia engineering teams to work with serialization formats other than JSON is just mean!  Wikimedia wants our software and data to be easy.

For better or worse, we are stuck with JSON.  This makes many things easy, but big data processing in Hadoop is not one of them.  Hadoop runs in the JVM, and it works more smoothly if its data is schema-ed and strongly typed.  Hive tables are schema-ed and strongly typed.  They can be mapped onto JSON HDFS files [using a JSON SerDe](http://ottomata.org/tech/too-many-hive-json-serdes/), but if the underlying data changes because someone renames a field, certain queries on that Hive table will break.  Wikimedia imports the latest JSON data from Kafka into HDFS every 10 minutes, and then does a batch transform and load process on each fully imported hour.

## Camus, Gobblin, Connect

LinkedIn created Camus to import Avro data from Kafka into HDFS.   JSON support was added by Wikimedia.  Camus’ shining feature is the ability to write data into HDFS directory hierarchies based on configurable time bucketing.  You specify the granularity of the bucket and which field in your data should be used as the event timestamp.
However, both LinkedIn and Confluent have dropped support for Camus.  It is an end-of-life piece of software.  Posited as replacements, LinkedIn has developed Gobblin, and Kafka ships with Kafka Connect.

[Gobblin](http://gobblin.readthedocs.io/en/latest/) is a generic HDFS import tool.  It should be used if you want to import data from a variety of sources into HDFS.  It does not support timestamp bucketed JSON data out of the box.  You’ll have to provide your own implementation to do this.

[Kafka Connect](http://docs.confluent.io/3.0.0/connect/) is generic Kafka import and export tool, and has a [HDFS Connector](https://github.com/confluentinc/kafka-connect-hdfs) that helps get data into HDFS.  It has limited JSON support, and requires that your JSON data conform to a Kafka Connect [specific envelope](https://github.com/apache/kafka/blob/trunk/connect/json/src/main/java/org/apache/kafka/connect/json/JsonSchema.java#L69-L82).  If you don’t want to reformat your JSON data to fit this envelope, you’ll have difficulty using Kafka Connect.

That leaves us with [Camus](https://github.com/confluentinc/camus).  For years, Wikimedia has successfully been using Camus to import JSON data from Kafka into HDFS.  Unlike the newer solutions, Camus does not do streaming imports, so it must be scheduled in batches. We’d like to catch up with more current solutions and use something like Kafka Connect, but until JSON is better supported we will continue to use Camus.

So, how is it done?  This [question](https://mail-archives.apache.org/mod_mbox/kafka-users/201511.mbox/%3CCAMkQo2QZ7YMa8LoGPq95dm0CbS3omtwVS59AhHjKGWzsg==h-g@mail.gmail.com%3E) [appears](https://groups.google.com/forum/#!searchin/confluent-platform/json%7Csort:relevance/confluent-platform/_5Iign8D30w/2pM89jXLHQAJ) [often](https://groups.google.com/forum/#!searchin/confluent-platform/json%7Csort:relevance/confluent-platform/Fz31DiKFUME/VArBOKGaAAAJ) [enough](https://groups.google.com/forum/#!searchin/confluent-platform/json%7Csort:relevance/confluent-platform/GuIc68M7GKA/Kg12czxYDgAJ) [on](https://groups.google.com/forum/#!searchin/confluent-platform/json%7Csort:relevance/confluent-platform/Q_BKR9VOHcE/USjbX-bRDAAJ) [Kafka](https://groups.google.com/forum/#!searchin/confluent-platform/json%7Csort:relevance/confluent-platform/iCzY2uf7gsA/gGvFYYN-DwAJ) related mailing lists, that we decided to write this blog post.

## Camus with JSON

Camus needs to be told how to read messages from Kafka, and in what format they should be written to HDFS.  JSON should be serialized and produced to Kafka as UTF-8 byte strings, one JSON object per Kafka message.  We want this data to be written as is with no transformation directly to HDFS.  We’d also like to compress this data in HDFS, and still have it be useable by MapReduce.  Hadoop’s [SequenceFile](https://wiki.apache.org/hadoop/SequenceFile) format will do nicely.  (If we didn’t care about compression, we could use the [StringRecordWriterProvider](https://github.com/confluentinc/camus/blob/confluent-master/camus-etl-kafka/src/main/java/com/linkedin/camus/etl/kafka/common/StringRecordWriterProvider.java) to write the JSON records \n delimited directly to HDFS text files.)

We’ll now create a camus.properties file that does what we need.

First, we need to tell Camus where to write our data, and where to keep execution metadata about this Camus job.  Camus uses HDFS to store Kafka offsets so that it can keep track of topic partition offsets from which to start during each run:

```
# Final top-level HDFS data output directory. A sub-directory
# will be dynamically created for each consumed topic.
etl.destination.path=hdfs:///path/to/output/directory
# HDFS location where you want to keep execution files,
# i.e. offsets, error logs, and count files.
etl.execution.base.path=hdfs:///path/to/camus/metadata
# Where completed Camus job output directories are kept,
# usually a sub-dir in the etl.execution.base.path
etl.execution.history.path=hdfs:///path/to/camus/metadata/history
```

Next, we’ll specify how Camus should read in messages from Kafka, and how it should look for event timestamps in each message.  We’ll use the [JsonStringMessageDecoder](https://github.com/confluentinc/camus/blob/confluent-master/camus-kafka-coders/src/main/java/com/linkedin/camus/etl/kafka/coders/JsonStringMessageDecoder.java), which expects each message to be  UTF-8 byte JSON string.  It will deserialize each message using the Gson JSON parser, and look for a configured timestamp field.

```
# Use the JsonStringMessageDecoder to deserialize JSON messages from Kafka.
camus.message.decoder.class=com.linkedin.camus.etl.kafka.coders.JsonStringMessageDecoder
```

`camus.message.timestamp.field` specifies which field in the JSON object should be used as the event timestamp, and `camus.message.timestamp.format` specifies the timestamp format of that field.  Timestamp interpolation is handled by Java’s [SimpleDateFormat](https://docs.oracle.com/javase/7/docs/api/java/text/SimpleDateFormat.html), so you should set `camus.message.timestamp.format` to something that SimpleDateFormat understands, unless your timestamp is already an integer UNIX epoch timestamp.  If it is, you should use `unix_seconds` or `unix_milliseconds`, depending on the granularity of your UNIX epoch timestamp.

[Wikimedia maintains a slight fork](https://github.com/wikimedia/analytics-camus/blob/wmf/camus-kafka-coders/src/main/java/com/linkedin/camus/etl/kafka/coders/JsonStringMessageDecoder.java) of JSONStringMessageDecoder that makes the `camus.message.timestamp.field` slightly more flexible.  In our fork, you can specify sub-objects using dotted notation, e.g. `camus.message.timestamp.field=sub.object.timestamp`. If you don’t need this feature, then don’t bother with our fork.

Here are a couple of examples:

#### Timestamp field is ‘dt’, format is an [ISO-8601](https://en.wikipedia.org/wiki/ISO_8601) string:

```
# Specify which field in the JSON object will contain our event timestamp.
camus.message.timestamp.field=dt
# Timestamp values look like 2017-01-01T15:40:17
camus.message.timestamp.format=yyyy-MM-dd'T'HH:mm:ss
```

#### Timestamp field is ‘meta.sub.object.ts’, format is a UNIX epoch timestamp integer in milliseconds:

```
# Specify which field in the JSON object will contain our event timestamp.
# E.g. { “meta”: { “sub”: { “object”: { “ts”: 1482871710123 } } } }
# Note that this will only work with Wikimedia’s fork of Camus.
camus.message.timestamp.field=meta.sub.object.ts
# Timestamp values are in milliseconds since UNIX epoch.
camus.message.timestamp.format=unix_milliseconds
```

If the timestamp cannot be read out of the JSON object, JsonStringMessageDecoder will log a warning and fall back to using System.currentTimeMillis().

Now that we’ve told Camus how to read from Kafka, we need to tell it how to write to HDFS. `etl.output.file.time.partition.mins` is important. It tells Camus the time bucketing granularity to use.  Setting this to 60 minutes will cause Camus to write files into hourly bucket directories, e.g. 2017/01/01/15. Setting it to 1440 minutes will write daily buckets, etc.

```
# Store output into hourly buckets.
etl.output.file.time.partition.mins=60
Use UTC as the default timezone.
etl.default.timezone=UTC
# Delimit records by newline.  This is important for MapReduce to be able to split JSON records.
etl.output.record.delimiter=\n
```

Use [SequenceFileRecordWriterProvider](https://github.com/confluentinc/camus/blob/confluent-master/camus-etl-kafka/src/main/java/com/linkedin/camus/etl/kafka/common/SequenceFileRecordWriterProvider.java) if you want to compress data.  To do so, set `mapreduce.output.fileoutputformat.compress.codec=Snappy` (or another splittable compression codec) either in your mapred-site.xml, or in this camus.properties file.

```
# SequenceFileRecordWriterProvider writes the records as Hadoop Sequence files
# so that they can be split even if they are compressed.
etl.record.writer.provider.class=com.linkedin.camus.etl.kafka.common.SequenceFileRecordWriterProvider
# Use Snappy to compress output records.
mapreduce.output.fileoutputformat.compress.codec=SnappyCodec
```

Finally, some basic Camus configs are needed:

```
# Replace this with your list of Kafka brokers from which to bootstrap.
kafka.brokers=kafka1001:9092,kafka1002:9092,kafka1003:9092
# These are the kafka topics camus brings to HDFS.
# Replace this with the topics you want to pull,
# or alternatively use kafka.blacklist.topics.
kafka.whitelist.topics=topicA,topicB,topicC
# If whitelist has values, only whitelisted topic are pulled.
kafka.blacklist.topics=
```

There are various other camus properties you can tweak as well.  You can see some of the ones Wikimedia uses [here](https://github.com/wikimedia/operations-puppet/blob/production/modules/camus/templates/webrequest.erb).

Once this camus.properties file is configured, we can launch a Camus Hadoop job to import from Kafka.

```
hadoop jar camus-etl-kafka-X.jar com.linkedin.camus.etl.kafka.CamusJob -P /path/to/camus.properties -Dcamus.job.name="my-camus-job"
```

_Note: replace X in the above command with your Camus version number, e.g. camus-etl-kafka-3.1.1.jar_

The first time this job runs, it will import as much data from Kafka as it can, and write its finishing topic-partition offsets to HDFS.  The next time you launch a Camus job with this with the same camus.properties file, it will read offsets from the configured `etl.execution.base.path` HDFS directory and start consuming from Kafka at those offsets.  Wikimedia schedules regular Camus Jobs using boring ol’ cron, but you could use whatever new fangled job scheduler you like.

After several Camus runs, you should see time bucketed directories containing Snappy compressed SequenceFiles of JSON data in HDFS stored in `etl.destination.path`, e.g. `hdfs:///path/to/output/directory/topicA/2017/01/01/15/`.  You could access this data with custom MapReduce or Spark jobs, or use Hive’s org.apache.hive.hcatalog.data.JsonSerDe and Hadoop’s org.apache.hadoop.mapred.SequenceFileInputFormat.  Wikimedia creates [an external Hive table doing just that](https://github.com/wikimedia/analytics-refinery/blob/master/hive/webrequest/create_webrequest_raw_table.hql), and then [batch processes](https://github.com/wikimedia/analytics-refinery/blob/master/oozie/webrequest/load/refine_webrequest.hql) this data into a more [refined and useful schema](https://github.com/wikimedia/analytics-refinery/blob/master/hive/webrequest/create_webrequest_table.hql) stored as Parquet for faster querying.

Here’s the camus.properties file in full:
```
#
# Camus properties file for consuming Kafka topics into HDFS.
#
# Final top-level HDFS data output directory. A sub-directory
# will be dynamically created for each consumed topic.
etl.destination.path=hdfs:///path/to/output/directory
# HDFS location where you want to keep execution files,
# i.e. offsets, error logs, and count files.
etl.execution.base.path=hdfs:///path/to/camus/metadata
# Where completed Camus job output directories are kept,
# usually a sub-dir in the etl.execution.base.path
etl.execution.history.path=hdfs:///path/to/camus/metadata/history
# Use the JsonStringMessageDecoder to deserialize JSON messages from Kafka.
camus.message.decoder.class=com.linkedin.camus.etl.kafka.coders.JsonStringMessageDecoder
# Specify which field in the JSON object will contain our event timestamp.
camus.message.timestamp.field=dt
# Timestamp values look like 2017-01-01T15:40:17
camus.message.timestamp.format=yyyy-MM-dd'T'HH:mm:ss
# Store output into hourly buckets.
etl.output.file.time.partition.mins=60
# Use UTC as the default timezone.
etl.default.timezone=UTC
# Delimit records by newline.  This is important for MapReduce to be able to split JSON records.
etl.output.record.delimiter=\n
# Concrete implementation of the Decoder class to use
camus.message.decoder.class=com.linkedin.camus.etl.kafka.coders.JsonStringMessageDecoder
# SequenceFileRecordWriterProvider writes the records as Hadoop Sequence files
# so that they can be split even if they are compressed.
etl.record.writer.provider.class=com.linkedin.camus.etl.kafka.common.SequenceFileRecordWriterProvider
# Use Snappy to compress output records.
mapreduce.output.fileoutputformat.compress.codec=SnappyCodec
# Max hadoop tasks to use, each task can pull multiple topic partitions.
mapred.map.tasks=24
# Connection parameters.
# Replace this with your list of Kafka brokers from which to bootstrap.
kafka.brokers=kafka1001:9092,kafka1002:9092,kafka1003:9092
# These are the kafka topics camus brings to HDFS.
# Replace this with the topics you want to pull,
# or alternatively use kafka.blacklist.topics.
kafka.whitelist.topics=topicA,topicB,topicC
# If whitelist has values, only whitelisted topic are pulled.
kafka.blacklist.topics=
# max historical time that will be pulled from each partition based on event timestamp
#  Note:  max.pull.hrs doesn't quite seem to be respected here.
#  This will take some more sleuthing to figure out why, but in our case
#  here it’s ok, as we hope to never be this far behind in Kafka messages to
#  consume.
kafka.max.pull.hrs=168
# events with a timestamp older than this will be discarded.
kafka.max.historical.days=7
# Max minutes for each mapper to pull messages (-1 means no limit)
# Let each mapper run for no more than 9 minutes.
# Camus creates hourly directories, and we don't want a single
# long running mapper keep other Camus jobs from being launched.
# We run Camus every 10 minutes, so limiting it to 9 should keep
# runs fresh.
kafka.max.pull.minutes.per.task=9
# Name of the client as seen by kafka
kafka.client.name=camus-00
# Fetch Request Parameters
#kafka.fetch.buffer.size=
#kafka.fetch.request.correlationid=
#kafka.fetch.request.max.wait=
#kafka.fetch.request.min.bytes=
kafka.client.buffer.size=20971520
kafka.client.so.timeout=60000
# Controls the submitting of counts to Kafka
# Default value set to true
post.tracking.counts.to.kafka=false
# Stops the mapper from getting inundated with Decoder exceptions for the same topic
# Default value is set to 10
max.decoder.exceptions.to.print=5
log4j.configuration=false
##########################
# Everything below this point can be ignored for the time being,
# will provide more documentation down the road. (LinkedIn/Camus never did! :/ )
##########################
etl.run.tracking.post=false
#kafka.monitor.tier=
kafka.monitor.time.granularity=10
etl.hourly=hourly
etl.daily=daily
etl.ignore.schema.errors=false
etl.keep.count.files=false
#etl.counts.path=
etl.execution.history.max.of.quota=.8
```
