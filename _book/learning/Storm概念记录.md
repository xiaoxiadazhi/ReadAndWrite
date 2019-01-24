### Storm概念记录

#### nimbus.seeds

​	The worker nodes need to know which machines are the candidate of master in order to download topology jars and confs

​	worker节点需要知道哪些机器是master的候选者（做高可用H/A），为了下载topology jars和confs

* http://storm.apache.org/releases/2.0.0-SNAPSHOT/Setting-up-a-Storm-cluster.html





### worker配置

worker.childopts:
  -Xmx4g
  -Xms4g
  -XX:+UseParNewGC
  -XX:+UseConcMarkSweepGC
  -XX:+CMSParallelRemarkEnabled
  -XX:CMSInitiatingOccupancyFraction=75
  -XX:+UseCMSInitiatingOccupancyOnly
  -XX:CMSMaxAbortablePrecleanTime=500
  -XX:+UseCMSCompactAtFullCollection
  -XX:CMSFullGCsBeforeCompaction=4
  -verbose:gc
  -XX:+PrintGCTimeStamps
  -XX:+PrintGCDetails
  -XX:+PrintGCDateStamps
  -Xloggc:artifacts/gc.log
  -Dsun.net.inetaddr.ttl=300 -Dsun.net.inetaddr.negative.ttl=10

