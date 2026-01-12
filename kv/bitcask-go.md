# 内存和磁盘设计
首先需要明确：
磁盘文件 `logRecord` 记录 `Key + Val + Type`
内存数据 `item` 记录 `key + fileID + offset`
因此可以得知取数据的流程是：`key -> fileID -> offset -> val`
内存设计引擎可以选择的数据结构有`BTree、BpTree、ART`
所以我们可以提供一个通用的抽象接口，用于接入不同的数据结构
```go
type Indexer interface {
	Put(key []byte, pos *data.LogRecordPos) bool
	Get(key []byte) *data.LogRecordPos
	Delete(key []byte) bool
}
```
我们抽离3个模块：
- `fio`：负责底层文件操作
- `data`：管理日志数据文件
- `index`：管理内存索引

## data
| 对比项  | DataFile（数据文件）       | LogRecord（日志记录） |
| ---- | -------------------- | --------------- |
| 定位   | 物理存储层                | 逻辑数据层           |
| 数据形式 | 字节流                  | 结构体             |
| 访问方式 | 按偏移量                 | 按字段解析           |
| 生命周期 | 持久化                  | 临时              |
| 错误处理 | IO 错误                | 数据损坏（CRC）       |
| 协作方式 | 存储 LogRecord 编码后的字节流 | 编码/解码逻辑数据       |

---
# 读写流程
涉及读写流程，设置 `bitcask` 存储引擎实例`db`协调 `fio、data、index`模块的工作
`bitcask` 存储引擎实例存放：
- `options`：数据库配置项
- `activeFile`：当前活跃文件
- `olderFiles`：旧的数据文件
- `index`：抽象索引接口

`bitcask`写数据：先写磁盘数据文件，再更新内存索引
追加写的逻辑：
```c
if 当前活跃文件 == null {
	初始化活跃文件
}

if 活跃文件 >= 文件阈值 {
	Sync 活跃文件
	打开新的活跃文件
}

向活跃文件中写记录
return 索引位置
```

>[!question] 需要注意
>1. 写入数据从当前活跃文件的偏移开始写，这是构造内存索引信息的 `offset`
>2. 需要根据配置项决定是否持久化，因为对于标准文件 `IO` 而言向文件写数据实际上会先写到内存缓冲区，并且等待操作系统进行调度，将其刷到磁盘当中

---
# 数据库启动流程

---
# Options
```go
type Options struct {
	DirPath            string      // 数据文件存储目录
	DataFileSize       int64       // 数据文件大小阈值
	IndexType          IndexerType // 索引类型
	SyncWrites         bool        // 是否每次同步写入数据文件
	BytesPerSync       int64       // 累计写入数据量，达到阈值时，同步数据文件
	MMapStartup        bool        // 启动时是否使用 MMap
	DataFileMergeRatio float64     // 数据文件合并的阈值
}
```
