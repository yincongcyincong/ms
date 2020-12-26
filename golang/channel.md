# channel

channel的整体结构图   
![image](https://github.com/yincongcyincong/ms/blob/main/image/channel1.png)

简单说明：

buf是有缓冲的channel所特有的结构，用来存储缓存数据。是个循环链表   
sendx和recvx用于记录buf这个循环链表中的~发送或者接收的~index    
lock是个互斥锁。    
recvq和sendq分别是接收(<-channel)或者发送(channel <- xxx)的goroutine抽象出来的结构体(sudog)的队列。是个双向链表    
源码位于/runtime/chan.go中(目前版本：1.11)。结构体为hchan。   
```
type hchan struct {
    qcount   uint           // total data in the queue
    dataqsiz uint           // size of the circular queue
    buf      unsafe.Pointer // points to an array of dataqsiz elements
    elemsize uint16
    closed   uint32
    elemtype *_type // element type
    sendx    uint   // send index
    recvx    uint   // receive index
    recvq    waitq  // list of recv waiters
    sendq    waitq  // list of send waiters

    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    lock mutex
}
```
下面我们来详细介绍hchan中各部分是如何使用的。

先从创建开始
我们首先创建一个channel。
```
ch := make(chan int, 3)
```
![image](https://github.com/yincongcyincong/ms/blob/main/image/channel2.png)    

创建channel实际上就是在内存中实例化了一个hchan的结构体，并返回一个ch指针，我们使用过程中channel在函数之间的传递都是用的这个指针，这就是为什么函数传递中无需使用channel的指针，而直接用channel就行了，因为channel本身就是一个指针。

channel中发送send(ch <- xxx)和recv(<- ch)接收
先考虑一个问题，如果你想让goroutine以先进先出(FIFO)的方式进入一个结构体中，你会怎么操作？ 加锁！对的！channel就是用了一个锁。hchan本身包含一个互斥锁mutex

channel中队列是如何实现的
channel中有个缓存buf，是用来缓存数据的(假如实例化了带缓存的channel的话)队列。我们先来看看是如何实现“队列”的。 还是刚才创建的那个channel
```
ch := make(chan int, 3)
```
![image](https://github.com/yincongcyincong/ms/blob/main/image/channel3.png)    

当使用send (ch <- xx)或者recv ( <-ch)的时候，首先要锁住hchan这个结构体。    

![image](https://github.com/yincongcyincong/ms/blob/main/image/channel4.png)   

然后开始send (ch <- xx)数据。   
一
```
ch <- 1
```
二
```
ch <- 1
```
三
```
ch <- 1
```
这时候满了，队列塞不进去了 动态图表示为：
![image](https://github.com/yincongcyincong/ms/blob/main/image/channel5.gif)     

然后是取recv ( <-ch)的过程，是个逆向的操作，也是需要加锁。   
![image](https://github.com/yincongcyincong/ms/blob/main/image/channel6.png)     


然后开始recv (<-ch)数据。 
一
```
<-ch
```
二
```
<-ch
```
三
```
<-ch
```
图为：
![image](https://github.com/yincongcyincong/ms/blob/main/image/channel7.gif)   
注意以上两幅图中buf和recvx以及sendx的变化，recvx和sendx是根据循环链表buf的变动而改变的。     
至于为什么channel会使用循环链表作为缓存结构，我个人认为是在缓存列表在动态的send和recv过程中，定位当前send或者recvx的位置、选择send的和recvx的位置比较方便吧，
只要顺着链表顺序一直旋转操作就好。   

缓存中按链表顺序存放，取数据的时候按链表顺序读取，符合FIFO的原则。

send/recv的细化操作
注意：缓存链表中以上每一步的操作，都是需要加锁操作的！

每一步的操作的细节可以细化为：   
第一，加锁    
第二，把数据从goroutine中copy到“队列”中(或者从队列中copy到goroutine中）。      
第三，释放锁    
每一步的操作总结为动态图为：(发送过程)
![image](https://github.com/yincongcyincong/ms/blob/main/image/channel8.gif)   
或者为：(接收过程)    
![image](https://github.com/yincongcyincong/ms/blob/main/image/channel9.gif)   
所以不难看出，Go中那句经典的话：Do not communicate by sharing memory; instead, share memory by communicating.的具体实现就是利用channel把数据从一端copy到了另一端！ 还真是符合channel的英文含义：

![image](https://github.com/yincongcyincong/ms/blob/main/image/channel10.gif)     

当channel缓存满了之后会发生什么？这其中的原理是怎样的？
使用的时候，我们都知道，当channel缓存满了，或者没有缓存的时候，我们继续send(ch <- xxx)或者recv(<- ch)会阻塞当前goroutine，但是，是如何实现的呢？

我们知道，Go的goroutine是用户态的线程(user-space threads)，用户态的线程是需要自己去调度的，Go有运行时的scheduler去帮我们完成调度这件事情。关于Go的调度模型GMP模型我在此不做赘述，如果不了解，可以看我另一篇文章(Go调度原理)

goroutine的阻塞操作，实际上是调用send (ch <- xx)或者recv ( <-ch)的时候主动触发的，具体请看以下内容：
```
//goroutine1 中，记做G1

ch := make(chan int, 3)

ch <- 1
ch <- 1
ch <- 1
```

![image](https://github.com/yincongcyincong/ms/blob/main/image/channel11.png)     
![image](https://github.com/yincongcyincong/ms/blob/main/image/channel12.png)     
这个时候G1正在正常运行,当再次进行send操作(ch<-1)的时候，会主动调用Go的调度器,让G1等待，并从让出M，让其他G去使用    
![image](https://github.com/yincongcyincong/ms/blob/main/image/channel13.png)     


同时G1也会被抽象成含有G1指针和send元素的sudog结构体保存到hchan的sendq中等待被唤醒。   

![image](https://github.com/yincongcyincong/ms/blob/main/image/channel14.gif)     

那么，G1什么时候被唤醒呢？这个时候G2隆重登场。   

![image](https://github.com/yincongcyincong/ms/blob/main/image/channel14.png)     


G2执行了recv操作p := <-ch，于是会发生以下的操作：        
![image](https://github.com/yincongcyincong/ms/blob/main/image/channel15.png)  


G2从缓存队列中取出数据，channel会将等待队列中的G1推出，将G1当时send的数据推到缓存中，然后调用Go的scheduler，唤醒G1，并把G1放到可运行的Goroutine队列中。
![image](https://github.com/yincongcyincong/ms/blob/main/image/channel16.gif)  


假如是先进行执行recv操作的G2会怎么样？
你可能会顺着以上的思路反推。首先：



这个时候G2会主动调用Go的调度器,让G2等待，并从让出M，让其他G去使用。 G2还会被抽象成含有G2指针和recv空元素的sudog结构体保存到hchan的recvq中等待被唤醒       
![image](https://github.com/yincongcyincong/ms/blob/main/image/channel17.gif)  


此时恰好有个goroutine G1开始向channel中推送数据 ch <- 1。 此时，非常有意思的事情发生了：    

![image](https://github.com/yincongcyincong/ms/blob/main/image/channel18.png)  

G1并没有锁住channel，然后将数据放到缓存中，而是直接把数据从G1直接copy到了G2的栈中。 这种方式非常的赞！在唤醒过程中，G2无需再获得channel的锁，然后从缓存中取数据。减少了内存的copy，提高了效率。

之后的事情显而易见：
