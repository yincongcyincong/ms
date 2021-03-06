#### ca证书怎么验证
  证书=公钥+申请者与颁发者信息+签名；  
（1）首先浏览器读取证书中的证书所有者、有效期等信息进行校验，校验证书的网站域名是否与证书颁发的域名一致，校验证书是否在有效期内    
（2）浏览器开始查找操作系统中已内置的受信任的证书发布机构CA，与服务器发来的证书中的颁发者CA比对，用于校验证书是否为合法机构颁发    
（3）如果找不到，浏览器就会报错，说明服务器发来的证书是不可信任的。    
（4）如果找到，那么浏览器就会从操作系统中取出颁发者CA 的公钥(多数浏览器开发商发布   
版本时，会事先在内部植入常用认证机关的公开密钥)，然后对服务器发来的证书里面的签名进行解密   
（5）浏览器使用相同的hash算法计算出服务器发来的证书的hash值，将这个计算的hash值与证书中签名做对比   
（6）对比结果一致，则证明服务器发来的证书合法，没有被冒充      


#### kafka的0-copy
从一个内核态的fd1传到内核态的fd2

#### 设置锁的超时时间超过了设置时间
移步使用lua续时，有一个版本的概念
 ```
//先判断是否对应线程 是否持有锁，
      //KEY[1] 是锁名称 ARGV[2]是唯一的标识 UUID:ThreadId
  "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
      //如果持有锁，那么就重新设置过期时间为 30S
      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
      "return 1; " +
  "end; " +
  "return 0;",
  ```

#### go 1.14
```
package main

import (
	"runtime"
	"time"
)

func main() {
	runtime.GOMAXPROCS(1)
	
	go func() {
		for {
		}
	}()
	
	time.Sleep(time.Millisecond)
	println("OK")
}
```

其中创建一个goroutine并挂起， main goroutine 优先调用了 休眠，此时唯一的 P 会转去执行 for 循环所创建的 goroutine，进而 main goroutine 永远不会再被调度，这会导致GC在等待所有goroutine停止时等待时间过长，从而导致GC延迟；甚至在一些特殊情况下，导致在STW（stop the world）时死锁。换一句话说在Go1.14之前，上边的代码永远不会输出OK，因为这种协作式的抢占式调度是不会使一个没有主动放弃执行权、且不参与任何函数调用的goroutine被抢占。    

Go1.14 实现了基于信号的真抢占式调度解决了上述问题。Go1.14程序启动时，在 runtime.sighandler 函数中注册了 SIGURG 信号的处理函数 runtime.doSigPreempt，在触发垃圾回收的栈扫描时，调用函数挂起goroutine，并向M发送信号，M收到信号后，会让当前goroutine陷入休眠继续执行其他的goroutine。  

同步协作式调度		
1.主动用户让权：通过 runtime.Gosched 调用主动让出执行机会；			
2.主动调度弃权：当发生执行栈分段时，检查自身的抢占标记，决定是否继续执行；		
异步抢占式调度
1.被动监控抢占：当 G 阻塞在 M 上时（系统调用、channel 等），系统监控会将 P 从 M 上抢夺并分配给其他的 M 来执行其他的 G，而位于被抢夺 P 的 M 本地调度队列中 的 G 则可能会被偷取到其他 M 中。		
2.被动 GC 抢占：当需要进行垃圾回收时，为了保证不具备主动抢占处理的函数执行时间过长，导致垃圾回收迟迟不得执行而导致的高延迟，而强制停止 G 并转为执行垃圾回收。		


#### 为什么不能跨goroutine的recover
每个goroutine保存自己的堆栈信息，panic是从g的defer里面找recover，需要遍历所有协程，或者维护调用关系，根据gmp模型，通过go关键字创建的goroutine都是top level的

#### 多服务调用超时
```
func requestWork(ctx context.Context, job interface{}) error {
    ctx, cancel := context.WithTimeout(ctx, time.Second*2)
    defer cancel()

    done := make(chan error, 1)
    panicChan := make(chan interface{}, 1)
    go func() {
        defer func() {
            if p := recover(); p != nil {
                panicChan <- p
            }
        }()

        done <- hardWork(job)
    }()

    select {
    case err := <-done:
        return err
    case p := <-panicChan:
        panic(p)
    case <-ctx.Done():
        return ctx.Err()
    }
}
```
分成两个不同的部分，操作数据库在select下面，业务处理在另一个goroutine里面，处理完看时间是否超时


#### mpg没有g
内核线程调度不是用户态控制，并且可能因为系统调用阻塞
1. 一般来讲，M 的数量都会多于 P。像在 Go 中，M 的数量最大限制是 10000，P 的默认数量的 CPU 核数。另外由于 M 的属性，也就是如果存在系统阻塞调用，阻塞了 M，又不够用的情况下，M 会不断增加。	
2. M 不断增加的话，如果本地队列挂载在 M 上，那就意味着本地队列也会随之增加。这显然是不合理的，因为本地队列的管理会变得复杂，且 Work Stealing 性能会大幅度下降。		
3. M 被系统调用阻塞后，我们是期望把他既有未执行的任务分配给其他继续运行的，而不是一阻塞就导致全部停止。		


#### 一个大文件有10亿个数，怎么判断一个数是否存在
对于这一类判断数字是否存在、判断数字是否重复的问题，位图法是一种非常高效的方法。以32位整型为例，它可以表示数字的个数为2^32.可以申请一个位图，让每个整数对应的位图中的一个bit，这样2^32个数需要的位图的大小为512MB。具体实现的思路为：申请一个512MB的位图，并把所有的位都初始化为0；接着遍历所有的整数，对遍历到的数字，把相应的位置上的bit设置为1.最后判断待查找的数对应的位图上的值是多少，如果是0，那么表示这个数字不存在，如果是1，那么表示这个数字存在。		
