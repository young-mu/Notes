# crypto basic

## 加密方法

### 单钥加密

也叫对称加密，使用同一套密码进行加解密，常用的算法有`AES`，`RC4`，`DES`和`3DES`等，单钥加密由于只有一套密码，因此密钥的保存非常重要，一旦密钥泄露，密码就被破解

### 双钥加密

也叫非对称加密，使用两套密码进行加解密，常用的算法有`RSA`，`DH`，`ECDHE`和`ECDH`等，双钥加密有两套密码，一个是公开的公钥（Public key），另一个是不公开的私钥（Private key），特点如下：

1. 公钥和私钥是一一对应关系，有一把公钥就一定有一把独一无二的私钥，反之成立
2. 所有的公钥私钥对都是不同的
3. 用公钥可以解开私钥加密的信息，用私钥也可以解开公钥加密的信息
4. 同时生成公钥和私钥比较容易，但是从公钥推算出私钥是不可能的
5. 私钥用来数字签名（内容经Hash函数生成Hash值，再用私钥生成signature），公钥可以将signature解密为生成的Hash值
6. 任何人都可以生成公钥/私钥对，因此防止有人散布伪造的公钥骗取信任，需要一个可靠的第三方机构来生成经过认证的公钥/私钥对，如美国数字服务认证商Verisign，主要业务是分发RSA数字证书（详细见数字签名和数字证书）

### 数字签名（Digital Signature）

