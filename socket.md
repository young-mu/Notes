# Socket

## C

### server

```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <netinet/in.h>
#include <arpa/inet.h>

static void socket_server_test(void)
{
    struct sockaddr_in local_addr;
    struct sockaddr_in remote_addr;

    local_addr.sin_family = AF_INET;
    local_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    local_addr.sin_port = htons(8888);

    int sfd = socket(AF_INET, SOCK_STREAM, 0);
    printf("sfd = %d\n", sfd);
    if (sfd < 0) {
        printf("socket error\n");
        exit(1);
    }

    int ret = bind(sfd, (struct sockaddr *)&local_addr, sizeof(local_addr));
    if (ret < 0) {
        printf("bind error\n");
        exit(1);
    }

    printf("start to listen 0.0.0.0:8888 ...\n");
    listen(sfd, 5);

    while (1) {
        int sin_size = sizeof(struct sockaddr_in);
        printf("\n> start to accept ...\n");
        int cfd = accept(sfd, (struct sockaddr *)&remote_addr, (socklen_t *)&sin_size);
        printf("> cfd = %d\n", cfd);
        if (cfd < 0) {
            printf("accept error\n");
            exit(1);
        }
        printf("> new connection comes\n");

        printf("> remote_addr IP: %s\n", inet_ntoa(remote_addr.sin_addr));

        struct sockaddr_in connAddr, peerAddr;
        int addrLen = sizeof(struct sockaddr_in);

        getsockname(cfd, (struct sockaddr *)&connAddr, (socklen_t*)&addrLen);
        printf("> host address: %s:%d\n", inet_ntoa(connAddr.sin_addr),
                                        ntohs(connAddr.sin_port));

        getpeername(cfd, (struct sockaddr *)&peerAddr, (socklen_t*)&addrLen);
        printf("> peer address: %s:%d\n", inet_ntoa(peerAddr.sin_addr),
                                        ntohs(peerAddr.sin_port));

        char buf[100];
        int len = recv(cfd, buf, sizeof(buf), 0);
        buf[len] = '\0';
        printf("> Recv: %s\n", buf);

        char msg[] = "I'm ESP32 server";
        len = send(cfd, msg, sizeof(msg), 0);

        close(cfd);
    }

    close(sfd);
}

int main(int argc, const char *argv[])
{
    socket_server_test();
    return 0;
}
```

### client

```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <netinet/in.h>
#include <string.h>
#include <arpa/inet.h>

static void socket_client_test(void)
{
    struct sockaddr_in remote_addr;
    memset(&remote_addr, 0, sizeof(remote_addr));
    remote_addr.sin_family = AF_INET;
    remote_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
    remote_addr.sin_port = htons(8888);

    int cfd = socket(AF_INET, SOCK_STREAM, 0);
    printf("cfd = %d\n", cfd);
    if (cfd < 0) {
        printf("socket error\n");
        exit(1);
    }

    int sin_size = sizeof(struct sockaddr);
    int ret = connect(cfd, (const struct sockaddr *)&remote_addr, sin_size);
    if (ret < 0) {
        printf("connect error\n");
        exit(1);
    }

    char msg[] = "I'm ESP32 client";
    int len = send(cfd, msg, sizeof(msg), 0);

    char buf[100];
    len = recv(cfd, buf, sizeof(buf), 0);
    buf[len] = '\0';
    printf("Recv: %s\n", buf);

    close(cfd);
}

int main(int argc, const char *argv[])
{
    socket_client_test();
    return 0;
}
```

### test

```shell
# ./server
sfd = 3
start to listen 0.0.0.0:8888 ...

> start to accept ...
> cfd = 4
> new connection comes
> remote_addr IP: 127.0.0.1
> host address: 127.0.0.1:8888
> peer address: 127.0.0.1:58321
> Recv: I'm ESP32 client

> start to accept ...
> cfd = 4
> new connection comes
> remote_addr IP: 127.0.0.1
> host address: 127.0.0.1:8888
> peer address: 127.0.0.1:58322
> Recv: I'm ESP32 client
```

```shell
# ./client
cfd = 3
Recv: I'm ESP32 server
# ./clent
cfd = 3
Recv: I'm ESP32 server
```

## Libuv / Libtuv

### server

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <uv.h>

uv_loop_t *loop;

void alloc_buffer(uv_handle_t *handle, size_t suggested_size, uv_buf_t *buf)
{
    buf->base = (char *)malloc(suggested_size);
    buf->len = suggested_size;
}

void write_cb(uv_write_t *req, int status)
{
    if (status < 0) {
        printf("write_cb error with %d\n", status);
        free(req);
        return;
    }

    free(req);
}

void read_cb(uv_stream_t *client, ssize_t nread, const uv_buf_t *buf)
{
    if (nread < 0) {
        if (nread != UV_EOF) {
            printf("read_cb error with %d\n", (int)nread);
        } else {
            printf("> client quit\n");
        }

        uv_close((uv_handle_t *)client, NULL);
        return;
    }

    char *data = (char *)malloc(sizeof(char) * (nread + 1));
    data[nread] = '\0';
    strncpy(data, buf->base, nread);

    printf("> Recv: %s\n", buf->base);

    free(data);
    free(buf->base);

    uv_write_t *req = (uv_write_t *)malloc(sizeof(uv_write_t));
    char msg[] = "I'm UV server!";
    uv_buf_t wrbuf = uv_buf_init(msg, strlen(msg) + 1);
    uv_write(req, (uv_stream_t *)client, &wrbuf, 1, write_cb);
}

