
# GC

![image](https://github.com/yincongcyincong/ms/blob/main/image/gc1.png)
![image](https://github.com/yincongcyincong/ms/blob/main/image/gc2.png)

标记-清扫GC算法首先会从一些固定的root节点开始，对于Go语言来说就是全局指针和 goroutine 栈上的指针，根据这些root节点进行递归标记。当标记完成后，所有被标记的对象就都是存活的，其余的对象即是可以清扫的。



Golang使用的是三色标记法方案，并且支持并行GC，即用户代码何以和GC代码同时运行。具体来讲，Golang GC分为几个阶段:Mark阶段该阶段又分为两个部分：    
    Mark Prepare：初始化GC任务，包括开启写屏障(write barrier)和辅助GC(mutator assist)，统计root对象的任务数量等，这个过程需要STW。    
   GC Drains: 扫描所有root对象，包括全局指针和goroutine(G)栈上的指针（扫描对应G栈时需停止该G)，将其加入标记队列(灰色队列)，并循环处理灰色队列的对象，直到灰色队列为空。该过程后台并行执行。    
Mark Termination阶段：该阶段主要是完成标记工作，重新扫描(re-scan)全局指针和栈。因为Mark和用户程序是并行的，所以在Mark过程中可能会有新的对象分配和指针赋值，这个时候就需要通过写屏障（write barrier）记录下来，re-scan 再检查一下，这个过程也是会STW的。    
Sweep: 按照标记结果回收所有的白色对象，该过程后台并行执行。   
Sweep Termination: 对未清扫的span进行清扫, 只有上一轮的GC的清扫工作完成才可以开始新一轮的GC。   
