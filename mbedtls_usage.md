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

1. client端必须加`auto_mode=optional`，若不加该选项最终配置为`required`
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

```c
// server.crt (mbedtls_test_srv_crt_rsa in library/certs.c)
-----BEGIN CERTIFICATE-----
MIIDNzCCAh+gAwIBAgIBAjANBgkqhkiG9w0BAQUFADA7MQswCQYDVQQGEwJOTDER
MA8GA1UEChMIUG9sYXJTU0wxGTAXBgNVBAMTEFBvbGFyU1NMIFRlc3QgQ0EwHhcN
MTEwMjEyMTQ0NDA2WhcNMjEwMjEyMTQ0NDA2WjA0MQswCQYDVQQGEwJOTDERMA8G
A1UEChMIUG9sYXJTU0wxEjAQBgNVBAMTCWxvY2FsaG9zdDCCASIwDQYJKoZIhvcN
AQEBBQADggEPADCCAQoCggEBAMFNo93nzR3RBNdJcriZrA545Do8Ss86ExbQWuTN
owCIp+4ea5anUrSQ7y1yej4kmvy2NKwk9XfgJmSMnLAofaHa6ozmyRyWvP7BBFKz
NtSj+uGxdtiQwWG0ZlI2oiZTqqt0Xgd9GYLbKtgfoNkNHC1JZvdbJXNG6AuKT2kM
tQCQ4dqCEGZ9rlQri2V5kaHiYcPNQEkI7mgM8YuG0ka/0LiqEQMef1aoGh5EGA8P
hYvai0Re4hjGYi/HZo36Xdh98yeJKQHFkA4/J/EwyEoO79bex8cna8cFPXrEAjya
HT4P6DSYW8tzS1KW2BGiLICIaTla0w+w3lkvEcf36hIBMJcCAwEAAaNNMEswCQYD
VR0TBAIwADAdBgNVHQ4EFgQUpQXoZLjc32APUBJNYKhkr02LQ5MwHwYDVR0jBBgw
FoAUtFrkpbPe0lL2udWmlQ/rPrzH/f8wDQYJKoZIhvcNAQEFBQADggEBAJxnXClY
oHkbp70cqBrsGXLybA74czbO5RdLEgFs7rHVS9r+c293luS/KdliLScZqAzYVylw
UfRWvKMoWhHYKp3dEIS4xTXk6/5zXxhv9Rw8SGc8qn6vITHk1S1mPevtekgasY5Y
iWQuM3h4YVlRH3HHEMAD1TnAexfXHHDFQGe+Bd1iAbz1/sH9H8l4StwX6egvTK3M
wXRwkKkvjKaEDA9ATbZx0mI8LGsxSuCqe9r9dyjmttd47J1p1Rulz3CLzaRcVIuS
RRQfaD8neM9c1S/iJ/amTVqJxA1KOdOS5780WhPfSArA+g4qAmSjelc3p4wWpha8
zhuYwjVuX6JHG0c=
-----END CERTIFICATE-----
```

