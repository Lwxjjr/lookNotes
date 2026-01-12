# Map是什么
Map: 存储 `key:value` 键值对的数据结构  
主要的数据结构有：哈希查找表`Hash table`、搜索树`Search tree`   
Hash算法核心特性：对同一个关键字进行哈希计算，每次计算都是相同的值

![[Pasted image 20250330141206.png]]

处理hash碰撞的2个思路：
1. ==链表法==（go解决方法）：把发生hash冲突的key的value组成一个链表，==处理冲突简单，无堆积现象==
2. 开放地址法：碰撞发生后，以一个的规律，在数组后面挑选"空位"，来放置新的key
	1. 线性探测：按照一定的顺序（通常是依次递增地址）探测下一个空闲位置，简单直接，但容易产生 “堆积” 现象，即连续的空闲位置被占用，影响查找效率
	2. 二次探测：探测步长是地址的二次函数，即探测位置为 `h(key)+i^2`，可以在一定程度上缓解线性探测中的 ”堆积” 问题，但是可能导致某些位置永远无法探测到
	3. 双重哈希：使用两个哈希函数
3. 再哈希法：需要多个哈希函数，发生冲突时，使用另外一个哈希函数，==不易产生聚集==
4. 建立公共溢出区


# Map内存模型
map是一个指针，占用8字节，指向`hmap`结构体
表示map的结构体是==`hmap`==：
- 一个哈希表里可以有多个哈希表节点，也即`bucket`
- 每个`bucket`就保存了map中的一个或一组键值对
```go
// runtime/map_noswiss.go
type hmap struct {
	count     int                // 元素个数，调用 len(map) 时，直接返回此值
	flags     uint8
	B         uint8              // buckets 的对数 log_2
	noverflow uint16	         // overflow 的 bucket 近似数
	hash0     uint32	         // 计算 key 的哈希的时候会传入哈希函数
	buckets    unsafe.Pointer    // 指向 buckets 数组，大小为 2^B
	// 等量扩容的时候，buckets 长度和 oldbuckets 相等
	// 双倍扩容的时候，buckets 长度会是 oldbuckets 的两倍
	oldbuckets unsafe.Pointer
	// 指示扩容进度，小于此地址的 buckets 迁移完成
	nevacuate  uintptr
	extra *mapextra // optional fields
}
```

1. `hmap`存储了`buckets`用于存储所有的`bucket`
2. `bucket`：存储`key:value`的数据结构，其中key放一快，value放一块
3. 链表法：每个`bucket`下有一个`overflow`属性用于指向其他的`bucket`
4. `overflow`：指向一个溢出桶，为了减少扩容次数引入的


![[Pasted image 20250509204938.png]]

---
# buckets
在 `src/runtime/hashmap.go` 中  
buckets是一个指针，最终指向一个结构体==bmap（桶）==：
```go
type bmap struct {
	tophash [bucketCnt]uint8
}
```
在编译期间会动态的创建一个新的结构：
每个`bucket`可以存储8个键值对
```go
type bmap struct {
    topbits  [8]uint8      // 存储哈希值的高8位
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr       // 溢出bucket的地址
}
```
![[Pasted image 20250426141456.png]]

![[Pasted image 20250330140836.png|350]]


## mapextra与溢出桶
`hmap`最后有`extra`字段->`mapextra`结构体
```go
type mapextra struct {
  overflow *[]*bmap     // 记录已使用的溢出桶的地址
  oldoverflow *[]*bmap  // 扩容阶段旧桶使用的溢出桶地址
  nextOverflow *bmap    // 指向下一个空闲溢出桶地址
}
```

![[Pasted image 20250330172055.jpg]]



---
# Key的定位过程
key经过哈希计算后得到哈希值，根据操作系统得到64/32bit  
低B位：定位bucket，计算落桶位置    
高8位：定位tophash，key在bucket的位置
![[Pasted image 20250330142214.png|315]]
![[Pasted image 20250330142619.png|475]]


