# node http

### apache配置

- apache根目录 `/Library/WebServer/Documents/`
- 启动apache `sudo apachectl start`

### php配置

- httpd配置文件`/etc/apache2/httpd.conf`去掉以下注释

```c
LoadModule php5_module libexec/apache2/libphp5.so
```

- 测试文件`test.php`

```php
<?php
	echo "Hello";
?>
```

- 重启apache `sudo apachectl restart`
- 访问`localhost/test.php`

### php测试文件

```php
// test_get.php
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

### http GET

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
    path: '/test?' + content, // test? = test.php?
    method: 'GET'
};

var req = http.request(options, function(res) {
    console.log('statusCode: ' + res.statusCode);
    //console.log('headers: ' + JSON.stringify(res.headers));
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

- 测试

```bash
> node get_client.js
statusCode: 200
value = 100
```

### http POST

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
    path: '/test',
    method: 'POST',
    headers: { // MUST have this header
        'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
    }
};

var req = http.request(options, function(res) {
    console.log('statusCode: ' + res.statusCode);
    //console.log('headers: ' + JSON.stringify(res.headers));
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

- 测试

```bash
> node post_client.js
statusCode: 200
GET: id = 001, value = 100
```
