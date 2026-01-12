# 消息队列设计要求
`MQ`的基本能力：
> [!tip] _一、要确保整个交互流程不出现消息丢失_

我们可以从三个方面去看待这个问题：
- producer 将 msg 投递到 mq 时不出现丢失
- msg 存放在 mq 时不出现丢失
- consumer 从 mq 消费 msg 时不出现丢失

针对第二点，各 mq 组件在实现上大抵上是基于==数据落盘+数据备份的方式保证的==
针对第一第三点，则是通过两个==交互环节中的 ack 机制保证的 + 消息幂去重==


> [!tip] 二、支持消息的存储

---
# 推送方式
根据`Consumer`消费的流程，将`MQ`分为`Push`型和`Pull`型

`Push`型：当 producer 将消息投递到 mq 时，由 mq 负责将消息以推送的形式主动发送给各个建立了订阅关系的消费方.
![[Pasted image 20250507223544.png]]

`pull`型：当 mq 中存在消息时，由 consumer 主动执行拉取消息的操作.
![[Pasted image 20250507223614.png]]

> [!tip] 优缺点
> `Push`
> 优点：
> - 流程实时性较强，消息来了就执行推送
> - 比较契合发布/订阅模型  
> 
> 缺点：
> - 对下游的Consumer保护力度不够，因为mq的核心功能是解耦、削峰，本质上是提供了一个缓冲的空间，让 consumer 能根据自己的消费能力在合适的时机进行消息处理，这个可以通过消费限流的方式加以弥补 
> 
> `Pull`
> 优点：
> - 下游握有消费操作的主动权，能选择在合适的时机执行消费操作
> 
> 缺点：
> - 实时性会弱一些，和主动 pull 的轮询机制有关
  


---
# Redis实现MQ的问题
- 存储昂贵
	redis 本身是基于内存实现的缓存组件，因此在存储消息时总容量相对有限.
- 数据丢失
	redis 存储消息时会不可避免地存在数据丢失的风险，可以从两个方面出发考虑：
	- 内存是易失性存储. 即便 redis 中有 rdb/aof 之类的持久化机制加以弥补，但这个持久化流程是异步执行的，无法提供百分百的保证力度
	- redis 走的是 ap 高可用流派，数据的主从复制流程是异步执行的，主从切换时数据存在弱一致问题

---
# List
> [!tip] 这种方式实现mq属于pull类型，消费方自行组织流程，在合适的时机进行消息的主动拉取

list 对应的 key 作为 MQ 的 topic 名称
生产者使用`lpush`推送消息
消费者使用`rpop`消费消息
使用`brpop`做到有数据才返回响应，否则令当前程序陷入阻塞，==解决了 consumer 合理阻塞消费数据的问题==
```shell
redis > brpop topic 0
1) "msg"
// 0：阻塞等待的时长，达到阈值仍仍未获取数据时会返回 nil. 如果设置为 0 ，则代表没有这个超时限制
```


## 无法支持发布/订阅模式
list 中的数据是独一份的，被 pop 出去后就不复存在了
==如果下游存在多个独立的消费组，各自都需要独立获取一份完整的数据==

## 无法支持消费端 ack 机制
倘若发生宕机或者其他意外错误，没有一种有效的手段能给予 mq 一个消息处理失败的反馈. ==这条消息一旦从 list 中被取走，就不再有机会被重新获取了，==

---
# pub/sub（发布/订阅模型）
消费者通过 subscribe 指令会对 channel 采用==阻塞模式进行监听，只有在有消息到来时，才会从阻塞状态中被唤醒==

消费方 subscriber 通过 subscribe 指令建立对某个 channel 的订阅关系
```shell
127.0.0.1:6379> subscribe my_channel_topic  
Reading messages... (press Ctrl-C to quit)  
1) "subscribe"  
2) "my_channel_topic"  
3) (integer) 1
```

生产方 publisher 通过 publish 指令往对应的 channel 中执行消息投递操作
```shell
127.0.0.1:6379> publish my_channel_topic msg  
(integer) 1
```

