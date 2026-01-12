使用 Docker Compose 部署 NSQ
编写 Docker Compose 配置文件
创建 docker-compose.yml 文件
```yaml
version: '3'
services:
  # 1. 服务发现节点：nsqlookupd
  nsqlookupd:
    image: nsqio/nsq        # 使用NSQ官方镜像
    command: /nsqlookupd    # 启动nsqlookupd服务
    ports:
      - "4160:4160"  # 暴露TCP端口（供nsqd连接）
      - "4161:4161"  # 暴露HTTP端口（供nsqadmin查询）
    restart: unless-stopped # 容器退出时自动重启（除非手动停止）

  # 2. 消息队列节点：nsqd
  nsqd:
    image: nsqio/nsq
    command: /nsqd --lookupd-tcp-address=nsqlookupd:4160 --broadcast-address=host.docker.internal
    ports:
      - "4150:4150"  # 暴露TCP端口（生产/消费消息）
      - "4151:4151"  # 暴露HTTP端口（HTTP API操作）
    depends_on:
      - nsqlookupd   # 确保nsqlookupd先启动
    restart: unless-stopped

  # 3. Web管理界面：nsqadmin
  nsqadmin:
    image: nsqio/nsq
    command: /nsqadmin --lookupd-http-address=nsqlookupd:4161
    ports:
      - "4171:4171"  # 暴露Web管理界面端口
    depends_on:
      - nsqlookupd
    restart: unless-stopped
```

进入后使用 `docker compose up -d` 启动启动所有服务

| 命令                       | 作用                       |
| ------------------------ | ------------------------ |
| `docker compose up -d`   | 启动所有服务（后台运行）             |
| `docker compose down`    | 停止并删除所有容器（保留镜像和数据）       |
| `docker compose logs`    | 查看所有容器的日志                |
| `docker compose restart` | 重启所有服务                   |
| `docker compose stop`    | 停止所有服务（容器保留，可通过 `up` 重启） |
# 架构
![[Pasted image 20251102201930.jpg]]

- nsqd：服务端集群
- nsqlookupd：服务注册与发现中心
- nsqadmin：web可视化界面

# 核心概念
![[Pasted image 20251102210258.jpg]]

- topic：人为定义的消息主题
- channel：消息方定义的消息频道
- producer、consumer
![[Pasted image 20251102202925.jpg]]
channel：topic = N ：1
topic 中有新消息到达，会拷贝成多份，逐一地发送到每个channel中
consumer 发起订阅时，需要显式指定 topic + channel 的二维消息
channel 下的每一条消息会被随机推送给订阅了该 channel 的一名 cosumer

因此可见，所有 subcribe 了**相同 channel 的 consumer 之间自动形成了一种类似消费者组的机制**，大家各自消费 topic 下数据的一部分，形成数据分治与负载均衡
如果想要消费全量数据，只需要 subcribe 一个不与其他人共享的 channel

nsq 与其他消息队列组件的另一大差异：topic 和 channel 采用懒创建
使用方无需显式执行 topic 或 channel 的创建操作
channel 由**首次针对该频道发起 subcribe 的 consumer 创建**
topic 由**首次针对该主题发起 publish 的 producer 或者 subcribe 的 consumer 创建**

# 源码解读
## 1. 连接交互
![[Pasted image 20251102214543.jpg]]
在 nsq 客户端部分，生产者和消费者，在与服务端交互时，都是通过客户端定义类 Conn 来封装表示两端之间的连接
> [!info] inFlight
> 某条消息已被客户端收到，但是还未被服务端 ACK 的状态

核心字段：
```go
type Conn struct {
    // 记录了有多少消息处于未 ack 状态
    messagesInFlight int64
    // ...


    // 互斥锁，保证临界资源一致性
    mtx sync.Mutex
    // ...


    // 真正的 tcp 连接
    conn    *net.TCPConn
    // ...
    // 连接的服务端地址
    addr    string
    // ...
    
    // 读入口
    r io.Reader
    // 写出口
    w io.Writer
    // writeLoop goroutine 用于接收用户指令的 channel
    cmdChan         chan *Command
    // write goroutine 通过此 channel 接收来自客户端的响应，发往服务端
    msgResponseChan chan *msgResponse
    // 控制 write goroutine 退出的 channel
    exitChan        chan int
    // 并发等待组，保证所有 goroutine 及时回收，conn 才能关闭
    wg        sync.WaitGroup
    // ...
}
```

