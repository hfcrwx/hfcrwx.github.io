---
title: "ZooKeeper"
published: true
---

# ZooKeeper: Distributed process coordination

https://zookeeper.apache.org/

https://dlcdn.apache.org/zookeeper/stable/

https://dlcdn.apache.org/zookeeper/stable/apache-zookeeper-3.6.3.tar.gz



https://archive.apache.org/dist/zookeeper/

https://archive.apache.org/dist/zookeeper/zookeeper-3.4.5/zookeeper-3.4.5.tar.gz

https://zookeeper.apache.org/doc/r3.4.5/zookeeperAdmin.html#sc_systemReq

	yum install java-1.6.0-openjdk
	
	tar -xvzf zookeeper-3.4.5.tar.gz
	mv conf/zoo_sample.cfg conf/zoo.cfg
		dataDir=/root/zookeeper-3.4.5/data



```
./bin/zkServer.sh start
```

```
./bin/zkServer.sh start-foreground
```



```
./bin/zkCli.sh
```

