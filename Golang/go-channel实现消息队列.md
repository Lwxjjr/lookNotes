`MQ（Message Queue）`

> [!没有mq]
> 邮件发送在请求登陆时进行，当密码验证成功后，就发送邮件，然后返回登陆成功
> 这样的缺陷：
> 1. 邮件发送出现问题，整个登录请求也出现错误，导致登录不成功（耦合太高）
> 2. 调用登录接口只需要`100ms`，但是需要做一次中间的发送邮件的等待，那么调用一次登录接口的时间就要增长（邮件的优先级不高）
> ![[Pasted image 20250429215214.png|450]]

> [!使用mq]
> 异步调用
> 我们将发送邮件请求放到`MQ`中，这样我们就能提高用户体验的吞吐量
> ![[Pasted image 20250429215240.png|450]]

# 实现
内部方法，需要在封装一层，客户端能直接调用的方法
```go
// broker.go
type Broker interface {
	publish(topic string, msg interface{}) error
	subscribe(topic string) (<-chan interface{}, error)
	unsubscribe(topic string, sub <-chan interface{}) error
	close()
	broadcast(msg interface{}, subscribers []chan interface{})
	setConditions(capacity int)
}
```
- `publish`：进行消息的推送，有两个参数即`topic`、`msg`，分别是订阅的主题、要传递的消息
- `subscribe`：消息的订阅，传入订阅的主题，即可完成订阅，并返回对应的`channel`通道用来接收数据
- `unsubscribe`：取消订阅，传入订阅的主题和对应的通道
- `close`：关闭消息队列的
- `broadCast`：这个属于内部方法，作用是进行广播，对推送的消息进行广播，保证每一个订阅者都可以收到
- `setConditions`：用来设置条件，条件是消息队列的容量，可以控制消息队列的大小

# 封装
```go
// client.go
type Client struct {
	bro *BrokerImpl
}

func NewClient() *Client {
	return &Client{
		bro: NewBroker(),
	}
}

func (c *Client)SetConditions(capacity int)  {
	c.bro.setConditions(capacity)
}

func (c *Client)Publish(topic string, msg interface{}) error{
	return c.bro.publish(topic,msg)
}

func (c *Client)Subscribe(topic string) (<-chan interface{}, error){
	return c.bro.subscribe(topic)
}

func (c *Client)Unsubscribe(topic string, sub <-chan interface{}) error {
	return c.bro.unsubscribe(topic,sub)
}

func (c *Client)Close()  {
	 c.bro.close()
}

// 获取订阅消息
func (c *Client)GetPayLoad(sub <-chan interface{}) interface{}{
	for val := range sub{
		if val != nil{
			return val
		}
	}
	return nil
}
```

# 消息队列实现结构
```go
type BrokerImpl struct {
	exit chan bool
	capacity int
	topics map[string][]chan interface{} // key： topic  value ： queue
	sync.RWMutex // 同步锁
}
```
- `exit`：控制消息队列的关闭
- `capacity`：设置消息队列的容量
- `topics`：存储订阅关系，键是 `topic`，值是订阅者的 channel 列表，==原因是我们一个topic可以有多个订阅者，所以一个订阅者对应着一个通道==
- `sync.RWMutex`：读写锁，这里是为了防止并发情况下，数据的推送出现错误，所以采用加锁的方式进行保证

# 具体方法
## close
关闭消息队列，`b.topics = make(map[string][]chan interface{})`是为了保证下一次使用该消息队列不发生冲突
```go
func (b *BrokerImpl) close()  {
	select {
	case <-b.exit:
		return
	default:
		close(b.exit)
		b.Lock()
		b.topics = make(map[string][]chan interface{})
		b.Unlock()
	}
	return
}
```

## 设置容量
```go
func (b *BrokerImpl)setConditions(capacity int)  {
	b.capacity = capacity
}
```

## publish
实现的思路很简单，只需要把传入的数据进行广播即可
通过`b.topics[topic]`获取订阅者列表
再通过`broadcast`推送信息
```go
// broker.go
func (b *BrokerImpl) publish(topic string, msg interface{}) error { 
	select {  
	case <-b.exit:  
	return errors.New("broker closed")  
	default:  
	}
	
	b.RLock()  
	subscribers, ok := b.topics[topic]  
	b.RUnlock()  
	if !ok {  
	    return nil  
	}  
	b.broadcast(msg, subscribers)  
	return nil  
}
```
## broadcast
这里是对数据进行广播，所以数据推送出去就可以了，没必要一直等待他推送成功过，所以这里我们采用`goroutine`，推送超过`5ms`就停止推送，进行下面的推送
通过`switch`计算订阅者数量和设置并发度
```go
// broker.go
func (b *BrokerImpl) broadcast(msg interface{}, subscribers []chan interface{}) {  
    count := len(subscribers)  
    concurrency := 1  

    switch {  
    case count > 1000:  
        concurrency = 3  
    case count > 100:  
        concurrency = 2  
    default:  
        concurrency = 1  
    }  

    pub := func(start int) {  
        idleDuration := 5 * time.Millisecond  
        idleTimeout := time.NewTimer(idleDuration)  
        defer idleTimeout.Stop()  
        for j := start; j < count; j += concurrency {  
            if !idleTimeout.Stop() {  
                select {  
                case <-idleTimeout.C:  
                default:  
                }  
            }  
            idleTimeout.Reset(idleDuration)  
            select {  
            case subscribers[j] <- msg:  
            case <-idleTimeout.C:  
            case <-b.exit:  
                return  
            }  
        }  
    }  

    for i := 0; i < concurrency; i++ {  
        go pub(i)  
    }  
}  
```

## subscribe
为订阅的主题创建一个`channel`，然后将订阅者加入对应的`topic`中，返回一个接收`channel`
```go
func (b *BrokerImpl) subscribe(topic string) (<-chan interface{}, error) {
	select {
	case <-b.exit:
		return nil, errors.New("broker closed")
	default:
	}

	ch := make(chan interface{}, b.capacity)
	b.Lock()
	b.topics[topic] = append(b.topics[topic], ch)
	b.Unlock()
	return ch, nil
}
```

## unsubscribe
```go
func (b *BrokerImpl) unsubscribe(topic string, sub <-chan interface{}) error {
	select {
	case <-b.exit:
		return errors.New("broker closed")
	default:
	}

	b.RLock()
	subscribers, ok := b.topics[topic]
	b.RUnlock()

	if !ok {
		return nil
	}
	// delete subscriber
	b.Lock()
	var newSubs []chan interface{}
	for _, subscriber := range subscribers {
		if subscriber == sub {
			continue
		}
		newSubs = append(newSubs, subscriber)
	}

	b.topics[topic] = newSubs
	b.Unlock()
	return nil
}
```