### 创建连接
由客户端实际向服务端发起一笔连接，对应的方法是 Conn.Connect()，核心步骤：
- 通过 net 包向服务端发起 tcp 连接
- 将 Conn 下的 writer 和 reader 设置为这笔 tcp 连接
- **异步启动 Conn 伴生的 readLoop 和 writeLoop goroutine**，持续负责接收和发送与服务端之间的往来数据

```go
func (c *Conn) Connect() (*IdentifyResponse, error) {
    dialer := &net.Dialer{
        LocalAddr: c.config.LocalAddr,
        Timeout:   c.config.DialTimeout,
    }
    
    conn, err := dialer.Dial("tcp", c.addr)
    // ...
    c.conn = conn.(*net.TCPConn)
    c.r = conn
    c.w = conn
    // var MagicV2 = []byte("  V2")
    _, err = c.Write(MagicV2)
    // ...
    c.wg.Add(2)
    // ...
    // 接收来自服务端的响应
    go c.readLoop()
    // 发送发往服务端的请求
    go c.writeLoop()
    return resp, nil
}
```

有关 readLoop 和 writeLoop 的设计，其核心优势包括：
- **解耦发送请求与接收响应**的串行流程，能够实现更加自由的**双工通信**
- 充分利用 goroutine 和 channel 的优势，通过 **for 自旋 + select 多路复用**的方式，保证两个 **loop goroutinie 能够在监听到退出指令时熔断流程**，比如 **context 的 cancel、timeout 事件**，比如 exitChan 的关闭事件等

==Look 小徐 HTTP 标准库实现原理==
## 2. 生产者
核心字段如下：
![[Pasted image 20251102214224.jpg]]
producer 在创建好与服务端交互的 Conn后，会启动一个常驻的 runtime goroutine，负责接收来自客户端的指令，并且复用 Conn 将指令发送到服务端

```go
// 生产者
type Producer struct {
    // 生产者标识 id
    id     int64
    // 连接的 nsqd 服务器地址
    addr   string
    // 内置的客户端连接，其实现类是 3.1 小节中的 Conn. 次数声明为 producerConn interface，是为 producer 屏蔽了一些生产者无需感知的细节
    conn   producerConn
    // ...
    // 生产者 router goroutine 接收服务端响应的 channel 
    responseChan chan []byte
    // 生产者 router goroutine 接收错误的 channel
    errorChan    chan []byte
    // 生产者 router goroutine 接收生产者关闭信号的 channel
    closeChan    chan int
    // 生产者 router goroutine 接收 publish 指令的 channel
    transactionChan chan *ProducerTransaction
    // ...
    // 生产者的状态
    state           int32
    // 当前有多少 publish 指令并发执行
    concurrentProducers int32
    // 生产者 router goroutine 接收退出信号的 channel
    exitChan            chan int
    // 并发等待组，保证 producer 关闭前及时回收所有 goroutine
    wg                  sync.WaitGroup
    // 互斥锁
    guard               sync.Mutex
}
```

## Publish
生产消息的入口是 Producer.Publish()，底层会组装成一个 PUB 指令，调用 Producer.sendCommand() 发送指令
```go
// Publish 同步地将消息体发布到指定的主题，如果发布失败则返回错误
func (w *Producer) Publish(topic string, body []byte) error {
    return w.sendCommand(Publish(topic, body))
}
```

PUB指令：
```go
// Publish 创建一个新的命令以向指定主题写入消息
func Publish(topic string, body []byte) *Command {
    var params = [][]byte{[]byte(topic)}
    return &Command{[]byte("PUB"), params, body}
}
```

## 发送指令
![[Pasted image 20251103200532.jpg]]
producer 发送指令的主流程如上图所示，核心步骤包括：
- 首次发送指令时，需要调用 Producer.connect() 方法创建一笔与服务端通信的 Conn
- 通过 Producer.transactionChan 通道，将指令发送到 Producer 的守护协程 router goroutine 当中
- router goroutine 接收到指令后，会调用 Conn.WriteCommand() 方法，将指令发放服务端

