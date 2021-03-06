# node http

### server (php)

```php
// test.php
<?php
    if (isset($_GET['id']) && isset($_GET['value'])) {
        echo 'GET: ';
        echo 'id = ', $_GET['id'], ', ';
        echo 'value = ', $_GET['value'];
    } else if (isset($_POST['id']) && isset($_POST['value'])) {
        echo 'POST: ';
        echo 'id = ', $_POST['id'], ', ';
        echo 'value = ', $_POST['value'];
    } else {
        echo 'error';
    }
?>
```

#### apache配置

- apache根目录 `/Library/WebServer/Documents/`
- 启动apache `sudo apachectl start`

#### php配置

- httpd配置文件`/etc/apache2/httpd.conf`去掉以下注释（**若更新到Sierra，请忽略这步**）

```c
LoadModule php5_module libexec/apache2/libphp5.so
```

- 测试文件`test.php`

```php
<?php
    echo "hello";
?>
```

- 重启apache `sudo apachectl restart`
- 访问`localhost/test.php`


### server (node)

```javascript
var http = require('http');
var qs = require('querystring');
var url = require('url');

http.createServer(function(req, res) {
    if (req.method == 'GET') {
        console.log('GET request from', req.headers.host);
        var pathname = url.parse(req.url).pathname; // req.url = /get?id=001&value=100
        var query = url.parse(req.url).query; // pathname = /get, query = id=001&value=100
        if (pathname == '/get') {
            data = qs.parse(query); // data = {id = '001', 'value = '100'}
            res.writeHead(200, {'content-type': 'text/plain'});
            res.end('GET: id = ' + data.id + ', value = ' + data.value);
        }
    } else if (req.method == 'POST') {
        console.log('POST request from', req.headers.host);
        var pathname = url.parse(req.url).pathname; // req.url = /post
        if (pathname == '/post') {
            req.on('data', function(chunk) { // chunk is a Buffer
                data = qs.parse(chunk.toString());
                res.writeHead(200, {'content-type': 'text/plain'});
                res.end('POST: id = ' + data.id + ', value = ' + data.value);
            });
        }
    }
}).listen(3000); // if 2nd arg is null, it will listen any connection

console.log('server starts to listen port 3000');
```

```shell
> node server.js
server starts to listen port 3000
GET request from localhost:3000
POST request from localhost:3000
```

### client GET (node)

```javascript
// get_client.js
var http = require('http');
var qs = require('querystring');

var get_data = {
    id: '001',
    value: 100
};
var content = qs.stringify(get_data); // id=001&value=100

var options = {
    hostname: '127.0.0.1',
    port: 80,
    path: '/test?' + content, // test? = test.php? only in Mac
    method: 'GET'
};

var req = http.request(options, function(res) {
    console.log('statusCode: ' + res.statusCode);
    res.setEncoding('utf-8');
    res.on('data', function(data) {
        console.log(data);
    });
});
req.end();

req.on('error', function(err) {
    console.log(err);
});
```

- 测试 apache server

```shell
> node get_client.js
statusCode: 200
GET: id = 001, value = 100
```

- 测试 node server（修改`port:3000`和`path:'/get?'`）

```shell
> node get_client.js
statusCode: 200
GET: id = 001, value = 100
```

### client POST (node)

```javascript
// post_client.js
var http = require('http');
var qs = require('querystring');

var post_data = {
    id: '001',
    value: 100
};
var content = qs.stringify(post_data);

var options = {
    hostname: '127.0.0.1',
    port: 80,
    path: '/test', // test = test.php only in Mac
    method: 'POST',
    headers: { // MUST have this header
        'content-type': 'application/x-www-form-urlencoded'
    }
};

var req = http.request(options, function(res) {
    console.log('statusCode: ' + res.statusCode);
    res.setEncoding('utf-8');
    res.on('data', function(data) {
        console.log(data);
    });
});
req.write(content);
req.end();

req.on('error', function(err) {
    console.log(err);
});
```

- 测试 apache server

```shell
> node post_client.js
statusCode: 200
POST: id = 001, value = 100
```

- 测试 node server（修改`port:3000`和`path:'/post?'`）

```shell
> node post_client.js
statusCode: 200
POST: id = 001, value = 100
```

### http.get

`http.get` 是 `http.request` 的简化版，默认为 `GET`，且不用调用 `req.end`

- 读取文本

```javascript
// http_get_text.js
var http = require('http');

var URI = 'http://localhost/test.txt';
var req = http.get(URI, function(res) {
    console.log('statusCode: ' + res.statusCode);
    res.setEncoding('utf-8');
    res.on('data', function(data) {
        console.log(data);
    });
});

req.on('error', function(err) {
    console.log(err);
});
```

```shell
> node http_get_text.js
test txt
```

- 读取照片流

```javascript
// http_get_stream.js
var http = require('http');
var fs = require('fs');

var URI = 'http://localhost/test.jpg';
var req = http.get(URI, function(res) {
    console.log('statusCode: ' + res.statusCode);
    res.pipe(fs.createWriteStream('./test2.jpg'));
});

req.on('error', function(err) {
    console.log(err);
});
```

```shell
> node http_get_stream.js
```

### 文件server

```javascript
var fs = require('fs');
var http = require('http');
var url = require('url');

var root = '/Users/Young/ABC/node_http/';

http.createServer(function(req, res) {

    console.log('new connection ...');

    console.log('\thost = ', req.headers.host); // 访问目标地址（IP+端口）
    console.log('\tip = ', req.connection.remoteAddress); // 访问者IP
    console.log('\taddr = ', req.connection.remotePort); // 访问者端口
    console.log('\tmethod = ', req.method); // 访问方式 (GET/POST)
    console.log('\turl = ', req.url); // 访问路径

    var file = root + req.url;
    fs.readFile(file, function (error, data) {
        if (error) {
            res.writeHead(404, {'content-type': 'text/plain'});
            res.end('Read Error');
        } else {
            res.writeHead(200, {'content-type': 'text/plain'});
            res.write(data);
            res.end();
        }
    });

}).listen(8000);

console.log('server starts to listen port 8000');
```

- 访问文件 `localhost:8000/test.txt`，成功显示文本
- 访问图片 `localhost:8000/test.jpg`，成功显示图片
- 访问错误 `localhost:8000/test.xxx`，不存在，显示`Read Error`