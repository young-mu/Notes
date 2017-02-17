# node socket

- **net** (TCP)
- **dgram** (UDP)

## TCP

### Net socket

- server (node)

```javascript
var server = net.createServer();
server.listen(6789);
```

- client (node)

```javascript
var client = net.connect({port: 6789, host: '127.0.0.1'});
```

- client (shell)

```shell
> telnet 127.0.0.1 6789
```

### Domain socket

- server (node)

```javascript
var server = net.createServer();
server.listen('/tmp/echo.sock');
```

- client (node)

```javascript
var client = net.connect({path: '/tmp/echo.sock'});
```

- client (shell)

```shell
> nc -U /tmp/echo.sock
```

### net socket example 1

- server (node)

```javascript
var net = require('net');

var PORT = 2222;

var server = net.createServer();
server.listen(PORT);

server.on('connection', function (sock) {
    console.log('\n> connected from: ' + sock.remoteAddress + ':' + sock.remotePort);

    sock.on('data', function (data) {
        console.log('> recv: ' + data);
        sock.write('I\'m server!');
    });

    sock.on('close', function (data) {
        console.log('> closed by: ' + sock.remoteAddress + ':' + sock.remotePort);
    });
});

server.on('listening', function() {
    console.log('> server starts to listen on ' + '*' +':'+ PORT);
});

server.on('close', function() {
    console.log('> server is closed ...');
});
```

```shell
$ node server.js
> server starts to listen on *:2222

> connected from: ::ffff:127.0.0.1:53754
> recv: I'm client!
> closed by: ::ffff:127.0.0.1:53754
```

- client (node)

```javascript
var net = require('net');

var HOST = '127.0.0.1';
var PORT = 2222;

var client = net.createConnection({port: PORT, host: HOST});

client.on('connect', function() {
    console.log('> connect to: ' + HOST + ':' + PORT);
    client.write('I\'m client!');
});

client.on('data', function (buffer) {
    console.log('> recv: ' + buffer.toString());
    client.destroy();
});

client.on('close', function() {
    console.log('> close connection');
});

client.on('error', function(error) {
    console.log('> error: ' + error);
});
```

```shell
$ node client.js
> connect to: 127.0.0.1:2222
> recv: I'm server!
> close connection
```

### net socket example (file)

- server (node)

```javascript
var net = require('net');
var fs = require('fs');

var server = net.createServer();
server.listen(8888);

var file = './dst';
var ws = fs.createWriteStream(file);

var idx = 0;
var recvSize = 0;

server.on('connection', function (sock) {
    console.log('\n> connected from `' +
        sock.remoteAddress + ':' + sock.remotePort + '`');

    sock.on('data', function (chunk) {
        recvSize += chunk.length;
        console.log('> recv #' + idx + ' chunk (' + chunk.length + ' bytes)');
        if (ws.write(chunk) === false) {
            sock.pause();
        }
        idx++;
    });

    ws.on('drain', function () {
        sock.resume();
    });

    sock.on('close', function () {
        console.log('> closed by `' +
            sock.remoteAddress + ':' + sock.remotePort + '`');
    });

    sock.on('end', function () {
        console.log('> recv done (' + recvSize + ' bytes in total)');
        idx = 0;
        recvSize = 0;
    });
});

server.on('listening', function() {
    console.log('> server starts to listen on *:8888');
});

server.on('close', function() {
    console.log('> server is closed ...');
});
```

- client (node)

```javascript
var net = require('net');
var fs = require('fs');

var HOST = '127.0.0.1';
var PORT = 8888;

var file = './src';

var rs = fs.createReadStream(file);

var idx = 0;
var sendSize = 0;

var client = net.createConnection({port: PORT, host: HOST});

client.on('connect', function() {
    console.log('> connect to: ' + HOST + ':' + PORT);

    rs.on('data', function (chunk) {
        sendSize += chunk.length;
        console.log('> send #' + idx + ' chunk (' + chunk.length + ' bytes) ...');
        if (client.write(chunk) === false) {
            rs.pause();
        }
        idx++;
    });

    rs.on('end', function () {
        console.log('> send done (' + sendSize + ' bytes in total)');
        client.destroy();
    });
});

client.on('data', function (buffer) {
    console.log('> recv: ' + buffer.toString());
});

client.on('drain', function () {
    rs.resume();
});

client.on('close', function() {
    console.log('> close connection');
});

client.on('error', function(error) {
    console.log('> error: ' + error);
});
```

- test

```shell
> server starts to listen on *:8888

> connected from `::ffff:127.0.0.1:56646`
> recv #0 chunk (65536 bytes)
> recv #1 chunk (65536 bytes)
> recv #2 chunk (65536 bytes)
> recv #3 chunk (65536 bytes)
> recv #4 chunk (65536 bytes)
> recv #5 chunk (65536 bytes)
> recv #6 chunk (65536 bytes)
> recv #7 chunk (14824 bytes)
> recv done (473576 bytes in total)
> closed by `::ffff:127.0.0.1:56646`
```

```shell
> connect to: 127.0.0.1:8888
> send #0 chunk (65536 bytes) ...
> send #1 chunk (65536 bytes) ...
> send #2 chunk (65536 bytes) ...
> send #3 chunk (65536 bytes) ...
> send #4 chunk (65536 bytes) ...
> send #5 chunk (65536 bytes) ...
> send #6 chunk (65536 bytes) ...
> send #7 chunk (14824 bytes) ...
> send done (473576 bytes in total)
> close connection
```

## UDP

node中，TCP是Stream实例，而UDP只是EventEmitter实例
