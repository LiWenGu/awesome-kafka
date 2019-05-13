基于0.10

# 1 kafka 入门

消息系统：作为消息系统的队列模式（点对点模式）和发布-订阅模式
存储系统：同步阻塞发发送消息，等待消息完全地复制到多个节点，才认为这条消息发送成功  
流处理系统：提供实时的流式数据处理能力：处理乱序、迟来的数据、重新处理输入数据、窗口和状态操作等

![][1]

四种核心 API：生产者、消费者、连接器、流处理  
![][2]

## 1.2 基本概念

### 1.2.1 分区模型

![][3]

每个主题（Topic）有多个分区（partition），每个消息入根据不同的 Topic 均匀的分散在其中的分区中，其中入分区的每个消息都带有一个自增的偏移量，分区通过偏移量（offset）来标识消费/写进度  
  
通过消费相同分区的消息，来保证有序性

### 1.2.2 消费模型

推模型：由消息中心 Broker 推送消息到消费者，缺点在于，Broker 需要记录消息已发送/已消费/未消费状态值  
拉模型：由消费者自己从消息中心 Broker 拉消息，这也是 kafka 选择的模式

kafka 保存所有消息，无论是否消费。这样消费者可以根据偏移量重复消费之前的消息，或者跳着消费，缺点在于磁盘占用空间大，需要合理的设置消息清理时间，kafka 两天清理一次

![][4]

### 1.2.3 分布式模型

消息中心，即 Broker 作为服务端，而生产者和消费者作为客户端  
Broker 主节点用于处理客户端们的消息读写，而 Broker 副节点用于消息的冗余，即为最小单位 partition 分区保证完整性，而采用分布式存储，主副节点支持故障转移  
  
消费者组支持纵向扩展，增加某个 Topic 消息吞吐量  

![][5]

## 1.3 Kafka 的设计与实现

### 1.3.1 文件系统的持久化与数据传输效率

预读：提前将一个比较大的磁盘块读入内存  
后写：将很多小的逻辑写操作合并起来组合成一个大的物理写操作  
磁盘缓存：将主内存剩余的所有空闲内存空间作为磁盘缓存，用于磁盘读写操作前的缓存  
因此，在某些情况下，磁盘顺序读写比随机内存读写快  
  
正常写入磁盘都是，先用应用程序写入内存，然后刷新到磁盘。但是 kafka 先存入磁盘缓存，然后刷新到磁盘（这不一样么，磁盘缓存在某种角度来说也是内存。。。）  
![][6]  

传统数据复制方案：操作系统将数据从磁盘读到内核空间的页面缓存->应用程序将数据从内核空间读到用户空间的缓存区->应用程序将数据从用户空间写回内核空间的 socket 缓存区->操作系统将数据从 socket 缓存区复制到王卡卡接口，通过网络发送出去    

kafka 的零拷贝方案：操作系统将数据从磁盘读到内核空间的页面缓存区->操作系统将数据直接通过网卡接口通过网络发送出去  

10 个消费者情况下，传统方案，需要 10 * 4 = 40 次  
零拷贝方案需要 10 + 1 = 11 次，其中的 1 次为从磁盘到内核空间的页面缓存

![][7]

### 1.3.2 生产者与消费者

生产者采用一种”在100ms内消息大小达到64字节要立即发送，如果在100ms时还没达到64字节，也要把已经收集的消息发送出去“，通过这种缓存机制，降低延迟以换取吞吐量  
  
消费者记录分区消费状态，好处在于消费者可以重新消费之前的消费。而消费状态是通过消费进度检查点文件实现，即在这个点之前的消息都已经被消费  
消费者拉模型的缺点在于，如果消息中心（Broker）没有消息，而消费者还是继续处于一种轮询阻塞的方式请求。解决方法在于：消费者请求 Broker 时，通过消费者设置的”最低消费字节数“来判断 Broker 消息是否足够，从而是否立即返回或继续阻塞

### 1.3.3 副本机制和容错处理

每个 partition 在同一个节点上，可能为主，也有可能为从
 
![][8]

为了避免数据热点问题（主数据全在一个机器上），尽量保证每个节点作为不同 partition 的主或从

副节点与主节点通信方式和客户端与主节点通信类似，只不过副节点将消息持久化，而客户端是将消息消费  
副节点正在同步中（in-sync）状态：1. 副节点与 zk 连接。2. 消息复制进度不能落后太多

