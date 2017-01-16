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

### net socket 完整例子

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

## UDP

node中，TCP是Stream实例，而UDP只是EventEmitter实例