---
# 哈希函数的选择
程序启动时，会检测 cpu 是否支持 aes
- 如果支持，则使用 aes hash，
- 否则使用 memhash

> [!info] 这是在 `src/runtime/alg.go` 中的 `alginit()`完成
> hash 函数，有加密型和非加密型。 加密型的一般用于加密数据、数字摘要等，典型代表就是 md5、sha1、sha256、aes256 这种； 非加密型的一般就是查找。在 map 的应用场景中，用的是查找。 选择 hash 函数主要考察的是两点：性能、碰撞概率。

根据key的类型，`_type` 结构体的`alg`字段会被设置成对应类型的`hash`和`equal`函数

![[Pasted image 20250330144044.png|650]]

---
# Map的扩容机制
## 扩容（装载因子）
==负载因子用于衡量一个哈希表冲突情况==
```bash
装载因子 = 元素数量 / bucket数量
```
==在装载因子 > 6.5触发2倍扩容（这是因为bucket的数量确定方式是以B的）==  
触发扩容的前提：
1. 负载因子 > 6.5，即平均每个`bucket`存储的键值对达到6.5个
2. overflow数量 > $2^{15}$

> [!info] 确定B
> 负载因子定位6.5，即平均每个bucket保存的元素≥6.5个时会触发扩容
> 所以，在初始化map时，元素个数确定的情况下，计算B值，就转变成至少分配几个bucket，能使当前的负载因子不超过6.5
> 再根据buckets数组的个数=2^B次方计算可得B值。

在源码中的makemap函数可以确定B的计算：
```go
B := uint8(0)
for overLoadFactor(hint, B) {
    B++
}
h.B = B
```

而 overLoadFactor函数如下:
```go
func overLoadFactor(count int, B uint8) bool {  
    return count > abi.OldMapBucketCount && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)  
}
```
是否增大B：
1. 检查count是否大于桶数量的阈值（8）
2. 检查是否大于负载因子（count > 负载因子 \* 当前桶数量）（13 x bucketShift(B) / 2

bucketShift函数的实现:
```go
func bucketShift(b uint8) uintptr {
    return uintptr(1) << (b && (sy 1.PtrSize*8 -1))
}
```


## 渐进式扩容
>[!example] 注意问题：
 假如我们在之前添加的2条key的hash后B位一样，扩容后不一定一样，由于B值的大小变化，此时他们就不会在同一个bucket里面了，但是重新计算大量的key/value的搬迁是比较消耗性能的，golang采取了一种==渐进式的迁移方式，即每次只迁移 2 个 bucket==  

> [!question] 什么时候会触发某个 bucket 的迁移呢？

插入或修改、删除 key 的时候，都会尝试进行搬迁 buckets 的工作，直到我们把所有的 bucket 迁移完  
旧的桶和新的桶同时存在，并通过指针标记来逐步迁移键值对  
当扩容开始时，会先将旧的 bucket 挂到 `oldbuckets` 上，再在 `buckets` 字段指向新的扩容之后的 bucket, 当所有的 bucket 都迁移到新的 bucket 上的时候，oldbuckets 的值就会变成 nil

## 等量扩容
因为 map 在删除一个键时，并不会真正从内存中 “移除” 这个键值对，而是：
1. tophash 对应位置标记为 "空"
2. 不改变桶（`bmap`）和溢出桶的链表结构
因此导致就算**桶内有空位，也可能无法复用，只能往溢出桶里塞**，最终导致溢出桶数量越来越多
虽然没有超过负载因子限制，但是==使用溢出桶过多==，就会触发**等量扩容**  
==一般发生在很多键值对被删除的情况下，这样会造成overflow的bucket数量增多，但负载因子又不高==  
这种情况下，B的大小不变，我们不需要对bucket的某个key重新计算，只需要把key搬迁到相同序号的bucket即可，这样==闲置率就下去了，查询效率就高起来了==

