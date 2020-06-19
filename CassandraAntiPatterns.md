
# Cassandra anti patterns:

No amount of performance tuning can mitigate a known anti-pattern. When you google 'antinpatterns in Cassandra' you will find lots of information. Datastax has done great a job listing many of them but that is not all. My aim is to jot down all antipatterns listed at one place. Which is very crucial to know for every Cassandra users.

1. Read before write: Two major draw back of read before write pattern is  a. Performance impact b. You will not be able to achieve atomic compare and set in case multiple client application are accessing same record in parallel.
To get rid of the later issue, users ends up using LWT (light weight transaction) in Cassandra. But this worsen the performance in terms of latency.
The best way to change your data model which avoid this pattern. [This](https://easyprograming.com/posts/2017/08/cassandra-deep-dive-read-before-write-evil/) is a very good post which has some good examples of data model change.

2. Collections are meant for storing/denormalizing relatively small amount of data. It is a anti-pattern to use a (single) collection to store large amounts of data. Although maximum  size of map and list is 2 GB, users should try to keep much lesser value than that in MBs.

3. Wide partition is root cause of multiple issues in Cassandra. A thumb rule says that don't go beyond 100 MB, however a good data model design should keep it much lesser. Tips: Bucketization can help here. For example storing sensor events for IoT application, rather than keeping primary key as (sensor_id, insert_time)  which will cause a wide partition per sensor, probably changing it  to ((sensor_id,date) insert_time) makes more sense. Event though [CASSANDRA-11206](https://issues.apache.org/jira/browse/CASSANDRA-11206) (version 3.5+) moved the barrier of wide partition to an extent but it is still recommended not to have too wide partition.

4. Storing big payload as a column with datatype text or blob is not wise. Recommended practical size is less than 1 MB but try keeping in Kbs. 
Tips: Put big files in bucket store like S3 and keep the metadata/reference link in Cassandra table or break the the single big blob into smaller chunk ans store in separate rows if Cassandra is the only option you have.

5. Select all  or select count without partition key will cause full table scan and should not be run on big dataset.

6. Materialized view is an experimental feature and should be avoided in production.

7. Use Cassandra secondary index very carefully. SI on high or low carnality field is not a wise decision. 

8. Queue type data structure (delete once consumed) : The queue anti-pattern serves as a reminder that any design that relies on the deletion of data is potentially a poorly performing design. Cassandra has provided tombstone_warn_threshold configuration parameter  to warn if you have designed your data-model like so.

9.  "ALLOW FILTERING" basically tells that it will require scanning a bunch of data to return some subset of it, equivalent to full table scan in sql database. One of the funniest definition i read is ALLOW FILTERING really means "ALLOW ME TO DESTROY MY APPLICATION AND CLUSTER." It means the data model does not support the query and will not scale. Having said that you can still use this if you really know what you are doing, for example using ALLOW FILTERING for a query by specifying partition key.

10.  Multi-partition batch : If we are using batch query and it is of big size it will cause latency at coordinator level. People coming from RDBMS should keep this thing in mind that multi-parition batch is not going to give better performance. Multi partition batch is kind of anti pattern in Cassandra so avoid that.  Batch (Logged) in Cassandra should be used to keep the write atomic in multiple de-normalized tables. Below are some configuration parameters related to batch.

    batch_size_warn_threshold_in_kb 
     (Default: 5KB per batch) Causes Cassandra to log a WARN message when any batch size exceeds this value in kilobytes.
     CAUTION:
     Increasing this threshold can lead to node instability.
     batch_size_fail_threshold_in_kb 
     (Default: 50KB per batch) Cassandra fails any batch whose size exceeds this setting. The default value is 10X the value of batch_size_warn_threshold_in_kb

11.  Running without the recommended [settings](https://docs.datastax.com/en/dse/6.7/dse-admin/datastax_enterprise/config/configRecommendedSettings.html) in production definetly "No".


12. Too less max heap size:  Cassandra in production on less than 8GB of heap looks too less in my experience. So 8 GB is the absolute minimum unless you are rarely reading. Stay with CMS if you have 8GB MAX_HEAP_SIZE. For G1 start with 16 GB or at-least 12 GB. 


13. Too much max heap size: It is not recommended going beyond 32GB even with G1. It will give diminishing results. The optimal value of MAX_HEAP_SIZE will depend on multiple factors like access pattern, data model, data per node etc so try tuning it and see which value works for you best. 

13. Multi-get by using IN clause is not recommended as it puts pressure on a single node because usually many nodes must be queried, rather concurrent multiple individual select is preferable.

14. List vs Set : Some operations on lists do perform read-before-write at local storage. Further, some lists operations are not idempotent by nature ,making their retry in case of timeout problematic. It is thus advised to prefer sets over lists when possible.

15. Using SAN or NAS becomes bottleneck in terms of IO and network route, so avoid using that rather use local disc SSD or spinning disk.

17. Keeping commitlog and ssd in same disk if using spinning disk due to conflicting io patterns. In case of SSD or EC2 it is fine.

16. Putting a load balancer in front of C* is completely unnecessary and only adds another point of failure. Rather use driver which has different types of loadbalacing policies.

18. If you’re running C* on AWS, EBS Volumes are problematic. It is not predictable and throughput is limited in many cases. Use ephemeral device instead by striping them.

19. The Row Cache : Row Cache has a much more limited use case. Row cache pulls entire partitions into memory. If any part of that partition has been modified, the entire cache for that row is invalidated. For large partitions this means the cache can be frequently caching and invalidating big pieces of memory. Because you really need mostly static partitions for this to be useful, for most use cases it is recommended that you do not use Row Cache.

21. Binding null values : Binding null values to prepared statement parameters will generate tombstones (java driver example: boundStatements.add(prepStmt.bind(1, null)) ), but leaving unset bound parameters in Cassandra 2.2+ combined with the DataStax Java Driver 3.0.0+ will not create tombstone. Hence don't add bind the value if it is null.

22. Intensive update on same column : This will cause performance impact during read as multiple sstable scan is required. To avoid this, use leveled compaction if your I/O can keep up. else change your data model see the below example:

```sql
Bad design: 
    CREATE TABLE sensor_data ( id long, value double, PRIMARY KEY (id)); 
    UPDATE sensor_data set value = ? WHERE id = ? --This is frequent operation on same id 
    SELECT value FROM sensor_data where id = ? --Here you will get read latency (worst case: timeout)

 Better Design: 
    CREATE TABLE sensor_data ( id long, date timestamp, value double, PRIMARY KEY (id, date)) WITH CLUSTERING ORDER(date DESC) ; 
    INSERT INTO sensor_data (id, date, value) VALUES (?, ?, ? ); 
    SELECT value FROM sensor_data where id = ? LIMIT 1; -- you will get the latest sensor value

```

23. Concurrent schema and topology change: An example of such design could be creating/dropping table every day (separate table for daily data). The issue comes up when you try to change the topology (for example adding/removing a node) and while your topology change is in progress your application tried to perform DDL as part of your daily routine. Concurrent schema change and topology change is an anti pattern. 

24. DDL or DCL operation in mixed version cluster : Please avoid them while upgrade is in progress..it should be done as either pre-step or post-step. 

25. Any activity which involves streaming like repair, scale-up or scale down  during update can put you in trouble so avoid that. Mixed version cluster only meant to support cross version read and write.

26. Don't try to keep two datacenter of same cluster in different version of Cassandra. This is only  valid scenario when you are in mid of upgrading your cluster.

27. Boot-strapping multiple nodes concurrently : Concurrent bootstrap may cause inconsistent token range movement between nodes.  To safely bootstrap each node try sequential bootstrap . While adding a separate dc nodes can be added by setting auto_bootstrap=false and following 2 minute pause rule between two consecutive startup in the new dc.

28. Running full repair (nodetool repair without any parameter) on all the nodes to avoid redundant repair of replicas: Try using sub-range repair preferably using tools like reaper or run nodetool repair -pr on each node sequentially. Keep in mind that Cassandra 4.0 has made great improvement in repair and fixed incremental repair  and will be worth trying.

29. Too many tables : Substantial performance degradation due to high memory usage and compaction issues can be caused by having too many tables in a cluster. Try to keep the count less than 200. 

30. Using the Byte Ordered Partitioner : The Byte Ordered Partitioner (BOP) is not recommended. Use virtual nodes (vnodes) instead.

31. CPU frequency scaling : As mentioned by Datastax, Recent Linux systems include a feature called CPU frequency scaling or CPU speed scaling. It allows a server's clock speed to be dynamically adjusted so that the server can run at lower clock speeds when the demand or load is low. This reduces the server's power consumption and heat output (which significantly impacts cooling costs). Unfortunately, this behavior has a detrimental effect on servers running DataStax products because throughput can get capped at a lower rate.

32. Lack of familiarity with Linux : Operator must be aware of some of the basics troublshooting linux commands to point out the issue. For example: top, dstat, iostat, mpstat, iftop, sar, lsof, netstat, htop, vmstat etc. A separate blog is required to cover all of them.

33. Insufficient testing : Be sure to test at scale and production loads. This the best way to ensure your system will function properly when your application goes live. To properly test, see Datastax [tips](https://docs.datastax.com/en/dse-planning/doc/planning/planningTesting.html) for testing your cluster before production. 

34. Preparing the same query more than once is generally an anti-pattern and will likely affect performance. Cassandra generate WARN message "Re-preparing already prepared query" for same as well.

35. Run Cassandra with RF=1. Why would you do that ? Are you not using Cassandra to avoid single point of failure?

36. Using SimpleStrategy in production. Simple replication strategy can be used as testing cluster. NetworkTopologyStrategy should be preferred over SimpleStrategy to make it easier to add new physical or virtual datacenters to the cluster later

37. If Using GossipingPropertyFileSnitch and  Cassandra-topology.properties is still present in the server may cause some unwanted behavior for example: different nodes appear to be down in the nodetool status output or nodetool status shows all down nodes under wrong dc name when restarting a node while others are down. The GossipingPropertyFileSnitch always loads Cassandra-topology.properties when that file is present. Remove the file from each node on any new cluster or any cluster migrated from the PropertyFileSnitch.

38. Don't mix normal write and LWT write on same records to avoid inconsistency during concurrent execution. Also LWT write with normal read (consistency QUORUM) on same record in parallel may show stale data so to avoid this inconsistency use read with consistency SERIAL/LOCAL_SERIAL. 
 	
Reference: 
- https://docs.datastax.com/en/dse-planning/doc/planning/planningAntiPatterns.html
- https://strange-loop-2012-notes.readthedocs.io/en/latest/tuesday/Cassandra.html
- https://www.slideshare.net/doanduyhai/Cassandra-nice-use-cases-and-worst-anti-patterns

