# net/http
## 1. 架构
http协议下，交互框架是由客户端和服务端组成C-S架构
![[Pasted image 20251110230556.jpg]]
启动服务:
```go
func main() {
	http.HandleFunc("/ping", func(w http.ResponseWriter, r-*http.Request) {
		w.Write([]byte("pong"))
	}) 
	http.ListenAndServe(":8091", nil) }
```


## 2. 服务端
### 2.1 数据结构
- Server
- Handler
- ServeMux

**Server**：整个 Http 服务端模块被封装在里面
Handler 是 Server 中最核心的成员字段
实现了从请求路径 path 到具体处理方法 handler 的注册和映射能力
> [!tip] 倘若其中的 Handler 字段未显式声明，则会取 net/http 包下的单例对象 DefaultServeMux（ServerMux 类型） 进行兜底

```go
type Server struct {
    // server 的地址
    Addr string
    // 路由器.
    Handler Handler // handler to invoke,http.DefaultServeMux if nil
    // ...
}
```

**Handler**：根据 http 请求 Request 中的请求路径 path 映射的 handler 处理函数，对请求进行处理和响应
```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

**ServeMux**：对 Handler 的具体实现，内部通过一个 map 维护了从 path 到 handler 的映射关系
```go
type ServeMux struct {
    mu sync.RWMutex
    m map[string]muxEntry
    es []muxEntry // slice of entries sorted from longest to shortest.
    hosts bool // whether any patterns contain hostnames
}
```

**muxEntry**：handler 单元（请求路径 + 处理函数）
```go
type muxEntry struct {
    h Handler
    pattern string 
}
```

### 2.2 注册 handler

![[Pasted image 20251111131335.jpg]]
```go
var DefaultServeMux = &defaultServeMux

var defaultServeMux ServeMux
```

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    DefaultServeMux.HandleFunc(pattern, handler)
}
```

在 ServeMux.HandleFunc 内部会将处理函数 handler 转为实现了 ServeHTTP 方法的 HandlerFunc 类型，将其作为 Handler interface 的实现类注册到 SeverMux 的路由 map 中
```go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    // ...
    mux.Handle(pattern, HandlerFunc(handler))
}

type HandlerFUnc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

核心逻辑注册 ServeMux.Handle 中
1. path 和 handler 包装成一个 muxEntry
2. 响应模糊匹配机制，以 '/' 结尾的 path，根据 path 长度将 muxEntry 有序插入到数组 ServeMux.es 中
```go
func (mux *ServeMux) Handle(pattern string, handler Handler) {
    mux.mu.Lock()
    defer mux.mu.Unlock()
    // ...
    
    // key: path
    e := muxEntry{h: handler, pattern: pattern}
    mux.m[pattern] = e
    if pattern[len(pattern)-1] == '/' {
        mux.es = appendSorted(mux.es, e)
    }
    // ...
}
```

```go
func appendSorted(es []muxEntry, e muxEntry) []muxEntry {
    n := len(es)
    i := sort.Search(n, func(i int) bool {
        return len(es[i].pattern) < len(e.pattern)
    })
    if i == n {
        return append(es, e)
    }
    es = append(es, muxEntry{}) // try to grow the slice in place, any entry works.
    copy(es[i+1:], es[i:])      // Move shorter entries down
    es[i] = e
    return es
}
```

### 2.3 启动 server
![[Pasted image 20251111134235.jpg]]
ListenAndServe 对服务端一键启动
```go
func ListenAndServe(addr string, handler Handler) error {
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}
```

根据用户传递的端口，申请到一个监听器 Listener
```go
func (srv *Server) ListenAndServe() error {
    // ...
    addr := srv.Addr
    if addr == "" {
        addr = ":http"
    }
    ln, err := net.Listen("tcp", addr)
    // ...    
    return srv.Serve(ln)
}
```

![[Pasted image 20251111134458.jpg]]
Server.Serve 的运行架构：for + listener.accept 模式
1. 将 serve 封装成一组 kv，添加到 context 中
2. for 循环，调用 Listener.Accept 方法阻塞等待新连接到达
3. 每有一个连接到达，创建一个 goroutine 异步执行 conn.serve

```go
var ServerContextKey = &contextKey{"http-server"}


type contextKey struct {
    name string
}


func (srv *Server) Serve(l net.Listener) error {
   // ...
   ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    for {
        rw, err := l.Accept()
        // ...
        connCtx := ctx
        // ...
        c := srv.newConn(rw)
        // ...
        go c.serve(connCtx)
    }
}
```

conn.serve 是响应客户端连接的核心方法
- 从 conn 中读取到封装到 response 结构体，以及请求参数 http.Request
- 调用 serveHandler.ServeHTTP 方法，根据请求的 path 为其分配handler
- 通过特定 handler 处理并响应请求

```go
func (c *conn) serve(ctx context.Context) {
    // ...
    c.r = &connReader{conn: c}
    c.bufr = newBufioReader(c.r)
    c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)


    for {
        w, err := c.readRequest(ctx)
        // ...
        serverHandler{c.server}.ServeHTTP(w, w.req)
        w.cancelCtx()
        // ...
    }
}
```

在 serveHandler.ServeHTTP 方法中，会对 Handler 作判断，==倘若其未声明，则取全局单例 DefaultServeMux 进行路由匹配，呼应了 http.HandleFunc 中的处理细节.==

```go
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }
    // ...
    handler.ServeHTTP(rw, req)
}
```








# Gin 核心特点
高性能、易用性、功能丰富
1. 高性能路由引擎（基数树）：Gin 的路由引擎的底层数据结构是基数树，路由匹配的时间复杂度是 O(k)，k是路由长度，与路由数量无关
2. 简洁的 API 设计（链式调用）
3. 丰富的中间件生态
4. 参数绑定（反射机制和标签系统）
# Gin架构
![[Pasted image 20251110225215.png]]
![[Pasted image 20251110225533.jpg]]
