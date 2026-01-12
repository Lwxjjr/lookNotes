简单来说：`context`就是在不同的`goroutine`之间同步请求特定的数据、取消信号以及处
理请求的截止日期
`context`的常见作用：
1. 并发的协调控制
2. 数据存储的介质

```go
type Context interface {
	Deadline (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```

# context使用
## 创建context
`context`包主要提供了两种方式创建`context`:
- `context.Backgroud()`
- `context.TODO()`

具体实践依靠`context`包提供的`with`系列函数来进行派生
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key, val interface{}) Context
```

![[1460000040917754.webp]]

## WithValue携带数据
日常业务开发希望能有一个`trace_id`==能串联同一请求所有的日志==，这需要我们打印日志时能够获取这个`trace_id`，我们可以使用`Context`来传递，使用`WithValue`来创建一个携带`trace_id`的`context`，然后不断透传下去，打印日志时输出即可
```go
const (
    KEY = "trace_id"
)

func NewRequestID() string {
    return strings.Replace(uuid.New().String(), "-", "", -1)
}

func NewContextWithTraceID() context.Context {
    ctx := context.WithValue(context.Background(), KEY, NewRequestID())
    return ctx
}

func PrintLog(ctx context.Context, message string)  {
    fmt.Printf("%s|info|trace_id=%s|%s",time.Now().Format("2006-01-02 15:04:05") , GetContextValue(ctx, KEY), message)
}

func GetContextValue(ctx context.Context,k string)  string{
    v, ok := ctx.Value(k).(string)
    if !ok{
        return ""
    }
    return v
}

func ProcessEnter(ctx context.Context) {
    PrintLog(ctx, "Golang梦工厂")
}


func main()  {
    ProcessEnter(NewContextWithTraceID())
}
```
使用`withValue`需要注意的事项：
- 不建议使用`context`值传递关键参数，关键参数应该显示的声明出来，`context`中最好是携带签名、`trace_id`这类值
- `context`传递的数据中`key`、`value`都是`interface`类型，这种类型编译期无法确定类型，所以不是很安全，所以在类型断言时别忘了保证程序的健壮性


## 超时控制
通常健壮的程序都是要设置超时时间的，==避免因为服务端长时间响应消耗资源==
所以一些`web`框架或`rpc`框架都会采用`withTimeout`或者`withDeadline`来做超时控制
他们都会通过传入的时间来自动取消`Context`，这里要注意的是他们都会返回一个`cancelFunc`方法，通过调用这个方法可以达到提前进行取消
==`WithTimeout`将持续时间作为参数输入而不是时间对象==
本质`withTimout`内部也是调用的`WithDeadline`

## withCancel取消控制
日常业务开发中我们往往为了完成一个复杂的需求会开多个`gouroutine`去做一些事情，这就导致我们会在一次请求中开了多个`goroutine`确无法控制他们，这时我们就可以使用`withCancel`来衍生一个`context`传递到不同的`goroutine`中，当我想让这些`goroutine`停止运行，就可以调用`cancel`来进行取消

---
# context数据结构
```go
type Context interface {  
    Deadline() (deadline time.Time, ok bool)  // 返回context的过期时间
    Done() <-chan struct{}                    // 返回context中的channel
    Err() error                               // 返回错误
    Value(key any) any                        // 返回context中的对应key的值
}
```
这个接口可以被三个类继承实现：
- emptyCtx
- ValueCtx
- cancelCtx

采用匿名接口的写法，这样可以对任意实现了该接口的类型进行重写
## emptyCtx
```go
type emptyCtx int  
  
func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {  
    return  
}  
  
func (*emptyCtx) Done() <-chan struct{} {  
    return nil  
}  
  
func (*emptyCtx) Err() error {  
    return nil  
}  
  
func (*emptyCtx) Value(key any) any {  
    return   
}

func (e *emptyCtx) String() string {
    switch e {
    case background:
        return "context.Background"
    case todo:
        return "context.TODO"
    }
    return "unknown empty Context"
}
```
- emptyCtx 是一个空的 context，本质上类型为一个整型；
- Deadline 方法会返回一个公元元年时间以及 false 的 flag，标识当前 context 不存在过期时间；
- Done 方法返回一个 nil 值，用户无论往 nil 中写入或者读取数据，均会陷入阻塞；
- Err 方法返回的错误永远为 nil；
- Value 方法返回的 value 同样永远为 nil.