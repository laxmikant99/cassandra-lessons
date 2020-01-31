## High disk usage in Cassandra:

Below are some of the troubleshooting steps for high disk usage in Cassandra cluster.

Find the root cause:
Improper dimension, more data written than expected, higher value of ttl or others.

### Options:

#### A. Add capacity - 
1. Add a disk, 
2. Use JBOD adding a second location folder for the sstables and move some of them around then restart Cassandra.
data_file_directories:
    - /var/lib/cassandra/data
    - /var/lib/cassandra/data-extra
Note: The JBOD feature can be useful in emergencies where disk space is urgently needed, however its usage in this case should be temporary.
3. Add a new node.

#### B. Reduce disk space used. 

#### Don'ts:
1. directly deleting sstables
2. running repair : it does not evict tombstone
3. executing delete operation.

#### Do's 

1. snapshots : Why it is an issue ? what is Hard link? 
**fix:**  'nodetool clearsnapshot' , what if this option is not present?

3. Other file present in Cassandra disk
**fix:** Remove any heap dump or other file that might be stored there

4. Forgot to run 'nodetool cleanup' after scale-up. 

5. Check 'nodetool compactionstats'. Current overhead of compaction by listing temporary sstables : *tmp*Data.db. 
**fix:** disable compaction. delete them. run compaction on one by one. 

6. Anti-compaction during repair. Why this is an issue ? 

    logs: 9e09c490-f1be-11e7-b2ea-b3085f85ccae   Anticompaction after repair     cargts   eventdata    147.3 GB   158.54 GB   bytes     92.91%

147GB*[compression ratio]. So with a compression ratio of 0.3 for example, that would be 44GB that will get reclaimed shortly after the anticompaction is over.

**fix:** use sub range repair

6. Some over-streaming that occurred during your repair. Check 'nodetool netstats'.

**fix :** throttle steaming. 'nodetool setstreamthroughput' .  stream_throughput_outbound_megabits_per_sec : 200 mbps

7. Tombstones: when the tombstones gets evicted:
 7.1 if tombstone creation time crossed GCGP
 7.2 if no older data with same partition is present in some other file.(overlapping sstables)

8. Ponder on Compaction stratgey : STCS vs LSCS. 

9. TWCS: Directly deleting data from cassandra can be cause data loss but it can be done carefuly if using twcs if sstable creation time corssed ttl. Else can use unsafe_aggressive_sstable_expiration = true which will not check for overlapping sstable files before removing.

10. Note: in STCS bigger sstable will not participate in compaction unless it comes under 

**fix:** Setting 'uncheck_tombstone_compaction: true'  often helps.


If not available use sstablemetadata and user defined compaction.

Using sstable metadata:

    for f in *.db; do meta=$(/home/user1/tools/bin/sstablemetadata $f); echo -e "Max:" $(date --date=@$(echo "$meta" | grep Maximum\ time | cut -d" "  -f3| cut -c 1-10) '+%Y/%m/%d %H:%M:%S') "Min: " $(date --date=@$(echo "$meta" | grep Minimum\ time | cut -d" "  -f3| cut -c 1-10) '+%Y/%m/%d %H:%M:%S') $(echo "$meta" | grep droppable) $(ls -lh $f | awk '{print $5" "$6" "$7" "$8" "$9}'); done | sort

    Max: 2017/12/07 10:06:46 Min:  2017/01/10 11:40:11 Estimated droppable tombstones: 0.524078481591479 4.4K Jan 21 07:07 ks-table-jb-2730206-Statistics.db
    Max: 2018/01/16 17:02:09 Min:  2017/01/10 09:52:06 Estimated droppable tombstones: 0.4882548629950296 4.5K Jan 21 07:07 ks-table-jb-2730319-Statistics.db
    Max: 2018/02/01 14:13:56 Min:  2017/01/10 08:15:39 Estimated droppable tombstones: 0.0784809244867484 5.9K Jan 21 07:07 ks-table-jb-2730216-Statistics.db
    Max: 2018/04/03 06:21:42 Min:  2017/01/10 08:13:04 Estimated droppable tombstones: 0.17692511128881072 5.9K Jan 21 07:07 ks-table-jb-2730344-Statistics.db

**User defined compaction:**

    SSTABLE_LIST="/Users/tlp/.ccm/2-2-9/node1/data0/ks/table-b260e6a0a7cd11e9a56a372dfba9b857/lb-46464-big-Data.db,\
    /Users/tlp/.ccm/2-2-9/node1/data0/ks/table-b260e6a0a7cd11e9a56a372dfba9b857/lb-46464-big-Data.db"
    
    JMX_CMD="run -b org.apache.cassandra.db:type=CompactionManager forceUserDefinedCompaction ${SSTABLE_LIST}"
    echo ${JMX_CMD} | java -jar jmxterm-1.0-alpha-4-uber.jar -l localhost:7100
    #calling operation forceUserDefinedCompaction of mbean org.apache.cassandra.db:type=CompactionManager
    #operation returns:
    null

OR nodetool  garbagecollect can also be effective if available in cassandra version. Note: This is io-intensive operation.

11. Check if you cluster is imbalance: 'nodetool status' load.
fix: make sure no manual error like num_token of each node are not equal or init_token not properly set. Not chosen you partition key properly.

12. Precaution when your disk space is >90% full.
Your compaction may be failing due to not enough space
**fix:** immediately get some space by moving 'unused' table to some other place or truncate them to give some room for compaction of other table.

13. Probably a bug in you Cassandra version:
For example: obsolete compacted files are not being deleted.
 Logs on Cassandra 2.0.14 after rolling restart while repair was also running:

        INFO [CompactionExecutor:1957] 2020-01-20 06:44:56,721 CompactionTask.java (line 120) Compacting [SSTableReader(path='/var/lib/cassandra/data/keyspace/columnfamily/keyspace-columnfamily-jb-123456-Data.db'), SSTableReader(path='/var/lib/cassandra/data/keyspace/columnfamily/keyspace-columnfamily-jb-234567-Data.db'), SSTableReader(path='/var/lib/cassandra/data/keyspace/columnfamily/keyspace-columnfamily-jb-345678-Data.db')]
        INFO [CompactionExecutor:1957] 2020-01-20 12:45:23,270 ColumnFamilyStore.java (line 795) Enqueuing flush of Memtable-compactions_in_progress@519967741(0/0 serialized/live bytes, 1 ops)
        INFO [CompactionExecutor:1957] 2020-01-20 12:45:23,502 CompactionTask.java (line 296) Compacted 3 sstables to [/var/lib/cassandra/data/keyspace/columnfamily/keyspace-columnfamily-jb-456789,].  136,795,757,524 bytes to 100,529,812,389 (~73% of original) in 21,626,781ms = 4.433055MB/s.  1,738,999,743 total partitions merged to 1,274,232,528.  Partition merge counts were {1:1049583261, 2:309997005, 3:23140824, }

**fix:** delete obsolete file and restart and upgrade to stable version soon.

**Reference:** Own experience and Cassandra Users mailing List. 
 Feel free to add more cases.
