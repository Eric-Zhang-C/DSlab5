## jdk1.8 install
1. 下载jdk https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
2. 上传到`～`目录下
    ```
    $ scp /Users/bian/Downloads/jdk-8u211-linux-x64.tar.gz  centos-ext-k1:~
    $ ssh centos-ext-k1
    $ cd ~ 
    $ sudo mv jdk-8u211-linux-x64.tar.gz ～
    $ cd ～
    $ sudo tar zxvf jdk-8u211-linux-x64.tar.gz
    ```
3. 环境变量配置内容
    - `$ sudo vi /etc/profile`
    ```
        #jdk
        export JAVA_HOME=～/jdk1.8.0_211
        export JRE_HOME=${JAVA_HOME}/jre   
        export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib   
        export PATH=${JAVA_HOME}/bin:$PATH 
    ```
    - ` source /etc/profile`
4. 验证
    ```
    [centos@k1 opt]$ java -version
    java version "1.8.0_211"
    Java(TM) SE Runtime Environment (build 1.8.0_211-b12)
    Java HotSpot(TM) 64-Bit Server VM (build 25.211-b12, mixed mode)
    ``` 


## zookeeper install
- ZooKeeper版本：ZooKeeper-3.4.14.tar.gz
1. download
    ```
    $ sudo yum -y install wget
    $ cd ～
    $ sudo wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.14/zookeeper-3.4.14.tar.gz
    ```
2. 提取tar文件
    ```
    $ sudo tar -zxf zookeeper-3.4.14.tar.gz 
    $ cd zookeeper-3.4.14
    ```
    - 创建data文件夹 用于存储数据文件
        - ```$ mkdir ～/zookeeper-3.4.14/data```
    - 创建logs文件夹 用于存储日志
        - ```$ mkdir ～/zookeeper-3.4.14/logs```
3. 创建配置文件
- zookeeper的主要配置文件，因为Zookeeper是一个集群服务，集群的每个节点都需要这个配置文件。为了避免出差错，zoo.cfg这个配置文件里没有跟特定节点相关的配置，所以每个节点上的这个zoo.cfg都是一模一样的配置。这样就非常便于管理了，比如我们可以把这个文件提交到版本控制里管理起来。其实这给我们设计集群系统的时候也是个提示：集群系统一般有很多配置，应该尽量将通用的配置和特定每个服务的配置(比如服务标识)分离，这样通用的配置在不同服务之间copy就ok了
```
$ vi ～/zookeeper-3.4.14/conf/zoo.cfg
```
- 编辑内容如下
    ```
    tickTime = 2000
    dataDir =  /home/centos/zookeeper-3.4.14/data
    dataLogDir = /home/centos/zookeeper-3.4.14/logs
    tickTime = 2000
    clientPort = 2181
    initLimit = 5
    syncLimit = 2

    server.1=zk:2888:3888
    server.2=k2:2888:3888
    server.3=k3:2888:3888
    ```
- 配置文件描述
    - tickTime
        - tickTime则是上述两个超时配置的基本单位，例如对于initLimit，其配置值为5，说明其超时时间为 2000ms * 5 = 10秒。
    - dataDir
        - 其配置的含义跟单机模式下的含义类似，不同的是集群模式下还有一个myid文件。myid文件的内容只有一行，且内容只能为1 - 255之间的数字，这个数字亦即上面介绍server.id中的id，表示zk进程的id。
    - dataLogDir
        - 如果没提供的话使用的则是dataDir。zookeeper的持久化都存储在这两个目录里。dataLogDir里是放到的顺序日志(WAL)。而dataDir里放的是内存数据结构的snapshot，便于快速恢复。为了达到性能最大化，一般建议把dataDir和dataLogDir分到不同的磁盘上，这样就可以充分利用磁盘顺序写的特性。
    - initLimit
        - ZooKeeper集群模式下包含多个zk进程，其中一个进程为leader，余下的进程为follower。  当follower最初与leader建立连接时，它们之间会传输相当多的数据，尤其是follower的数据落后leader很多。initLimit配置follower与leader之间建立连接后进行同步的最长时间。
    - syncLimit
        - 配置follower和leader之间发送消息，请求和应答的最大时间长度。
    - server.id=host:port1:port2
        - server.id 其中id为一个数字，表示zk进程的id，这个id也是data目录下myid文件的内容
        - host 是该zk进程所在的IP地址
        - port1 表示follower和leader交换消息所使用的端口
        - port2 表示选举leader所使用的端口
