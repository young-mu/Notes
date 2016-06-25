# tls

- server

```javascript
var tls = require('tls');
var fs = require('fs');

var options = {
    key: fs.readFileSync('./keys/server.key'),
    cert: fs.readFileSync('./keys/server.crt'),
    requestCert: true,
    ca: [ fs.readFileSync('./keys/ca.crt') ]
};

var server = tls.createServer(options, function(stream) {
    console.log('server connected! stream',
            stream.authorized ? 'authorized' : 'unauthorized');
    stream.write('welcome!\n');
    stream.setEncoding('utf8');
    stream.pipe(stream);
});

server.listen(8000, function() {
    console.log('server bound 127.0.0.1:8000');
});

```

```shell
erver bound 127.0.0.1:8000
server connected! stream authorized
```

- client

```javascript
var tls = require('tls');
var fs = require('fs');

var options = {
    key: fs.readFileSync('./keys/client.key'),
    cert: fs.readFileSync('./keys/client.crt'),
    ca: [ fs.readFileSync('./keys/ca.crt') ]
};

var stream = tls.connect(8000, options, function() {
    console.log('client connected! stream',
            stream.authorized ? 'authorized' : 'unauthorized');
    process.stdin.pipe(stream);
});

stream.setEncoding('utf8');

stream.on('data', function(data) {
    console.log(data);
});

stream.on('end', function() {
    stream.close();
});
```

```shell
> node client.js
client connected authorized
welcome!
```