---
# 遍历
由于缩扩容的原因，map的key所在的bucket会一直改变  
因此如果我们对 map 进行遍历，那么我们得到的 key 的顺序和我们的插入顺序是不一致的  
> [!注意]
> **go 的设计者为了避免开发者误认为 map 的遍历是有序的，所以每次遍历 map 的时候会随机从某个 bucket 开始**

```go
r := uintptr(fastrand()) 
if h.B > 31-bucketCntBits {
	r += uintptr(fastrand()) << 31 
} 
// 从哪个 bucket 开始遍历 
it.startBucket = r & (uintptr(1)<<h.B - 1) 
// 从 bucket 的哪个 cell 开始遍历 
it.offset = uint8(r >> h.B & (bucketCnt - 1))
```


---
# Map的赋值（mapassign）
![[Pasted image 20250331191017.png]]
核心仍然是双层循环：  
==外层遍历`bucket`和`overflow bucket`，内层遍历单个bucket`的所有槽位`==  
## 需要注意的点：
1. 首先会检查`flags`写标志位，`flags = 1`说明有其他协程正在执行"写"操作，由于`assign`本身也是写操作，因此产生了并发写，直接使程序panic，==说明map不是协程安全的==
	```go
	package main
	
	import (  
		"fmt"  
		"sync"  
	)
	
	var m = make(map[int]string)
	var wg sync.WaitGroup
	
	func main() {    
		const numGoroutines = 10  
		wg.Add(numGoroutines)  
	  
		for i := 0; i < numGoroutines; i++ {    
		    go func() {  
		        defer wg.Done()    
			        for j := 0; j < 10; j++ {  
			            key := j   
			            value := fmt.Sprintf("value_%d", j)  
			            m[key] = value
			        }  
			    }()  
		}  
		wg.Wait()   
		fmt.Println("Final map content:")  
		for k, v := range m {  
		    fmt.Printf("Key: %d, Value: %s\n", k, v)  
		}  
	}
	```
2. 如果map 处在扩容的过程中，那么当定位key到了某个bucket 后，需要 确保这个bucket 对应的老 bucket 已经完成了迁移过程，==只有在完成了迁移操作后==，才能安全地在新 bucket 里定位 key 要安置的地址，再进行之后的 赋值操作

---
# map的key可以是什么值
Go 语言中只要是==可比较的类型==都可以作为 key，除了slice、map、functions
## float类型
float 型可以作为 key，但是由于精度的问题，会导致一些诡异的问题出现，故慎用之  
因为float64作为key的时候，要先将其转换为uint64，再插入key中

---
# 比较2个map
两个map深度相等的条件如下（3选1）：
1. 都为nil
2. 非空，长度相等，指向同一个 map 实体对象
3. 相应的 key 指向的 value “深度”相等

---
# make/new
都是内存分配的内建函数
1. 适用类型不同
	- make：slice、map、channel等==引用类型==，初始化并分配内存
	- new: int型、数组、结构体等==值类型==，分配内存但不初始化
2. 调用形式不同：
	- func make(t Type, size ...IntegerType) Type
	- func new(Type) \*Type

# slice 和 map 分别作为函数参数时有什么区别？
1. makemap函数返回的结果是 \*hmap
2. makeslice函数返回的结果 silce

所以，在函数内部对 map 的操作会影响 map 结构体；而对 slice 操作却不会


---
# Map的并发安全问题
==首先Map就不是并发安全的==
> [!tip] Go官方这么说的：
> Map access is unsafe only when updates are occurring. As long as all goroutines are only reading—looking up elements in the map, including iterating through it using a for range loop—and not changing the map by assigning to elements or doing deletions, it is safe for them to access the map concurrently without synchronization.

