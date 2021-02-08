# https

#### https流程
![image](https://github.com/yincongcyincong/ms/blob/main/image/https.png)
#### (1).client_hello    
客户端发起请求，以明文传输请求信息，包含版本信息，加密套件候选列表，压缩算法候选列表，随机数，扩展字段等信息，相关信息如下：    
(a)支持的最高TSL协议版本version，从低到高依次 SSLv2 SSLv3 TLSv1 TLSv1.1 TLSv1.2，当前基本不再使用低于 TLSv1 的版本;    
(b) 客户端支持的加密套件 cipher suites 列表， 每个加密套件对应前面 TLS 原理中的四个功能的组合：认证算法 Au (身份验证)、密钥交换算法 KeyExchange(密钥协商)、对称加密算法 Enc (信息加密)和信息摘要 Mac(完整性校验);    
(c)支持的压缩算法 compression methods 列表，用于后续的信息压缩传输;   
(d) 随机数 random_C，用于后续的密钥的生成;    
(e)扩展字段 extensions，支持协议与算法的相关参数以及其它辅助信息等，常见的 SNI 就属于扩展字段，后续单独讨论该字段作用。        
#### (2).server_hello+server_certificate+sever_hello_done
(a) server_hello, 服务端返回协商的信息结果，包括选择使用的协议版本 version，选择的加密套件 cipher suite，选择的压缩算法 compression method、随机数 random_S 等，其中随机数用于后续的密钥协商;    
(b)server_certificates, 服务器端配置对应的证书链，用于身份验证与密钥交换;               
(c)server_hello_done，通知客户端 server_hello 信息发送结束;       
#### (3).证书校验    
客户端验证证书的合法性，如果验证通过才会进行后续通信，否则根据错误情况不同做出提示和操作，合法性验证包括如下：   
(a)[证书链]的可信性 trusted certificate path，方法如前文所述;    
(b)证书是否吊销 revocation，有两类方式离线 CRL 与在线 OCSP，不同的客户端行为会不同;      
(c)有效期 expiry date，证书是否在有效时间范围;       
(d)域名 domain，核查证书域名是否与当前的访问域名匹配，匹配规则后续分析;              
#### (4).client_key_exchange+change_cipher_spec+encrypted_handshake_message
(a) client_key_exchange，合法性验证通过之后，客户端计算产生随机数字 Pre-master，并用证书公钥加密，发送给服务器;   
(b) 此时客户端已经获取全部的计算协商密钥需要的信息：两个明文随机数 random_C 和 random_S 与自己计算产生的 Pre-master，计算得到协商密钥;    enc_key=Fuc(random_C, random_S, Pre-Master)    
(c) change_cipher_spec，客户端通知服务器后续的通信都采用协商的通信密钥和加密算法进行加密通信;    
(d) encrypted_handshake_message，结合之前所有通信参数的 hash 值与其它相关信息生成一段数据，采用协商密钥 session secret 与算法进行加密，然后发送给服务器用于数据与握手验证;       
#### (5).change_cipher_spec+encrypted_handshake_message   
(a) 服务器用私钥解密加密的 Pre-master 数据，基于之前交换的两个明文随机数 random_C 和 random_S，计算得到协商密钥:enc_key=Fuc(random_C, random_S, Pre-Master);    
(b) 计算之前所有接收信息的 hash 值，然后解密客户端发送的 encrypted_handshake_message，验证数据和密钥正确性;   
(c) change_cipher_spec, 验证通过之后，服务器同样发送 change_cipher_spec 以告知客户端后续的通信都采用协商的密钥与算法进行加密通信;   
(d) encrypted_handshake_message, 服务器也结合所有当前的通信参数信息生成一段数据并采用协商密钥 session secret 与算法加密并发送到客户端;       
#### (6).握手结束
客户端计算所有接收信息的 hash 值，并采用协商密钥解密 encrypted_handshake_message，验证服务器发送的数据和密钥，验证通过则握手完成;    
#### (7).加密通信
开始使用协商密钥与算法进行加密通信。 注意：
(a) 服务器也可以要求验证客户端，即双向认证，可以在过程2要发送 client_certificate_request 信息，客户端在过程4中先发送 client_certificate与certificate_verify_message 信息，证书的验证方式基本相同，certificate_verify_message 是采用client的私钥加密的一段基于已经协商的通信信息得到数据，服务器可以采用对应的公钥解密并验证;  
(b) 根据使用的密钥交换算法的不同，如 ECC 等，协商细节略有不同，总体相似;
(c) sever key exchange 的作用是 server certificate 没有携带足够的信息时，发送给客户端以计算 pre-master，如基于 DH 的证书，公钥不被证书中包含，需要单独发送;
(d) change cipher spec 实际可用于通知对端改版当前使用的加密通信方式，当前没有深入解析;
(e) alter message 用于指明在握手或通信过程中的状态改变或错误信息，一般告警信息触发条件是连接关闭，收到不合法的信息，信息解密失败，用户取消操作等，收到告警信息之后，通信会被断开或者由接收方决定是否断开连接。   

#### 具体讲解
1.客户端验证server是他自身而不是伪造，用rsa验证服务器证书，这里用到非对称加密    
2.使用DCHE算法交换密钥，简单来说就是p=n*m,通过只交换p，服务端解出n，客户端解出m，然后客户端和服务端用m和n算出对称加密密钥，没有用到加密，用到了密钥交换算法    
3.传输数据，根据前面的密钥各自进行加密，服务端给客户端的数据加密的密钥和客户端给服务端的数据加密的密钥是不相同的
