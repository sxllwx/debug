# golang memory

环境信息如下:

- golang版本: `1.14.12`
- os版本: `Linux VM-32-129-centos 4.18.0`

现象:

> 遇到有个问题想请教一下，go 1.14版本内存释放是用的MADV_FREE，我们用stress模拟把系统剩余物理内存都用完了，top看进程的rss还是没有释放。**并且发生了OOM的现象**。

问题分析如下(可能性大小排序):

1. 有可能是app处在一个很高的负载，大量的内存在栈上。
2. golang GC有问题。
3. OS畸形行为导致, 在内存紧缩的情况下不进行标记为Free的内存块的释放。

## 几个基本知识点:

1. 以`MADV_FREE`释放的内存不会减少RSS, 以`MADV_DONOTNEED`释放的内存会减少RSS
2. golang的内存分析应该从Stack和Heap，拆开看。

> `MADV_DONOTNEED`标记的内存会在访问的时候出现缺页(PageFault)但是`MADV_FREE`不会, 前者代价很高

## 相关文档如下:

- https://github.com/golang/go/issues/36398   **可以重点关注一下mknyszek的回答**


## 一些顾虑

内存的信息: 
```
# runtime.MemStats
... (省略无关信息)
# Sys = 73351424 //从OS层面已经申请了多少内存
# HeapSys = 66420736 // 从OS层面看申请了多少关于堆的内存
# Stack = 688128 / 688128 // 第一个是StackInuse, 第二个从OS层面已经申请了多少内存
```

- golang的GC和内存申请是异步的。1.14.13版本中GC被触发的路径非常多，有可能是ForceGCInterval, 也有可能是malloc路径之上
- pprof MemStats 展示内容(curl http://localhost:6060/debug/pprof/heap\?debug\=2)
- 锁的竞争可能会导致Mem表现畸形，如果大量的Goroutine争抢Lock，会导致栈内的内存和堆内的内存都处于Inused，无法被释放，**吞量高，但是吐量小**的的业务尤其要小心这种

## Container环境和BearMetal环境下的内存紧缩

```bash
-sh-4.2# cat memory.soft_limit_in_bytes
9223372036854771712
-sh-4.2# pwd
/sys/fs/cgroup/memory/kubepods/pod29eda02f-3b4b-47f9-ac43-519a9a600612/03cafdd2a9552240eba8c44f7173c4a499842baba2d95c9bf839fae38506d360
-sh-4.2# cat memory.limit_in_bytes
1073741824
```

Pod的Request和Limit会触发Containerd，或者其他的CRI，设置cgroup部分选项，包括(memory.soft_limit_in_bytes), 这个会触发Kernal更加激进的对标记为MADV_FREE的内存进行释放。

> 结论: 容器化为Pod设置了Request和Limit之后不会遇到这种情况

## 为了更加明确这个问题

使用相同规格的机器，使用相同的压测软件，使用相同的并发测试两个相同的App(golang 1.12~1.16之间编译出来的App)，唯一的区别就是运行前是否导出`GODEBUG=madvdontneed=1`

有几个重点需要关注一下

- 是否两个都会OOM
- 运行过程中关注一下**栈**上的内存是否已经非常的高
- 可以对上述Cgroup参数进行设置来观察业务需求是否能够完成
