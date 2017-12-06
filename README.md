# HBase-Online-Merger

We seen some overburn in RS which hosts too many regions on single region server. Utility helps us to merge the two adjacent region without impacting the exiting transactions (Online Merge), this tool more likely the HBase Normalization.

We have two option here,

**invokemerge**: *It starts the merger to do the merge tasks*.

**printmerge**: *It prints an overall summary of the table, is tell about the adjacent region, Hot region, and region count*.

### Design:

We maintain two separate Queue to trigger merge.

**1. WaitQueue**

We mark the region as "Hot" if major_compacting. and adding them to WaitQueue. It also includes the adjacent region of "Hot" region.
Merge will be postponed until compaction finishes.
This thread awake every 2 minutes and check the status for triggering merge.

**2. GeneralQueue**

All eligible regions are added to GeneralQueue and merge will be triggered immediately.

---

Merge will be Skipped for the following regions:

1. HotRegion

 2. (regionASize + regionBSize) < getMaxMergeSize()

Note: **MaxMergeSize is hardcoded to 70% of MAX_FILESIZE/hbase.hregion.max.filesize**.

---

We maintain the meta table in HBase, "hbase-online-merger". It has a boolean flag which indicates the following set. It does the Pre-Check.

1) Is already running?

2) Is Over-merged?

3) Was error in the last run?


Table Schema:- FAMILY:QUALIFIER

d:Status   => Was successful in the last run?

d:Running => Is already running?

d:Error => Was any error in the last run?

---


**USAGE**:

#sudo su - hbase

$Kinit

```
$java -cp hbase-merger-util.jar:`hbase classpath`: com.hwx.hbase.util.MRunner

           [--zookpeer-server <comma seperated list>]

           [--zookeper-znode <znode>]

           [--zookeeper-port <zk port>]

           --table-name <tablename|all>

           --cmd <invokemerge|printmerge>




 -c,--cmd <arg>                <invokemerge printmerge>

 -n,--zookeeper-znode <arg>    zookeeper znode, zookeeper.znode.parent

 -p,--zookeeper-port <arg>     zookeeper port,
                               hbase.zookeeper.property.clientPort

 -t,--table-name <arg>         table name,for all tables --table-name=all

 -z,--zookeeper-server <arg>   comma seperated zk server list,
                               hbase.zookeeper.quorum

```



**Example: printmerge**

```

To know the table summary:

$ java -cp hbase-merger-util.jar:`hbase classpath`: com.hwx.hbase.util.MRunner -t test -c printmerge

***********************************
         TABLENAME: test

MERGE. RegionA b63404d4cdd769d89e2c4c0e17b64a7c and RegionB db3b3a0e11040b69fbfaf23cd8d27f08
MERGE. RegionA 6f74297e3d3ba65dde3b52ccb5454896 and RegionB c2cc15430e3394bd30f44dfcb91c2f26
MERGE. RegionA da30e2f9f8d04fd51534027f695a7d4a and RegionB 3d1c35298abcd34a8d3873668b27de84
MERGE. RegionA 8ed0660945254a8712c54872d9d62354 and RegionB 31682d7ca4600055aa2703b67f99a153
MERGE. RegionA c7ff3cb7848c83f02525bc9a202aa7f3 and RegionB 5a534c8f0686e1108161d27485040706
MERGE. RegionA 284c953dc51ae6607f5a90d053c4cdd2 and RegionB f5eed4ac3a2ddcdca85c74e4189376c9

         OVERVIEW
         ***
         TABLENAME: test
         REGIONS: 12
         MERGE POSTPONED[HOT]: 0
         COUNT WILL BE AFTER MERGE:6


```


**Example: invokemerge**

