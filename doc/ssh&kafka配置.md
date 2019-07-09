###主机名称修改

- hostnamectl set-hostname <hostname>
- reboot
- k1,k2,k3,k4

### SSH配置

- 在本机配置config免密登录

  ![image-20190705144740053](/Users/gavin/Library/Application Support/typora-user-images/image-20190705144740053.png)

- 集群相互免密登录:在k1配置config,复制到其他集群节点

  - `for a in {2..3}; do scp {config,r1_rsa} k$a:~/.ssh/; done`
  - ` scp ~/.ssh/k1_rsa zk:~/.ssh/`

- 在集群上修改hosts映射关系
  - 通过 `scp` 命令复制发送到集群的每一个节点
  - 复制其他authorized_keys到centos-ext`for a in {2..3}; do ssh k$a cat ~/.ssh/authorized_keys >> ~/.ssh/authorized_keys; done`
  - 转发给其他集群`for a in {2..3}; do scp ~/.ssh/authorized_keys k$a:~/.ssh/authorized_keys; done`

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

  - --from-beginnig`为从头接收消息