```c
// server.key (mbedtls_test_srv_key_rsa in library/certs.c)
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAwU2j3efNHdEE10lyuJmsDnjkOjxKzzoTFtBa5M2jAIin7h5r
lqdStJDvLXJ6PiSa/LY0rCT1d+AmZIycsCh9odrqjObJHJa8/sEEUrM21KP64bF2
2JDBYbRmUjaiJlOqq3ReB30Zgtsq2B+g2Q0cLUlm91slc0boC4pPaQy1AJDh2oIQ
Zn2uVCuLZXmRoeJhw81ASQjuaAzxi4bSRr/QuKoRAx5/VqgaHkQYDw+Fi9qLRF7i
GMZiL8dmjfpd2H3zJ4kpAcWQDj8n8TDISg7v1t7HxydrxwU9esQCPJodPg/oNJhb
y3NLUpbYEaIsgIhpOVrTD7DeWS8Rx/fqEgEwlwIDAQABAoIBAQCXR0S8EIHFGORZ
++AtOg6eENxD+xVs0f1IeGz57Tjo3QnXX7VBZNdj+p1ECvhCE/G7XnkgU5hLZX+G
Z0jkz/tqJOI0vRSdLBbipHnWouyBQ4e/A1yIJdlBtqXxJ1KE/ituHRbNc4j4kL8Z
/r6pvwnTI0PSx2Eqs048YdS92LT6qAv4flbNDxMn2uY7s4ycS4Q8w1JXnCeaAnYm
WYI5wxO+bvRELR2Mcz5DmVnL8jRyml6l6582bSv5oufReFIbyPZbQWlXgYnpu6He
GTc7E1zKYQGG/9+DQUl/1vQuCPqQwny0tQoX2w5tdYpdMdVm+zkLtbajzdTviJJa
TWzL6lt5AoGBAN86+SVeJDcmQJcv4Eq6UhtRr4QGMiQMz0Sod6ettYxYzMgxtw28
CIrgpozCc+UaZJLo7UxvC6an85r1b2nKPCLQFaggJ0H4Q0J/sZOhBIXaoBzWxveK
nupceKdVxGsFi8CDy86DBfiyFivfBj+47BbaQzPBj7C4rK7UlLjab2rDAoGBAN2u
AM2gchoFiu4v1HFL8D7lweEpi6ZnMJjnEu/dEgGQJFjwdpLnPbsj4c75odQ4Gz8g
sw9lao9VVzbusoRE/JGI4aTdO0pATXyG7eG1Qu+5Yc1YGXcCrliA2xM9xx+d7f+s
mPzN+WIEg5GJDYZDjAzHG5BNvi/FfM1C9dOtjv2dAoGAF0t5KmwbjWHBhcVqO4Ic
BVvN3BIlc1ue2YRXEDlxY5b0r8N4XceMgKmW18OHApZxfl8uPDauWZLXOgl4uepv
whZC3EuWrSyyICNhLY21Ah7hbIEBPF3L3ZsOwC+UErL+dXWLdB56Jgy3gZaBeW7b
vDrEnocJbqCm7IukhXHOBK8CgYEAwqdHB0hqyNSzIOGY7v9abzB6pUdA3BZiQvEs
3LjHVd4HPJ2x0N8CgrBIWOE0q8+0hSMmeE96WW/7jD3fPWwCR5zlXknxBQsfv0gP
3BC5PR0Qdypz+d+9zfMf625kyit4T/hzwhDveZUzHnk1Cf+IG7Q+TOEnLnWAWBED
ISOWmrUCgYAFEmRxgwAc/u+D6t0syCwAYh6POtscq9Y0i9GyWk89NzgC4NdwwbBH
4AgahOxIxXx2gxJnq3yfkJfIjwf0s2DyP0kY2y6Ua1OeomPeY9mrIS4tCuDQ6LrE
TB6l9VGoxJL4fyHnZb8L5gGvnB1bbD8cL6YPaDiOhcRseC9vBiEuVg==
-----END RSA PRIVATE KEY-----
```

