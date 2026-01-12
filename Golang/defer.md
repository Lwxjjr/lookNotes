defer常用于一些成对的操作：打开/关闭连接、加/释放锁、打开/关闭文件
# defer规则
1. 延迟函数的参数m在defer语句出现时就已经确定了
2. defer执行按照栈的顺序
3. return之后的defer不会被执行
# 拆解defer
```go
return xxx
```
这条语句经过编译后，实际上生成了3条指令，==说明`return`不是一条原子语句==：
1. 设置返回值：`result = xxx`
2. 调用`defer`语句
3. 空`return`

```go
func f() (r int) { 
	 t := 5 
     defer func() { 
       t = t + 5 
     }() 
     return t 
} 
```
拆解后：
```go
func f() (r int) { 
     t := 5 
     r = t 
     func() {         
         t = t + 5 
     } 
     return 
} 
```


---
# 闭包
闭包 = 函数 + 引用环境
匿名函数也叫闭包，没有函数名，不能独立存在，但是可以直接调用或者赋值于某个变量
==一个闭包继承了函数声明时的作用域==
==闭包的核心概念是**函数内部可以引用外部作用域的变量**
即使在函数内部外部作用域已经结束==
有个不太恰当的例子：可以把闭包看成是一个类，一个闭包函数调用就是实例化一个类
闭包在运行时可以有多个实例，它会将同一个作用域里的变量和常量捕获下来，无论闭包在什么地方被调用（实例化）时，都可以使用这些变量和常量
==闭包捕获的变量和常量是引用传递，不是值传递==

==当一个匿名函数**捕获**其外部变量时，它就成了一个**闭包**。虽然所有闭包都是匿名函数，但并不是所有的匿名函数都是闭包==
## 使用场景
### 中间件
我们在定义 web 中间件时经常会看到以下形式的代码:
```go
func makeHandler(fn func(http.ResponseWriter, *http.Request, string)) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        m := validPath.FindStringSubmatch(r.URL.Path)
        if m == nil {
            http.NotFound(w, r)
            return
        }
        fn(w, r, m[2]) // 如果没问题则继续执行 fn
    }
}
```
我们返回了一个 `http.HandlerFunc`, 这个函数里面调用了 fn, 这样的话我们就可以实现链式操作，既执行了中间件代码，又可以继续执行函数，非常方便
### 状态共享
闭包可以用来共享多次执行函数的状态， 常见的例子是迭代器:
```go
package main

import "fmt"

func main() {
	num := []int{1, 2, 3, 4}

	iterator := func(arr []int) func([]int) (int, bool) {
		i := -1
		return func(arr []int) (int, bool) {
			i ++
			if i < len(arr) {
				return arr[i], true
			}
			return 0, false
		}
	}

	iter := iterator(num)

	for {
		value, ok := iter(num)
		if !ok {
			return
		}

		fmt.Println(value)
	}
}

// 结果
//1
//2
//3
//4
```


---
# defer配合recover
实际项目里经常出现各种 error 满天飞
正常的代码逻辑里有很多 error 处理的代码块。函数总是会返回一个 error，留给调用者处理
- 错误被认为是一种可以预期的结果
- 异常则是一种非预期的结果
Go 语言推荐使用 `recover` 函数将内部异常转为错误处理，这使得用户可以真正的关心业务相关的错误处理

==有些时候，需要从异常中恢复==。比如服务器程序遇到严重问题，产生了 panic，这时至少可以 在程序崩溃前做一些“扫尾工作”，比如关闭客户端的连接，防止客户端一直等待等
==panic 会停掉当前正在执行的程序，而不只是当前线程==
==recover的位置（一定要在defer里面==）：
1. 不能。直接调用 recover，返回 nil。
```go
func main() { 
    recover() 
    panic(404) 
} 
```
2. 要在defer函数里调用recover
```go
func main() { 
    defer recover() 
    panic(404) 
} 
```
## recover失效的条件
三个条件让`recover()`返回`nil`：
1. `panic`指定参数为`nil`（一般`panic`语句如`panic("xxx failed...")
2. 当前协程没有发生`panic`
3. `recover`没有被`defer`方法直接调用（如`defer`->`function`->`function`->`recover`）


---
# 异常捕获
### 难以捕获的异常类型，无法通过`recover`捕获
- 内存溢出：预分配空间过大导致内存溢出，返回`runtime: out of memory`
- `map`并发读写：当 map 并发读写时，会返回 `concurrent map read and map write`
- 栈内存耗尽: 当栈内存耗尽时，会返回 `runtime: goroutine stack exceeds 1000000000-byte limit`
- Goroutine运行空函数：返回 `runtime: goroutine running on NULL machine`
- 全部 Goroutine 休眠时：会返回 `all goroutines are asleep - deadlock!`
### 可以捕获的异常
- 数组越界：返回 `panic: runtime error: index out of range`
- 空指针引用：返回 `panic: runtime error: invalid memory address or nil pointer dereference`
- 类型断言失败：返回 `panic: interface conversion: interface {} is nil, not int`
- 调用不存在的方法：返回 `panic: runtime error: invalid memory address or nil pointer dereference`