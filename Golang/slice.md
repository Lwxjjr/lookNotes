slice的底层数组是数组，slice是对数组的封装  
数组的定长的，数组的长度是类型的一部分，限制了它的表达能力，比如[3]int，[4]int是不同的类型  
silce实际上是一个结构体
```go
type slice struct {
	array  unsafe.Pointer
	len    int
	cap    int
}
```

# 拷贝区别
```go
// array
x := [5]int{1, 2, 3, 4, 5}
y := x
```
数组y是x的副本
```go
x := []int{1, 2, 3, 4, 5}
y := x
```
切片y和x指向同一个底层数组  

# 切片扩容
```go
func growslice(et *_type, old slice, cap int) slice {
   //  ......
	newcap := old.cap
	doublecap := newcap + newcap     //双倍扩容（原容量的两倍）
	if cap > doublecap {             //如果所需容量大于 两倍扩容，则直接扩容到所需容量
		newcap = cap
	} else {
		const threshold = 256        //这里设置了一个 阈值 -- 256
		if old.cap < threshold {     //如果旧容量 小于 256，则两倍扩容
			newcap = doublecap   
		} else {
		    // 检查 0 < newcap 以检测溢出并防止无限循环。
			//如果新容量 > 0  并且 原容量 小于 所需容量
			for 0 < newcap && newcap < cap {   
            /* 
            从小片的增长2x过渡到大片的增长1.25x。
            这个公式给出了两者之间的平滑过渡。
            (这里的系数会随着容量的大小发生变化，从2.0到无线接近1.25)
            */
				newcap += (newcap + 3*threshold) / 4
              //当newcap计算溢出时，将newcap设置为请求的上限。
			if newcap <= 0 {   // 如果发生了溢出，将新容量设置为请求的容量大小
				newcap = cap
			}
		}
	}
}
```
1. newcap > 2 * cap，newcap = newcap 
2. newcap < 2 * cap：
	1. cap < 256，newcap = 2 * cap
	2. cap > 256，newcap = 1.25cap + 3/4 threshold

# 切片作为函数参数
给出结论：slice在作为函数参数进行传递时，分两种情况：
1. 如果不发生扩容，则是引用传递，也就是函数内对切片的操作会影响函数外的切片。
2. 如果发生扩容，则函数内的切片的内存地址发生变化，此时对函数内切片的操作不会影响函数外。

因此，如果想要函数内对切片的操作始终影响函数外，那么在传参时使用**指针**即可