```c
// client.crt (mbedtls_test_cli_crt_rsa in library/certs.c)
-----BEGIN CERTIFICATE-----
MIIDPzCCAiegAwIBAgIBBDANBgkqhkiG9w0BAQUFADA7MQswCQYDVQQGEwJOTDER
MA8GA1UEChMIUG9sYXJTU0wxGTAXBgNVBAMTEFBvbGFyU1NMIFRlc3QgQ0EwHhcN
MTEwMjEyMTQ0NDA3WhcNMjEwMjEyMTQ0NDA3WjA8MQswCQYDVQQGEwJOTDERMA8G
A1UEChMIUG9sYXJTU0wxGjAYBgNVBAMTEVBvbGFyU1NMIENsaWVudCAyMIIBIjAN
BgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAyHTEzLn5tXnpRdkUYLB9u5Pyax6f
M60Nj4o8VmXl3ETZzGaFB9X4J7BKNdBjngpuG7fa8H6r7gwQk4ZJGDTzqCrSV/Uu
1C93KYRhTYJQj6eVSHD1bk2y1RPD0hrt5kPqQhTrdOrA7R/UV06p86jt0uDBMHEw
MjDV0/YI0FZPRo7yX/k9Z5GIMC5Cst99++UMd//sMcB4j7/Cf8qtbCHWjdmLao5v
4Jv4EFbMs44TFeY0BGbH7vk2DmqV9gmaBmf0ZXH4yqSxJeD+PIs1BGe64E92hfx/
/DZrtenNLQNiTrM9AM+vdqBpVoNq0qjU51Bx5rU2BXcFbXvI5MT9TNUhXwIDAQAB
o00wSzAJBgNVHRMEAjAAMB0GA1UdDgQWBBRxoQBzckAvVHZeM/xSj7zx3WtGITAf
BgNVHSMEGDAWgBS0WuSls97SUva51aaVD+s+vMf9/zANBgkqhkiG9w0BAQUFAAOC
AQEAAn86isAM8X+mVwJqeItt6E9slhEQbAofyk+diH1Lh8Y9iLlWQSKbw/UXYjx5
LLPZcniovxIcARC/BjyZR9g3UwTHNGNm+rwrqa15viuNOFBchykX/Orsk02EH7NR
Alw5WLPorYjED6cdVQgBl9ot93HdJogRiXCxErM7NC8/eP511mjq+uLDjLKH8ZPQ
8I4ekHJnroLsDkIwXKGIsvIBHQy2ac/NwHLCQOK6mfum1pRx52V4Utu5dLLjD5bM
xOBC7KU4xZKuMXXZM6/93Yb51K/J4ahf1TxJlTWXtnzDr9saEYdNy2SKY/6ZiDNH
D+stpAKiQLAWaAusIWKYEyw9MQ==
-----END CERTIFICATE-----
```

