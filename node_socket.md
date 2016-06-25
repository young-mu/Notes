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

### Net socket 完整例子

- server (node)

```javascript
var net = require('net');

var HOST = '127.0.0.1';
var PORT = 6969;

var server = net.createServer();

server.on('connection', function(sock) {
    console.log('CONNECTED FROM: ' +
                sock.remoteAddress + ':' + sock.remotePort);
    sock.on('data', function(data) {
        console.log('DATA: ' + data);
        sock.write('You said "' + data + '"');
    });
    sock.on('close', function(data) {
        console.log('CLOSED BY: ' +
                    sock.remoteAddress + ':' + sock.remotePort);
    });
}).listen(PORT, HOST);

server.on('listening', function() {
    console.log('Server starts to listen on ' + HOST +':'+ PORT);
});
```

```shell
> node server.js
Server starts to listen on 127.0.0.1:6969
CONNECTED FROM: 127.0.0.1:61626
DATA: I'm Young!
CLOSED BY: 127.0.0.1:61626
```

- client (node)

```javascript
var net = require('net');

var HOST = '127.0.0.1';
var PORT = 6969;

var client = new net.Socket();

client.connect(PORT, HOST, function() {
    console.log('CONNECT TO: ' + HOST + ':' + PORT);
    client.write('I\'m Young!');
});

client.on('data', function(data) {
    console.log('DATA: ' + data);
    client.destroy();
});

client.on('close', function() {
    console.log('CLOSE CONNECTION');
});
```

```shell
> node client.js
CONNECT TO: 127.0.0.1:6969
DATA: You said "I'm Young!"
CLOSE CONNECTION
```

## UDP

node中，TCP是Stream实例，而UDP只是EventEmitter实例
