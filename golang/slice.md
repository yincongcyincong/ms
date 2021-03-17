# slice

#### 并发安全
真实的输出并没有达到我们的预期，len(slice) < n。 问题出在哪？我们都知道slice是对数组一个连续片段的引用，
当slice长度增加的时候，可能底层的数组会被换掉。当出在换底层数组之前，切片同时被多个goroutine拿到，并执行append操作。
那么很多goroutine的append结果会被覆盖，导致n个gouroutine append后，长度小于n。

#### 具体结构
```
type slice struct {
	array unsafe.Pointer // 元素指针
	len   int // 长度 
	cap   int // 容量
}
```

#### make和new
make 的作用是初始化内置的数据结构，也就是我们在前面提到的切片、哈希表和 Channel，make 关键字的 OMAKE 节点根据参数类型的不同转换成了 OMAKESLICE、OMAKEMAP 和 OMAKECHAN 三种不同类型的节点，这些节点会调用不同的运行时函数来初始化相应的数据结构，然后指针指向此数据结构。    
new 的作用是根据传入的类型分配一片内存空间并返回指向这片内存空间的指针，开辟栈空间，指针指向此空间；

#### append扩容
先将旧的slice容量乘以2，如果乘以2后的容量仍小于新的slice容量，则取新的slice容量(append多个elems)
如果旧的slice容量大于1024，则新slice容量取旧slice容量乘以1.25
