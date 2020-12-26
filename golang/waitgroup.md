# waitgroup

结构
```
  type WaitGroup struct {
    state1 [3]uint32
  }
```
state1是个长度为3的数组，其中包含了state和一个信号量，而state实际上是两个计数器：   

counter： 当前还未执行结束的goroutine计数器    
waiter count: 等待goroutine-group结束的goroutine数量，即有多少个等候者    
semaphore: 信号量    

### Add(delta int)
Add()做了两件事，一是把delta值累加到counter中，因为delta可以为负值，也就是说counter有可能变成0或负值，所以第二件事就是当counter值变为0时，跟据waiter数值释放等量的信号量，
把等待的goroutine全部唤醒，如果counter变为负值，则panic.

Add()伪代码如下：   
```
  func (wg *WaitGroup) Add(delta int) {
      statep, semap := wg.state() //获取state和semaphore地址指针

      state := atomic.AddUint64(statep, uint64(delta)<<32) //把delta左移32位累加到state，即累加到counter中
      v := int32(state >> 32) //获取counter值
      w := uint32(state)      //获取waiter值

      if v < 0 {              //经过累加后counter值变为负值，panic
          panic("sync: negative WaitGroup counter")
      }

      //经过累加后，此时，counter >= 0
      //如果counter为正，说明不需要释放信号量，直接退出
      //如果waiter为零，说明没有等待者，也不需要释放信号量，直接退出
      if v > 0 || w == 0 {
          return
      }

      //此时，counter一定等于0，而waiter一定大于0（内部维护waiter，不会出现小于0的情况），
      //先把counter置为0，再释放waiter个数的信号量
      *statep = 0
      for ; w != 0; w-- {
          runtime_Semrelease(semap, false) //释放信号量，执行一次释放一个，唤醒一个等待者
      }
  }   
```
### Wait()
Wait()方法也做了两件事，一是累加waiter, 二是阻塞等待信号量    
```
  func (wg *WaitGroup) Wait() {
      statep, semap := wg.state() //获取state和semaphore地址指针
      for {
          state := atomic.LoadUint64(statep) //获取state值
          v := int32(state >> 32)            //获取counter值
          w := uint32(state)                 //获取waiter值
          if v == 0 {                        //如果counter值为0，说明所有goroutine都退出了，不需要待待，直接返回
              return
          }

          // 使用CAS（比较交换算法）累加waiter，累加可能会失败，失败后通过for loop下次重试
          if atomic.CompareAndSwapUint64(statep, state, state+1) {
              runtime_Semacquire(semap) //累加成功后，等待信号量唤醒自己
              return
          }
      }
  }
```
这里用到了CAS算法保证有多个goroutine同时执行Wait()时也能正确累加waiter。

### Done()
Done()只做一件事，即把counter减1，我们知道Add()可以接受负值，所以Done实际上只是调用了Add(-1)。

源码如下：
```
  func (wg *WaitGroup) Done() {
    wg.Add(-1)
  }
```
Done()的执行逻辑就转到了Add()，实际上也正是最后一个完成的goroutine把等待者唤醒的。

### 编程Tips
Add()操作必须早于Wait(), 否则会panic
Add()设置的值必须与实际等待的goroutine个数一致，否则会panic