4. 创建myid 文件
- 在data里会放置一个myid文件，里面就一个数字，用来唯一标识这个服务。这个id是很重要的，一定要保证整个集群中唯一
- ZooKeeper会根据这个id来取出server.x上的配置。比如当前id为1，则对应着zoo.cfg里的server.1的配置
    - `$ echo "1" > ～/zookeeper-3.4.14/data/myid`
- 这样一台k4机器就配置完了

- problem
    ```
    [centos@zk data]$ sudo echo "1" > ～/zookeeper-3.4.14/data/myid
    -bash: ～/zookeeper-3.4.14/data/myid: 权限不够
    ```
    - solution:`sudo sh -c "echo "1" > ～/zookeeper-3.4.14/data/myid"`

5. 复制集群配置
- 在集群k4上执行,复制配置好的zookeeper到其他两台主机上
```
for a in {2..3} ; do scp -r /home/centos/zookeeper-3.4.14/ k$a:/home/centos/zookeeper-3.4.14 ; done
```
- 在集群k4上执行 ,批量修改myid 文件
```
for a in {2..4} ; do ssh k$a "source /etc/profile; echo $a > /home/centos/zookeeper-3.4.14/data/myid" ; done
```

6. 启动集群
- 在集群任意一台机器上执行
```
$ for a in {2..4} ; do ssh k$a "source /etc/profile; /home/centos/zookeeper-3.4.14/bin/zkServer.sh start" ; done
```
- 响应
    ```
    ZooKeeper JMX enabled by default
    Using config: /home/centos/zookeeper-3.4.14/bin/../conf/zoo.cfg
    Starting zookeeper ... STARTED
    Killed by signal 1.
    ZooKeeper JMX enabled by default
    Using config: /home/centos/zookeeper-3.4.14/bin/../conf/zoo.cfg
    Starting zookeeper ... STARTED
    Killed by signal 1.
    ZooKeeper JMX enabled by default
    Using config: /home/centos/zookeeper-3.4.14/bin/../conf/zoo.cfg
    Starting zookeeper ... STARTED
    Killed by signal 1.
    ```

