nat 类型分4种   

1、全锥形 full cone   

　　　A 与 主机B交互，nat转换 A的内部地址及端口为  ip1 port1，ip1和port1为对外地址，任何机器能访问。

2、ip 受限制（对B而言）    

          A 与 主机B交互，nat转换 A的内部地址及端口为  ip1 port1，B要想访问A，需要A先访问过B（不管是否失败），并且B的ip不能变，但是B的端口可以变。    

3、端口 受限制（对B而言）    

          A 与 主机B交互，nat转换 A的内部地址及端口为  ip1 port1，B要想访问A，需要A先访问过B（不管是否失败），并且B的ip不能变，B端口也不能变。    

4、对称型   

      以上种，再nat转换时，都会转换成ip1 port1的形式，但是这种，访问主机B,转为ip1,port1，访问主机C,转为ip1,port2,或者ip2,port2    

 　　  A 与 主机B交互，nat转换 A的内部地址及端口为  ip1 port1，B要想访问A,，需要A先访问过B（不管是否失败），并且B的ip不能变，B端口也不能变   

nat穿透：    

　　局域网的A访问一个外部主机，这个主机返回A它的nat转换后的ip1 port1。另一个局域网B访问外部主机（stun），外部主机返回B被nat转换后的ip2，port2。A访问B，就是A去访问ip2，port2。B访问A,就是访问ip1,port1。

   非对称与非对称：由于A（客户机）无论访问哪个主机，A的nat都是将它转换为ip1 port1（ip1 与port1都不会变化），因此A B都去访问stun，得到的ip与port不会变化。因此可以打洞。   

　　一端对称，一端非对称：A(非对称Nat，且只能是ip不变，port变化的那种情况)，一端非对称B,且只能是 full cone 或者ip受限。首先B访问A，B记录A的ip1（只要ip1的信息发过来，就能收到），B的数据必然被A的nat丢掉，但是A就可以访问B了，这儿假定A的nat转换后的ip是不变的（A一般都是这种）。    

   对称与对称：A去访问stun 得到的为Aip1 Aport1。B去访问stun,得到Bip1 Bport1。 A去访问B， A net记录Bip1 Bport1， B去访问A，自身nat将其转换为BIp2, Bport2（Bip1,Bport1是访问stun得到的），但是A的洞只为Bip1,Bport1留着，Bip2,Bport2根本链接不上A,所以2个都改为对称Nat，根本没法打穿。    