此时，之前对这个 channel 执行过 subscribe 操作的 subscriber 都会接收到这则消息
```shell
1) "message"
2) "my_channel_topic"
3) "msg"
```

redis pub/sub 模式的实现原理：
pub/sub 对于 channel 以及 subscribers 之间的实时映射关系存在强依赖
执行顺序：
- 先执行 subscribe 指令
- 执行 publish 执行
![[Pasted image 20250507230412.png]]

## 缺乏消息存储能力
redis pub/sub 机制就有点类似于 golang 中的无缓冲型 channel.
相当于只维护了channel 和 subscribers 的映射关系
==但是每条被投递的消息都是即来即走==
如果某一消费方宕机、redis宕机、subscriber 消费能力弱导致消息积压都会出现消息丢失的问题
subscriber 对应的缓冲区容量阈值可以在 redis.conf 文件中进行配置
```shell
client-output-buffer-limit pubsub 32mb 8mb 60
```
倘若某个 subscriber 的缓冲区 buffer 大小达到 32MB，则 subscriber 会被踢下线；倘若缓冲区内数据量在连续 60s 内达到 8MB 大小，subscriber 也会踢下线

## **无法支持消费端 ack 机制**
这个问题和 redis list 相同， subscriber 在获取到消息后，没有对应的 ack 机制，因此倘若处理失败，==想要执行消息的重放操作是无法做到的==


---
# Streams
**生产消息**：
通过 XADD 指令往 topic 中投入一组 kv 对消息
```shell
redis > XADD my_streams_topic * key1 val1  
1) "1638515664470-0"  
redis > XADD topic1 * key2 val2  
2) "1638515672769-0"
```
- my_streams_topic：topic 名称
- \*：消息自动生成唯一标识 id，基于时间戳+自增序号生成
- key1/val1、key2/val2：消息数据 kv 对

**消费消息**:
通过 XREAD 指令从对应 topic 中获取消息
```shell
redis > XREAD STREAMS my_streams_topic 0-0  
1) 1) "my_streams_topic"  
   2) 1) 1) "1638515664470-0"  
         2) 1) "key1"  
            2) "val1"  
      2) 1) "1638515672769-0"  
         2) 1) "key2"  
            2) "val2"
```

- my_streams_topic: topic 名称
- 0-0：从头开始消费. 倘若这里填为某条消息 id，则代表从这条消息之后（不包含这条消息）开始消费

**阻塞模式消费消息**：
streams 支持在消费时，采用阻塞模式进行消费. 倘若存在数据则即时返回处理，否则会阻塞消费流程
```shell
# BLOCK 0 表示阻塞等待时没有超时时间上限  
redis > XREAD BLOCK 0 STREAMS my_streams_topic 1638515672769-0  
(nil)
```

**发布订阅模型，能保证消息被多个消费者组 consumer group 同时消费到**：
1. 首先需要进行消费者组的创建
```shell
redis > XGROUP CREATE my_streams_topic my_group 0-0  
OK
```
- my_streams_topic：topic 名称
- my_group：消费者组名称
- 0-0：从头开始消费
- 基于消费者组消费消息

2. 通过 XReadGroup 指令，以消费者组的身份进行消费
```shell
redis > XREADGROUP GROUP my_group consumer BLOCK 0 STREAMS my_streams_topic >  
1) 1) "topic1"  
   2) 1) 1) "1638515664470-0"  
         2) 1) "key1"  
            2) "val1"  
      2) 1) "1638515672769-0"  
         2) 1) "key2"  
            2) "val2"
```
- my_group: 消费者组名称
- consumer：消费者名称
- my_streams_topic：topic 名称
- block 0: 采用阻塞等待的模式，0 代表没有超时上限
- > : 读最新的消息 (尚未分配给某个 consumer 的消息)

还有另一种消费模式，读取的是已分配给当前消费者，但是还未经确认的老消息:
```shell
redis > XREADGROUP GROUP my_group consumer STREAMS my_streams_topic 0-0
1) 1) "topic1"  
   2) 1) 1) "1638515664470-0"  
         2) 1) "key1"  
            2) "val1"  
      2) 1) "1638515672769-0"  
         2) 1) "key2"  
            2) "val2"
```
- 0-0：标识读取已分配给当前 consumer ，但是还没经过 xack 指令确认的消息

