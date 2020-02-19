
Below are probable cause of Cassandra read latency:

**1. Wide partition :**  Father of modern medicine Hippocrates once said that “All disease begins in the gut” , In the similar fashion I believe all major problems in Cassandra begins with Wide partition. Hence this can be a good reason for you slow read. nodetool tablehistogram "Partition Size" can give an idea, theoretically partition size should not be more than 100 mb. In practice it should be much lesser than 100 mb. 

**Fix:** Try bucketizing the partition key for the table to have thin partition.
e.g: partition key on organization name may be a bad choice however (organization, department) may be a fix.

**2. Bad query:** Below are some example of bad queries which can cause Coordinator level latency.

 - full table scan : e.g. select * from employee; select count(*) from
   employee;
 - using allow filtering.
 - using IN clause.

Fix: For full scan you may use spark . If that is not possible you can try 

    cqlsh -e "copy keyspace.table_name (first_partition_key_name) to '/dev/null'" | sed -n 5p | sed 's/ .*//'

For others try avoid running them as they are not scalable and will face as your data grows.
If you find any specific query is taking time then You can do 'TRACING ON'  in cqlsh, then run a problematic query or 'nodetool settraceprobability 0.001' - (here, be careful with implications of setting this value too high) and check the trace to see what is going on.

**3. Number of sstable touched:** If more number (eg: 95 Percentile touching more than 2 sstables ) of sstables are being touched during read then you can expect some scope of improvement in you cassandra cluster. You can check this using nodetool table/cfhistograms

**Fix:** Make sure your compaction is not lagging behind. Try to tune either the compaction strategy or the concurrent_compactors or compaction_throughput options.

**4. Blocking Read Repair:** Blocking Read Repair will trigger due to digest mismatch during read.  You can check the debug log digestmismatch exception (not some numbers of this exception is expected but it should not be in tons)  

**Fix:** Keep your cluster consistent and healthy by running anti-entropy repair in regular interval.  Reading data as soon as you wrote it may also cause this, check the possibility to give some delay.

**5. Speculative Retry:** Cassandra is designed to be fault tolerant so a failed node can be take care easily but a faulty node can difficult to handle .In other words, with Cassandra node going completely down is fine but if a node status is fluctuating (frequently being unresponsive) then it will cause read latency due to speculative retry as a node is not completely down but gets. 

**Fix:** Needs to identify the faulty node and trouble shoot it. Also check that your application is distributing the load evenly. Make sure you partition keys are such that it distributed evenly. Also try disabling dynamic_snitch in cassandra.yaml and test.

**6. Secondary indexes:** If not chosen wisely will cause latency at co-ordinator level. Well If you create SI on very high cardinality field (eg: email id or any unique id) or very low cardinality field (eg: sex:M/F) then you are in trouble. 
**7. Consistency level and Cross-DC traffic :** As you increase your consistency level your read operation will slow down. Using a QUOURM consistency in multi dc cluster can cause Cross-DC traffic and hence slow read.

**Fix:** If possible, try using LOCAL_QUORUM in case of multi dc cluster. If you don't want strong consistency you can probably use read consistency lesser than QUORUM.  Avoid using ALL consistency in production.

**8. Tombstones:** If you has lots of tombstones in you table and range read will be slow and you will see as warn if it cross tombstone_warn_threshold in new version of Cassandra.	

    WARN org.apache.cassandra.db.ReadCommand Read 0 live rows and 87051 tombstone cells for query SELECT * FROM example.table

You can also check sstable metadata to check tombstone ratio. 

**Fix:** If possible try to avoid creating tombstones at first place. You can also check my first [article](https://medium.com/analytics-vidhya/how-to-resolve-high-disk-usage-in-cassandra-870674b636cd) how to take care of tombstones by tuning.

**9.  Huge reads per seconds (rkB/s):** Check nodetool tpstats output to get an idea. Lots of pending/block readStage  can cause increased read latency.  

**Fix:** Generally resolved by adding nodes or reducing read throughput.   

**10. Insufficient cache:** Not having enough ram can slow down read as kernal has to read from disk than page cache. So it is good to have enough page cache. Cassandra also provide some cache like key cache, row cache which can be tuned only on very specific use-cases (e.g: reading some rows frequently)


**11. Hardware issue:** Local ssd is recommended for fast read in cassandra. 
**Fix:** avoid using NAS or SAN.

**12. Bloom filter:** Check nodetool tablestats,  high number of Bloom filter false positives counts can cause read latency. 
**Fix:** Tuning bloom_filter_fp_chance : Adjust based on ram available and no. of sstables, for slow read and lots of sstable, lower the fp change to decrease the disk io. Note that this will increase memory usage.

**13. Network latency and throughput:** You can check nodetool proxyhistogram and and at system level you can check latency using ping, traceroute or mtr and network throughput using iftop command

**14  Garbage collection:** Check gc logs and nodetool gcstas to see the gc pauses.  GC can be due to multiple reason like bad data model, bad gc tuning parameters etc. 

**15. Resource bound:** Check if your system is not cpu or io bound during read. you can use command like top/htop or iostat for that. Page cache can make the query fast so You can also check if enough page cache is available or not by using free command. 
A higher readahead value (> 128 kb) can cause high i/o and hence read latency.

**Fix:** Apache Cassandra has a good [article](http://cassandra.apache.org/doc/4.0/troubleshooting/use_tools.html) regarding same   Also check readahead value using blockdev - report command. and set it using blockdev- setra command.
 
**Conclusion:** Cassandra can be an great fit to your application only if you have suitable data model and access pattern. Cassandra tuning is complex and in this blog i have given high level idea of points to check while resolving read latency in Cassandra. 
Feel free to add more points in comments if you have faced in the comment.
