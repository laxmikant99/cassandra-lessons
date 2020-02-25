
Cassandra is know for its fast write due it its simple write path. However if you are facing write latency, try to observe the symptoms and identify the bottleneck. Below are some reasons and troubleshooting steps:

1. Bad disk: SAN or NAS is considered as anti-pattern in Cassandra so we should avoid that. Local SSD is preferred for fast read and write.

2. Large batch write : If we are using batch query and it is of big size it will cause latency at coordinator level.Multi partition batch is kind of anti pattern in Cassandra so avoid that. 
Batch in cassandra should be used to keep the write atomic in multiple de-normalized tables. Below are some configuration parameters related to batch.

 

       batch_size_warn_threshold_in_kb 
        (Default: 5KB per batch) Causes Cassandra to log a WARN message when any batch size exceeds this value in kilobytes.
        CAUTION:
        Increasing this threshold can lead to node instability.
        batch_size_fail_threshold_in_kb 
        (Default: 50KB per batch) Cassandra fails any batch whose size exceeds this setting. The default value is 10X the value of batch_size_warn_threshold_in_kb

3. Large row write : If you writing very big blobs or big files into a single row, it is an issue. A row size should not cross more than 1 MB. 
Fix: Break it into smaller chunks and store them in parallel.

4. Network latency: you may face write latency If your facing network latency specially cross-dc latency . Your consistency level can also cause cross-dc traffic if you are using consistency QUORAM. If possible chose LOCAL_QUORAM in multi dc cluster. 
nodetool tablehistograms shows local write latency however proxyhistograms gives total write latency including response time from replicas, hence it is a network issue if your proxyhistograms write latency is unexpectedly higher than tablehistograms write latency. 

5. Huge writes/second : If you cluster is not large enough to handle your write throughput then the nodes will become overloaded and it will impact write performance. Overloaded nodes has symptoms like memory/cpu/io/threads crunch and eventually results latency. In this case  

6. You may be running out of threads, check nodetool tpstats can indicate that with pending/blocked MutationStage or Native-Transport-Requests . Try to keep concurrent_writes to no higher than 128 as increasing further does not really help in my experience. You may try to tune the pool by increasing max_queued_native_transtport_requests and native_transport_max_threads value if you see blocked Native-Transport-Requests. However if that does not help then actual problem may lie somewhere else and can be fixed adding a node, tuning hardware and configuration, and/or updating data models.  

7. On the client side: Three tips to remember at client side: 
	- Use Prepared statements 
	- Prefer ExecuteAsync, it can really speed things up. 
	- Don't write serially rather add some concurrency into the process.  

8. Using LWT : LWT uses paxos consensus model and it is an expensive operation. Hence at least 3 times more response time is expected with such operation. It is advisable to avoid or use them less frequently if required.

9. Counter : Updating counter is more expensive than normal write. you may increase counter cache in case you are doing counter updates.

10. Noisy neighbor suspicion : Is this a one-time or occasional load or more frequently? apart form Cassandra's own repair and compaction, Is there any other process (e.g.: scheduled script, manually triggering full table scan) apart from your application which is eating up the resources.

11. High 95% latency: One interesting reason for higher write latency at the 95%+ could be large volumes of data for particular rows.

12. If your compaction is taking too long or your repair/anti-compaction is eating up all IO resources this can cause temporary latency when these background process is running. To avoid Anticompaction during anti-entropy repair, you can opt for sub-range repair rather than full. you may reduce compaction threads/throughput to control resource consumption during compaction.

13. CPU crunch: At your system level you may have few cores or for any other compute process is eating up cpu, then you may have bottleneck at CPU end. 

14. Garbage collection is the reason for all types latency either read or write. Find the actual reason behind gc pauses.

15.  Check if write latency is on Single table or multiple tables. If it is for a single table then it may be due to data model,  access pattern and configuration of that particular table rather that system level issue.  
 
