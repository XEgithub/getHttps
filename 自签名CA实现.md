##关于自签名CA证书实现流程    
###实现原理  
SSL是利用公开密钥的加密技术（RSA）来作为用户端与服务器端在传送机密资料时的加密通讯协定,通过基于ssl开发的开放源代码的软件库包openssl来进行安全通信，避免窃听，同时确认另一端连接者的身份，来达到加密传输的目的。密钥和证书管理是PKI的一个重要组成部分，OpenSSL为之提供了丰富的功能，支持多种标准。通过openssl，加密传输数据，保障通信安全，并通过自签名CA证书管理平台，有效的识别连接者的身份，防止数据传输过程中被劫持后被中间人攻击。
关于RSA算法安全性：RSA中的每一个公钥都有唯一的私钥与之对应，任一公钥只能解开对应私钥加密的内容。换句话说，其它私钥加密的内容，这个公钥是解不开的。公钥加密然后私钥解密，可以用于通信中拥有公钥的一方向拥有私钥的另一方传递机密信息，不被第三方窃听。 公钥加密数据,然后私钥解密的情况被称为加密解密,私钥加密数据,公钥解密一般被称为签名和验证签名.
![原理](images/jiami.png)

步骤：     
CA生成自己的私钥、生成自签名证书    
CA修改配置文件后，可以向其他申请者签署证书    
web服务器，生成自己的私钥、证书申请文件，将证书申请根文件发送给CA请求CA签署证书。

###实现过程###
基础环境：对外提供web访问的服务器均须具有开放的443端口，其中一台服务器作为ca(ca.crt)证书机构，给其他的服务器(server.crt)进行其所生成证书进行签发认证。
服务器环境：centos7
web环境：pache/2.4.6 (CentOS)
>1.在Apache服务器下，首先检查是否存在mod_ssl模块，不存在则安装：
``` httpd -M|grep ssl_module``` 

>>```yum -y install mod_ssl```

>2.搭建CA服务器：   
>>在负责进行CA签名认证的服务器上，为CA建立自签名证书:   
>>CA私钥文件: (umask 077; openssl genrsa -out /etc/pki/CA/private/cakey.pem 2048)    
>>在根CA服务器上创建密钥，密钥的位置必须为/etc/pki/CA/private/cakey.pem，这个是openssl.cnf中指定的路径，只要与配置文件中指定的匹配即可。   
>>  ![creatcrt](images/carts.png)       
>> 生成CA自签名证书文件: openssl req -new -x509 -key /etc/pki/CA/private/cakey.pem -out /etc/pki/CA/certs/ca.crt -days 365   
>> 可以加证书过期时间选项 "-days 365",代表证书有效时间1年    
>> 生成证书管理索引文件 touch /etc/pki/CA/index.txt   
>>生成证书序列文件 echo "00">>/etc/pki/CA/serial

>3.为web服务器端准备公钥、私钥,web服务器的server.csr文件必须经过CA的签名才可形成证书.
>>  生成服务器端私钥    
``` (umask 077; openssl genrsa -out  /etc/pki/tls/private/server.key 2048) ```   
>>  生成服务器端证书签名请求文件(csr文件):     
``` openssl req -new -key /etc/pki/tls/private/server.key -out server.csr  ```

>> 生成Certificate Signing Request（CSR）,生成的csr文件交给CA签名后形成服务端自己的证书.    
>>    ![CA生成](images/creatca.png)    
>>    注意: Common Name (e.g. server FQDN or YOUR name) []: 这一项，是最后可以访问的域名，如果是为了给网站生成证书，需要写成 xxxx.com 。因为SSL会话只能基于IP地址和端口进行，基于同一个IP的多个虚拟网站，只有一个能进行SSL会话。如有多个web环境，则需要在ssl.conf下配置域名。     

>4.利用CA证书文件对CA证书请求文件进行进行签名：    
>>将web服务器端CA签名请求文件传送给CA服务器,然后签名：   
>> openssl ca -in server.csr -out server.crt -cert /etc/pki/CA/certs/ca.crt -keyfile /etc/pki/CA/private/cakey.pem -days 365   
![签名成功](images/cacsr.png)
 




>5.配置并启用证书：  
>>将签完名生成的server.crt文件传回服务器，并修改/etc/httpd/conf.d/ssl.conf文件中证书的路径：   
![ca](images/saveca.png)    
![path](images/path.png)


>6.停止防火墙服务   
>>setenforce 0   
>>systemctl stop firewalld.service     
>>重启httpd服务 ：systemctl restart httpd   