```

To run the Merger:
$ java -cp hbase-merger-util.jar:`hbase classpath`: com.hwx.hbase.util.MRunner -t test -c invokemerge

..
..
2017-12-06 16:57:27,335 INFO com.hwx.hbase.util.MImpl: MERGING.. RegionA b63404d4cdd769d89e2c4c0e17b64a7c and RegionB db3b3a0e11040b69fbfaf23cd8d27f08
2017-12-06 16:57:27,393 INFO com.hwx.hbase.util.MImpl: MERGED. RegionA b63404d4cdd769d89e2c4c0e17b64a7c and RegionB db3b3a0e11040b69fbfaf23cd8d27f08
2017-12-06 16:58:07,448 INFO com.hwx.hbase.util.MImpl: MERGING.. RegionA 6f74297e3d3ba65dde3b52ccb5454896 and RegionB c2cc15430e3394bd30f44dfcb91c2f26
2017-12-06 16:58:07,503 INFO com.hwx.hbase.util.MImpl: MERGED. RegionA 6f74297e3d3ba65dde3b52ccb5454896 and RegionB c2cc15430e3394bd30f44dfcb91c2f26
2017-12-06 16:58:47,545 INFO com.hwx.hbase.util.MImpl: MERGING.. RegionA da30e2f9f8d04fd51534027f695a7d4a and RegionB 3d1c35298abcd34a8d3873668b27de84
2017-12-06 16:58:47,618 INFO com.hwx.hbase.util.MImpl: MERGED. RegionA da30e2f9f8d04fd51534027f695a7d4a and RegionB 3d1c35298abcd34a8d3873668b27de84
2017-12-06 16:59:27,658 INFO com.hwx.hbase.util.MImpl: MERGING.. RegionA 8ed0660945254a8712c54872d9d62354 and RegionB 31682d7ca4600055aa2703b67f99a153
2017-12-06 16:59:27,702 INFO com.hwx.hbase.util.MImpl: MERGED. RegionA 8ed0660945254a8712c54872d9d62354 and RegionB 31682d7ca4600055aa2703b67f99a153
2017-12-06 17:00:07,752 INFO com.hwx.hbase.util.MImpl: MERGING.. RegionA c7ff3cb7848c83f02525bc9a202aa7f3 and RegionB 5a534c8f0686e1108161d27485040706
2017-12-06 17:00:07,809 INFO com.hwx.hbase.util.MImpl: MERGED. RegionA c7ff3cb7848c83f02525bc9a202aa7f3 and RegionB 5a534c8f0686e1108161d27485040706
2017-12-06 17:00:47,840 INFO com.hwx.hbase.util.MImpl: MERGING.. RegionA 284c953dc51ae6607f5a90d053c4cdd2 and RegionB f5eed4ac3a2ddcdca85c74e4189376c9
2017-12-06 17:00:47,887 INFO com.hwx.hbase.util.MImpl: MERGED. RegionA 284c953dc51ae6607f5a90d053c4cdd2 and RegionB f5eed4ac3a2ddcdca85c74e4189376c9
2017-12-06 17:01:27,892 INFO com.hwx.hbase.util.MImpl: Merge completed for Table: test
..
..


Note: Log will be written to "/var/log/hbase/hbase-online-merger.log"

```



You may also get following exception upon the request.


1] ERROR: **com.hwx.hbase.exceptions.MergerException: You are forcing the region over-merge, Table:tableName. Exiting current Merger**.

To enforce the over-merge. *Be carful. Not advisible*.

hbase_shell> put 'hbase-online-merger','<table_name>','d:Status','false'



2] ERROR: **com.hwx.hbase.exceptions.LastRunException: There is some error in the last run. Please check, Table:tableName. Exiting current Merger**.

To know the last run error:

hbase_shell> get 'hbase-online-merger','<table_name>','d:Message'

To run the merger without fixing last failures:

hbase_shell> put 'hbase-online-merger','<table_name>','d:Error','false'



3] ERROR: **com.hwx.hbase.exceptions.AlreadyRunningException: Merger is running already, Table:tableName. Exiting current Merger**.

Make sure there is no merger running already.  You can update if you confirmed. *Be careful*.

hbase_shell> put 'hbase-online-merger','<table_name>','d:Running','false'

