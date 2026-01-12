# CSP（Communicating Sequential Processes）

> [!tip] CSP的并发哲学：  
> Do not communicate by sharing memory;   
> instead, share memory by communicating.  
> 不要通过共享内存来通信，而要通过通信来实现内存共享
> CSP是 C.A.R Hoare 在 1978 年发表在 ACM 的一篇论文的主题。论文里 指出一门编程语言应该重视 input 和 output，尤其是注重并发编程

Go 的并发原则非常优秀，目标很简单：
- 尽量使用 channel；
- 把 goroutine 当作免费的资源，随便使用

---
# channel操作情况

|  操作   | nil channel | closed channel |               not nil, not closed channel               |
| :---: | :---------: | :------------: | :-----------------------------------------------------: |
| close |    panic    |     panic      |                          正常关闭                           |
| <-ch  |     阻塞      |   读到对应类型的零值    |    阻塞<br>（缓存channel为空/<br>非缓存channel没有等待的发送者）<br>正常读    |
| ch-<  |     阻塞      |     panic      | 阻塞<br>（缓存channel的buffer满/<br>非缓存channel没有等待的接收者）<br>正常写 |
- nil channel：`var ch chan int`：没有初始化，值为 nil
---
# channel的应用
1. 停止信号
2. 定时任务(等待 100 ms 后，如果 s.stopc 还没有读出数据或者被关闭，就直接结束)
```go
	select { 
		case <-time.After(100 * time.Millisecond);
		case <-s.stopc;
			return false 
	}
	```
3. 解耦生产法和消费方
	通过使用channel，生产方和消费方可以在不同速率下运行，而不会互相阻塞。生产方可以持续向channel发送数据，消费方则可以按自己的节奏从channel接收数据。当channel的缓冲区满时，生产方会被阻塞，直到消费方接收了数据腾出空间。
4. 控制并发数


---
# 从一个关闭的通道仍然能读出数据吗？
从一个有缓冲的 channel 里读数据。当 channel 被关闭，依然能读出有效值，只有当返回的 ok 为 false 时，读出的数据才是无效的

---
# channel的不便：
1. 不改变 `channel` 自身状态的情况下， 无法得知一个 `channel` 是否关闭
2. 关闭一个 `closed channel` 会导致 `panic`，如果关闭 `channel` 的一方不知道 `channel` 是否关闭就去贸然关闭 `channel` 是很危险的事情
3. 向一个 `closed channel` 发送数据会导致 `panic`

关闭channel的最本质的原则：`Don’t close (or send values to) closed Channels`
## 不那么优雅地关闭channel
1. 使用`defer-recover`机制，即使发生了panic，也有`defer-recover`兜底
2. 使用`sync.Once`来保证只关闭一次
	```go
	// 使用sync.Once来确保close(ch)只被执行一次
	var once sync.Once 
	// 定义一个关闭channel的函数
	func closeChannel() { 
		once.Do(func() {
			fmt.Println("关闭channel...")
			close(ch)
		}) 
	}
	```

## 优雅地关闭channel
根据`sender`和`receiver`的个数，分下面几种情况：
1. `sender`:`receiver` = 1 : 1
2. `sender`:`receiver` = 1 : M
3. `sender`:`receiver` = N : 1
4. `sender`:`receiver` = N : M

对于前2种只有一个`sender`的情况，直接从`sender`端关闭就好  
重点关注第3、4种情况  
### sender：receiver = N：1
> [!tip] 唯一的接收者通过关闭一个第三方充当信号的 channel，来关闭 channel

==增加一个传递关闭信号的 channel==，receiver 通过关闭信号 channel下达关闭数据 channel 的指令，当 senders 监听到关闭信号后，停止发送数据  
```go
func main() { 
    rand.Seed(time.Now().UnixNano()) 
 
    const Max = 100000 
    const NumSenders = 1000 
 
    dataCh := make(chan int, 100) 
    stopCh := make(chan struct{}) 
 
    // senders 
    for i := 0; i < NumSenders; i++ { 
        go func() { 
            for { 
                select { 
                case <- stopCh: 
                    return 
                case dataCh <- rand.Intn(Max): 
                } 
            } 
        }() 
    } 
 
    // the receiver 
    go func() { 
        for value := range dataCh { 
            if value == Max-1 { 
                fmt.Println("send stop signal to senders.") 
                close(stopCh) 
                return 
            } 
 
            fmt.Println(value) 
        } 
    }() 
 
    select { 
    case <- time.After(time.Hour): 
    } 
} 
```

