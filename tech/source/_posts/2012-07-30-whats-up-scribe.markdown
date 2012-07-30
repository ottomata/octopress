---
layout: post
title: "What&rsquo;s up Scribe?"
date: 2012-07-30 11:59
comments: true
categories: logging bigdata analytics
---


I first used [Scribe](https://github.com/facebook/scribe) at [CouchSurfing](http://www.couchsurfing.org) back in 2009.  We used it for application logs and eventually for collecting webserver access logs as well.  It worked for years with zero hiccups.  CouchSurfing's scale is nothing near Facebook's, but we weren't no dinky little website either.  Scribe just worked.  However, we had to compile it and build our own RPMs.  Fine fine, Scribe was only recently open sourced at the time, so that's fair.  

Now it is 2012, and I'm on the the [Wikimedia Foundation's](http://wikimediafoundation.org/wiki/Home) new [Analytics](https://blog.wikimedia.org/2012/07/25/meet-the-analytics-team/) team.  We've been considering using Scribe as a way to get data into the [analytics cluster](http://www.mediawiki.org/wiki/Analytics/Kraken) we're building.  By now, you'd think it would be as easy as `apt-get install scribe`.  But, nopers, you've still got to build it all yourself.  This means choosing the correct [Thrift](http://thrift.apache.org/) and fb303 versions.  I also needed to build Java bindings for Thrift and Scribe, which meant modifying Makefiles.  On top of this, I can't just `make install!` (Does anyone actually do this in production?)  I needed to build .deb packages.  

Welp, it's done!  [simplegeo](https://github.com/simplegeo) had already done the hard work writing the `debian/` directory.  I forked each of their [thrift](https://github.com/wmf-analytics/thrift), [fb303](https://github.com/wmf-analytics/thrift-fb303) and [scribe](https://github.com/wmf-analytics/scribe) repositories, modified the `debian/` configuration and any required Makefile changes.  I wanted this process to be reproducible for folks in the future, so I scripted up the process over at the [scribe-debian](https://github.com/wmf-analytics/scribe-debian) repository.  The .debs I built for Ubuntu Lucid amd64 are available there as well.  

The forks I used build Scribe 2.2 against Thrift 0.2.0.  Thrift is currently at 0.8.0.  I struggled for a while building Scribe against newer Thrift versions, but was never quite able to make it work.  Someone else has [figured this out](http://ycavatars.blogspot.com/2012/05/build-scribe-on-ubuntu-1204.html), and I wish I had seen this before I had settled on Thrift 0.2.0.  I need to go back and try again.  But, before I (or you) do, consider the following.

As far as I can tell, Facebook is no longer supporting or maintaining Scribe.  From [this Quora article](http://www.quora.com/Why-did-Facebook-develop-Puma-pTail-instead-of-using-existing-ones-like-Flume):
> We actually don't use "scribe".

Instead, Facebook has written a new series of tools for streaming log processing on top of HDFS.  Watch Sam Rash's presentation from the 2011 Hadoop Summit on that Quora article.  In summary, they have 'rewritten' Scribe in Java as Calligraphus.  Calligraphus is a pub/sub distributed logging service / message queue, with a persistence layer provided by HDFS.  They then use ptail to stream logs out of HDFS.  You can think of the whole piece as a distributed Scribe buffered network source.  Instead of Scribe writing buffered or file logs to disk on a single node, Calligraphus writes logs to HDFS, and a consumer (e.g. ptail) reads them off at will.  Unfortunately, Calligraphus has not yet been open sourced, although they say they plan to make is so soon.

Anyway, the fact that Facebook seems to be phasing out Scribe doesn't bode well for Scribe's future.  [fluentd](http://fluentd.org/) might be a contender for those who don't mind (or who want) a distributed logger written in Ruby, and for those where extremely high throughput isn't a big concern.  If you are going for big big data, you might want to check out [Flume](https://cwiki.apache.org/FLUME/) or [Kafka](http://incubator.apache.org/kafka/).

See also:

* [Prebuilt .debs for Debian Squeeze](http://sandrotosi.blogspot.gr/2010/10/scribe-thrift-for-debian-squeeze.html)
* [Building Scribe for Ubuntu](http://ycavatars.blogspot.com/2012/05/build-scribe-on-ubuntu-1204.html)
* [Scribe fork](https://github.com/traviscrawford/scribe) with LZO and ZooKeeper node discovery
* [log4j scribe appender .deb packaging](https://github.com/wmf-analytics/log4j-scribe-appender)
* [Thrift debian upstream packaging effort](http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=648451)
