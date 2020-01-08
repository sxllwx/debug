# 手动Track一次go-ethereum的“小”bug

哈哈，刚刚好开年。新年的第7天，就解决了一个对我个人来说意义比较“小”的Bug。有趣的是还给 [以太坊](https://github.com/ethereum/go-ethereum) 提交了一个PR，既然在8个小时内被***Merge***。说实话，有些小激动，虽然只加了两行代码，但是找到他可不容易。废话不多说开启story。

## 初现端倪

线上部署的一个以太坊节点（Geth）我们称为“干死”（手动滑稽），差点“干死”一台48C，128G的机器。虽然说区块链节点本身挖矿就会消耗大量的CPU资源，但是该节点并未挖矿仅仅同步Block，为啥消耗了100%（甚至更多）的CPU资源，并且还在持续增长。于是，一直作为**咸鱼**的我被“捞”出来解决这个问题。

## 上线盲人摸象

Golang大法好，pporf安排上，火焰图下“神鬼现身”。一个名为runtime.timerproc的goroutine吃掉了90%的计算资源。定位完毕，golang runtime有bug，全剧终。。。

![图片alt](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1578477279891&di=f552a5dcf524a262092989db8b8e5f8b&imgtype=jpg&src=http%3A%2F%2Fimg4.imgtn.bdimg.com%2Fit%2Fu%3D1218938014%2C1025602925%26fm%3D214%26gp%3D0.jpg)

哈哈，当然不是。毕竟golang作为一个拥有67k star的开源项目写了诸如kubernetes，docker等项目的语言，不应该会有这样的bug，不应该这么Low，也不会。

## “老司机”上线，黑盒Debug

幸好部门内操作系统大佬 《A君》也是我的直接Leader。通过“神仙操作”很快定位到我们的进程大部分时间处在syscall。并且通过gstack和strace的定位到“干死”进程大量的调用**futex**和**pselect**。

GET！

但是，说实话，并不知道大佬说的是啥。（手动捂脸🤦‍♂️）

## 万里长征第一步

内存的Debug相对还好做，这个CPU的，还真没做过。BUT，作为一个老咸鱼，当然是从“车祸”现场看起 https://github.com/golang/go/blob/367da16e91af75888e44b02932d228af7c9b0244/src/runtime/time.go#L247:6 就是他了，一个永远都不会return的for循环。上面写着注释，该函数是用来运行“时间驱动”的事件。

```go
func timerproc(tb *timersBucket) {
	tb.gp = getg()
	for {
    		...
		now := nanotime()
		...
    		delta := int64(-1)
		for {
			...
            		// 取出堆顶第一个timer
			t := tb.t[0]
			delta = t.when - now
			if delta > 0 {
                	// 还需要继续等...
				break
			}
			ok := true
			if t.period > 0 {
                		// 是一个ticker
				t.when += t.period * (1 + -delta/t.period)
                		// 调整timer堆
				...
			} else {
                		// 是一个timer
				// 从timer堆中移除，并且调整timer堆
			}
			...
            		// 发送当前时间到timer或者ticker的C中。
			f(arg, seq)
			lock(&tb.lock)
		}
		if delta < 0 || faketime > 0 {
			// timer堆中没有timer，将该goroutine挂起。
			goparkunlock(&tb.lock, waitReasonTimerGoroutineIdle, traceEvGoBlock, 1)
			continue
		}
        	// 清理内容 balabala
		...
		notetsleepg(&tb.waitnote, delta) //关键来了（继续往下走）
	}
}

```

关键性的调度代码

```go
// same as runtime·notetsleep, but called on user g (not g0)
// calls only nosplit functions between entersyscallblock/exitsyscall
func notetsleepg(n *note, ns int64) bool {
	gp := getg()
	if gp == gp.m.g0 {
		throw("notetsleepg on g0")
	}

	// 当前这个goroutine需要进入syscall了，
  	// 1. 首先保存当前goroutine的PC等运行信息
	// 2. 将当前goroutine的状态从running改为 syscall
  	// 3. 放弃绑在当前的goroutine上的P
  	// 4. 调用 futex
	entersyscallblock() 
	ok := notetsleep_internal(n, ns) 
  	//  恢复现场
	exitsyscall() 
	return ok
}
```

**符合和大佬DEBUG的结果🎉🎉🎉🎉🎉🎉🎉**

基本定位是 $GOROOT time包下 ticker或者timer 使用过多导致。BUT 即使知道是timer在"搞怪"，但是总不能grep所有time.NewTicker和time.NewTimer的所有函数一点点看吧。

## 自定义“骚操作”

```bash
scott at localhost in ~/go-ethereum (master●)
$ find . -name "*.go" |xargs cat|wc -l
1104086
```

这个“鬼东西”110万行代码。我哪知道是哪行代码在频繁的NewTimer呢。“投机取巧”，直接在time包下， 在NewTimer和NewTicker处增加print，并且输出当前的调用栈。编译运行。回“沙滩“ 翻个面继续”晒“自己。

## 等**鱼**上钩

”晒“了两天，900M的日志已经躺在了我的硬盘里。

- cat debug | grep XXXX 
- 简单的统计 NewTicker和NewTimer的数量
- 找到栈尾的调用处

OK，whisper，你居然启动了N个goroutine，光管NewTicker不管Stop。 果断加上Stop的逻辑。反复测试2天，发现CPU消耗已经正常，3%左右。

## 给官方提交PR

简单描述PR在Fix啥bug，8小时后，merged！

## 尾声

2019年初，我还是个golang”弟中弟“（虽然现在也是，😛）。不过经过一年的学习，已经可以有模有样的排不怎么了解的项目的错了。 2020， 加油！

golang大法，还是好，哈哈哈哈哈哈。
