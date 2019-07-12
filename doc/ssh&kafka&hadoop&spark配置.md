### 主机名称修改

- hostnamectl set-hostname <hostname>
- reboot
- k1,k2,k3,k4

### 修改hosts

- 通过 `scp` 命令复制发送到集群的每一个节点
- 复制其他authorized_keys到centos-ext`for a in {2..3}; do ssh k$a cat ~/.ssh/authorized_keys >> ~/.ssh/authorized_keys; done`
- 转发给其他集群`for a in {2..3}; do scp ~/.ssh/authorized_keys k$a:~/.ssh/authorized_keys; done`

### SSH配置

- 在本机配置config免密登录

  ![image-20190705144740053](/Users/gavin/Library/Application%20Support/typora-user-images/image-20190705144740053.png)

- 集群相互免密登录

  - `/etc/ssh/sshd_config`下

    - ```shell
      PermitRootLogin yes #允许root远程登录
      PubkeyAuthentication yes  #开启公钥验证
      PasswordAuthentication yes #开启密码验证
      UsePAM yes #通过PAM验证
      ```

  - 将集群k1 修改后的 `/etc/ssh/sshd_config ` 通过 `scp` 命令复制发送到集群的每一个节点`for a in {2..4} ; do scp /etc/ssh/sshd_config k$a:/etc/ssh/sshd_config ; done`

  - 在集群的每一个节点节点输入命令 `ssh-keygen -t rsa`，生成 key，一律回车

  - 将集群每一个节点的公钥`id_rsa.pub`放入到自己的认证文件中authorized_keys,`for a in {1..4}; do ssh k$a cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys; done`

  - 将自己的认证文件 authorized_keys 通过 scp 命令复制发送到每一个节点上去:/root/.ssh/authorized_keys, `for a in {2..4}; do scp /root/.ssh/authorized_keys k$a:/root/.ssh/authorized_keys ; done`

  - 每个节点重启ssh

    - `systemctl restart sshd.service`

- Tips:

  - `systemctl restart sshd.service`重启失败，提示`Job for ssh.service failed because the control process exited with error code`
    - sudo /usr/sbin/sshd -T  ,查看为什么失败和哪里失败

  - ssh出现问题 可以 less -N /var/secure 查看ssh日志

### kafka配置

- 安装kafka

  - `cd ~`
  - `sudo yum install wget`
  - `sudo wget https://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.3.0/kafka_2.12-2.3.0.tgz`
  - `sudo tar -zxvf kafka_2.12-2.3.0.tgz`

- 在每个结点配置kafka

  - `sudo vi kafka_2.12-2.3.0/configs/server.properties`

  - ```c
    broker.id=0  每台服务器不能重复
    
    #设置zookeeper的集群地址
    zookeeper.connect=192.168.252.121:2181,192.168.252.122:2181,192.168.252.123:2181
    ```

- 先启动zookeeper

  - `for a in {2..4}; do ssh k$a "source /etc/profile; ~/zookeeper-3.4.14/bin/zkServer.sh start"; done`
  - 查看zk状态`for a in {2..4}; do ssh k$a "source /etc/profile; ~/zookeeper-3.4.14/bin/zkServer.sh status"; done`

- 启动kafka

  - `for a in {1..3};do ssh k$a "source /etc/profile; ~/kafka_2.12-2.3.0/bin/kafka-server-start.sh ~/kafka_2.12-2.3.0/config/server.properties" ;done`

  - 以守护进程方式后台启动kafka（&结尾 或可以使用nohup）`for a in {1..3};do ssh k$a "source /etc/profile; ~/kafka_2.12-2.3.0/bin/kafka-server-start.sh -daemon ~/kafka_2.12-2.3.0/config/server.properties > /dev/null 2>&1 &" ;done`

  - 停止kafka`for a in {1..3};do ssh k$a "source /etc/profile; ~/kafka_2.12-2.3.0/bin/kafka-server-stop.sh -daemon ~/kafka_2.12-2.3.0/config/server.properties" ;done`

  - jps(jvm process status)命令查看kafka是否启动

  - 过程中报错,删除

    - ```c
      kafka.common.KafkaException: Failed to acquire lock on file .lock in /tmp/kafka-logs. A Kafka instance in another process or thread is using this directory.
      
      $ rm -rf /tmp/kafka-logs
      ```

- 创建topic
  - topic  wzw`kafka_2.12-2.3.0/bin/kafka-topics.sh --create --zookeeper 10.0.0.5:2181,10.0.0.54:2181,10.0.0.79:2181 --replication-factor 2 --partitions 1 --topic wzw`
  - --replication-factor 备份
  - --partitions 分区
  - --topic 主题

- 查看topic
  - 列出topic`kafka_2.12-2.3.0/bin/kafka-topics.sh --list --zookeeper 10.0.0.5:2181,10.0.0.54:2181,10.0.0.79:2181`
  - --zookeeper zk连接端口
  - topic详情`kafka_2.12-2.3.0/bin/kafka-topics.sh --describe --zookeeper 10.0.0.5:2181,10.0.0.54:2181,10.0.0.79:2181 --topic wzw`

- 生产消息

  - 在任一结点上生产 9092为kafka工作端口

  - `kafka_2.12-2.3.0/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic wzw`


- 消费消息

  - `kafka_2.12-2.3.0/bin/kafka-console-consumer.sh --bootstrap-server k3:9092 --from-beginning --topic wzw`

  - `--from-beginnig`为从头接收消息

### Hadoop&Spark配置

- 每个Node准备

  - hadoop镜像`wget https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-3.1.2/hadoop-3.1.2.tar.gz`
  - spark镜像 spark-2.4.3-bin-hadoop2.7

  - Jdk1.8.0_211

  - Scala 2.11.0

  - `/etc/profile`

    - ```shell
      export JAVA_HOME=/opt/jdk1.8.0_211
      export JRE_HOME=$JAVA_HOME/jre
      export HADOOP_HOME=/home/centos/hadoop-3.1.2
      export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
      export CLASSPATH=.:$JAVA_HOME/lib:$CLASSPATH
      export SCALA_HOME=/usr/share/scala
      export PATH=$SCALA_HOME/bin:$PATH
      ```


- Hadoop配置

  - namenode上`/hadoop-3.1.2/etc/hadoop/`下配置
    - core-site.xml
    - hadoop-ev.sh
    - hdfs-site.xml
      - 配置dfs.http.address
    - workers
      - K2,k3,k4
    - yarn-site.xml
    - mapred-site.xml

  - scp到其他node
  - ./sbin/start-all.sh启动集群
  - jps查看是否成功

- Spark配置

  - master上配置`/spark-2.4.3-bin-hadoop2.7/conf`

    - mv spark-env.sh.template spark-ev.sh

      - ```shell
        export JAVA_HOME=/opt/jdk1.8.0_211
        export HADOOP_CONF_DIR=/home/centos/hadoop-3.1.2/etc/hadoop
        SPARK_MASTER_IP=k1
        SPARK_LOCAL_DIRS=/home/centos/spark-2.4.3-bin-hadoop2.7
        ```

    - mv slaves.template slaves

      - k2,k3,k4

    - 在`spark-2.4.3-bin-hadoop2.7/sbin`下
      - mv start-all.sh stat-spark-all.sh
      - mv  stop-all.sh stop-spark-all.sh
      - 防止和hadoop同名出错
      - 修改start-master.sh可以配置spark的web-ui端口（默认8080）

  - scp到其他node

  - ./sbin/start-spark-all.sh启动集群

  - jps查看是否成功