```go
func (w *Producer) sendCommand(cmd *Command) error {
    doneChan := make(chan *ProducerTransaction)
    err := w.sendCommandAsync(cmd, doneChan, nil)
    // ...
    t := <-doneChan
    return t.Error
}
func (w *Producer) sendCommandAsync(cmd *Command, doneChan chan *ProducerTransaction, args []interface{}) error {
	// 跟踪我们正在处理的未完成生产者数量
	// 以便之后确保我们将它们全部清理干净...
    atomic.AddInt32(&w.concurrentProducers, 1)
    defer atomic.AddInt32(&w.concurrentProducers, -1)
    
    if atomic.LoadInt32(&w.state) != StateConnected {
        err := w.connect()
        // ...
    }
    
    t := &ProducerTransaction{
        cmd:      cmd,
        doneChan: doneChan,
        Args:     args,
    }
    
    select {
    case w.transactionChan <- t:
    case <-w.exitChan:
        return ErrStopped
    }
    
    return nil
}
```

在 Producer.connect() 方法中：
- 创建了一个 Conn 实例
- **调用 Conn.Connect() 方法**实际向服务端发起连接（这个过程中也会完成 readLoop 和 writeLoop goroutine 的启动）
- **异步启动了 router goroutine，负责持续接收指令，并通过 Conn 与服务端交互**
```go
func (w *Producer) connect() error {
    w.guard.Lock()
    defer w.guard.Unlock()
    
    // ...
    w.conn = NewConn(w.addr, &w.config, &producerConnDelegate{w})
    // ...
    // producer 创建和 nsqd 之间的连接
    _, err := w.conn.Connect()
    // ...
    atomic.StoreInt32(&w.state, StateConnected)
    w.closeChan = make(chan int)
    w.wg.Add(1)
    go w.router()
    
    return nil
}
```

在 router goroutine 的运行框架中，通过 for + select 的形式，持续接收来自 Producer.transactionChan 通道的指令，将其发送到服务端

## 接收指令
![[Pasted image 20251103203227.jpg]]

1. 读取来自服务端的响应，通过调用 Producer.onConnResponse() 方法，将数据发送到 Producer.responseChan 当中
```go
func (c *Conn) readLoop() {
    delegate := &connMessageDelegate{c}
    for {
        // ...


        frameType, data, err := ReadUnpackedResponse(c, c.config.MaxMsgSize)
        // ...


        switch frameType {
        case FrameTypeResponse:
            c.delegate.OnResponse(c, data)
        // ...
        }
    }
    // ...
}
func (d *producerConnDelegate) OnResponse(c *Conn, data []byte)       { d.w.onConnResponse(c, data) }
func (w *Producer) onConnResponse(c *Conn, data []byte) { w.responseChan <- data }
```

2. 而在 connect() 异步启动的 router goroutine 通过 Producer.responseChan 接收到响应数据，通过调用 Producer.popTransaction(...) 方法，将响应推送到 doneChan 当中
```go
func (w *Producer) router() {
    for {
        select {
        // ...
        case data := <-w.responseChan:
            w.popTransaction(FrameTypeResponse, data)
        case data := <-w.errorChan:
            w.popTransaction(FrameTypeError, data)
        // ...
    }
    // ...
}

func (w *Producer) popTransaction(frameType int32, data []byte) {
    // ...
    t := w.transactions[0]
    // ...
    t.finish()
}

func (t *ProducerTransaction) finish() {
    if t.doneChan != nil {
        t.doneChan <- t
    }
}
```

3. 客户端通过在 Producer.sendCommand 方法中阻塞等待来自 doneChan 的数据，接收到后将其中的错误返回给上层
```go
func (w *Producer) sendCommand(cmd *Command) error {
    doneChan := make(chan *ProducerTransaction)
    err := w.sendCommandAsync(cmd, doneChan, nil)
    // ...
    t := <-doneChan
    return t.Error
}
```

# 3. 消费者
