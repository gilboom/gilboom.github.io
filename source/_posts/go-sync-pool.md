---
title: go-sync-pool
date: 2021-11-01 23:54:58
tags: Golang
---

# Go 语言中的 sync.Pool

写下这篇博客的原因是，在看孔令飞大佬在极客时间上的《Go 语言项目开发实战》课程的时候，有个从零开始实现日志的章节里面运用到了 sync.Pool，我之前也了解过 sync.Pool 但是又想不起来了，所以写下这篇文章记录一下加深自己的印象

## sync.Pool 的作用

引用某位大佬的某篇文章的一句话：
> 保存和复用临时对象，减少内存分配，减少 GC 压力 [<sup>1</sup>](#refer-anchor-1)

Go 语言的内存管理是不像 C/C++ 那样需要手动管理内存的分配和释放，而是由内存分配器 + 垃圾回收器去管理的，对象在创建的时候，有可能分配在栈上，有可能分配在堆上（如果对象比较大，或者对象的指针被返回到外部，是会被分配到堆内存上的），堆上的内存是需要垃圾回收器去回收的。如果在高并发情况下频繁的创建临时对象，可能会导致堆内存上出现大量的还来不及回收的临时对象。当大量临时对象占用了堆内存空间的时候，会导致没有足够的堆内存去分配其他对象，就会触发强制 GC，虽然 Go 已经对 GC 算法做了很多优化，比如将清理过程和程序执行过程并行，但是还是避免不了 STW 带来的影响。所以 sync.Pool 就派上了用场了。

## 例子

在 Go 标准库中的 fmt 包就用到了 sync.Pool

进入到 fmt 包中，我们先从 fmt.Printf() 这个方法入手看看里面是怎么使用 sync.Pool 的

```Golang
// Printf formats according to a format specifier and writes to standard output.
// It returns the number of bytes written and any write error encountered.
func Printf(format string, a ...interface{}) (n int, err error) {
	return Fprintf(os.Stdout, format, a...)
}
// Fprintf formats according to a format specifier and writes to w.
// It returns the number of bytes written and any write error encountered.
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
	p := newPrinter()
	p.doPrintf(format, a)
	n, err = w.Write(p.buf)
	p.free()
	return
}
```

可以看到 Printf 方法其实是调用了 Fprintf 方法，这里是做了一个代码复用，Printf 是更高级的方法，Fprintf 是 low-level 的方法。然后 Fprintf 调用 newPrinter 方法。

```Golang
var ppFree = sync.Pool{
	New: func() interface{} { return new(pp) },
}

// newPrinter allocates a new pp struct or grabs a cached one.
func newPrinter() *pp {
	p := ppFree.Get().(*pp)
	p.panicking = false
	p.erroring = false
	p.wrapErrs = false
	p.fmt.init(&p.buf)
	return p
}
```

可以看到这个 newPrinter 方法就是用 ppFree，也就是一个 sync.Pool 对象去获取空闲的 pp 对象，然后做一些初始化，这个 pp 对象用来实际执行 doPrintf 的逻辑。然后在 Fprintf 方法中执行完打印工作以后，就调用 p.free 方法，去执行一些清理操作，最后把 pp 对象放回了 sync.Pool。

```Golang
// free saves used pp structs in ppFree; avoids an allocation per invocation.
func (p *pp) free() {
	// Proper usage of a sync.Pool requires each entry to have approximately
	// the same memory cost. To obtain this property when the stored type
	// contains a variably-sized buffer, we add a hard limit on the maximum buffer
	// to place back in the pool.
	//
	// See https://golang.org/issue/23199
	if cap(p.buf) > 64<<10 {
		return
	}

	p.buf = p.buf[:0]
	p.arg = nil
	p.value = reflect.Value{}
	p.wrappedErr = nil
	ppFree.Put(p)
}
```

可以看到在这里使用 sync.Pool 有两个好处：

1. 每次从 ppFree 中拿到 pp 对象以后，不需要对 pp 对象加锁，因为从 sync.Pool 中获取临时对象以后，取出来的临时对象就不在 sync.Pool 中了，也就是其他 Gorouting 也拿不到相同的临时对象了，所以减少了锁的开销
2. 临时对象 pp 用完以后，只要做一些清理工作，就能把 pp 放回 ppFree 中复用了，减少临时对象的创建，减少 GC

## 参考

<div id="refer-anchor-1">
- [1][Go sync.Pool] https://geektutu.com/post/hpg-sync-pool.html