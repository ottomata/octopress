---
layout: post
title: "Puppet and CDH4"
date: 2012-10-02 16:38
comments: true
categories: bigdata analytics hadoop
---
I just pushed the first bits of a complete Puppet module for Cloudera's CDH4 distribution for Hadoop.  Currently it only supports YARN (not MapReduce v1), and assumes that your NameNode also runs your ResourceManager.

[puppet-cdh4](https://github.com/wikimedia/puppet-cdh4)

There is [another](https://github.com/robbkidd/cloudera-cdh4-puppet) CDH4 puppet module out there, but it uses MapReduce v1, and also adds the HDFS data directories as [hardcoded facter variables](https://github.com/robbkidd/cloudera-cdh4-puppet/blob/master/modules/hadoop/lib/facter/facts.rb).  [puppet-cdh4](https://github.com/wikimedia/puppet-cdh4) will allows you to specify Hadoop directories using a config class.  As long as your JBOD mounts have been partitioned and are mounted, then you should be able to configure a DataNode like this:

```puppet
include cdh4

class { "cdh4::hadoop::config":
    namenode_hostname => "namenode.hostname.org",
    mounts            => [
        "/var/lib/hadoop/data/a",
        "/var/lib/hadoop/data/b",
        "/var/lib/hadoop/data/c"
    ],
    dfs_name_dir      => ["/var/lib/hadoop/name", "/mnt/hadoop_name"],
}

# Installs and starts the DataNode and NodeManager services.
include cdh4::hadoop::worker
```


This is a work in progress, so [leave me a comment on github](https://github.com/wikimedia/puppet-cdh4/issues/new) if you try to use it and have troubles.