7. 连接集群
```
$ /home/centos/zookeeper-3.4.14/bin/zkCli.sh -server k4:2181,k2:2181,k3:2181
```
- 响应
```
[centos@k4 zookeeper-3.4.14]$ /home/centos/zookeeper-3.4.14/bin/zkCli.sh -server k4:2181,k2:2181,k3:2181
Connecting to k4:2181,k2:2181,k3:2181
2019-07-05 15:49:12,123 [myid:] - INFO  [main:Environment@100] - Client environment:zookeeper.version=3.4.14-4c25d480e66aadd371de8bd2fd8da255ac140bcf, built on 03/06/2019 16:18 GMT
2019-07-05 15:49:12,129 [myid:] - INFO  [main:Environment@100] - Client environment:host.name=k4
2019-07-05 15:49:12,129 [myid:] - INFO  [main:Environment@100] - Client environment:java.version=1.8.0_211
2019-07-05 15:49:12,132 [myid:] - INFO  [main:Environment@100] - Client environment:java.vendor=Oracle Corporation
2019-07-05 15:49:12,132 [myid:] - INFO  [main:Environment@100] - Client environment:java.home=/opt/jdk1.8.0_211/jre
2019-07-05 15:49:12,132 [myid:] - INFO  [main:Environment@100] - Client environment:java.class.path=/home/centos/zookeeper-3.4.14/bin/../zookeeper-server/target/classes:/home/centos/zookeeper-3.4.14/bin/../build/classes:/home/centos/zookeeper-3.4.14/bin/../zookeeper-server/target/lib/*.jar:/home/centos/zookeeper-3.4.14/bin/../build/lib/*.jar:/home/centos/zookeeper-3.4.14/bin/../lib/slf4j-log4j12-1.7.25.jar:/home/centos/zookeeper-3.4.14/bin/../lib/slf4j-api-1.7.25.jar:/home/centos/zookeeper-3.4.14/bin/../lib/netty-3.10.6.Final.jar:/home/centos/zookeeper-3.4.14/bin/../lib/log4j-1.2.17.jar:/home/centos/zookeeper-3.4.14/bin/../lib/jline-0.9.94.jar:/home/centos/zookeeper-3.4.14/bin/../lib/audience-annotations-0.5.0.jar:/home/centos/zookeeper-3.4.14/bin/../zookeeper-3.4.14.jar:/home/centos/zookeeper-3.4.14/bin/../zookeeper-server/src/main/resources/lib/*.jar:/home/centos/zookeeper-3.4.14/bin/../conf:.:/opt/jdk1.8.0_211/lib:/opt/jdk1.8.0_211/jre/lib
2019-07-05 15:49:12,133 [myid:] - INFO  [main:Environment@100] - Client environment:java.library.path=/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
2019-07-05 15:49:12,133 [myid:] - INFO  [main:Environment@100] - Client environment:java.io.tmpdir=/tmp
2019-07-05 15:49:12,133 [myid:] - INFO  [main:Environment@100] - Client environment:java.compiler=<NA>
2019-07-05 15:49:12,133 [myid:] - INFO  [main:Environment@100] - Client environment:os.name=Linux
2019-07-05 15:49:12,133 [myid:] - INFO  [main:Environment@100] - Client environment:os.arch=amd64
2019-07-05 15:49:12,133 [myid:] - INFO  [main:Environment@100] - Client environment:os.version=3.10.0-957.1.3.el7.x86_64
2019-07-05 15:49:12,133 [myid:] - INFO  [main:Environment@100] - Client environment:user.name=centos
2019-07-05 15:49:12,134 [myid:] - INFO  [main:Environment@100] - Client environment:user.home=/home/centos
2019-07-05 15:49:12,134 [myid:] - INFO  [main:Environment@100] - Client environment:user.dir=/home/centos/zookeeper-3.4.14
2019-07-05 15:49:12,135 [myid:] - INFO  [main:ZooKeeper@442] - Initiating client connection, connectString=k4:2181,k2:2181,k3:2181 sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@5ce65a89
Welcome to ZooKeeper!
2019-07-05 15:49:12,163 [myid:] - INFO  [main-SendThread(k4:2181):ClientCnxn$SendThread@1025] - Opening socket connection to server k4/fe80:0:0:0:f816:3eff:fe7c:5fda%2:2181. Will not attempt to authenticate using SASL (unknown error)
JLine support is enabled
2019-07-05 15:49:12,219 [myid:] - INFO  [main-SendThread(k4:2181):ClientCnxn$SendThread@879] - Socket connection established to k4/fe80:0:0:0:f816:3eff:fe7c:5fda%2:2181, initiating session
2019-07-05 15:49:12,237 [myid:] - INFO  [main-SendThread(k4:2181):ClientCnxn$SendThread@1299] - Session establishment complete on server k4/fe80:0:0:0:f816:3eff:fe7c:5fda%2:2181, sessionid = 0x1000046a7d40003, negotiated timeout = 30000

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[zk: k4:2181,k2:2181,k3:2181(CONNECTED) 0] 
```
- problem 
    ```
    2019-07-05 15:33:41,761 [myid:] - WARN  [main-SendThread(k3:2181):ClientCnxn$SendThread@1164] - Session 0x0 for server k3:2181, unexpected error, closing socket connection and attempting reconnect
    ```
    - 在k2中运行
        ```
        [centos@k2 ~]$ sudo systemctl stop firewalld.service
        Failed to stop firewalld.service: Unit firewalld.service not loaded.
        ```

8. 集群状态
```
$ for a in {2..4} ; do ssh k$a "source /etc/profile; /home/centos/zookeeper-3.4.14/bin/zkServer.sh status" ; done
```
响应

```
$ for a in {2..4} ; do ssh k$a "source /etc/profile; /home/centos/zookeeper-3.4.14/bin/zkServer.sh stop" ; done
```


