

### zookeeper

#### 下载

```bash
#下载
wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/stable/apache-zookeeper-3.5.9-bin.tar.gz
mkdir -p /opt/apache
tar xzvf apache-zookeeper-3.5.9-bin.tar.gz
cd /opt/apache/apache-zookeeper-3.5.9-bin
mkdir data
```

#### 配置

```bash
#修改配置
cd apache-zookeeper-3.5.9
cd conf/
cp zoo_sample.cfg  zoo.cfg
vim zoo.cfg

#按照 server.id = ip:port:port    ，在zoo.cfg 文件结尾添加配置
server.1=172.19.87.174:2888:3888
server.2=172.19.87.173:2888:3888
server.3=172.19.87.175:2888:3888
```

```bash
#配置完成的zoo.cfg文件为

# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/opt/apache/apache-zookeeper-3.5.9-bin/data
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
 
server.1=172.19.87.174:2888:3888
server.2=172.19.87.173:2888:3888
server.3=172.19.87.175:2888:3888
```

将该配置文件复制到其他两台机器对应目录下，内容保持不变。

**注意：****dataDir=**  参数 配置的目录应该为空目录，不能复用有数据的目录，否则会导致zookeeper启动失败。

#### 配置myid

根据 server.**id** = **ip**:port:port 中， id 和 ip 的对应关系，分别在 三台节点机器的 dataDir=/var/lib/zookeeper/ 指定的目录下， 新建 myid 文件并且填入对应的数字。

```bash
vim  /var/lib/zookeeper/myid
# 节点1 （ ip 为192.168.111.129 ）
1

# 节点2： （ip 为192.168.111.130 ）
2

#节点3： （ip 为192.168.111.131）
3
```

**注意：**myid 文件只能填数字（具体要求如下引用），除此之外不能添加其他内容，空格也不能有，否则会报错。

> The myid file consists of a single line containing only the text of that machine's id. So *myid* of server 1 would contain the text "1" and nothing else. The id must be unique within the ensemble and should have a value between 1 and 255. **IMPORTANT:** if you enable extended features such as TTL Nodes (see below) the id must be between 1 and 254 due to internal limitations.



### 测试

（1）关闭防火墙

首先必须关闭每台机器的防火墙，否则会导致zookeeper启动失败。

```bash
systemctl stop firewalld
```

（2）启动服务

```bash
# 在第一台机器启动zookeeper 服务
cd apache-zookeeper-3.5.9-bin/
./bin/zkServer.sh start

# 查看启动状态
./bin/zkServer.sh status

# 查看日志
tail -f ./logs/zookeeper-root-server-node1.out

```

（3）查看程序状态

```bash
# 节点1
./bin/zkServer.sh status

# 节点2
./bin/zkServer.sh status

# 节点3
./bin/zkServer.sh status
```

（4）测试

```bash
bin/zkCli.sh -server 192.168.111.129:2181,192.168.111.130:2181,192.168.111.131:2181
```






##### 参考

https://blog.csdn.net/abcdu1/article/details/113591746

