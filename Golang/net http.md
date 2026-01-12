# http处理
Go 实现 Web 服务的工作模式
![[Pasted image 20251030211546.png]]
1. 创建Listen Socket, 监听指定的端口, 等待客户端请求到来。
2. Listen Socket接受客户端的请求, 得到Client Socket, 接下来通过Client Socket与客户端通信。
3. 处理客户端的请求, 首先从Client Socket读取HTTP请求的协议头, 如果是POST方法, 还可能要读取客户端提交的数据, 然后交给相应的handler处理请求, handler处理完毕准备好客户端需要的数据, 通过Client Socket写给客户端。

```go
func (srv *Server) Serve(l net.Listener) error {
    defer l.Close()
    var tempDelay time.Duration // how long to sleep on accept failure
    for {
        rw, e := l.Accept()
        if e != nil {
            if ne, ok := e.(net.Error); ok && ne.Temporary() {
                if tempDelay == 0 {
                    tempDelay = 5 * time.Millisecond
                } else {
                    tempDelay *= 2
                }
                if max := 1 * time.Second; tempDelay > max {
                    tempDelay = max
                }
                log.Printf("http: Accept error: %v; retrying in %v", e, tempDelay)
                time.Sleep(tempDelay)
                continue
            }
            return e
        }
        tempDelay = 0
        c, err := srv.newConn(rw)
        if err != nil {
            continue
        }
        go c.serve()
    }
}
```
![[Pasted image 20251030211802.png]]
1. `net.Listen("tcp", addr)` 是底层网络监听的核心机制，类似 Socket 解释中的 `BIND` 和 `LISTEN`，服务器现在已经在网络上宣告 "我在这里等着接收连接了！"
2. `rw := l.Accept()` 核心网络操作，类似 Socket 解释中的 `accept` 步骤。阻塞直到有新的客户端连接到来。一旦有客户端尝试连接，`Accept()` 方法就会返回一个 `net.Conn` 接口
3. 创建一个 `Conn`，单独开一个 `goroutine` 来处理客户端连接
4. `c.readRequest` 从 `net.Conn` 读取原始的字节流解析成HTTP请求的结构体，解析出 `http.Request` 和 `http.ResponseWritter`
# Conn的goroutine
Go为了实现高并发和高性能, 使用了goroutines来处理Conn的读写事件, 这样每个请求都能保持独立，相互不会阻塞，可以高效的响应网络事件
```go
c, err := srv.newConn(rw)
if err != nil {
    continue
}
go c.serve()
```
Conn 保存这次请求的信息，然后传递到对应的 handler，该handler中便可以读取到相应的header信息

# ServeMux
`conn.server` 实际内部调用了 `http` 包默认的路由器，通过路由器把本次请求的信息传递到了后端的处理函数
```go
type ServeMux struct {
    mu sync.RWMutex   //锁，由于请求涉及到并发处理，因此这里需要一个锁机制
    m  map[string]muxEntry  // 路由规则，一个string对应一个mux实体，这里的string就是注册的路由表达式
    hosts bool // 是否在任意的规则中带有host信息
}
```

`muxEntry`：
```go
type muxEntry struct {
    explicit bool   // 是否精确匹配
    h        Handler // 这个路由表达式对应哪个handler
    pattern  string  //匹配字符串
}
```

`Handler`：
```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)  // 路由实现器
}
```

假定`handler`是`sayhelloName`
`HandlerFunc` 会
```go
1. http.HandleFunc("/", sayhelloName)
   ↓
   DefaultServeMux.HandleFunc("/", sayhelloName)
   ↓
   DefaultServeMux.Handle("/", http.HandlerFunc(sayhelloName))  // sayhelloName 被包装成 http.Handler 接口

2. http.ListenAndServe(":9090", nil)
   ↓
   创建一个 http.Server 结构体，其 Handler 字段被设置为 nil
   ↓
   http.Server.ListenAndServe() 方法内部检测到 Handler 为 nil
   ↓
   将 http.Server 的 Handler 字段设置为 DefaultServeMux

3. http.Server 启动监听并接受连接
   ↓
   为每个新连接启动一个 goroutine
   ↓
   在 goroutine 中，调用 http.Server.Handler.ServeHTTP(w, r)
   ↓ (此时 http.Server.Handler 就是 DefaultServeMux)
   DefaultServeMux.ServeHTTP(w, r)
   ↓
   DefaultServeMux 根据请求路径 "/" 查找匹配的 Handler
   ↓
   找到之前注册的 http.HandlerFunc(sayhelloName)
   ↓
   调用 http.HandlerFunc(sayhelloName).ServeHTTP(w, r)
   ↓
   执行原始的 sayhelloName(w, r) 函数
```


# Socket编程


# gin
![[Pasted image 20251105110805.png]]