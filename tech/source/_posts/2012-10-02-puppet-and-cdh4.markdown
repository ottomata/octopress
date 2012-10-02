---
layout: post
title: "Puppet and CDH4"
date: 2012-10-02 16:38
comments: true
categories: bigdata analytics hadoop
---
I just pushed the first bits of a complete Puppet module for Cloudera's CDH4 distribution for Hadoop.  Currently it only supports YARN (not MapReduce v1), and assumes that your NameNode also runs your ResourceManager.  

[cloudera-cdh4-puppet](https://github.com/wmf-analytics/cloudera-cdh4-puppet)

This is a work in progress, so [leave me a comment on github](https://github.com/wmf-analytics/cloudera-cdh4-puppet/issues/new) if you try to use it and have troubles.
