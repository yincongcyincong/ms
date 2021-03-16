
# GC

![image](https://github.com/yincongcyincong/ms/blob/main/image/gc1.png)
![image](https://github.com/yincongcyincong/ms/blob/main/image/gc2.png)

标记-清扫GC算法首先会从一些固定的root节点开始，对于Go语言来说就是全局指针和 goroutine 栈上的指针，根据这些root节点进行递归标记。当标记完成后，所有被标记的对象就都是存活的，其余的对象即是可以清扫的。


