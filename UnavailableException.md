**Site information:** Cassandra version: 2.1.16. Cluster has  two dc (primary and secondry) and each has 3 nodes, each dc has replication factor as 3. 


**What we did:** Trying to emulate one site crash scenario.we cleanup all data (deleted data directory from each node) on primary site and restarted  nodes. 

**Issue:**  Not able to connect to cqlsh on primary dc. Getting error 

> code=0100 [Bad credentials]
> message="org.apache.cassandra.exceptions.UnavailableException: Cannot
> achieve consistency level LOCAL_ONE

    /usr/share/cassandra/bin/cqlsh -u <uname> -p <pw> --ssl --cqlshrc=/etc/cassandra/conf/cqlshrc <host>  <port>
    Connection error: ('Unable to connect to any servers', {'host1_priv': AuthenticationFailed(u'Failed to authenticate to host1_priv: code=0100 [Bad credentials] message="org.apache.cassandra.exceptions.UnavailableException: Cannot achieve consistency level LOCAL_ONE"',)})


**Trouble shooting steps:**

**Step-1** is to assure that the credential are correct as the error was "Bad credential"

Observation:  The credential were correct . 

**Step-2**  
When you see  UnavailableException first thing comes into your mind is that 
The  request reaches the coordinator and there is not enough replica alive to achieve the requested consistency level so check the status of the cluster as  consistency could not be achieved 
Node tool status :

Observation :  All node were up and running on both side. 

**Step-3** As all nodes are up so now check the actual replication factor of system_auth keyspace 

    vs@cqlsh> DESCRIBE keyspace system_auth;
    CREATE KEYSPACE system_auth WITH replication = {'class': 'NetworkTopologyStrategy', 'DC2': '3'}  AND durable_writes = true;

Observation :  We observed  that in replication of system_auth keyspace were set to only to  one DC.   

**Fix:** So we first altered the replication factor of system_auth

**Step -1**

    [root@vmc0257 vs]# Alter KEYSPACE system_auth WITH REPLICATION ={'class' : 'NetworkTopologyStrategy','DC1':3, 'DC2':3};
    Warning: schema version mismatch detected, which might be caused by DOWN nodes; if this is not the case, check the schema versions of your nodes in system.local and system.peers.
    OperationTimedOut: errors={}, last_host=host1_priv

The above warning message is confusing and make you doubtful whether the alter was successful or not.

**Step -2** just assure the above alter operation issue is successful.

    vs@cqlsh> DESCRIBE keyspace system_auth;
    CREATE KEYSPACE system_auth WITH replication = {'class': 'NetworkTopologyStrategy', 'DC1': '3', 'DC2': '3'}  AND durable_writes = true;

**Note:  Ignore the above warning!** 

**Step -3** Now run repair across for  all the nodes sequentially so that data is consistent across all nodes in cluster.
nodetool repair system_auth

And your problem is fixed !! :)


**Lesson learnt:**  
In this case replicas can not get automatically the copy of system_auth keyspace. This results in missing data in the replicas and caused "Bad credential" error. So, we needed to repair the keyspace on all nodes to sync with each other.