A自己生成一个公钥/私钥对，公钥交给B，私钥自己保留，文件内容通过Hash函数生成Hash值，Hash值再通过私钥生成数字签名，A将文件（包含文件内容和数字签名）发送给B，B用公钥解密数字签名为Hash值，用Hash函数处理文件内容得到Hash值，两者相等，证明此份文件是A发来的(AB公有Hash函数和公钥）

### 数字证书（Digital Certificate）

为了防止公钥被别人恶意篡改以骗取信任（比如C用自己的公钥和B换，B以为此时的公钥是A的，但实际是C的），B和A可以到证书中心（Certificate Authority，CA）为A的公钥做认证，CA采用自己的私钥对A的公钥和一些相关信息加密，生成数字证书给A和B，这样A以后发送文件包含**文件内容**，**数字签名**和**数字证书**三部分

B就可以先利用CA的公钥（可靠的第三方机构的，无法被人恶意篡改）对数字证书解密得到A的公钥
利用A的公钥解密数字签名，得到Hash值，再利用Hash函数处理文件内容得到Hash值，两者相等，证明此文件是A发来的

## SSL/TLS

### SSL（Secure Sockets Layer，安全套接层协议

NetScape推出第一个Web浏览器时，提出的SSL协议v1.0，目的是保证两个应用在Internet间通信的保密性和可靠性，可在服务器和客户端同时实现支持，SSL协议建立在传输层TCP协议之上，与应用层协议独立无关，应用层协议（如HTTP，FTP和TELNET等）可透明地建立在SSL协议之上，SSL协议在应用层协议通信之前就已完成加密密码，通信密钥的协商及服务器认证工作

### TLS（Transport Layer Security，安全传输层协议）

该协议目的也是保证两个通信应用程序至今提供数据完整性和保密性，由两层组成，TLS记录协议（TLS Record）和TLS握手协议（TLS Handshake），底层的是TLS Record，位于可靠的传输协议（如TCP协议）上面，与应用层协议无关

### SSL/TLS的关系

SSL由Netscape于1995年发布，TLS由互联网协会ISOC（Internet SOCiety）于1999年发布，版本对应关系如下：

| SSL     | TLS     |
| ------- | ------- |
| SSLv3.1 | TLSv1.0 |
| SSLv3.2 | TLSv1.1 |
| SSLv3.3 | TLSv1.2 |

### 为什么使用SSL/TLS

不使用SSL/TLS的HTTP协议通信，会面临三大风险（对应三个阶段）：

1. 窃听风险（Eavesdropping）：第三方可以获知通信信息（过程）
2. 篡改风险（Tampering）：第三方可以修改通信内容（接收）
3. 冒充风险（Pretending）：第三方可以冒充他人身份参与通信（发送）

因此SSL/TTL就是解决这三大风险设计的，希望达到的目的：

1. **message privacy**，所有信息都是加密传播，第三方无法窃听
2. **message integrity**，具有检验机制，一旦被篡改，通信双方会立刻发现
3. **mutual authentication**，配备身份证书，防止身份被冒充

### 运行原理

采用公钥加密法，即客户端向服务器索要公钥，然后用公钥加密信息，服务器收到密文后，用自己的私钥解密

服务器发送公钥给客户端时，为保证公钥不被篡改，将公钥放在数字证书中，为了减少公钥加密计算量问题，在每次对话中，客户端和服务器都生成一个对话密钥（Session Key）用来加密信息，该对话密钥是对称加密的，所有运算速度非常快，而公钥只用于加密对话密钥本身，这样就减少了加密运算的时间消耗

### 握手阶段的详细步骤

1. Client给Server发送协议版本号，客户端随机数`Client Random`以及Client支持的加密方法
2. Server确认双方使用的加密方法，并给出数字证书以及服务器随机数`Server Random`
3. Client确定数字证书有效并提取Server公钥，生成新客户端随机数`Premaster Secret`并使用Server公钥加密发送给Server
4. Server使用自己私钥解密Premaster Secret（Client和Server现在都拥有三个随机数，目的是为了更随机）
5. Client和Server根据约定的加密方法，使用前面三个随机数，生成`Master Secret`，再用它生成对话密钥，用来加密接下来的整个对话过程

![SSL](http://image.beekka.com/blog/2014/bg2014092003.png)

## 各种算法

### 非对称密钥交换算法

对称加密的一个很大的问题是如何安全生成和保管对称密钥，非对称密钥交换算法就是为了解决此问题的，常见的算法有`RSA`，`DH`，`ECDHE`和`ECDH`等

- `RSA` 需要比较大的素数（2048位），因此比较消耗CPU
- `DH` Diffie-Hellman算法，比较消耗CPU（相当做两次RSA）
- `ECDHE` 使用椭圆曲线（ECC）的DH快速算法
- `ECDH` 可以理解为低配置的ECDHE算法（不推荐使用）

通常所说的`ECDHE_RSA`指的是使用`ECDHE`做密钥协商，使用`RSA`进行签名认证

### 块加密算法

对称加密可分为流加密和块加密，流加密常见算法有`RC4`（不再安全）和`ChaCha20`，块加密常见算法有`AES`，`DES`和`3DES`等

`AES`属于主流的块加密算法，其中共有五种分组加密模式，其中主流有两种：

- **CBC（Cipher Block Chaining，密码分组链接）**

1. 原理：明文跟初始化向量异或，再用密钥进行加密，结果作为下个块的初始化向量
2. 特点：只能做串行操作，而且相对不安全
3. 举例：`AES-128-CBC`指的是AES对称加密算法中密钥长度（同时也指块大小）为128位并且分组模式为CBC

- **CTR（CounTeR，计数器）**

1. 原理：用密钥对输入的计数器加密，然后同明文异或得到密文
2. 特点：可做并行操作，但计算量大
3. 扩展：`GCM（Galois/Counter Mode)`**（主流）**是CTR模式的扩展
4. 举例：`AES-256-GCM`指的是AES对称加密算法中密钥长度为256位并且分组模式为CTR的扩展GCM

### 消息验证算法

消息验证算法`MAC`，用于对验证一段内容的正确完整性，有两种模式：

- **HMAC（Hash-based Message Authenticaation Code）**

基于Hash函数的MAC算法，如`HMAC-SHA256`指的是HMAC算法中采用的Hash算法为256位的SHA

- **AEAD（Authenticated-Encryption with Additional Data）**

加密和MAC直接集成为一个算法的模式，GCM就是AEAD中最重要的一种，因此GCM不需要MAC算法

## Reference

1. [密码学笔记](http://www.ruanyifeng.com/blog/2006/12/notes_on_cryptography.html)
2. [数字签名](http://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html)
3. [SSL/TLS](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)
4. [图解TLS协议](http://www.ruanyifeng.com/blog/2014/09/illustration-ssl.html)
5. [TLS协议分析](http://blog.csdn.net/yzhou86/article/details/51211167)
6. [AES五种加密模式](http://www.cnblogs.com/starwolf/p/3365834.html?utm_source=tuicool&utm_medium=referral)
7. [AES之CBC和CTR](http://www.metsky.com/archives/585.html)
8. [非对称密钥交换](http://studygolang.com/articles/2984)
9. [OCSP Stapling](https://segmentfault.com/a/1190000004045710)
10. [Session ID和Session Ticket](http://www.ruanyifeng.com/blog/2014/09/illustration-ssl.html)