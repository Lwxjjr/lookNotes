## FIFO（First In First Out）
先进先出，FIFO 认为，最早添加的记录，其不再被使用的可能性比刚添加的可能性大
FIFO的实现：创建队列，新增记录添加到队尾，内存不够，淘汰队首
缺点：
- 缓存命中率低
## LFU（Least Frequently Used）
最少使用，也就是淘汰缓存中访问频率最低的记录
LFU 认为，如果数据过去被访问多次，那么将来被访问的频率也更高
LFU 的实现需要维护一个==按照访问次数排序的队列==
LFU算法的命中率是比较高的
缺点：
- 维护每个记录的访问次数，对==内存的消耗==是很高的
- ==受历史数据的影响比较大==，如历史上访问次数奇高，但是在某个时间点之后不再被访问，却迟迟不能淘汰

## LRU（Least Recently Used）
最近最少使用，LRU 认为，如果数据最近被访问过，那么将来被访问的概率也会更高
LRU的实现维护一个队列，某条记录被访问了，就移动到队尾，队首即最近最少访问的数据

## LRU实现
### 数据结构
- `map`：存储kv的映射关系，==字典查找/插入复杂度为 `O(1)`==
- `double linked list`：把值放入双向链表，==链表移动/插入/删除数据复杂度为 `O(1)`==


![[Pasted image 20250427234631.jpg]]

```go
package lru  
  
import "container/list"  
  
// Cache is a LRU cache. It is not safe for concurrent access.  
type Cache struct {  
	maxBytes int64  
	nbytes   int64  
	ll       *list.List  
	cache    map[string]*list.Element  
	// optional and executed when an entry is purged.  
	OnEvicted func(key string, value Value)  
}  
  
type entry struct {  
	key   string  
	value Value  
}  
  
// Value use Len to count how many bytes it takes  
type Value interface {  
	Len() int  
}
```
- 双向链表`list.list`
- 字典`map[string] *list.Element`，值是双向链表中对应节点的指针
- `maxBytes` 是允许使用的最大内存
- `nbytes` 是当前已使用的内存
- `OnEvicted` 是某条记录被移除时的回调函数，可以为 nil
- `entry`：双向链表节点的数据类型

### 实例化
```go
// New is the Constructor of Cache  
func New(maxBytes int64, onEvicted func(string, Value)) *Cache {  
	return &Cache{  
		maxBytes:  maxBytes,  
		ll:        list.New(),  
		cache:     make(map[string]*list.Element),  
		OnEvicted: onEvicted,  
	}  
}
```

### 查找
1. 从字典中找到对应的双向链表的节点
2. 把该节点移动到队尾
```go
// Get look ups a key's value  
func (c *Cache) Get(key string) (value Value, ok bool) {  
	if ele, ok := c.cache[key]; ok {  
		c.ll.MoveToFront(ele)  
		kv := ele.Value.(*entry)  
		return kv.value, true  
	}  
	return  
}
```
- 如果键对应的链表节点存在，则将对应节点移动到队尾，并返回查找到的值。
- `c.ll.MoveToFront(ele)`，即将链表中的节点 `ele` 移动到队尾（双向链表作为队列，队首队尾是相对的，在这里约定 front 为队尾）

### 删除（缓存淘汰）
```go
// RemoveOldest removes the oldest item  
func (c *Cache) RemoveOldest() {  
	ele := c.ll.Back()  
	if ele != nil {  
		c.ll.Remove(ele)  
		kv := ele.Value.(*entry)  
		delete(c.cache, kv.key)  
		c.nbytes -= int64(len(kv.key)) + int64(kv.value.Len())  
		if c.OnEvicted != nil {  
			c.OnEvicted(kv.key, kv.value)  
		}  
	}  
}
```
- `c.ll.Back()` 取到队首节点，从链表中删除。
- `delete(c.cache, kv.key)`，从字典中 `c.cache` 删除该节点的映射关系。
- 更新当前所用的内存 `c.nbytes`。
- 如果回调函数 `OnEvicted` 不为 nil，则调用回调函数

### 新增/修改
```go
// Add adds a value to the cache.  
func (c *Cache) Add(key string, value Value) {  
	if ele, ok := c.cache[key]; ok {  
		c.ll.MoveToFront(ele)  
		kv := ele.Value.(*entry)  
		c.nbytes += int64(value.Len()) - int64(kv.value.Len())  
		kv.value = value  
	} else {  
		ele := c.ll.PushFront(&entry{key, value})  
		c.cache[key] = ele  
		c.nbytes += int64(len(key)) + int64(value.Len())  
	}  
	for c.maxBytes != 0 && c.maxBytes < c.nbytes {  
		c.RemoveOldest()  
	}  
}
```
- 如果键存在，则更新对应节点的值，并将该节点移到队尾。
- 不存在则是新增场景，首先队尾添加新节点 `&entry{key, value}`, 并字典中添加 key 和节点的映射关系。
- 更新 `c.nbytes`，如果超过了设定的最大值 `c.maxBytes`，则移除最少访问的节点