考虑到存在性能损失，官方没有将`map`设计成原子操作，所以在并发读写会有问题
`runtime/map.go`中的并发检测部分：
```go
const (
    hashWriting  = 4 // a goroutine is writing to the map
)

// A header for a Go map.
type hmap struct {
    flags     uint8
}

func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // ...
    // 往map写数据时，会将flags的hashWriting位置为1
    h.flags ^= hashWriting
    // ...
    // 最后清掉hashWriting位的1
    h.flags &^= hashWriting
}

// 各个读取的地方
if h.flags&hashWriting != 0 {
    fatal("concurrent map read and map write")
}
```
## 解决方案
#### 1. 加锁（sync.RWMutex)
我们对`map`的几个常用操作进行了封装，加了读写锁。这样  就可以在并发环境下安全的使用`map`了
```go
type RWMap struct {
	sync.RWMutex
	m map[string]int // 基于封装的思路，我们不希望能被外部的使用方直接的使用到
}

func (m *RWMap) Get(key string) (int, bool) {
    // 读锁保护
	m.RLock()
	defer m.RUnlock()
	val, ok := m.m[key]
	return val, ok
}

func (m *RWMap) Set(key string, val int) {
    // 写锁保护
	m.Lock()
	defer m.Unlock()
	m.m[key] = val
}

func (m *RWMap) Del(key string) {
    // 写锁保护
	m.Lock()
	defer m.Unlock()
	delete(m.m, key)
}

func (m *RWMap) Len() int {
    // 读锁保护
	m.RLock()
	defer m.RUnlock()
	return len(m.m)
}

func (m *RWMap) Range(f func(key string, val int) bool) {
    // 读锁保护
	m.RLock()
	defer m.RUnlock()

	for k, v := range m.m {
		if !f(k, v) {
			break
		}
	}
}
```

#### 2. sync.Map
`golang`官方提供了一个并发安全的`map`实现：`sync.Map`
```go
package sync // import "sync"

func (m *Map) Store(key, value any)
    将键值对存储到Map中

func (m *Map) Load(key any) (value any, ok bool)
    第一个返回值是key对应的value，第二个返回值表示是否存在这个key
```

```go
func main() {
	sm := sync.Map{}
	wg := sync.WaitGroup{}

	// 100个goroutine并发写入sync.Map
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			sm.Store(i, i)
		}(i)
	}

	// 100个goroutine并发读取sync.Map
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			if value, ok := sm.Load(i); ok {
				fmt.Println(value)
			}
		}(i)
	}

	wg.Wait()
}
```

#### 3. 分片锁
- 加个读写锁很简单，但是在==高并发的情况下，读写锁很可能会成为性能瓶颈==
- `sync.Map`虽然并发安全，但是它的适用场景有限。只适合读多写少的场景

> [!question] 官方的FAQ里面：
> As an aid to correct map use, some implementations of the language contain a special check that automatically reports at run time when a map is modified unsafely by concurrent execution. Also there is a type in the sync library called sync.Map that works well for certain usage patterns such as static caches, although it is not suitable as a general replacement for the builtin map type.

可见如果对性能敏感的话，`sync.Map`也不是很好的选择
在`golang`中，必定会有大量并发请求，所以还是要考虑锁，如何提升锁的效率？
1. 降低锁的持有时间
2. 减少锁的粒度

针对1，这是业务代码需要考虑的事情
针对2，减少锁的粒度
分片锁/分段锁就是减少锁的粒度一种思路。==它将一把锁拆分成了多个小的锁，每个小锁负责一部分的数据==。这样在读写时，只需要锁住对应的小锁，而不是整个大锁。从而减少了锁的粒度

![[Pasted image 20250426235907.png|322]]

分片锁的`map`实现库：[concurrent-map](https://github.com/orcaman/concurrent-map)
本质上通过将`map`拆分成多个`shard`(根据对key进行某种算法拆分)，每个`shard`有自己的读写锁。这样在读写时，只需要锁住对应的`shard`，而不是整个`map`