```c
// client.key (mbedtls_test_cli_key_rsa in library/certs.c)
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAyHTEzLn5tXnpRdkUYLB9u5Pyax6fM60Nj4o8VmXl3ETZzGaF
B9X4J7BKNdBjngpuG7fa8H6r7gwQk4ZJGDTzqCrSV/Uu1C93KYRhTYJQj6eVSHD1
bk2y1RPD0hrt5kPqQhTrdOrA7R/UV06p86jt0uDBMHEwMjDV0/YI0FZPRo7yX/k9
Z5GIMC5Cst99++UMd//sMcB4j7/Cf8qtbCHWjdmLao5v4Jv4EFbMs44TFeY0BGbH
7vk2DmqV9gmaBmf0ZXH4yqSxJeD+PIs1BGe64E92hfx//DZrtenNLQNiTrM9AM+v
dqBpVoNq0qjU51Bx5rU2BXcFbXvI5MT9TNUhXwIDAQABAoIBAGdNtfYDiap6bzst
yhCiI8m9TtrhZw4MisaEaN/ll3XSjaOG2dvV6xMZCMV+5TeXDHOAZnY18Yi18vzz
4Ut2TnNFzizCECYNaA2fST3WgInnxUkV3YXAyP6CNxJaCmv2aA0yFr2kFVSeaKGt
ymvljNp2NVkvm7Th8fBQBO7I7AXhz43k0mR7XmPgewe8ApZOG3hstkOaMvbWAvWA
zCZupdDjZYjOJqlA4eEA4H8/w7F83r5CugeBE8LgEREjLPiyejrU5H1fubEY+h0d
l5HZBJ68ybTXfQ5U9o/QKA3dd0toBEhhdRUDGzWtjvwkEQfqF1reGWj/tod/gCpf
DFi6X0ECgYEA4wOv/pjSC3ty6TuOvKX2rOUiBrLXXv2JSxZnMoMiWI5ipLQt+RYT
VPafL/m7Dn6MbwjayOkcZhBwk5CNz5A6Q4lJ64Mq/lqHznRCQQ2Mc1G8eyDF/fYL
Ze2pLvwP9VD5jTc2miDfw+MnvJhywRRLcemDFP8k4hQVtm8PMp3ZmNECgYEA4gz7
wzObR4gn8ibe617uQPZjWzUj9dUHYd+in1gwBCIrtNnaRn9I9U/Q6tegRYpii4ys
c176NmU+umy6XmuSKV5qD9bSpZWG2nLFnslrN15Lm3fhZxoeMNhBaEDTnLT26yoi
33gp0mSSWy94ZEqipms+ULF6sY1ZtFW6tpGFoy8CgYAQHhnnvJflIs2ky4q10B60
ZcxFp3rtDpkp0JxhFLhiizFrujMtZSjYNm5U7KkgPVHhLELEUvCmOnKTt4ap/vZ0
BxJNe1GZH3pW6SAvGDQpl9sG7uu/vTFP+lCxukmzxB0DrrDcvorEkKMom7ZCCRvW
KZsZ6YeH2Z81BauRj218kQKBgQCUV/DgKP2985xDTT79N08jUo3hTP5MVYCCuj/+
UeEw1TvZcx3LJby7P6Xad6a1/BqveaGyFKIfEFIaBUBItk801sDDpDaYc4gL00Xc
7lFuBHOZkxJYlss5QrGpuOEl9ZwUt5IrFLBdYaKqNHzNVC1pCPfb/JyH6Dr2HUxq
gxUwAQKBgQCcU6G2L8AG9d9c0UpOyL1tMvFe5Ttw0KjlQVdsh1MP6yigYo9DYuwu
bHFVW2r0dBTqegP2/KTOxKzaHfC1qf0RGDsUoJCNJrd1cwoCLG8P2EF4w3OBrKqv
8u4ytY0F+Vlanj5lm3TaoHSVF1+NWPyOTiwevIECGKwSxvlki4fDAA==
-----END RSA PRIVATE KEY-----
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

#define SERVER_PORT "3000"
#define SERVER_NAME "localhost"
#define GET_REQUEST "GET / HTTP/1.0\r\n\r\n"
#define DEBUG_LEVEL 0

const char *cafile = "/Users/Young/Keys/ca.crt";

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
    mbedtls_x509_crt cacert;

    mbedtls_net_init(&tls_server_fd);
    mbedtls_ssl_init(&ssl);
    mbedtls_ssl_config_init(&conf);
    mbedtls_entropy_init(&entropy);
    mbedtls_ctr_drbg_init(&ctr_drbg);
    mbedtls_x509_crt_init(&cacert);

    mbedtls_debug_set_threshold(DEBUG_LEVEL);

    // start the connection
    printf(" . Connecting to tcp %s:%s ...", SERVER_NAME, SERVER_PORT);
    if ((ret = mbedtls_net_connect(&tls_server_fd, SERVER_NAME, SERVER_PORT,
                    MBEDTLS_NET_PROTO_TCP)) != 0) {
        printf(" failed\n ! mbedtls_net_connect return %d\n\n", ret);
        goto exit;
    }
    printf(" ok\n");

    // load the CA cert
    printf(" . Loading the CA certificate ...");
    if ((ret = mbedtls_x509_crt_parse_file(&cacert, cafile)) != 0) {
        printf(" failed\n ! mbedtls_x509_crt_parse_file returned -0x%x\n\n", -ret);
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
    mbedtls_ssl_conf_ca_chain(&conf, &cacert, NULL);

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
    do {
        ret = mbedtls_ssl_close_notify(&ssl);
    } while (ret == MBEDTLS_ERR_SSL_WANT_WRITE);
    printf(" done\n");

exit:
    mbedtls_net_free(&tls_server_fd);
    mbedtls_ssl_free(&ssl);
    mbedtls_ssl_config_free(&conf);
    mbedtls_entropy_free(&entropy);
    mbedtls_ctr_drbg_free(&ctr_drbg);
    mbedtls_x509_crt_free(&cacert);

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
#define SERVER_NAME "localhost"
#define DEBUG_LEVEL 0

#define HTTP_RESPONSE                                       \
    "HTTP/1.0 200 OK\r\nContent-Type: text/html\r\n\r\n"    \
    "<h2>mbed TLS Test Server</h2>\r\n"                     \
    "<p>Successful connection using: %s</p>\r\n"

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
    mbedtls_x509_crt server_crt;
    mbedtls_pk_context server_key;

    mbedtls_net_init(&tls_listen_fd);
    mbedtls_net_init(&tls_client_fd);
    mbedtls_ssl_init(&ssl);
    mbedtls_ssl_config_init(&conf);
    mbedtls_entropy_init(&entropy);
    mbedtls_ctr_drbg_init(&ctr_drbg);
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

    mbedtls_ssl_conf_dbg(&conf, my_debug, stdout);
    mbedtls_ssl_conf_rng(&conf, mbedtls_ctr_drbg_random, &ctr_drbg);

    if ((ret = mbedtls_ssl_conf_own_cert(&conf, &server_crt, &server_key)) != 0) {
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
        printf("ret = %d\n", ret);
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
    mbedtls_x509_crt_free(&server_crt);
    mbedtls_pk_free(&server_key);

    return ret;
}
```