需要说明的是，上面的代码并没有明确关闭 dataCh  
在 Go 语言中，对于一个 channel，如果 最终没有任何 goroutine 引用它，不管 channel 有没有被关闭，最终都会被 GC 回收  
==所谓的优雅地关闭 channel 就是不关闭 channel，让 GC 代劳==

### sender：receiver = N：M
> [!tip] 通知中间人来关闭一个额外的信号 channel，从而关闭channel

直接采取第3种解决方案，由receiver直接关 闭stopCh的话，M个receiver就会重复关闭一个channel，导致 panic

---
# 通道内存泄漏
goroutine 操作 channel 后，处于发送或接收阻塞状态，而 channel 处于满或空 的状态，一直得不到改变。同时，垃圾回收器也不会回收此类资源，进而导致 gouroutine 会一直处 于等待队列中

---
# chan
==channel是Go在语言层面提供goroutine间的通信方式==
如果需要跨进程通信，建议使用分布式系统的方法来解决  
## chan数据结构
`src/runtime/chan.go:hchan`定义`channel`的数据结构；
```go
type hchan struct {
    qcount   uint           // 当前队列中剩余元素个数
    dataqsiz uint           // 环形队列长度，即可以存放的元素个数
    buf      unsafe.Pointer // 环形队列指针
    elemsize uint16         // 每个元素的大小
    closed   uint32            // 标识关闭状态
    elemtype *_type         // 元素类型
    sendx    uint           // 队列下标，指示元素写入时存放到队列中的位置
    recvx    uint           // 队列下标，指示元素从队列的该位置读出
    recvq    waitq          // 等待读消息的goroutine队列
    sendq    waitq          // 等待写消息的goroutine队列
    lock mutex              // 互斥锁，chan不允许并发读写
}
```
从数据结构可以看出channel由==队列、类型信息、goroutine等待队列组成==
### 环形队列
chan内部实现了一个环形队列作为缓冲区，队列长度由创建chan时指定
![[Pasted image 20250504180658.png]]
- `dataqsiz`：指示了队列长度为6，即可缓存6个元素；
- `buf`：指向队列的内存，队列中还剩余两个元素；
- `qcount`：表示队列中还有两个元素；
- `sendx`：指示后续写入的数据存储的位置，取值[0, 6)
- `recvx`：指示从该位置读取数据, 取值[0, 6)

### 等待队列
从channel中读/写数据，channel缓冲区为空/满或者没有缓冲区，当前goroutine会被阻塞
被阻塞的goroutine将会被挂在channel的等待队列中，直到写/读数据才会被唤醒
![[Pasted image 20250504180723.png]]
一般来说，recvq和sendq至少一个为空，只有一种例外，就是goroutine使用select语句向channel一边写数据，一个读数据
### 类型信息
一个`channel`只能传递一种类型的值，类型信息存储在==`hchan`数据结构==
- elemtype代表类型，用于数据传递过程中的赋值
- elemsize代表类型大小，用于在buf中定位元素位置



---
# 常见用法
## 单向channel
单向channel指只能用于发送或接收数据
所谓单向channel只是对channel的一种使用限制，和C语言使用const修饰函数参数为只读是一个道理
通过形参限定函数：
- `func readChan(chanName <-chan int)`：内部只能从channel中读取数据
- `func writeChan(chanName chan<- int)`：内部只能向channel中写入数据

## select
使用select可以监控多channel，比如监控多个channel，当其中某一个channel有数据时，就从其读出数据
==从channel中读出数据的顺序是随机的==
==select的case语句读channel不会阻塞，尽管channel中没有数据==
这是==由于case语句编译后调用读channel时会明确传入不阻塞的参数==，此时读不到数据时不会将当前goroutine加入到等待队列，而是直接返回
```go
package main

import (
    "fmt"
    "time"
)

func addNumberToChan(chanName chan int) {
    for {
        chanName <- 1
        time.Sleep(1 * time.Second)
    }
}

func main() {
    var chan1 = make(chan int, 10)
    var chan2 = make(chan int, 10)

    go addNumberToChan(chan1)
    go addNumberToChan(chan2)

    for {
        select {
        case e := <- chan1 :
            fmt.Printf("Get element from chan1: %d\n", e)
        case e := <- chan2 :
            fmt.Printf("Get element from chan2: %d\n", e)
        default:
            fmt.Printf("No element in chan1 and chan2.\n")
            time.Sleep(1 * time.Second)
        }
    }
}
```
```bash
Get element from chan1: 1
Get element from chan2: 1
No element in chan1 and chan2.
Get element from chan2: 1
Get element from chan1: 1
No element in chan1 and chan2.
Get element from chan2: 1
Get element from chan1: 1
No element in chan1 and chan2
```

