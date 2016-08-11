# mbedtls usage

## Sites

- [homepage](https://tls.mbed.org/)
- [github](https://github.com/ARMmbed/mbedtls)

## Compile and install

```shell
> cmake -H. -Bbuild -GNinja
> cd build
> ninja build
> ninja install
```

编译PIC的静态库

```shell
> cmake -H. -Bbuild -GNinja -DCMAKE_C_FLAGS="-fPIC"
> cd buid
> ninja build
```

## Usages

### server + client1

- server + client1

```bash
> ./build/programs/ssl/ssl_server
> ./build/programs/ssl/ssl_client1
```

Changes made with `ssl_client1.c` (after which there will be no errors output)

```c
define DEBUG_LEVEL 0 // 不输出任何debug log
```
```c
mbedtls_ssl_conf_authmode( &conf, MBEDTLS_SSL_VERIFY_REQUIRED ); // 严格检查certificate
```
```c
if( ( ret = mbedtls_ssl_set_hostname( &ssl, "localhost" ) ) != 0 )
```
```c
// mbedtls_printf( " %d bytes read\n\n%s", len, (char *) buf );
if (ret > 0 && buf[len-1] == '\n') {
    ret = 0;
    break;
}
```

### server2 + client2

- **server2 + client2 + 默认CA**

```bash
> ./build/programs/ssl/ssl_server2
> ./build/programs/ssl/ssl_client2
```

Notes:

1. 对于server，选项`auth_mode`默认为`none`，server将不检查client certificate ([link](https://github.com/ARMmbed/mbedtls/blob/master/library/ssl_tls.c#L4221))
2. 对于client，选项`auth_mode`默认为`required`，client将严格检查server certificate
3. server和client加载的`CA certificate`为`RSA`和`ECDSA`
4. 默认server加载的`Server certificate`为`RSA`和`ECDSA`两种，但用`ECDSA`
5. 默认client加载的`Client certificate`为`RSA`

- **server2 + client2 + 自定义CA**

```bash
> ./build/programs/ssl/ssl_server2 crt_file=/Users/Young/ABC/https/keys/server.crt key_file=/Users/Young/ABC/https/keys/server.key
> ./build/programs/ssl/ssl_client2 ca_file=/Users/Young/ABC/https/keys/ca.crt auth_mode=optional
```

Notes:

1. client端必须加`auth_mode=optional`，若不加该选项最终配置为`required`
2. 选项`auth_mode`设置为`optional`会报`Server certificate`不合法，设置为`none`将不检查，设置为`required`握手阶段就会失败（因为`CA certificate`不合法）

- **server2 + client.js + 自定义CA**

```bash
> ./build/programs/ssl/ssl_server2 crt_file=/Users/Young/ABC/https/keys/server.crt key_file=/Users/Young/ABC/https/keys/server.key server_port=3000
> cd /Users/Young/ABC/https; node client.js
```

- **server.js + client2 + 自定义CA**

```bash
> cd /Users/Young/ABC/https; node server.js
> ./build/programs/ssl/ssl_client2 ca_file=/Users/Young/ABC/https/keys/ca.crt auth_mode=none server_port=3000
```

## workflow

### client

1. seed the random number generator (JS -> C)
2. load CA certificate (JS -> C)
3. load CLIENT certificate if needed (JS -> C)
4. start the TCP connection (JS net module)
5. setup TLS stuff like CA and CLIENT certificate etc. (JS -> C)
6. perform handshake (JS -> C)
7. verify SERVER certificate (JS -> C)
8. ssl read and write (JS -> C)

### server

1. seed the random number generator (JS -> C)
2. load CA certificate if needed (JS -> C)
3. load SERVER certificate and SERVER key (JS -> C)
4. setup listening TCP socket (JS net module)
5. start to listen (JS net module)
6. setup TLS stuff like CA and SERVER certificate etc. (JS -> C)
7. perform handshake (JS -> C)
8. ssl read and write (JS -> C)

## Use mbedTLS in client application

### server

```javascript
var https = require('https');
var fs = require('fs');

var options = {
    cert: fs.readFileSync('/Users/Young/Keys/server.crt'),
    key: fs.readFileSync('/Users/Young/Keys/server.key'),
};

https.createServer(options, function(req, res) {
    console.log('ok...');
    res.writeHead(200);
    res.end('hello world\n');
}).listen(3000, 'localhost', function() {
    console.log('start to listen port 3000');
});
```

```bash
> node server.js
start to listen port 3000
ok...
```

### client

```c
// client.c
#include <stdio.h>
#include <string.h>

#include "mbedtls/platform.h"
#include "mbedtls/debug.h"
#include "mbedtls/net.h"
#include "mbedtls/ssl.h"
#include "mbedtls/entropy.h"
#include "mbedtls/ctr_drbg.h"
#include "mbedtls/x509_crt.h"
#include "mbedtls/certs.h"

#define SERVER_PORT "4433"
#define SERVER_NAME "127.0.0.1"
#define GET_REQUEST "GET / HTTP/1.0\r\n\r\n"
#define DEBUG_LEVEL 0

const char *ca_crt_file = "/Users/Young/Keys/ca.crt";
const char *client_crt_file = "/Users/Young/Keys/client.crt";
const char *client_key_file = "/Users/Young/Keys/client.key";

static void my_debug(void *ctx, int level, const char *file, int line, const char *str)
{
    fprintf((FILE *)ctx, "[%d] %s:%04d: %s", level, file, line, str);
    fflush((FILE *)ctx);
}

int main(int argc, const char *argv[])
{
    int ret, len;
    unsigned char buf[512];

    const char *custom = "mbedtls-test-client";
    mbedtls_net_context tls_server_fd;
    mbedtls_ssl_context ssl;
    mbedtls_ssl_config conf;
    mbedtls_entropy_context entropy;
    mbedtls_ctr_drbg_context ctr_drbg;
    mbedtls_x509_crt ca_crt;
    mbedtls_x509_crt client_crt;
    mbedtls_pk_context client_key;

    mbedtls_net_init(&tls_server_fd);
    mbedtls_ssl_init(&ssl);
    mbedtls_ssl_config_init(&conf);
    mbedtls_entropy_init(&entropy);
    mbedtls_ctr_drbg_init(&ctr_drbg);
    mbedtls_x509_crt_init(&ca_crt);
    mbedtls_x509_crt_init(&client_crt);
    mbedtls_pk_init(&client_key);

    mbedtls_debug_set_threshold(DEBUG_LEVEL);

    // start the connection
    printf(" . Connecting to tcp %s:%s ...", SERVER_NAME, SERVER_PORT);
    if ((ret = mbedtls_net_connect(&tls_server_fd, SERVER_NAME, SERVER_PORT,
                    MBEDTLS_NET_PROTO_TCP)) != 0) {
        printf(" failed\n ! mbedtls_net_connect return %d\n\n", ret);
        goto exit;
    }
    printf(" ok\n");

    // load the ca certificate
    printf(" . Loading the ca certificate ...");
    if ((ret = mbedtls_x509_crt_parse_file(&ca_crt, ca_crt_file)) != 0) {
        printf(" failed\n ! mbedtls_x509_crt_parse_file returned -0x%x\n\n", -ret);
        goto exit;
    }
    printf(" ok\n");

    // load the client certificate
    printf(" . Loading the client certificate ...");
    if ((ret = mbedtls_x509_crt_parse_file(&client_crt, client_crt_file)) != 0) {
        printf(" failed\n ! mbedtls_x509_crt_parse_file returned -0x%x\n\n", -ret);
        goto exit;
    }
    printf(" ok\n");

    // load the client key
    printf(" . Loading the client key ...");
    if ((ret = mbedtls_pk_parse_keyfile(&client_key, client_key_file, "")) != 0) {
        printf(" failed\n ! mbedtls_x509_pk_parse_keyfile returned -0x%x\n\n", -ret);
        goto exit;
    }
    printf(" ok\n");

    // init the RNG and configure the TLS structure
    printf(" . Initializing the RNG ...");
    if ((ret = mbedtls_ctr_drbg_seed(&ctr_drbg, mbedtls_entropy_func, &entropy,
                    (const unsigned char *)custom, strlen(custom))) != 0) {
        printf(" failed\n ! mbedtls_ctr_drbg_seed return %d\n", ret);
        goto exit;
    }
    printf(" ok\n");

    printf(" . Setting up the SSL/TLS structure ...");
    if ((ret = mbedtls_ssl_config_defaults(&conf, MBEDTLS_SSL_IS_CLIENT,
                    MBEDTLS_SSL_TRANSPORT_STREAM, MBEDTLS_SSL_PRESET_DEFAULT)) != 0) {
        printf(" failed\n ! mbedtls_ssl_config_defaults returned %d\n\n", ret);
        goto exit;
    }
    printf(" ok\n");

    mbedtls_ssl_conf_authmode(&conf, MBEDTLS_SSL_VERIFY_REQUIRED);
    mbedtls_ssl_conf_dbg(&conf, my_debug, stdout);
    mbedtls_ssl_conf_rng(&conf, mbedtls_ctr_drbg_random, &ctr_drbg);
    mbedtls_ssl_conf_ca_chain(&conf, &ca_crt, NULL);

    if ((ret = mbedtls_ssl_conf_own_cert(&conf, &client_crt, &client_key)) != 0) {
        printf(" failed\n ! mbedtls_ssl_config_defaults returned %d\n\n", ret);
        goto exit;
    }

    if ((ret = mbedtls_ssl_setup(&ssl, &conf)) != 0) {
        printf(" failed\n ! mbedtls_ssl_setup returned %d\n\n", ret);
        goto exit;
    }
    if ((ret = mbedtls_ssl_set_hostname(&ssl, "localhost")) != 0) {
        printf(" failed\n ! mbedtls_ssl_set_hostname returned %d\n\n", ret);
        goto exit;
    }
    mbedtls_ssl_set_bio(&ssl, &tls_server_fd, mbedtls_net_send, mbedtls_net_recv, NULL);

    // perform the TLS handshake and print info
    printf(" . Performing the SSL/TLS handshake ...");
    while ((ret = mbedtls_ssl_handshake(&ssl)) != 0) {
        if (ret != MBEDTLS_ERR_SSL_WANT_READ && ret != MBEDTLS_ERR_SSL_WANT_WRITE) {
            printf(" failed\n ! mbedtls_ssl_handshake returned -0x%x\n\n", -ret);
            goto exit;
        }
    }
    printf(" ok\n");

    printf("   [ Protocol is %s]\n", mbedtls_ssl_get_version(&ssl));
    printf("   [ Ciphersuite is %s]\n", mbedtls_ssl_get_ciphersuite(&ssl));
    printf("   [ Record expansion is %d]\n", mbedtls_ssl_get_record_expansion(&ssl));
    printf("   [ Maximum fragment length is %u]\n", (unsigned int)mbedtls_ssl_get_max_frag_len(&ssl));

    // verify peer cert and print info
    uint32_t flags;
    printf(" . Verifying peer X509 certificate ...");
    if ((flags = mbedtls_ssl_get_verify_result(&ssl)) != 0) {
        printf(" failed\n");
        mbedtls_x509_crt_verify_info((char *)buf, sizeof(buf), " ! ", flags);
        printf("%s\n", buf);
    } else {
        printf(" ok\n");
    }

    if (mbedtls_ssl_get_peer_cert(&ssl) != NULL) {
        printf(" . Peer certificate information ... ok\n");
        mbedtls_x509_crt_info((char *)buf, sizeof(buf) - 1, "   ", mbedtls_ssl_get_peer_cert(&ssl));
        printf("%s\n", buf);
    }

    // write the GET request
    len = sprintf((char *)buf, GET_REQUEST);
    printf(" > Write to server (%d bytes):", len);
    while ((ret = mbedtls_ssl_write(&ssl, buf, len)) <= 0) {
        if (ret != 0) {
            printf(" failed\n ! write returned %d\n\n", ret);
            goto exit;
        }
    }
    printf(" %d bytes written\n\n%s", ret, (char *)buf);

    // read the HTTP response
    printf(" < Read from server:");
    do {
        len = sizeof(buf) - 1;
        memset(buf, 0, sizeof(buf));
        ret = mbedtls_ssl_read(&ssl, buf, len);

        if (ret <= 0) {
            if (ret == MBEDTLS_ERR_SSL_PEER_CLOSE_NOTIFY) {
                ret = 0;
                goto close_notify;
            }
            printf(" failed\n ! ssl_read returned %d\n\n", ret);
            break;
        }
        printf(" %d bytes read\n\n%s", ret, (char *)buf);
    } while (1);

close_notify:
    printf(" . Closing the connection ...");
//    do {
//        ret = mbedtls_ssl_close_notify(&ssl);
//    } while (ret == MBEDTLS_ERR_SSL_WANT_WRITE);
    printf(" done\n");

exit:
    mbedtls_net_free(&tls_server_fd);
    mbedtls_ssl_free(&ssl);
    mbedtls_ssl_config_free(&conf);
    mbedtls_entropy_free(&entropy);
    mbedtls_ctr_drbg_free(&ctr_drbg);
    mbedtls_x509_crt_free(&ca_crt);
    mbedtls_x509_crt_free(&client_crt);
    mbedtls_pk_free(&client_key);

    return ret;
}
```

```c
// server.c
#include <stdio.h>
#include <string.h>

#include "mbedtls/platform.h"
#include "mbedtls/debug.h"
#include "mbedtls/net.h"
#include "mbedtls/ssl.h"
#include "mbedtls/entropy.h"
#include "mbedtls/ctr_drbg.h"
#include "mbedtls/x509_crt.h"
#include "mbedtls/certs.h"

#define SERVER_PORT "4433"
#define SERVER_NAME "127.0.0.1"
#define DEBUG_LEVEL 0

#define HTTP_RESPONSE                                       \
    "HTTP/1.0 200 OK\r\nContent-Type: text/html\r\n\r\n"    \
    "<h2>mbed TLS Test Server</h2>\r\n"                     \
    "<p>Successful connection using: %s</p>\r\n"

const char *ca_crt_file = "/Users/Young/Keys/ca.crt";
const char *server_crt_file = "/Users/Young/Keys/server.crt";
const char *server_key_file = "/Users/Young/Keys/server.key";

static void my_debug(void *ctx, int level, const char *file, int line, const char *str)
{
    fprintf((FILE *)ctx, "[%d] %s:%04d: %s", level, file, line, str);
    fflush((FILE *)ctx);
}

int main(int argc, const char *argv[])
{
    int ret, len;
    unsigned char buf[512];

    const char *custom = "mbedtls-test-server";
    mbedtls_net_context tls_listen_fd, tls_client_fd;
    mbedtls_ssl_context ssl;
    mbedtls_ssl_config conf;
    mbedtls_entropy_context entropy;
    mbedtls_ctr_drbg_context ctr_drbg;
    mbedtls_x509_crt ca_crt;
    mbedtls_x509_crt server_crt;
    mbedtls_pk_context server_key;

    mbedtls_net_init(&tls_listen_fd);
    mbedtls_net_init(&tls_client_fd);
    mbedtls_ssl_init(&ssl);
    mbedtls_ssl_config_init(&conf);
    mbedtls_entropy_init(&entropy);
    mbedtls_ctr_drbg_init(&ctr_drbg);
    mbedtls_x509_crt_init(&ca_crt);
    mbedtls_x509_crt_init(&server_crt);
    mbedtls_pk_init(&server_key);

    mbedtls_debug_set_threshold(DEBUG_LEVEL);

    // setup the listensing TCP socket
    printf(" . Bind on https://%s:%s/ ...", SERVER_NAME, SERVER_PORT);
    if ((ret = mbedtls_net_bind(&tls_listen_fd, SERVER_NAME, SERVER_PORT,
                    MBEDTLS_NET_PROTO_TCP)) != 0) {
        printf(" failed\n ! mbedtls_net_bind return %d\n\n", ret);
        goto exit;
    }
    printf(" ok\n");

    // load the ca certificate
    printf(" . Loading the ca certificate ...");
    if ((ret = mbedtls_x509_crt_parse_file(&ca_crt, ca_crt_file)) != 0) {
        printf(" failed\n ! mbedtls_x509_crt_parse_file returned -0x%x\n\n", -ret);
        goto exit;
    }
    printf(" ok\n");

    // load the server certificate
    printf(" . Loading the server certificate ...");
    if ((ret = mbedtls_x509_crt_parse_file(&server_crt, server_crt_file)) != 0) {
        printf(" failed\n ! mbedtls_x509_crt_parse_file returned -0x%x\n\n", -ret);
        goto exit;
    }
    printf(" ok\n");

    // load the server key
    printf(" . Loading the server key ...");
    if ((ret = mbedtls_pk_parse_keyfile(&server_key, server_key_file, "")) != 0) {
        printf(" failed\n ! mbedtls_x509_pk_parse_keyfile returned -0x%x\n\n", -ret);
        goto exit;
    }
    printf(" ok\n");

    // init the RNG and configure the TLS structure
    printf(" . Initializing the RNG ...");
    if ((ret = mbedtls_ctr_drbg_seed(&ctr_drbg, mbedtls_entropy_func, &entropy,
                    (const unsigned char *)custom, strlen(custom))) != 0) {
        printf(" failed\n ! mbedtls_ctr_drbg_seed return %d\n", ret);
        goto exit;
    }
    printf(" ok\n");

    printf(" . Setting up the SSL/TLS structure ...");
    if ((ret = mbedtls_ssl_config_defaults(&conf, MBEDTLS_SSL_IS_SERVER,
                    MBEDTLS_SSL_TRANSPORT_STREAM, MBEDTLS_SSL_PRESET_DEFAULT)) != 0) {
        printf(" failed\n ! mbedtls_ssl_config_defaults returned %d\n\n", ret);
        goto exit;
    }
    printf(" ok\n");

    mbedtls_ssl_conf_authmode(&conf, MBEDTLS_SSL_VERIFY_REQUIRED);
    mbedtls_ssl_conf_dbg(&conf, my_debug, stdout);
    mbedtls_ssl_conf_rng(&conf, mbedtls_ctr_drbg_random, &ctr_drbg);
    mbedtls_ssl_conf_ca_chain(&conf, &ca_crt, NULL);

    if ((ret = mbedtls_ssl_conf_own_cert(&conf, &server_crt, &server_key)) != 0) {
        printf(" failed\n ! mbedtls_ssl_config_defaults returned %d\n\n", ret);
        goto exit;
    }

    if ((ret = mbedtls_ssl_setup(&ssl, &conf)) != 0) {
        printf(" failed\n ! mbedtls_ssl_setup returned %d\n\n", ret);
        goto exit;
    }

reset:
    mbedtls_net_free(&tls_client_fd);
    mbedtls_ssl_session_reset(&ssl);

    // wait until a client connection
    printf(" . Waiting for a client connection ...");
    fflush(stdout);
    if ((ret = mbedtls_net_accept(&tls_listen_fd, &tls_client_fd, NULL, 0, NULL)) != 0) {
        printf(" failed\n ! mbedtls_ssl_handshake returned -0x%x\n\n", -ret);
        goto exit;
    }
    printf(" ok\n");
    mbedtls_ssl_set_bio(&ssl, &tls_client_fd, mbedtls_net_send, mbedtls_net_recv, NULL);

    // perform the TLS handshake and print info
    printf(" . Performing the SSL/TLS handshake ...");
    while ((ret = mbedtls_ssl_handshake(&ssl)) != 0) {
        if (ret != MBEDTLS_ERR_SSL_WANT_READ && ret != MBEDTLS_ERR_SSL_WANT_WRITE) {
            printf(" failed\n ! mbedtls_ssl_handshake returned -0x%x\n\n", -ret);
            goto exit;
        }
    }
    printf(" ok\n");

    printf("   [ Protocol is %s]\n", mbedtls_ssl_get_version(&ssl));
    printf("   [ Ciphersuite is %s]\n", mbedtls_ssl_get_ciphersuite(&ssl));
    printf("   [ Record expansion is %d]\n", mbedtls_ssl_get_record_expansion(&ssl));
    printf("   [ Maximum fragment length is %u]\n", (unsigned int)mbedtls_ssl_get_max_frag_len(&ssl));

    // verify peer cert and print info
    uint32_t flags;
    printf(" . Verifying peer X509 certificate ...");
    if ((flags = mbedtls_ssl_get_verify_result(&ssl)) != 0) {
        printf(" failed\n");
        mbedtls_x509_crt_verify_info((char *)buf, sizeof(buf), " ! ", flags);
        printf("%s\n", buf);
    } else {
        printf(" ok\n");
    }

    if (mbedtls_ssl_get_peer_cert(&ssl) != NULL) {
        printf(" . Peer certificate information ... ok\n");
        mbedtls_x509_crt_info((char *)buf, sizeof(buf) - 1, "   ", mbedtls_ssl_get_peer_cert(&ssl));
        printf("%s\n", buf);
    }

    // read the HTTP request
    printf(" < Read from server:");
    do {
        len = sizeof(buf) - 1;
        memset(buf, 0, sizeof(buf));
        ret = mbedtls_ssl_read(&ssl, buf, len);

        if (ret <= 0) {
            if (ret == MBEDTLS_ERR_SSL_PEER_CLOSE_NOTIFY) {
                ret = 0;
                goto close_notify;
            }
            printf(" failed\n ! ssl_read returned %d\n\n", ret);
            break;
        }
        printf(" %d bytes read\n\n%s", ret, (char *)buf);

        if (ret > 0) {
            break;
        }
    } while (1);

    // write the 200 response
    len = sprintf((char *)buf, HTTP_RESPONSE, mbedtls_ssl_get_ciphersuite(&ssl));
    printf(" > Write to server (%d bytes):", len);
    while ((ret = mbedtls_ssl_write(&ssl, buf, len)) <= 0) {
        if (ret != 0) {
            printf(" failed\n ! write returned %d\n\n", ret);
            goto exit;
        }
    }
    printf(" %d bytes written\n\n%s", ret, (char *)buf);

close_notify:
    printf(" . Closing the connection ...");
    do {
        ret = mbedtls_ssl_close_notify(&ssl);
    } while (ret == MBEDTLS_ERR_SSL_WANT_WRITE);
    printf(" done\n");

    goto reset;

exit:
    mbedtls_net_free(&tls_listen_fd);
    mbedtls_net_free(&tls_client_fd);
    mbedtls_ssl_free(&ssl);
    mbedtls_ssl_config_free(&conf);
    mbedtls_entropy_free(&entropy);
    mbedtls_ctr_drbg_free(&ctr_drbg);
    mbedtls_x509_crt_free(&ca_crt);
    mbedtls_x509_crt_free(&server_crt);
    mbedtls_pk_free(&server_key);

    return ret;
}
```

```bash
> gcc client.c -o client -I/usr/local/include /usr/local/lib/libmbedtls.a /usr/local/lib/libmbedx509.a /usr/local/lib/libmbedcrypto.a
> ./client
 . Initializing the RNG ... ok
 . Loading the CA certificate ... ok
 . Connecting to tcp localhost:3000 ... ok
 . Setting up the SSL/TLS structure ... ok
 . Performing the SSL/TLS handshake ... ok
   [ Protocol is TLSv1.2]
   [ Ciphersuite is TLS-ECDHE-RSA-WITH-AES-128-GCM-SHA256]
   [ Record expansion is 29]
   [ Maximum fragment length is 16384]
 . Verifying peer X509 certificate ... ok
 . Peer certificate information ... ok
   cert. version     : 3
   serial number     : 02
   issuer name       : C=NL, O=PolarSSL, CN=PolarSSL Test CA
   subject name      : C=NL, O=PolarSSL, CN=localhost
   issued  on        : 2011-02-12 14:44:06
   expires on        : 2021-02-12 14:44:06
   signed using      : RSA with SHA1
   RSA key size      : 2048 bits
   basic constraints : CA=false

 > Write to server: 18 bytes written

GET / HTTP/1.0

 < Read from server: 87 bytes read

HTTP/1.1 200 OK
Date: Sat, 11 Jun 2016 13:58:36 GMT
Connection: close

hello world
 . Closing the connection ... done
```

### Note

要使用信任的`CA certificate`，`Server certificate`和`Server key`（mbedTLS内部的），若使用自己生成的在handshake时会报警告（如果`authmode`选择`required`）

## Internals

### debug level

```c
mbedtls_debug_set_threshold(int threshold)
```

The default level is 0 (No debug), we can set the threshold by the API above, debug messages that have a level over the `threshold` will be discarded.

| level | comment       |
| ----- | ------------- |
| 0     | No debug      |
| 1     | Error         |
| 2     | State change  |
| 3     | Informational |
| 4     | Verbose       |

### customized debug callback

```c
mbedtls_ssl_conf_dbg(mbedtls_ssl_config *conf,
                     void (*f_dbg)(void *, int, const char *, int, const char *),
                     void *p_dbg)
```
configure customized debug callback (`f_dbg`) and the output file stream (`p_dbg`)

The customized debug callback prototype is

```c
void (*f_dbg)(void *context, int debug_level, const char *file_name, int line_number, const char *message);
```

## Refs

1. [mbedTLS tutorial](https://tls.mbed.org/kb/how-to/mbedtls-tutorial)
2. [TLS协议详解](http://blog.csdn.net/yzhou86/article/details/51211167)
3. [tnodir/luapolarssl](https://github.com/tnodir/luapolarssl)