**确认消息**：
通过 xack 指令，携带上消费者组、topic 名称以及消息 id，能够完成对某条消息的确认操作
```shell
redis > XACK my_streams_topic my_group 1638515664470-0  
(integer) 1
```

- my_streams_topic：topic 名称
- my_group：消费者组名称
- 1638515664470-0：消息 id

## 支持发布/订阅模式
redis streams 引入了消费者组 group 的概念，因此是能够保证各个消费者组 consumer group 均能够获取到一份独立而完整的消息数据

## 数据可持久化
redis 中的 streams 和 string、list 等数据类型一样，都能够通过 rdb( redis database)、aof( append only file) 的持久化机制进行落盘存储，能够在很大程度上降低数据丢失的概率

## 支持消费端 ack 机制
redis streams 中另一项非常重要的改进，是支持 consumer 的 ack 能力. consumer 在处理好某条消息后，==能通过 xack 指令对该消息进行确认. 这样对于没经过 ack 确认的消息，redis streams 还是为 consumer 保留了重新消费的能力==

![[Pasted image 20250507233108.png]]

## 支持消息缓存
和 pub/sub 模式不同的是，redis streams 中会实际开辟内存空间用于存储 streams 中的数据. 因此哪怕某个 consumer group 是在消息生产之后才完成注册操作，也能够进行消息溯源，从 topic 起点开始执行消息的消费操作

redis streams 支持在每次投递消息时，显式设定一个 topic 中能缓存的数据长度，来人为限制这个缓存空间的容量
这里可以通过在 XADD 指令中加上 maxlen 参数，用于指定 topic 中能缓存的数据长度：
```shell
redis > XADD topic1 MAXLEN 10000 * key1 val1
```
- MAXLEN 10000：最多缓存 10000 条数据
==这样倘若 topic 数据容量超限，则新消息的到达会把最老的消息挤出队列==，意味着也可能存在数据丢失的风险，因此大家在使用时需要合理设置 maxlen 参数

![[Pasted image 20250507233307.png]]

---
# 对比
| **mq 实现方案** | **发布/订阅能力** | **消费端ACK机制** | **消息缓存能力** | **数据丢失风险** |
| ----------- | ----------- | ------------ | ---------- | ---------- |
| list        | 不支持         | 不支持          | 支持         | 低          |
| pub/sub     | 支持          | 不支持          | 不支持        | 高          |
| streams     | 支持          | 支持           | 支持         | 低          |

来看看 redis stream 存在哪些方面的优劣：

| **mq 组件**     | **消息存储介质** | **消息分区/并发能力** | **数据丢失风险** | **运维成本** |
| ------------- | ---------- | ------------- | ---------- | -------- |
| redis streams | 内存         | 不支持           | 低          | 低        |
| kafka         | 磁盘         | 支持            | 理论上不存在     | 偏高       |
可以看到：
- redis streams 在存储介质上需要使用内存，因此==消息存储容量相对有限==
- 同一个 topic 的数据由于对应为同一个 key，因此会被分发到相同节点，==无法实现数据的纵向分治==
- 基于 redis 实现的 mq ==一定是存在消息丢失的风险的==，尽管在生产端和消费端，producer/consumer 在和 mq 交互时可以通过 ack 机制保证在交互环节不出现消息的丢失，原因是：
	- redis 数据基于内存存储：哪怕通过最严格 aof 等级设置，由于==持久化流程本身是异步执行的==，也无法保证数据绝对不丢失
	- redis 走的是 ap 高可用流派：为保证可用性，redis 会在一定程度上牺牲数据一致性. ==在主从复制时，采用的是异步流程==，倘若主节点宕机，从节点的数据可能存在滞后，这样在主从切换时消息就可能丢失

==倘若业务流程对于数据的精度没有特别严格的要求==，那此时使用 redis streams 这样一种轻量化的 mq 实现方案未尝不是一种好的选择和尝试
