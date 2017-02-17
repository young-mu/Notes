## stream

copy util using stream

```javascript
var fs = require('fs');
var out = process.stdout;

var srcFile = './src';
var dstFile = './dst';

var readStream = fs.createReadStream(srcFile);
var writeStream = fs.createWriteStream(dstFile);

var stat = fs.statSync(srcFile);
var passSize = 0;
var lastSize = 0;
var totalSize = stat.size;

var startTime = Date.now();

readStream.on('data', function (chunk) {
    passSize += chunk.length;
    if (writeStream.write(chunk) === false) {
        readStream.pause();
    }
});

readStream.on('end', function () {
    writeStream.end();
});

writeStream.on('drain', function () {
    readStream.resume();
});

setTimeout(function showStatus () {
    var percent = (passSize / totalSize) * 100;
    var passSizeMB = (passSize / 1000000);
    var diffSize = passSizeMB - lastSize;

    lastSize = passSizeMB;
    out.clearLine();
    out.cursorTo(0);
    out.write('Complete ' + passSizeMB.toFixed(2) + 'MB, ' +
              percent.toFixed(2) + '% (' +(diffSize * 2).toFixed(2) + 'MB/s)');

    if (passSize < totalSize) {
        setTimeout(showStatus, 500);
    } else {
        var endTime = Date.now();
        console.log();
        console.log('Total: ' + (endTime - startTime) / 1000 + 's');
    }
}, 500);
```
