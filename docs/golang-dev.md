
<!-- TOC -->

- [1.go的调度模型理解](#1go的调度模型理解)
- [2.golang map如何顺序读取](#2golang-map如何顺序读取)
- [3.go struct能不能比较](#3go-struct能不能比较)
- [4.Golang中除了加Mutex锁以外还有哪些方式安全读写共享变量？](#4golang中除了加mutex锁以外还有哪些方式安全读写共享变量)

<!-- /TOC -->
##### 1.go的调度模型理解
go的调度原理是基于GMP模型，G代表一个goroutine，不限制数量；M=machine，代表一个线程，最大1万，所有G任务还是在M上执行；P=processor代表一个处理器，每一个允许的M都会绑定一个G，默认与逻辑CPU数量相等（通过runtime.GOMAXPROCS(runtime.NumCPU())设置）。
##### 2.golang map如何顺序读取

##### 3.go struct能不能比较
普通的能比较， 但是包含 map、slice，如果struct包含这些类型的字段，则不能比较。

struct 包含普通的struct 也能比较


##### 4.Golang中除了加Mutex锁以外还有哪些方式安全读写共享变量？
Golang中Goroutine 可以通过 Channel 进行安全读写共享变量。