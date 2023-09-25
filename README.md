# Spark dynamic allocation on YARN cluster in Docker

Apache Spark is an open-source, distributed processing system used for big data workloads.

In this demo, a Spark container uses a Hadoop YARN cluster as a resource management and job scheduling technology to perform distributed data processing.

This Docker image contains Spark binaries prebuilt and uploaded in Docker Hub.

## Start Swarm cluster

1. start swarm mode in node1
```shell
$ docker swarm init --advertise-addr <IP node1>
$ docker swarm join-token worker  # issue a token to add a node as worker to swarm
```

2. add 3 more workers in swarm cluster (node2, node3, node4)
```shell
$ docker swarm join --token <token> <IP nodeN>:2377
```

3. label each node to anchor each container in swarm cluster
```shell
docker node update --label-add hostlabel=hdpmst node1
docker node update --label-add hostlabel=hdp1 node2
docker node update --label-add hostlabel=hdp2 node3
docker node update --label-add hostlabel=hdp3 node4
```

4. create an external "overlay" network in swarm to link the 2 stacks (hdp and spk)
```shell
docker network create --driver overlay mynet
```

5. start the Hadoop cluster (with HDFS and YARN) and spark client
```shell
$ docker stack deploy -c docker-compose.yml hdp
$ docker stack ps spk
ID             NAME            IMAGE                              NODE      DESIRED STATE   CURRENT STATE           ERROR     PORTS
vammehpu2iz4   spk_hdp1.1      mkenjis/ubhdpclu_vol_img:latest    node2     Running         Running 3 minutes ago             
xvasm5pexn3d   spk_hdp2.1      mkenjis/ubhdpclu_vol_img:latest    node3     Running         Running 3 minutes ago             
iqly7vcoz8c8   spk_hdp3.1      mkenjis/ubhdpclu_vol_img:latest    node4     Running         Running 3 minutes ago             
mdep15vxvrav   spk_hdpmst.1    mkenjis/ubhdpclu_vol_img:latest    node1     Running         Running 3 minutes ago             
vh1i5d3g4ipb   spk_spk_cli.1   mkenjis/ubspkcli_yarn_img:latest   node1     Running         Running 3 minutes ago
```

## Set up YARN with Spark Shuffle

1. access spark client node
```shell
$ docker container exec -it <spk_cli ID> bash
```

2. find jar file for spark_shuffle on YARN
```shell
$ cd $SPARK_HOME
$ find . -name '*yarn*jar'
./yarn/spark-2.3.2-yarn-shuffle.jar
./jars/hadoop-yarn-server-common-2.7.3.jar
./jars/hadoop-yarn-server-web-proxy-2.7.3.jar
./jars/hadoop-yarn-api-2.7.3.jar
./jars/hadoop-yarn-client-2.7.3.jar
./jars/hadoop-yarn-common-2.7.3.jar
./jars/spark-yarn_2.11-2.3.2.jar
```

3. copy the jar file to Hadoop YARN master and slaves
```shell
$ scp ./yarn/spark-2.3.2-yarn-shuffle.jar root@hdpmst:/usr/local/hadoop-2.7.3/share/hadoop/yarn
spark-2.3.2-yarn-shuffle.jar                                             100% 9476KB   2.3MB/s   00:04    
$ scp ./yarn/spark-2.3.2-yarn-shuffle.jar root@hdp1:/usr/local/hadoop-2.7.3/share/hadoop/yarn
Warning: Permanently added 'hdp1,10.0.1.5' (ECDSA) to the list of known hosts.
spark-2.3.2-yarn-shuffle.jar                                             100% 9476KB   2.0MB/s   00:04    
$ scp ./yarn/spark-2.3.2-yarn-shuffle.jar root@hdp2:/usr/local/hadoop-2.7.3/share/hadoop/yarn
Warning: Permanently added 'hdp2,10.0.1.8' (ECDSA) to the list of known hosts.
spark-2.3.2-yarn-shuffle.jar                                             100% 9476KB   6.1MB/s   00:01    
$ scp ./yarn/spark-2.3.2-yarn-shuffle.jar root@hdp3:/usr/local/hadoop-2.7.3/share/hadoop/yarn
Warning: Permanently added 'hdp3,10.0.1.11' (ECDSA) to the list of known hosts.
spark-2.3.2-yarn-shuffle.jar                                             100% 9476KB   1.9MB/s   00:04
```

4. stop YARN resource manager and node managers

In hdp1,hdp2,hdp3 run :
```shell
$ yarn-daemon.sh stop nodemanager
```

In hdpmst, run :
```shell
$ stop-yarn.sh
```

5. edit yarn-site.xml in hdpmst and copy to spkcli and yarn nodes :
```shell
$ cd $HADOOP_HOME/etc/hadoop
$ vi yarn-site.xml

  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle,spark_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services.spark_shuffle.class</name>
    <value>org.apache.spark.network.yarn.YarnShuffleService</value>
  </property>


$ scp yarn-site.xml root@spk_cli:/usr/local/spark-2.3.2-bin-hadoop2.7/conf
$ scp yarn-site.xml root@hdp1:/usr/local/hadoop-2.7.3/etc/hadoop
$ scp yarn-site.xml root@hdp2:/usr/local/hadoop-2.7.3/etc/hadoop
$ scp yarn-site.xml root@hdp3:/usr/local/hadoop-2.7.3/etc/hadoop
```

6. start YARN resource manager and node managers
```shell
$ start-yarn.sh
```

## Set up Spark client

1. access spark client node
```shell
$ docker container exec -it <spk_cli ID> bash
```

2. edit spark-env.sh and add lines below
```shell
$ cd $SPARK_HOME/conf
$ vi spark-defaults.conf

spark.shuffle.service.enabled true
spark.dynamicAllocation.enabled true
spark.dynamicAllocation.initialExecutors 4
spark.dynamicAllocation.minExecutors 2
spark.dynamicAllocation.maxExecutors 20
spark.dynamicAllocation.schedulerBacklogTimeout 1m
spark.dynamicAllocation.executorIdleTimeout 1m

```

3. start spark-shell
```shell
$ spark-shell --master yarn
2021-12-05 11:09:14 WARN  NativeCodeLoader:62 - Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
2021-12-05 11:09:40 WARN  Client:66 - Neither spark.yarn.jars nor spark.yarn.archive is set, falling back to uploading libraries under SPARK_HOME.
Spark context Web UI available at http://802636b4d2b4:4040
Spark context available as 'sc' (master = yarn, app id = application_1638723680963_0001).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.3.2
      /_/
         
Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_181)
Type in expressions to have them evaluated.
Type :help for more information.

scala> 
```


