## 指令

### 基本

```sh
# 获取kafka某个topic的所有partition的offset
kafka-run-class.sh kafka.tools.GetOffsetShell --bootstrap-server 10.16.21.23:9092,10.16.21.24:9092,10.16.21.25:9092 --topic HICON_VEHICLES_TRACE
# 从指定offset开始消费
./kafka-console-consumer.sh --bootstrap-server 10.16.21.23:9092,10.16.21.24:9092,10.16.21.25:9092 --topic HICON_VEHICLES_TRACE --offset 35000000 --partition 0

# kafka从指定时间开始消费
## 指定某个group的offset到某个时间点
kafka-consumer-groups.sh \
--bootstrap-server 10.16.21.23:9092,10.16.21.24:9092,10.16.21.25:9092 \
--group jida \
--topic HICON_VEHICLES_TRACE \
--reset-offsets \
--to-datetime 2023-07-03T09:30:00.000 \
-execute
## 通过该group进行消费
./kafka-console-consumer.sh \
-topic HICON_VEHICLES_TRACE \
--bootstrap-server 10.16.21.23:9092,10.16.21.24:9092,10.16.21.25:9092 \
--group jida 
# 打印其他属性
--property print.offset=true \
--property print.partition=true \
--property print.headers=true \
--property print.timestamp=true \
--property print.key=true
# 指定序列化与反序列化方式
--key-deserializer "org.apache.kafka.common.serialization.LongDeserializer"	\
--value-deserializer "org.apache.kafka.common.serialization.DoubleDeserializer"

# 创建topic
.\kafka-topics.bat --create --topic tess-trace --replication-factor 1 --partitions 10 --bootstrap-server localhost:9092

./kafka-topics.sh --create --topic tess-trace --replication-factor 1 --partitions 10 --bootstrap-server 10.16.21.23:9092,10.16.21.24:9092,10.16.21.25:9092

# 删除topic
.\kafka-topics.bat --delete --topic HICON_VEHICLES_TRACE --bootstrap-server localhost:9092
# 列出topic
./kafka-topics.sh --list --bootstrap-server localhost:9092 --topic tess-trace
```

## 遇到的错误

### Local: Message timed out

```sh
[2023-03-09 11:48:44.400682]ERR KafkaProducer[e227173a-50261bf0] FAIL | [thrd:123.60.14.186:9092/bootstrap]: 123.60.14.186:9092/0: 2 request(s) timed out: disconnect (after 1001ms in state UP)
== Message delivery failed: Local: Message timed out
```

#### 解决

- 修改参数：

  ```json
  { "message.max.bytes", "200000000" },
  { "batch.num.messages", "1000000" },
  { "batch.size", "2147483647" }
  ```

  将参数限制增大

## 其它

### Topic命名规范

- 不能点"." 或者下划线"_"
- 不要创建字母相同，仅大小写不同的topic

### modern-kafka的对象作为全局变量，会出错

==原因未知==

wgfz-350-1682301858933
wgfz-350-1682302409497

### 时间戳

>[Kafka消息时间戳(kafka message timestamp) - huxihx - 博客园 (cnblogs.com)](https://www.cnblogs.com/huxi2b/p/6050778.html)
