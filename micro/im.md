敏感词服务，用户服务，推送服务，消息记录服务

发消息接收到发mq，从mq取出，生成id，存redis的sort set，   
发mq触发另一个连接接收消息    
如果是离线就发离线消息   
敏感词过滤， 通过redis记录用户的连接信息，接入层连接哪个用户，然后发送到哪个接入层，没有登陆的用户会收到离线推送 

![image](https://user-images.githubusercontent.com/24344673/149328174-409d623a-1e11-49b4-862d-3929bdf8bffa.png)
