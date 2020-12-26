# os

#### page cache
page cache的大小为一页，通常为4K。在linux读写文件时，它用于缓存文件的逻辑内容，从而加快对磁盘上映像和数据的访问。

#### mmap
进程读取文件时，把文件内容映射到进程内存

#### 缺页
page fault缺页异常分为两种类型，一种叫做major page fault，这种类型的缺页可以通过 Disk IO来满足。   
另一种叫做minor page fault，这种缺页可以直接利用内存中的缓存页满足。