```c
// ca.crt (TEST_CA_CRT_RSA in library/certs.c)
-----BEGIN CERTIFICATE-----
MIIDhzCCAm+gAwIBAgIBADANBgkqhkiG9w0BAQUFADA7MQswCQYDVQQGEwJOTDER
MA8GA1UEChMIUG9sYXJTU0wxGTAXBgNVBAMTEFBvbGFyU1NMIFRlc3QgQ0EwHhcN
MTEwMjEyMTQ0NDAwWhcNMjEwMjEyMTQ0NDAwWjA7MQswCQYDVQQGEwJOTDERMA8G
A1UEChMIUG9sYXJTU0wxGTAXBgNVBAMTEFBvbGFyU1NMIFRlc3QgQ0EwggEiMA0G
CSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDA3zf8F7vglp0/ht6WMn1EpRagzSHx
mdTs6st8GFgIlKXsm8WL3xoemTiZhx57wI053zhdcHgH057Zk+i5clHFzqMwUqny
50BwFMtEonILwuVA+T7lpg6z+exKY8C4KQB0nFc7qKUEkHHxvYPZP9al4jwqj+8n
YMPGn8u67GB9t+aEMr5P+1gmIgNb1LTV+/Xjli5wwOQuvfwu7uJBVcA0Ln0kcmnL
R7EUQIN9Z/SG9jGr8XmksrUuEvmEF/Bibyc+E1ixVA0hmnM3oTDPb5Lc9un8rNsu
KNF+AksjoBXyOGVkCeoMbo4bF6BxyLObyavpw/LPh5aPgAIynplYb6LVAgMBAAGj
gZUwgZIwDAYDVR0TBAUwAwEB/zAdBgNVHQ4EFgQUtFrkpbPe0lL2udWmlQ/rPrzH
/f8wYwYDVR0jBFwwWoAUtFrkpbPe0lL2udWmlQ/rPrzH/f+hP6Q9MDsxCzAJBgNV
BAYTAk5MMREwDwYDVQQKEwhQb2xhclNTTDEZMBcGA1UEAxMQUG9sYXJTU0wgVGVz
dCBDQYIBADANBgkqhkiG9w0BAQUFAAOCAQEAuP1U2ABUkIslsCfdlc2i94QHHYeJ
SsR4EdgHtdciUI5I62J6Mom+Y0dT/7a+8S6MVMCZP6C5NyNyXw1GWY/YR82XTJ8H
DBJiCTok5DbZ6SzaONBzdWHXwWwmi5vg1dxn7YxrM9d0IjxM27WNKs4sDQhZBQkF
pjmfs2cb4oPl4Y9T9meTx/lvdkRYEug61Jfn6cA+qHpyPYdTH+UshITnmp5/Ztkf
m/UTSLBNFNHesiTZeH31NcxYGdHSme9Nc/gfidRa0FLOCfWxRlFqAI47zG9jAQCZ
7Z2mCGDNMhjQc+BYcdnl0lPXjdDK6V0qCg1dVewhUBcW5gZKzV7e9+DpVA==
-----END CERTIFICATE-----
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
