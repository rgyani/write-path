Lets compare the read-write path of various Big Data Technologies

### HDFS

* To write a file in HDFS, a client needs to interact with master i.e. namenode (master).  
* Now namenode provides the address of the datanodes (slaves) on which client will start writing the data. 
* Client directly writes data on the datanodes, now the datanode will create a data write pipeline.
  * The first datanode while receiving data, will copy the block to another datanode, which in-turn will  copy it to the third datanode.
  * Once it creates the replicas of blocks, it sends back the acknowledgment.
* If a node fails during this process (or delays acknowledgement), the block will be under-replicated which will be taken care of later by the namenode.


* To read a file from HDFS, a client needs to interact with namenode (master) as namenode is the centerpiece of Hadoop cluster (it stores all the metadata i.e. data about the data).
* Now namenode checks for required privileges, if the client has sufficient privileges then namenode provides the address of the slaves where a file is stored.
*  Now the client will interact directly with the respective datanodes to read the data blocks.

![](hadoop.png)

### HBASE

* There is a special HBase Catalog table called the META table, which holds the location of the regions in the cluster.
* ZooKeeper stores the location of the META table.
* When the first time a client reads or writes to HBASE, the client gets the Region server that hosts the META table from ZooKeeper.
* The client will query the .META. server to get the region server corresponding to the row key it wants to access (read or write)
* The client caches this information along with the META table location. It will send the CRUD request to the corresponding Region Server.
* After the request is received by the right region server, 
  * the change cannot be written to a HFile immediately because the data in a HFile must be sorted by the row key. This allows searching for random rows efficiently when reading the data.
  * each change is stored in a place in memory called the memstore, which cheaply and efficiently supports random writes. Data in the memstore is sorted in the same manner as data in a HFile.
  *  When the memstore accumulates enough data, the entire sorted set is written to a new HFile in HDFS

![](hbase.png)

![](hbase2.png)

### ZOOKEEPER
* Once a ZooKeeper ensemble starts, it will wait for the clients to connect.
* Clients will connect to one of the nodes in the ZooKeeper ensemble.
  * It may be a leader or a follower node.
  * Once a client is connected, the node assigns a session ID to the particular client and sends an acknowledgement to the client.
  * If the client does not get an acknowledgement, it simply tries to connect another node in the ZooKeeper ensemble.
  * Once connected to a node, the client will send heartbeats to the node in a regular interval to make sure that the connection is not lost.
* If a client wants to read a particular znode, it sends a read request to the node with the znode path and the node returns the requested znode by getting it from its own database.
  * For this reason, reads are fast in the ZooKeeper ensemble.
* If a client wants to store data in the ZooKeeper ensemble, it sends the znode path and the data to the server.
  * The connected server will forward the request to the leader and then the leader will reissue the writing request to all the followers.
  * If only a majority of the nodes respond successfully, then the write request will succeed, and a successful return code will be sent to the client. Otherwise, the write request will fail. The strict majority of nodes is called Quorum.

![](zookeeper.png)

In order to achieve high read-availability, Zookeeper guarantees a weak-consistency over the replicates: a read can always be answered by a client node, and the answer returned may be a stale value (even a new version has been committed through the leader).  
Then it is the users' responsibility to decide whether the answer for a read is "stale-able" or not, since not all applications require the up-to-date information.   
So the following choices are provided:
1) If your application does not need up-to-date values for reads, you can get high read-availability by requesting data directly from the client.
2) If your application requires up-to-date values for reads, you should use the "sync" API before your read request to sync the client-side version with the leader.  
So as a conclusion, Zookeeper provides a customizable consistency guarantee, and users can decide the balance between availability and consistency.


It is worth mentioning that the zookeeper also stores a version-id with the znode data. If a client decides to overwrite the data, it can specify the version-id that it has seen, in the write request. The leader will reject the write request, if the version of the data that the client has seen, is an older version than what the leader has. This helps if the client misses a write between the sync request and write request.


### KAFKA
* Kafka Brokers handle read and writes. Each topic has one or more partitions, and each partition has a LEADER broker. 
* A Broker also forwards messages for replication.
* Each Broker also compacts its own logs replica (without influence to any other copies)
* Even when you have a replicated partition on a different broker, Kafka wouldn’t let you read or write from it because in each replicated set of partitions, there is a LEADER and the rest of them are just mere FOLLOWERS serving as backup. 
  * The followers keep on syncing the data from the leader partition periodically, waiting for their chance to shine.
  * When the leader goes down, one of the in-sync follower partitions is chosen as the new leader and now you can consume data from this partition.

![](kafka.png)

### CASSANDRA
* In Cassandra, the cluster of nodes is stored as a “ring” of nodes and writes are replicated to N nodes using a replication placement strategy.
* When a client sends a write request to a single, random Cassandra node. 
  * The node who receives the request acts as a coordinator and writes the data to the cluster. 
  * It is the job of the Coordinator node to write the data to N nodes for redundancy and it will write the first copy to the primary node for that data, the second copy to the next node in the ring in another data centre, and the rest of the copies to machines in the same data centre as the proxy.
* When you write to a table in Cassandra (inserting data, for example), you can specify the write consistency level.
  *  The write consistency level is the number of replica nodes that have to acknowledge the coordinator that its local insert was successful (success here means that the data was appended to the commit log and written to the memtable). 
  * As soon as the coordinator gets WRITE_CONSISTENCY_LEVEL success acknowledgements from the replica nodes, it returns success back to the client and doesn't wait for the remaining replicas to acknowledge success.
  
* When a write request reaches a node, mainly two things happen:
  * The write request is appended to the commit log in the disk. This ensures data durability (the write request data would permanently survive even in a node failure scenario)
  * The write request is sent to the memtable (a structure stored in the memory).
  * When the memtable is full, the data is flushed to a SSTable on disk using sequential I/O and the data in the commit log is purged.