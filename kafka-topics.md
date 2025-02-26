## kafka-topics.sh

1. 查看版本
   ```
   ~/kafka_2.13-3.3.1/bin/kafka-topics.sh --version
   ==> 3.3.1 (Commit:e23c59d00e687ff5)
   ```
3. 查看查看所有的Topic
   ```
   ~/kafka_2.13-3.3.1/bin/kafka-topics.sh --list --bootstrap-server localhost:9092
   ==> test1
   ```
5. 創建Topic，預設
   ```
   ~/kafka_2.13-3.3.1/bin/kafka-topics.sh --create --bootstrap-server 127.0.0.1:9092 --topic ${topic name}
   ~/kafka_2.13-3.3.1/bin/kafka-topics.sh --create --bootstrap-server 127.0.0.1:9092 --topic test1
   ==> Created topic test1.
   ```
7. 創建Topic，建立副本
   ```
   ~/kafka_2.13-3.3.1/bin/kafka-topics.sh --create --bootstrap-server ${IP:Port} --replication-factor ${int} --partitions ${int} --topic ${topic name}
   ```
9. Topic，詳細描述
    ```
    ~/kafka_2.13-3.3.1/bin/kafka-topics.sh --describe --bootstrap-server ${IP:Port}  --topic ${topic name}
    ```
```
   ~/kafka_2.13-3.3.1/bin/kafka-topics.sh --create --bootstrap-server ${IP:Port} --replication-factor ${int} --partitions ${int} --topic ${topic name}

   --topic：指定主題名稱。
   --partitions：指定主題的分區數量。
   --replication-factor：指定每個分區的複製因子（副本數量）。將每一個分區複製到其中兩個 broker 上。
   --bootstrap-server：指定 Kafka 伺服器的連線位置，例如 localhost:9092。
```
10. 刪除 Topic
   ~/kafka_2.13-3.3.1/bin/kafka-topics.sh -delete --bootstrap-server localhost:9092 --topic test1








case 1. 設定黨為預設條件, 指令優先處理
confing/server.preperies 
partition = 4 
分成了四組partition, 但是因為沒有給予replication-factor, 所以Leader 跟Replicas跟Isr都是 0, 也就是本機
```
   ~/kafka_2.13-3.3.1/bin/kafka-topics.sh --create --bootstrap-server 127.0.0.1:9092 --topic test1
   ==> Created topic testP.

   --replication-factor 副本數為2
   --partitions：指定主題的分區數量為3。

==> Topic: test1	TopicId: aKGLhOFLSw2TJlcEaTKQOQ	PartitionCount: 4	ReplicationFactor: 1	Configs: 
   	Topic: test1	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
	  Topic: test1	Partition: 1	Leader: 0	Replicas: 0	Isr: 0
	  Topic: test1	Partition: 2	Leader: 0	Replicas: 0	Isr: 0
	  Topic: test1	Partition: 3	Leader: 0	Replicas: 0	Isr: 0
```

case 2. 一個zookpeer, 三個break.

準備三個設定黨
```
cd ~/kafka_2.13-3.3.1/confing/
cp server.properties server-1.properties
cp server.properties server-2.properties
cp server.properties server-3.properties
```
依序修改設定檔
1. 調整 port, 每台都要不一樣
2. 每個 broker 的 id 必須不同
3. log.dirs 就是讓訊息所謂持久化存放的位置，依照 broker 區分不能重複
```
broker.id=0 # 與Leader對應
listeners=PLAINTEXT://localhost:9092  # port要不一樣
log.dirs=/tmp/kafka-logs-1 # path要不一樣
```

啟動broker
```
cd   ~/kafka_2.13-3.3.1/
~/kafka_2.13-3.3.1/bin/kafka-server-start.sh ~/kafka_2.13-3.3.1/confing/server-1.properties
~/kafka_2.13-3.3.1/bin/kafka-server-start.sh ~/kafka_2.13-3.3.1/confing/server-2.properties
~/kafka_2.13-3.3.1/bin/kafka-server-start.sh ~/kafka_2.13-3.3.1/confing/server-3.properties
```

建置topic
```
   ~/kafka_2.13-3.3.1/bin/kafka-topics.sh --create --bootstrap-server 127.0.0.1:9092 --replication-factor 2 --partitions 3 --topic testP
   ==> Created topic testP.

   --replication-factor 副本數為2
   --partitions：指定主題的分區數量為3。

    Topic: testP	TopicId: 2IUT_zjzRQqKYQeuBVmhTA	PartitionCount: 3	ReplicationFactor: 2	Configs: 
	  Topic: testP	Partition: 0	Leader: 2	Replicas: 2,0	Isr: 2,0
	  Topic: testP	Partition: 1	Leader: 1	Replicas: 1,2	Isr: 1,2
	  Topic: testP	Partition: 2	Leader: 0	Replicas: 0,1	Isr: 0,1
```
Isr: 1, 0 => 代表我們當前可以在 broker2 和 broker0 上訪問該 partition 0 的資料
=> 如果broker 0掛掉，Isr 會從 Isr: 1,0 變成 Isr: 1
=> Isr 的全名為in-sync replica，也就是已同步的副本


將副本放置於其他主機上
     ~/kafka_2.13-3.3.1/bin/kafka-topics.sh --create --bootstrap-server localhost:9091,localhost:9092,localhost:9093 --replication-factor 2 --partitions 4 --topic testP3
確認topic 詳細資料, 同時確認連線 (有連線問題時), 會顯示 Broker may not be available , 因為9095跟9090是不存在的
    ~/kafka_2.13-3.3.1/bin/kafka-topics.sh --describe --topic testPt --bootstrap-server localhost:9095,localhost:9090,localhost:9093
```
[2023-08-14 06:51:45,731] WARN [AdminClient clientId=adminclient-1] Connection to node -1 (localhost/172.20.10.6:9095) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
[2023-08-14 06:51:45,742] WARN [AdminClient clientId=adminclient-1] Connection to node -2 (localhost/172.20.10.6:9090) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
Topic: testPt	TopicId: XG6Q51S2STK4dycU3G_PdQ	PartitionCount: 4	ReplicationFactor: 2	Configs: 
	Topic: testPt	Partition: 0	Leader: 2	Replicas: 2,0	Isr: 2,0
	Topic: testPt	Partition: 1	Leader: 1	Replicas: 1,2	Isr: 1,2
	Topic: testPt	Partition: 2	Leader: 0	Replicas: 0,1	Isr: 0,1
	Topic: testPt	Partition: 3	Leader: 2	Replicas: 2,1	Isr: 2,1
```


模擬其中一個 broker 壞掉後再恢復的情況
注意, 即便恢復了 kafka也不會馬上復原,
而是需要再透過指令重新指派leader
```
kafka-topics --describe --zookeeper 127.0.0.1:2181 --topic topicWithThreeBroker

Topic: topicWithThreeBroker	TopicId: BAocHAwHR_STmwAUlI3YMw	PartitionCount: 3	ReplicationFactor: 2	Configs:
	Topic: topicWithThreeBroker	Partition: 0	Leader: 1	Replicas: 1,0	Isr: 1
	Topic: topicWithThreeBroker	Partition: 1	Leader: 2	Replicas: 2,1	Isr: 2,1
	Topic: topicWithThreeBroker	Partition: 2	Leader: 2	Replicas: 0,2	Isr: 2
```




















