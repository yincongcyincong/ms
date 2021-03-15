# k8s

#### Schedule:
主要是调度pod放在哪个节点上(添加pod)，有三种方式：
预选 g.findNodesThatFit()
优选  g.prioritizeNodes()
抢占  sched.preempt()

#### Controller-manager
注册contorller
StartControllers()->NewControllerInitializers()
主要是watch api-server，每当etcd变化时，APIserver会通知controller，然后由它判断是否重启，或者删除pod，endpoint，service这些资源

#### Api-server
CreateServerChain->CreateKubeAPIServer->new
// 这里就是加载核心资源对应存储设置了。   
// 目前k8s的的SchemeGroupVersion group为空，version为v1   
// 在k8s体系中，通常storage、store对应的属性都是和存储相关，这些存储包括缓存和后端的存储模块etcd，    
// k8s中的每种资源都实现了自己的存储方式。在接收到客户端请求处理时候调用预先配置的storage进行存储。    
->new->InstallLegacyAPI->InstallLegacyAPIGroup->installAPIResources->InstallREST->installer.Install->registerResourceHandlers
注册外部handler

主要是改etcd，kubelet报告pod的信息，通过接口上报给apiserver，apiserver报给controller-manager，controller-manager进行处理，返回结果给apiserver，然后apiserver改etcd，
如果是添加pod，还会经过schedule进行选取node

#### Kubelet：
Work更改存在本地的pod状态cache
readinessManager    
livenessManager   
startupManager    
这三个manager判断是否有更新，有就存入数据
然后这三个manager从cache的channel取数据，取出后判断是否改变，改变了就传入    
podStatusChannel    
这个channel，然后总manager取出，如果需要删除pod就调用apiserver进行删除操作    


#### Flannel网络
Vxlan模式：Etcd更改ARP和FDB   
brctl show： 查看网桥    
arp –e：Arp 查看mac地址和对应pod的ip网段，对应的网卡   
ip route查看ip对应的网段   
bridge fdb：查看mac对应的node的ip    

cat /proc/sys/net/ipv4/neigh/flannel.1/app_solicit    
查看pod对应的id














