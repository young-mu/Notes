# node https

### 自定义CA

- 生成一个CA私钥`ca.key`和其对应的公钥`ca.pub`

```bash
openssl genrsa -out ca.key 1024
openssl rsa -in ca.key -pubout -out ca.pub
```

- 用CA私钥`ca.key`和辅助信息配置文件`ca_csr.cfg`生成CSR证书请求文件`ca.csr`

```bash
openssl req -new -key ca.key -config ca_csr.cfg -out ca.csr
```

```c
// ca_csr.cfg
[req]
distinguished_name = req_distinguished_name
[req_distinguished_name]
countryName = Country Name (2 letter code)
countryName_default = CN
localityName = Locality Name (eg, city)
localityName_default = Shanghai
organizationalUnitName = Organizational Unit Name (eg, section)
organizationalUnitName_default = Ruff
commonName = Common Name (e.g. server FQDN or YOUR name)
commonName_default = Young
```

- 用CA私钥`ca.key`和CSR文件`ca.csr`生成CA证书`ca.crt`

```bash
openssl x509 -req -signkey ca.key -in ca.csr -out ca.crt
```

- 用CA证书`ca.crt`也可以获得CA公钥`ca.pub2`（与`ca.pub`一致）

```bash
openssl x509 -in ca.crt -pubkey -out /dev/null > ca.pub2
```

### 自定义Server证书

- 生成Server私钥`server.key`和其对应的公钥`server.pub`

```bash
openssl genrsa -out server.key 1024
openssl rsa -in server.key -pubout -out server.pub
```

- 用Server私钥`server.key`和辅助信息配置文件`server_csr.cfg`生成Server证书请求文件`server.csr`

```bash
openssl req -new -key server.key -config server_csr.cfg -out server.csr
```

```c
// server_csr.cfg
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
[req_distinguished_name]
countryName = Country Name (2 letter code)
countryName_default = CN
localityName = Locality Name (eg, city)
localityName_default = Shanghai
organizationalUnitName = Organizational Unit Name (eg, section)
organizationalUnitName_default = Ruff
commonName = Common Name (e.g. server FQDN or YOUR name)
commonName_default = Lucia
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
IP.1 = 127.0.0.1
```

- 用CA私钥`ca.key`和Server的CSR文件`server.csr`生成Server证书`server.crt`

```bash
openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -in server.csr -extensions v3_req -extfile server_csr.cfg -out server.crt
```

- 用Server证书`server.crt`也可以获得Server公钥`server.pub2`（与`server.pub`一致）

```bash
openssl x509 -in server.crt -pubkey -out /dev/null > server.pub2
```

### 测试

- server (node)

```javascript
var https = require('https');
var fs = require('fs');

var options = {
    key: fs.readFileSync('./keys/server.key'),
    cert: fs.readFileSync('./keys/server.crt'),
};

https.createServer(options, function(req, res) {
    console.log('ok...');
    res.writeHead(200);
    res.end('hello world');
}).listen(3000, '127.0.0.1', function() {
    console.log('start to listen port 3000');
});
```

```shell
> node server.js
start to listen port 3000
ok...
```

- client (shell)

```shell
# -k|--insecure: 由于系统没有内置私有证书，因此如果不加该参数会报错
> curl -k|--insecure https://127.0.0.1:3000
hello world
```

```shell
# --cacert <crt_path>: 加载私有证书
> curl --cacert keys/ca.crt https://127.0.0.1:3000
hello world
```

- client (node)

```javascript
var https = require('https');
var fs = require('fs');

var options = {
    host: 'localhost', // 不能设置127.0.0.1，server.crt中不支持
    port: 3000,
    path: '/',
    method: 'GET',
    // rejectUnauthorized: false // 如果不指定ca，需要加该参数，否则出错
    ca: [fs.readFileSync('./keys/ca.crt')],
};

options.agent = new https.Agent(options);

var req = https.get(options, function(res) {
    res.setEncoding('utf-8');
    res.on('data', function(data) {
        console.log(data);
    });
});
req.on('error', function(err) {
    console.log(err);
});
```

```bash
> node client.js
hello world
```

### TLS验证

#### 单向验证，Client验证Server

- 对于Client，它需要拥有CA证书`ca.crt`
- 对于Server，它需要拥有Server私钥`server.key`和Server证书`server.crt`

1. Client向Server发送**`client random`**
2. Server向Client发送**`server random`**和Server证书`server.crt`
3. Client用**CA证书`ca.crt`中抽出的CA公钥**验证Server证书`server.crt`的有效性（**验证**）
4. CLient从**Server证书`server.crt`中抽出Server公钥**，加密**`premaster secret`**，然后发送给Server
4. Server用Server私钥`server.key`解密`premaster secret`

#### 双向验证，Client和Server互验证

- 对于Client，它需要拥有CA证书`ca.crt`，Client私钥`client.key`和Client证书`client.crt`
- 对于Server，它需要拥有CA证书`ca.crt`，Server私钥`server.key`和Server证书`server.crt`

1. Client向Server发送**`client random`**
2. Server向Client发送**`server random`**和Server证书`server.crt`
3. Client从CA证书`ca.crt`中抽出CA公钥验证Server证书`server.crt`的有效性（**验证**）
4. Client从Server证书`server.crt`中抽出Server公钥，加密**`premaster secret`**，然后发送给Server，并发送Client证书`client.crt`
5. Server从CA证书`ca.crt`中抽出CA公钥验证Client证书`client.crt`的有效性（**验证**）
6. Server用Server私钥`server.key`解密`premaster secret`

### 参考

1. [SSL/TLS协议运行机制的概述](http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)
2. [用node.js创建自签名的https服务器](http://cnodejs.org/topic/54745ac22804a0997d38b32d)