void connection_cb(uv_stream_t *server, int status)
{
    if (status < 0) {
        printf("connection_cb error with %d\n", status);
        return;
    }

    uv_tcp_t *client = (uv_tcp_t *)malloc(sizeof(uv_tcp_t));
    uv_tcp_init(loop, client);

    printf("\n> enter connection callback, start to accept ...\n");
    if (uv_accept(server, (uv_stream_t *)client) == 0) {
        printf("> new connection comes\n");

        struct sockaddr host, peer;
        struct sockaddr_in host_in, peer_in;
        int len = sizeof(struct sockaddr);
        uv_tcp_getsockname(client, &host, &len);
        uv_tcp_getpeername(client, &peer, &len);

        char ip[17] = { 0 };
        uv_ip4_name((const struct sockaddr_in *)&host, ip, 16);
        host_in = *(struct sockaddr_in *)&host;
        printf("> host address: %s:%d\n", ip, ntohs(host_in.sin_port));

        uv_ip4_name((const struct sockaddr_in *)&peer, ip, 16);
        peer_in = *(struct sockaddr_in *)&peer;
        printf("> peer address: %s:%d\n", ip, ntohs(peer_in.sin_port));

        uv_read_start((uv_stream_t *)client, alloc_buffer, read_cb);
    } else {
        uv_close((uv_handle_t *)client, NULL);
    }
}

int main(int argc, const char *argv[])
{
    loop  = uv_default_loop();

    uv_tcp_t server;
    uv_tcp_init(loop, &server);

    struct sockaddr_in addr;
    int ret = uv_ip4_addr("0.0.0.0", 8888, &addr);
    printf("ret = %d\n", ret);

    uv_tcp_bind(&server, (const struct sockaddr*)&addr, 0);

    printf("start to listen on 0.0.0.0:8888 ...\n");
    int r = uv_listen((uv_stream_t *)&server, 5, connection_cb);
    if (r) {
        printf("uv_listen failed with %d\n", r);
        return 1;
    }

    ret = uv_run(loop, UV_RUN_DEFAULT);
    printf("ret = %d\n", ret);

    return 0;
}
```

### client

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <uv.h>

uv_loop_t *loop = NULL;
uv_tcp_t *client = NULL;

void alloc_buffer(uv_handle_t *handle, size_t suggested_size, uv_buf_t *buf)
{
    buf->base = malloc(suggested_size);
    buf->len = suggested_size;
}

void read_cb(uv_stream_t *client, ssize_t nread, const uv_buf_t *buf)
{
    if (nread < 0) {
        if (nread != UV_EOF) {
            printf("read_cb error with %d\n", (int)nread);
        }

        uv_close((uv_handle_t *)client, NULL);
        free(client);

        return;
    }

    char *data = (char *)malloc(sizeof(char) * (nread + 1));
    data[nread] = '\0';
    strncpy(data, buf->base, nread);

    printf("Recv: %s\n", data);

    free(data);
    free(buf->base);

    uv_close((uv_handle_t *)client, NULL);
    free(client);
}

void write_cb(uv_write_t *req, int status)
{
    if (status < 0) {
        printf("write_cb error with %d\n", status);
        free(req);
        return;
    }
    free(req);

    uv_read_start((uv_stream_t *)req->handle, alloc_buffer, read_cb);
}

void connect_cb(uv_connect_t *req, int status)
{
    if (status < 0) {
        printf("connect_cb error with %d\n", status);
        free(req);
        return;
    }

    uv_write_t *wreq = (uv_write_t *)malloc(sizeof(uv_write_t));
    char msg[] = "I'm UV client!";
    uv_buf_t wrbuf = uv_buf_init(msg, strlen(msg) + 1);
    uv_write(wreq, (uv_stream_t *)client, &wrbuf, 1, write_cb);
}

int main(int argc, const char *argv[])
{
    loop  = uv_default_loop();

    client = (uv_tcp_t *)malloc(sizeof(uv_tcp_t));
    uv_tcp_init(loop, client);

    struct sockaddr_in addr;
    uv_ip4_addr("127.0.0.1", 8888, &addr);

    uv_connect_t *creq = (uv_connect_t *)malloc(sizeof(uv_connect_t));
    uv_tcp_connect(creq, client, (const struct sockaddr*)&addr, connect_cb);

    int ret;
    ret = uv_run(loop, UV_RUN_DEFAULT);
    printf("ret = %d\n", ret);

    return 0;
}
```

### test

```shell
# ./server
start to listen on 0.0.0.0:8888 ...

> enter connection callback, start to accept ...
> new connection comes
> host address: 127.0.0.1:8888
> peer address: 127.0.0.1:58282
> Recv: I'm UV client!
> client quit

> enter connection callback, start to accept ...
> new connection comes
> host address: 127.0.0.1:8888
> peer address: 127.0.0.1:58284
> Recv: I'm UV client!
> client quit
```

```shell
# ./client
Recv: I'm UV server!
ret = 0
# ./client
Recv: I'm UV server!
ret = 0
```
