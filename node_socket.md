# node socket

- Server

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

- Client

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

- 测试

```shell
> node server.js
Server starts to listen on 127.0.0.1:6969
CONNECTED FROM: 127.0.0.1:61626
DATA: I'm Young!
CLOSED BY: 127.0.0.1:61626

> node client.js
CONNECT TO: 127.0.0.1:6969
DATA: You said "I'm Young!"
CLOSE CONNECTION
```