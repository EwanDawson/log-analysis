# Batch processing httpd logs using logstash and elasticsearch

The following describes how I set up elasticsearch and logstash to index 60 million apache httpd access logs. The parameters I chose are based on  research, and I haven't experimented with other values. So no doubt this configuration can be improved on. However, I did find I was able to index 15 million records per hour, whereas when using the default configuration, indexing ground to a halt after 8 hours with only 20 million  records indexed.

System: Intel Core i7 8-core 2.8GHz CPU and 16GB of RAM, running CentOS 5.9, writing data to an ext3 filesystem.

I've included references at the end, and attached my elasticsearch and logstash configs.

* Get the latest versions of logstash, elasticsearch and kibana from elasticsearch.org.  I used logstash-1.4.0, elasticsearch-1.1.0, and kibana-3.0.0.
* Make sure you're running on Linux and Java1.7.  I tried running on windows, and it would eventually crash,  even running inside an Ubuntu VMWare machine.
* Make sure you've got plenty of memory - I used a 16GB machine with 8GB allocated to elasticsearch, and 2GB allocated to logstash.
* Configure your OS to allow elasticsearch to reserve the entire 8GB heap in one allocation. This reduces memory fragmentation and boosts the performance of elasticsearch. Also increase the number of open file handles permitted. Replace <user> with the user under which elasticsearch will be running. You'll need to log out and back in for the changes to take effect.
* Edit /etc/security/limits.conf
  * <user> hard nofile 64000
  * <user>  soft nofile 64000
  * <user>  hard memlock unlimited
  * <user>  soft memlock unlimited
* Set the ES_HEAP_SIZE environment variable determine the amount of RAM allocated to elasticsearch.  I set it to 8g (8GB).
* Edit conf/elasticsearch.yml
  * index.number_of_shards: 1 (since we'll only be using a single development machine.  More than one shard only makes sense when your cluster of ES nodes is bigger than the number of replicas you want. For example, in a 3-node HA config with 2 replicas, you still only want 1 shard.  After 3 nodes, adding extra shards (max one per node above 3) will increase search performance, but decrease indexing performance).
  * index.number_of_replicas: 0 (the max value for this is always number_of_nodes - 1, so should be set to 0 for a single node setup.  Add nodes and increase this number for resiliance. For example, a good setup for high-availability is 3 nodes with number_of_replicas set to 2. This allows one node to be taken offline for rolling upgrade or maintenance, and still leave 2 nodes running.
  * path.*: best set to a location outside the elasticsearch installation folder, on a fast local disk for max IO performance.
  * bootstrap.mlockall: true (this allows the JVM to lock a block of memory for the entire head size at startup, improving speed by reducing memory fragmentation. This only works on Linux, and the host OS needs to be configured to allow this.  If the OS isn't correctly configures, you'll see an error message in the logs on startup. Note, this setting will result in a long pause on startup while the memory block is reserved).
  * indices.memory.index_buffer_size: 80% (this allocates almost all memory towards indexing, at the expense of searching. Remember to set it back to <50% (default is 10%) after the bulk index is finished, or you won't be able to search your data!
  * index.translog.flush_threshold_ops: 50000 (Reduces the frequency of transaction log flushes to increase performance. Might want to reset back to the default (5000) after bulk indexing is complete.
  * threadpool.search.type: fixed
  * threadpool.search.size: 20
  * threadpool.search.queue_size: 100
  * threadpool.index.type: fixed
  * threadpool.index.size: 60
  * threadpool.index.queue_size: 200 (overrides the default thread allocation to favour indexing. You may want to increase or decrease the numbers depending on the number of cores on your machine.  The machine I used has 8 cores. Once batch insert is complete, swap the numbers over).
* Now you want to configure logstash to send the logs to elasticsearch for indexing. For full details of httpd access log parsing, see the attached logstash config. The only parameter important for our purposes is the elasticsearch output flus_size parameter. The default value is 5000, but I increased this to 50000 since the individual log messages are rather small.
* Set the environment variable LS_HEAP_SIZE=2g (default is 500MB) to reduce pauses due to GC.
* Logstash defaults to running the event filter processes on a single thread. The filtering is the bottleneck in our setup, so when running logstash, use the -w parameter to set the number of threads used for filtering. I used 10.

References:
* http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/indices-update-settings.html
* http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/setup-configuration.html#_environment_variables
* http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-cluster.html
* http://elasticsearch-users.115913.n3.nabble.com/write-throughput-test-on-elasticsearch-9-high-configuration-nodes-cluster-td4033887.html
* http://jablonskis.org/2013/elasticsearch-and-logstash-tuning/index.html
* http://blog.trifork.com/2014/01/07/elasticsearch-how-many-shards/
* http://elasticsearch-users.115913.n3.nabble.com/Unknown-mlockall-error-0-td4007300.html
* http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/modules-threadpool.html
