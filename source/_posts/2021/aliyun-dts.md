---
title: 阿里云DTS同步ES详解
date: "2021/7/24 20:46:25"
tags: [Aliyun, ES]
categories: ES
---

DTS是数据传输服务，Data Transmission Service的简称Data Transmission Service
![20230204234335](https://image.codingoer.top/blog/20230204234335.jpg)

## Apache Avro

[官网地址](https://avro.apache.org/docs/current/index.html)

Apache Avro是一个序列化工具，其优点是二进制传输，简洁高效，主要用于RPC序列化等。

它的主要特点有：支持二进制序列化方式，可以便捷，快速地处理大量数据；动态语言友好，Avro提供的机制使动态语言可以方便地处理Avro数据。

### Schemas

相关介绍可以看：https://www.jianshu.com/p/a5c0cbfbf608

![20230204012040](https://image.codingoer.top/blog/20230204012040.jpg)

### 对比其他系统的优点

当前市场上有很多类似的序列化系统，如Google的Protocol Buffers, Facebook的Thrift。

DTS中数据订阅Kafka客户端为什么选择了Avro作为序列化工具？他相比其他序列化系统有什么好处呢？

图片来源官网，看看就好！

![20230204234605](https:image.codingoer.top/blog/20230204234605.jpg)

### 相关扩展（Protocol）

Google protocol buffer是谷歌开源系列化工具，Google开源gRPC框架序列化工具

定义proto文件

```proto
// any.proto, api.proto, descriptor.proto, duration.proto, empty.proto, field_mask.proto, source_context.proto, struct.proto, timestamp.proto, type.proto, wrappers.proto
syntax = "proto3";
 
option java_multiple_files = true;
option java_package = "com.lionel.test.grpc.grpc";
option java_outer_classname = "OperationItfDescr";
option objc_class_prefix = "IOI";
 
import "google/protobuf/any.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/timestamp.proto";
 
package operationitf;
 
service OperationItf{
    rpc operate (OperateRequest) returns (google.protobuf.Empty) {}
}
 
message OperationDto {
    enum Source {
        ORDER = 0;
        CMS = 1;
    }
    enum Type {
        LOCK = 0;
        DEDUCT_LOCKED = 1;
        DEDUCT = 2;
    }
    int32 count = 1;
    string descr = 2;
    int64 goodsId = 3;
    Source optSource = 4;
    int64 skuId = 5;
    bool sync = 6;
    Type type = 7;
    string uuid = 8;
}
 
message OperateRequest {
    OperationDto arg0 = 1;
}
```

使用插件根据proto文件，生成客户端存根代码，实现rpc调用。

## DTS数据订阅Kafka客户端Demo

从Github上下载即可，在阿里云官方文档上有。

## Demo代码结构

下载Demo工程后，导入IDEA，先看下目录结构

![20230204234835](https:image.codingoer.top/blog/20230204234835.jpg)

- avro目录
里面包含了反序列文件和工具
- javaimpl
Java实现代码

<!-- more --> 

现在就详细分析下Javaimpl目录下的代码，可以看到分为几个package

![20230204234938](https:image.codingoer.top/blog/20230204234938.jpg)

1. **boot**
Boot.java：启动类，创建数据生产者和消费者，启动线程
MysqlRecordPrinter：将Avro定义的Record记录打印成mysql语句

2. **com.alibaba.dts.formats.avro**
这个包下面的所有类就是根据定义的Record.avsc生成的，也就是根据schema生成Java类。

3. **common**
CheckPoint：消费位点
Context：上下文信息，里面有生产者和消费者
RecordListener：数据消费监听接口，数据处理的地方
UserCommitCallBack：数据消费成功后，定义的监听接口
UserRecord：数据对象，最终我们使用这个对象解析数据
WorkThread：工作线程

4. **metastore【kafka消费位点处理】**

MetaStore：kafka消费者偏移量处理接口
LocalFileMetaStore：本地处理消费位点
KafkaMetaStore：kafka处理消费位点
MetaStoreCenter：相当于工厂类

5. **recordgenerator【生产者，kafka的consumer】**

ClusterSwithListener：kafka集群切换，减少重复数据消费
ConsumerWrap：kafka消费者包装器，消息的来源
ConsumerWrapFactory：kafka消费者包装器工厂，获取kafka里面的消费者
Names：常亮，没什么说的
OffsetCommitCallBack：kafka消费偏移量提交回调接口
RecordGenerator：数据生产者，kafka里面的comsumer，线程

6. **recordprocessor【消费者】**

AvroDeserializer：反序列化
EtlRecordProcessor：数据消费者
MysqlFieldConverter：数据转换用
FieldConverter：数据转换用
FieldValue：数据转换用

### 线程分析

使用kafka客户端来订阅数据，一共有三个相关的线程。

- 数据生产者，等同于kafka中的cosumer
- 数据消费者
- 数据消费者中的提交消费偏移量的线程

本地启动Demo示例：

![20230204235418](https:image.codingoer.top/blog/20230204235418.jpg)

注：Finalizer和Reference Hander是GC线程。

从Spring Boot Admin中查看线程情况也是一样，可以看到涉及的三个线程。

![20230204235437](https:image.codingoer.top/blog/20230204235437.jpg)

![20230204235458](https:image.codingoer.top/blog/20230204235458.jpg)

![20230204235517](https:image.codingoer.top/blog/20230204235517.jpg)

**RecordProcessor线程**

数据消费者线程代码（中间省略了数据处理的部分，这里只说明线程情况）

```java
@Override
public void run() {
    while (!existed) {
        ConsumerRecord<byte[], byte[]> toProcess = null;
        Record record = null;
        int fetchFailedCount = 0;
        try {
            while (null == (toProcess = toProcessRecord.peek()) && !existed) {
                sleepMS(5);
                fetchFailedCount++;
                if (fetchFailedCount % 1000 == 0) {
                    log.info("EtlRecordProcessor: haven't receive records from generator for  5s");
                }
            }
            if (existed) {
                return;
            }
            // 省略数据处理代码。。。。
            toProcessRecord.poll();
        } catch (Exception e) {
            log.error("EtlRecordProcessor: process record failed, raw consumer record [" + toProcess + "], parsed record [" + record + "], cause " + e.getMessage(), e);
            existed = true;
        }
    }
}
```

从上面代码可以看出，当线程启动后会一直从数据消费队列中获取数据，如果没有数据，睡眠5毫秒，达到1000次后，也就是5秒还没有数据，打印一次日志。

**RecordGenerator线程**

数据生产者（也就是kafka消费者）线程。（中间省略了数据获取的部分，这里只说明线程情况）

```java
public void run() {
    int haveTryTime = 0;
    String message = "first start";
    ConsumerWrap kafkaConsumerWrap = null;
    while (!existed) {
        // 从上下文中获取数据消费者，也就是数据处理者processor
        EtlRecordProcessor recordProcessor = context.getRecordProcessor();
        try {
            kafkaConsumerWrap = getConsumerWrap(message);
            // 省略kafka consumer获取数据........
            while (!existed) {
                    // 数据进入队列
                    while (!recordProcessor.offer(1000, TimeUnit.MILLISECONDS, record) && !existed) {
                        if (++offerTryCount % 10 == 0) {
                            log.info("RecordGenerator: offer record has failed for a period (10s) [ " + record + "]");
                        }
                    }
                }
            }
        } catch (Throwable e) {
            if (isErrorRecoverable(e) && haveTryTime++ < tryTime) {
                log.warn("RecordGenerator: error meet cause " + e.getMessage() + ", recover time [" + haveTryTime + "]", e);
                sleepMS(tryBackTimeMS);
                message = "reconnect";
            } else {
                log.error("RecordGenerator: unrecoverable error  " + e.getMessage() + ", have try time [" + haveTryTime + "]", e);
                this.existed = true;
            }
        }
    }
 
}
```

从上面代码可以看出，数据生产者当线程启动后，获取到kafkaConsumer后，一直拉取kafka中的消息，如果有数据放入消费者队列。
如果失败，可能是网络原因，线程睡眠10秒（可配置），进行重试。

**RecordProcess#Commit**

提交kafka消费位点线程

```java
private WorkThread getCommitThread() {
    WorkThread workThread = new WorkThread(new Runnable() {
        @Override
        public void run() {
            while (!existed) {
                sleepMS(5000);
                commit();
            }
        }
    }, "Record Processor Commit");
    return workThread;
}
```

每五秒提交一次消费点位。

**总结**

以上，就是这三个线程来完成数据同步工作。总的来说就是：

1. 数据生产者线程（kafka消费者）获取到数据到，将数据加入数据消费者队列
2. 数据消费者线程从队列中获取数据，进行反序列化后处理处理，设置最新的消费位点
3. 提交偏移量的线程每五秒提交一次位点

### 启动流程分析

![20230204235856](https:image.codingoer.top/blog/20230204235856.jpg)

从上面流程图来看，启动流程很简单，最终的部分在构建生产者和消费者两个线程。下面来分析一下

创建生产者：

```java
private static RecordGenerator getRecordGenerator(Context context, Properties properties) {
 
    RecordGenerator recordGenerator = new RecordGenerator(properties, context,
            parseCheckpoint(properties.getProperty(INITIAL_CHECKPOINT_NAME)),
            new ConsumerWrapFactory.KafkaConsumerWrapFactory());
    context.setStreamSource(recordGenerator);
    return recordGenerator;
}
```

创建消费者

```java
private static EtlRecordProcessor getEtlRecordProcessor(Context context, Properties properties, RecordGenerator recordGenerator) {
 
    EtlRecordProcessor etlRecordProcessor = new EtlRecordProcessor(new OffsetCommitCallBack() {
        @Override
        public void commit(TopicPartition tp, long timestamp, long offset, String metadata) {
            // 回调之后设置信息消费点位
            recordGenerator.setToCommitCheckpoint(new Checkpoint(tp, timestamp, offset, metadata));
        }
    }, context);
    context.setRecordProcessor(etlRecordProcessor);
    return etlRecordProcessor;
}
```

创建生产者和消费者都是通过构造方法，将一些属性赋值。下面分别看下生产者和消费者线程运行时的逻辑

![20230204235953](https:image.codingoer.top/blog/20230204235953.jpg)

### 消费位点提交

消息成功消费后，需要告知Kafka，这部分是通过MetaStoreCenter来处理的。

![20230205000015](https:image.codingoer.top/blog/20230205000015.jpg)

MetaStoreCenter里面维护了一个Map，在创建生产者的时候将LocalFileMetaStore注册到map中。

**本地消费位点存储**

通过文件来实现，定义了AtomicFileStore。

**Kafka消费位点提交**

使用的是commitAsync方法。

### 回调函数

Demo示例中使用了回调接口

1. UserRecord中的UserCommitCallBack接口，在用户处理数据时的回调
2. 生产者RecordProcessor中的OffsetCommitCallBack接口，在消费者处理消费位点时的回调，通知kafka消费者。

在最外层创建了RecordListener接口（接口匿名实现），处理完数据后提交消费位点

```java
public static Map<String, RecordListener> buildRecordListener() {
     
    RecordListener mysqlRecordPrintListener = record -> {
        String ret = MysqlRecordPrinter.recordToString(record.getRecord());
        log.info("【Record Listener】 consume record is {}", ret);
        // 处理数据逻辑省略
        // 提交消费位点
        record.commit(String.valueOf(record.getRecord().getSourceTimestamp()));
    };
     
    return Collections.singletonMap("mysqlRecordPrinter", mysqlRecordPrintListener);
}
```

UserRecord中定义了UserCommitCallBack接口，而接口的实现是在创建消费者中定义的。

```java
record = fastDeserializer.deserialize(consumerRecord.value());
for (RecordListener recordListener : recordListeners.values()) {
    recordListener.consume(new UserRecord(new TopicPartition(consumerRecord.topic(), consumerRecord.partition()), consumerRecord.offset(), record, new UserCommitCallBack() {
        @Override
        public void commit(TopicPartition tp, Record commitRecord, long offset, String metadata) {
            commitCheckpoint = new Checkpoint(tp, commitRecord.getSourceTimestamp(), offset, metadata);
        }
    }));
}
```

RecordProcessor中定义了OffsetCommitCallBack接口，接口的实现是在创建消费者时定义，将值传递给了生产者。

```java
private static EtlRecordProcessor getEtlRecordProcessor(Context context, Properties properties, RecordGenerator recordGenerator) {
    EtlRecordProcessor etlRecordProcessor = new EtlRecordProcessor(new OffsetCommitCallBack() {
        @Override
        public void commit(TopicPartition tp, long timestamp, long offset, String metadata) {
            recordGenerator.setToCommitCheckpoint(new Checkpoint(tp, timestamp, offset, metadata));
        }
    }, context);
    context.setRecordProcessor(etlRecordProcessor);
    return etlRecordProcessor;
}
```

这样通过一层层回调函数，将消费位点传给了KafkaConsumer。这种设计思想可以学习。

### 数据节点

1. kafkaConsumer.poll

```java
ConsumerRecords<byte[], byte[]> records = kafkaConsumerWrap.poll();
```

![20230205000335](https:image.codingoer.top/blog/20230205000335.jpg)

![20230205000405](https:image.codingoer.top/blog/20230205000405.jpg)

2. Avro Record

```java
Record record = fastDeserializer.deserialize(consumerRecord.value());
```

![20230205000430](https:image.codingoer.top/blog/20230205000430.jpg)

![20230205000441](https:image.codingoer.top/blog/20230205000441.jpg)

3. UserRecord

```java
recordListener.consume(new UserRecord(new TopicPartition(consumerRecord.topic(), consumerRecord.partition()), consumerRecord.offset(), record, new UserCommitCallBack() {
    @Override
    public void commit(TopicPartition tp, Record commitRecord, long offset, String metadata) {
        commitCheckpoint = new Checkpoint(tp, commitRecord.getSourceTimestamp(), offset, metadata);
    }
}));
```

![20230205000514](https:image.codingoer.top/blog/20230205000514.jpg)

4. MysqlRecordPrinter.recordToString

```log
recordID [521705329]source [{"sourceType": "MySQL", "version": "5.6.47-log"}]dbTable [null.null]recordType [BEGIN]recordTimestamp [1624467825]extra tags [{Q_SQL_MODE_CODE=1075838976, Q_CHARSET_CODE=45, GTID=9d437549-9a4c-11ea-9659-9efc635af2ae:558572086, thread_id=55153591, readerThroughoutTime=1624467825852, Q_FLAGS2_CODE=0},]
```

```
recordID [521754300]source [{"sourceType": "MySQL", "version": "5.6.47-log"}]dbTable [dcr.t_dcr_rectify]recordType [UPDATE]recordTimestamp [1624468556]extra tags [{pk_uk_info={"PRIMARY":["id"]}, thread_id=55156706, readerThroughoutTime=1624468556441},]
Field [id]Before [66]After [66]
Field [rectify_no]Before [RE1381622516602527744]After [RE1381622516602527744]
Field [rectify_process_id]Before [605]After [605]
Field [contract_id]Before [710919]After [710919]
Field [status]Before [7]After [7]
Field [handle_bd_code]Before [GSZN11468]After [GSZN11468]
Field [bd_department_whole_id]Before [10124-50563001642-50563003553-10061]After [10124-50563001642-50563003553-10061]
Field [bd_channel_whole_id]Before []After []
Field [distribute_bd_code]Before [GSZN5150]After [GSZN5150]
Field [distribute_bd_department_whole_id]Before [10124-10036-50563003549-10009-50563002040-50563006677]After [10124-10036-50563003549-10009-50563002040-50563006677]
Field [distribute_bd_channel_whole_id]Before []After []
Field [distribute_time]Before [2021-04-12 22:57:32]After [2021-04-12 22:57:32]
Field [remark]Before [测试同步啊a]After [测试同步啊aaa]
Field [is_deleted]Before [0]After [0]
Field [gmt_create]Before [2021-04-12 22:57:31]After [2021-04-12 22:57:31]
Field [gmt_modify]Before [2021-06-24 01:15:48]After [2021-06-24 01:15:56]
Field [stop_push_terminate_rectify]Before [0]After [0]
Field [stop_push_terminate_rectify_time]Before [1618239451.000000]After [1618239451.000000]
```