如何保证消息被消费者看到后，消息是真实存在磁盘中的：生产者发送消息提交到主节点，只有当副节点从主节点复制完消息后，该消息才会被消费者可见，这就是消息提交机制


# 2 生产者

## 2.1 新生产者客户端

### 2.1.1 同步和异步发送消息

#### 2.1.1.1 为消息选择分区

消息如果没有带键，则通过 round-robin 选择 Broker，如果带有键，则对键散列后选则 Broker  
主Broker负责外部的消息读写，然后与副Broker同步消息进度  
![][2_1] 

#### 2.1.1.2 客户端记录收集器

生产者发送的消息先在客户端缓存到记录收集器 RecordAccumulator  
![][2_2]

### 2.1.2 客户端消息发送线程

1. 根据分区进行发送消息：假如有两台服务器，每台服务器有三个分区，那么需要分别对分区发送消息，一共需要六次请求  
2. 根据服务器节点发送消息：如上，只需要三个分区发一次即可，一共需要两次请求，kafka使用此方式  
![][2_3]

#### 2.1.2.1 从记录收集器获取数据

#### 2.1.2.2 创建生产者客户端请求


### 2.1.3 客户端网络连接对象

#### 2.1.3.1 准备发送客户端请求

#### 2.1.3.2 客户端轮询并调用回调函数

inFlightRequests 缓存还没有收到响应的客户端请求，同一个服务端，如果上一个客户端请求还没有发送完成，则不允许发送新的客户端请求  

1. 不需要响应的流程：在于请求发送成功后从 inFlightRequests 队列中移除
2. 需要响应的流程：请求发送后，接收到完整响应后，从 inFlightRequests 队列中移除

#### 2.1.3.3 客户端请求和客户端响应的关系

1. 客户端请求（ClientRequest）：包含客户端发送的请求和回调处理器
2. 客户端响应（ClientResponse）：包含了客户端请求头和客户端请求的回调函数以及其它响应信息

### 2.1.4 选择器处理网络请求

1. SocketChannel：channel.read(buffer)/channel.write(buffer) 缓冲区和通道数据交换
2. Selector：发生听到的事件有读/写，选择器通过选择键的方式监听读写事件的发生  
3. SelectionKey：将通道注册到选择器上， channel.register(selector) 返回对应选择键，进行逻辑处理

#### 2.1.4.1 客户端连接服务端并建立 Kafka 通道

KafkaChannel 抽象

#### 2.1.4.2 Kafka 通道和网络传输层

#### 2.1.4.3 Kafka 通道上的读写操作

#### 2.1.4.4 选择器的轮询

通过不断的注册事件、执行事件处理、取消事件  
![][2_4]

## 2.3 服务端网络连接

基于 Scala 的 Reactor模式，具体参考 Netty 的主从 Reactor 多线程模式：接收器线程+多个处理器线程

### 2.3.1 服务端使用接收器接收客户端的连接

### 2.3.2 处理器使用选择器的轮询处理网络请求 

### 2.3.3 请求通道的请求队列和响应队列

![][2_5]

### 2.3.4 Kafka 请求处理线程

一个接收器线程、多个处理器  
一个请求通道、一个请求队列、多个响应队列  
一个请求处理线程连接池、多个请求处理线程、一个服务端请求入口

### 2.3.5 服务端的请求处理入口

# 3 消费者：高级API和低级API

![][3_1]  
  
一个分区只可被消费组中的一个消费者所消费：  
1. 一个消费组中，一个消费者可以消费多个分区
2. 每个消费组都会全量消费所有分区
3. 同一个消费组下的消费者们不重复的共同消费所有分区








[1]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Kafka%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95/1_1.png
[2]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Kafka%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95/1_2.jpg
[3]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Kafka%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95/1_3.png
[4]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Kafka%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95/1_4.png
[5]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Kafka%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95/1_5.png
[6]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Kafka%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95/1_6.png
[7]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Kafka%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95/1_7.png
[8]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Kafka%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95/1_8.png
[2_1]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Kafka%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95/2_1.png
[2_2]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Kafka%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95/2_2.png
[2_3]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Kafka%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95/2_3.png
[2_4]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Kafka%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95/2_4.png
[2_5]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Kafka%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95/2_5.png
[3_1]: https://leran2deeplearnjavawebtech.oss-cn-beijing.aliyuncs.com/learn/Kafka%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95/3_1.png