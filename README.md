High Availability Hadoop in Docker with JournalNode Quorum and Zookeeper, with Kubernetes in mind. Based on the official guide [Hadoop HA with JournalNode Quorum](https://hadoop.apache.org/docs/r2.7.3/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithQJM.html).

# Package details
* Java 8
* Hadoop 2.7.3

# Quickstart
For the sake of simplicity we will use 1 Zookeeper, 1 JournalNode, 2 NameNodes and 1 DataNode to be able to run on a single Docker host.
It goes without saying that you should adjust these numbers in production.

1. Create a common docker network  
```docker network create hadoop```

2. Start Zookeeper  
```docker run --net=hadoop --name zk-1 --restart always -d zookeeper```

3. Start JournalNode  
```docker run -d --name=jn-1 -e "NNODE1_IP=nn1" -e "NNODE2_IP=nn2" -e "JN_IPS=jn-1:8485" -e "ZK_IPS=zk-1:2181" --net=hadoop -v /tmp/hadoop-jn:/mnt/hadoop bastipaeltz/hadoop-ha-docker /etc/bootstrap.sh -d journalnode```

4. Format the active NameNode  
```docker run --hostname=nn1 --name=nn1 -it -e "NNODE1_IP=nn1" -e "NNODE2_IP=nn2" -e "JN_IPS=jn-1:8485" -e "ZK_IPS=zk-1:2181" --net=hadoop -v /tmp/hadoop-nn1:/mnt/hadoop bastipaeltz/hadoop-ha-docker /etc/bootstrap.sh -d format```

5. Sync the initial state to the standby NameNode  
```docker run --hostname=nn2 --name=nn2 -it -e "NNODE1_IP=nn1" -e "NNODE2_IP=nn2" -e "JN_IPS=jn-1:8485" -e "ZK_IPS=zk-1:2181" --net=hadoop -v /tmp/hadoop-nn2:/mnt/hadoop -v /tmp/hadoop-nn1:/mnt/shared/nn1 bastipaeltz/hadoop-ha-docker /etc/bootstrap.sh -d standby```  

Notice that the volume from nn1 - which now holds the initial cluster state - is just mounted to a certain directory where all data will be copied to nn2's volume.  
At this point both volumes hold the initial cluster state and can be used as a mountpoint in actual NameNode images.

6. Start both NameNodes (separate terminals)  
```docker run --hostname=nn1 -p 50060:50070 --name=nn1 -it -e "NNODE1_IP=nn1" -e "NNODE2_IP=nn2" -e "JN_IPS=jn-1:8485" -e "ZK_IPS=zk-1:2181" --net=hadoop -v /tmp/hadoop-nn1:/mnt/hadoop bastipaeltz/hadoop-ha-docker /etc/bootstrap.sh -d namenode```  
```docker run --hostname=nn2 -p 50070:50070 --name=nn2 -it -e "NNODE1_IP=nn1" -e "NNODE2_IP=nn2" -e "JN_IPS=jn-1:8485" -e "ZK_IPS=zk-1:2181" --net=hadoop -v /tmp/hadoop-nn2:/mnt/hadoop bastipaeltz/hadoop-ha-docker /etc/bootstrap.sh -d namenode```

Now both NameNodes should be running, check it by visiting the WebUI on Port 50060 (nn1) and 50070 (nn2). nn2 should be `standby` while nn1 is `active`.

7. Start DataNodes

```docker run -d -e "NNODE1_IP=nn1" -e "NNODE2_IP=nn2" -e "JN_IPS=jn-1:8485" -e "ZK_IPS=zk-1:2181" --net=hadoop -v /tmp/hadoop-dn-1:/mnt/hadoop bastipaeltz/hadoop-ha-docker /etc/bootstrap.sh -d datanode```

8. Kill the active NameNode to trigger failover

Just press CTRL-C on the terminal which is attached to the active NameNode. Now watch on the WebUI how the standby NameNode gets active.  
DataNodes are still connected. Wait a bit and restart the formerly active NameNode. Now it will be the standby Node.

## A word on Kubernetes
Kubernetes is generally not that good in doing one thing before another one (steps 4 and 5).
Steps 4 and 5 can be accomplished by using InitContainers to pass around the volumes similar how it is done in [this example](https://kubernetes.io/docs/tutorials/stateful-application/run-replicated-stateful-application/#statefulset).

## Extra bootstrap parameters
* ```-d``` - Runs the service continuously instead of auto quiting
* ```-b``` - Opens a bash terminal after starting to inspect its internal file system.

## Cluster name
By default the cluster will start with the name "cluster". You can set this name with ```$CLUSTER_NAME```

## Mandatory environment variables
For the containers to run you need to set 3 environment variable on the docker run command.

* ```NNODE1_IP``` : Address to NameNode 1 without port
* ```NNODE2_IP``` : Address to NameNode 2 without port
* ```JN_IPS```: Comma separated addresses for JournalNodes with port 8485. (At least 3 or more as long as it is an uneven number.)
* ```ZK_IPS```: Comma separated addresses for Zookeeper nodes with port (default is 2181).  (At least 3 or more as long as it is an uneven number.)

## Storage
* ```/mnt/hadoop/dfs/name``` - For NameNode storage
* ```/mnt/hadoop/dfs/data``` - For DataNode storage
* ```/mnt/hadoop/journal/data``` - For Journal storage
* ```/usr/local/hadoop/logs/``` - For the logs. You can also replace the ```/usr/local/hadoop/etc/log4j.properties``` with an attach docker volume to that file to customize the logging settings

## Networking

It is important that the IPs of the namenode are the IP address/DNS name of the containers because Hadoop actually binds to those addresses.

# Fencing
In certain situations the NameNodes need to fence for a proper failover. Now the Fence will always return true without doing anything. Replace ```/etc/fence.sh`` with a docker volume attach for your own fencing algorithm. Probably something like a call to your docker scheduler to close down the other NameNode.

