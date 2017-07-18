## Crypto

### 功能

- Hash
- HMAC
- Cipher
- Decipher
- Sign
- Verify

> 这里把 HASH 和 HMAC 列为加密算法同级，是因为他们属于不可逆算法（主体都为 HASH 算法）

### Hash

最简单的加密算法，以一个**任意长度**的字符串作为输入，生成一个**固定长度**加密串作为输出

#### 方法列表

`crypto.getHashes();`

- md5
- sha1
- sha512
- ...

#### 示例

```javascript
var crypto = require('crypto');

var algorithm = 'md5';
var buf = 'foobar';

var md5 = crypto.createHash(algorithm);
md5.update(buf);
var out = md5.digest('hex');

console.log(out);
```

### HMAC

HMAC (Keyed-Hash Message Authentication Code)

更复杂些的加密算法，以一个密钥和一个消息作为输入，生成一个加密串作为输出

#### 示例

```javascript
var crypto = require('crypto');

var algorithm = 'sha1';
var key = 'abc';
var buf = 'foo';

var hmac = crypto.createHash(algorithm, key);
hmac.update(buf);
var out = hmac.digest('hex');

console.log(out);
```

### Cipher & Decipher

和 HASH/HMAC 不同，指既能加密又能解密的算法

#### 方法列表

`crypto.getCiphers();`

- aes128
- blowfish
- des
- ...

#### 示例

```javascript
var crypto = require('crypto');

var algorithm = 'blowfish';
var key = 'abc';
var buf = 'Hello World';

var encrypted = '';
var enc = crypto.createCipher(algorithm, key);
encrypted += enc.update(buf, 'utf8', 'hex'); // input, output
encrypted += enc.final('hex'); // output

console.log(encrypted);

var decrypted = '';
var dec = crypto.createDecipher(algorithm, key);
decrypted += dec.update(encrypted, 'hex', 'utf8');
decrypted += dec.final('utf8');

console.log(decrypted);
```

### Sign & Verify

判断数据在传输过程中，是否真实和完整。

数字签名（signature）的具体过程：

> 也可对内容直接加密，而不对生成的 Hash 值加密

1. 数据进行 Hash 处理得到 Hash 值
2. 私钥对 Hash 值进行加密，得到数字签名
3. 传输（数据 + 数字签名）
4. 公钥对数字签名进行解密，得到 Hash 值（a）
5. 数据进行 Hash 处理得到 Hash 值（b），与（a）进行比对

数字证书（certificate）的具体过程：

> 防止公钥被篡改，发送者将公钥／私钥到证书中心 CA 进行认证，生成数字证书

1. 数据进行 Hash 处理得到 Hash 值
2. 私钥对 Hash 值进行加密，得到数字签名
3. 传输（数据 + 数据签名 + **数字证书**）
4. **用 CA 公钥对数字证书解密，得到公钥**
5. 公钥对数字签名进行解密，得到 Hash 值（a）
6. 数据进行 Hash 处理得到 Hash 值（b），与（a）进行比对

**1/2 为 Sign 过程， 4/5/6 为 Verify 过程**

![Flow](http://blog.fens.me/wp-content/uploads/2015/03/screenshot.62.png)

#### 示例

- Sign

```javascript
var crypto = require('crypto');
var sign = crypto.createSign('RSA-SHA256');

var priKey = getPrivateKeySomehow();

sign.update('some data to sign');

console.log(sign.sign(priKey, 'hex'));
```

- Verify

```javascript
var crypto = require('crypto');
var verify = crypto.createVerify('RSA-SHA256');

var pubKey = getPublicKeySomehow();
var signature = getSignatureSomehow();

verify.update('some data to verify');

console.log(verify.verify(pubKey, signature)); // true|false
```

### 参考

1. [crypto API](https://nodejs.org/dist/latest-v6.x/docs/api/crypto.html)
2. [crypto 某博客](http://blog.fens.me/nodejs-crypto/)