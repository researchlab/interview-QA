
<!-- TOC -->

- [1.go的调度模型理解](#1go的调度模型理解)
- [2.golang map如何顺序读取](#2golang-map如何顺序读取)
- [3.go struct能不能比较](#3go-struct能不能比较)
- [4.Golang中除了加Mutex锁以外还有哪些方式安全读写共享变量？](#4golang中除了加mutex锁以外还有哪些方式安全读写共享变量)
- [5.go defer在return之前还是之后执行?](#5go-defer在return之前还是之后执行)
- [6.slice使用需要注意什么?](#6slice使用需要注意什么)
- [7.当go服务部署到线上了，发现有内存泄露，该怎么处理](#7当go服务部署到线上了发现有内存泄露该怎么处理)
- [8.字符串转成byte数组，会发生内存拷贝吗？](#8字符串转成byte数组会发生内存拷贝吗)
- [9.go的GC原理](#9go的gc原理)
- [10.mysql索引为什么要用B+树？](#10mysql索引为什么要用b树)
- [11.mysql语句性能评测？](#11mysql语句性能评测)
- [12.对已经关闭的的chan进行读写，会怎么样？为什么？](#12对已经关闭的的chan进行读写会怎么样为什么)

<!-- /TOC -->
##### 1.go的调度模型理解
go的调度原理是基于GMP模型，G代表一个goroutine，不限制数量；M=machine，代表一个线程，最大1万，所有G任务还是在M上执行；P=processor代表一个处理器，每一个允许的M都会绑定一个G，默认与逻辑CPU数量相等（通过runtime.GOMAXPROCS(runtime.NumCPU())设置）。

go调用过程,
> 创建一个G对象
> 
> 如果还有空闲的的P，创建一个M
> 
> M会启动一个底层线程，循环执行能找到的G
> 
> G的执行顺序是先从本地队列找，本地没找到从全局队列找。一次性转移(全局G个数/P个数）个，再去其> 它P中找（一次性转移一半）
> 
> 以上的G任务是按照队列顺序执行（也就是go函数的调用顺序）。
> 
> 另外在启动时会有一个专门的sysmon来监控和管理，记录所有P的G任务计数schedtick。如果某个P的> schedtick一直没有递增，说明这个P一直在执行一个G任务，如果超过一定时间就会为G增加标记，并且> 该G执行非内联函数时中断自己并把自己加到队尾。

##### 2.golang map如何顺序读取
map的底层是hash table(hmap类型)，对key值进行了hash，并将结果的低八位用于确定key/value存在于哪个bucket（bmap类型）。再将高八位与bucket的tophash进行依次比较，确定是否存在。出现hash冲撞时，会通过bucket的overflow指向另一个bucket，形成一个单向链表。每个bucket存储8个键值对。

如果要实现map的顺序读取，需要使用一个slice来存储map的key并按照顺序进行排序。

```go
m := make(map[int]int, 10)
keys := make([]int, 0, 10)
for i := 0; i < 10; i++ {
    m[i] = i
    keys = append(keys, i)
}
//降序
sort.Slice(keys, func(i, j int) bool {
    if keys[i] > keys[j] {
        return true
    }
    return false
})
for _, v := range keys {
    fmt.Println(m[v])
}
```
##### 3.go struct能不能比较
普通的能比较， 但是包含 map、slice，如果struct包含这些类型的字段，则不能比较。

struct 包含普通的struct 也能比较


##### 4.Golang中除了加Mutex锁以外还有哪些方式安全读写共享变量？
Golang中Goroutine 可以通过 Channel 进行安全读写共享变量。

##### 5.go defer在return之前还是之后执行?
go defer 在return之后，函数退出之前执行;

##### 6.slice使用需要注意什么?

slice，len，cap，共享，扩容

len：切片的长度，访问时间复杂度为O(1)，go的slice底层是对数组的引用。

cap：切片的容量，扩容是以这个值为标准。默认扩容是2倍，当达到1024的长度后，按1.25倍。

扩容：每次扩容slice底层都将先分配新的容量的内存空间，再将老的数组拷贝到新的内存空间，因为这个操作不是并发安全的。所以并发进行append操作，读到内存中的老数组可能为同一个，最终导致append的数据丢失。

共享：slice的底层是对数组的引用，因此如果两个切片引用了同一个数组片段，就会形成共享底层数组。当sliec发生内存的重新分配（如扩容）时，会对共享进行隔断。详细见下面例子：

```go
a := []int{1, 2, 3}
b := a[:2]
a[0] = 10
//此时a和b共享底层数组
fmt.Println(a, "a cap:", cap(a), "a len:", len(a))
fmt.Println(b, "b cap:", cap(b), "b len:", len(b))
fmt.Println("-----------")
b = append(b, 999)
//虽然b append了1,但是没有超出cap，所以未进行内存重新分配
//等同于b[2]=999，因此a[2]一并被修改
fmt.Println(a, "a cap:", cap(a), "a len:", len(a))
fmt.Println(b, "b cap:", cap(b), "b len:", len(b))
fmt.Println("-----------")
a[2] = 555 //同上，未重新分配，所以，a[2] b[2]都被修改
fmt.Println(a, "a cap:", cap(a), "a len:", len(a))
fmt.Println(b, "b cap:", cap(b), "b len:", len(b))
fmt.Println("-----------")
b = append(b, 777) //超出了cap，这时候b进行重新分配,b[3]=777,cap(b)=6
a[2] = 666         //这时候a和b不再共享
fmt.Println(a, "a cap:", cap(a), "a len:", len(a))
fmt.Println(b, "b cap:", cap(b), "b len:", len(b))
```

##### 7.当go服务部署到线上了，发现有内存泄露，该怎么处理

[pprof](https://www.cnblogs.com/weiweng/p/12497274.html)

[如何读懂火焰图？](http://www.ruanyifeng.com/blog/2017/09/flame-graph.html)

> y 轴表示调用栈，每一层都是一个函数。调用栈越深，火焰就越高，顶> 部就是正在执行的函数，下方都是它的父函数。
> 
> x 轴表示抽样数，如果一个函数在 x 轴占据的宽度越宽，就表示它被> 抽到的次数多，即执行的时间长。注意，x 轴不代表时间，而是所有的> 调用栈合并后，按字母顺序排列的。
> 
> 火焰图就是看顶层的哪个函数占据的宽度最大。只要有"平顶"> （plateaus），就表示该函数可能存在性能问题。
> 
> 颜色没有特殊含义，因为火焰图表示的是 CPU 的繁忙程度，所以一般选择暖色调。

##### 8.字符串转成byte数组，会发生内存拷贝吗？
字符串转成切片，会产生拷贝。严格来说，只要是发生类型强转都会发生内存拷贝。那么问题来了。

频繁的内存拷贝操作听起来对性能不大友好。有没有什么办法可以在字符串转成切片的时候不用发生拷贝呢？
解释
```go
package main

import (
 "fmt"
 "reflect"
 "unsafe"
)

func main() {
 a :="aaa"
 ssh := *(*reflect.StringHeader)(unsafe.Pointer(&a))
 b := *(*[]byte)(unsafe.Pointer(&ssh))  
 fmt.Printf("%v",b)
}
```
StringHeader 是字符串在go的底层结构。
```go
type StringHeader struct {
 Data uintptr
 Len  int
}
```
SliceHeader 是切片在go的底层结构。
```go
type SliceHeader struct {
 Data uintptr
 Len  int
 Cap  int
}
```

那么如果想要在底层转换二者，只需要把 StringHeader 的地址强转成 SliceHeader 就行。那么go有个很强的包叫 unsafe 。

unsafe.Pointer(&a)方法可以得到变量a的地址。

(*reflect.StringHeader)(unsafe.Pointer(&a)) 可以把字符串a转成底层结构的形式。

(*[]byte)(unsafe.Pointer(&ssh)) 可以把ssh底层结构体转成byte的切片的指针。
再通过 *转为指针指向的实际内容。

##### 9.go的GC原理
http://alblue.cn/articles/2020/07/07/1594131614114.html

mysql
##### 10.mysql索引为什么要用B+树？
节省空间（非叶子节点不存储数据，相对b tree的优势），减少I/O次数（节省的空间全部存指针地址，让树变的矮胖），范围查找方便（相对hash的优势）。

##### 11.mysql语句性能评测？
explain

其他的见：

https://blog.csdn.net/chengxuyuanyonghu/article/details/61431386

##### 12.对已经关闭的的chan进行读写，会怎么样？为什么？

- 读已经关闭的 chan 能一直读到东西，但是读到的内容根据通道内关闭前是否有元素而不同。
如果 chan 关闭前，buffer 内有元素还未读 , 会正确读到 chan 内的值，且返回的第二个 bool 值（是否读成功）为 true。
如果 chan 关闭前，buffer 内有元素已经被读完，chan 内无值，接下来所有接收的值都会非阻塞直接成功，返回 channel 元素的零值，但是第二个 bool 值一直为 false。

- 写已经关闭的 chan 会 panic

https://github.com/lifei6671/interview-go/blob/master/question/q018.md