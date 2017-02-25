# uv poll handle

- poll.c

```c
#include <stdio.h>
#include <unistd.h>
#include <uv.h>

uv_loop_t *loop;
int fd;

void poll_callback(uv_poll_t *poll_handle, int status, int events)
{
    static int counter = 0;
    printf("status = %d, events = %d\n", status, events);
    sleep(1);

	// 立刻读出fd的内容，否则一直进入该callback
//    int num;
//    int ret = read(fd, (void*)&num, 4);
//    printf("read %d bytes: %d\n", ret, num);

    counter++;
    // 如果5s没有读出来的话，就主动停掉poll handle
    if (counter == 5) {
        printf("stop the poll handle actively\n");
        uv_poll_stop(poll_handle);
    }
}

int main(int argc, const char *argv[])
{
    loop  = uv_default_loop();

    fd = open("./fpipe", O_RDWR);

    uv_poll_t poll_handle;
    uv_poll_init(loop, &poll_handle, fd);
    uv_poll_start(&poll_handle, UV_READABLE, poll_callback);

    int ret;
    ret = uv_run(loop, UV_RUN_DEFAULT);
    printf("ret = %d\n", ret);

    return 0;
}
```

- wpipe.c

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

int main(int argc, const char *argv[])
{
    int fd = open("./fpipe", O_RDWR);
    int num = 1;
    int ret = write(fd, (void*)&num, 4);
    printf("write %d bytes\n", ret);
    return 0;
}
```

- 编译

```shell
> gcc poll.c -o poll -I/usr/local/include -L/usr/local/lib -luv
> gcc wpipe.c -o wpipe
> mkfifo fpipe
```

- 测试一（注释，不读fd内容）

```shell
> ./poll
status = 0, events = 1
status = 0, events = 1
status = 0, events = 1
status = 0, events = 1
status = 0, events = 1
stop the poll handle actively
ret = 0
```

```shell
> ./wpipe
write 4 bytes
```

- 测试二（不注释，读fd内容）

```shell
> ./poll
status = 0, events = 1
read 4 bytes: 1
(wait)
```

```shell
> ./wpipe
write 4 bytes
```

> [Ref libuv poll](http://docs.libuv.org/en/v1.x/poll.html#c.uv_poll_cb)
