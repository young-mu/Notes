## IO

三种IO复用方式

- select
- poll
- epoll

### 优缺点

1. select的fdset最多存放1024个fd，每次select调用要将3N个fdset（读/写/错误）由用户层传至内核层，且在用户层需要采用轮询遍历的方式查看三组fdset，性能随N增加而线性降低
2. poll需要将N个pollfd由用户层传至内核层，且在用户层需要采用轮询遍历的方式查看pollfd的revents，性能随N增加而线性降低
3. epoll通过epoll_create创建一个epfd（用户层fd管理内核层fdset），然后通过epoll\_ctl添加或删除内核层fdset的fd，避免一起将N个fdset由用户层传至内核层，且内核层将触发的事件写在events中，用户层不必轮询遍历，性能不随N增加而降低

### Links

1. [POLLIN等关键字的解释](http://www.2cto.com/shouce/linuxman/socket.7.html)
2. [IO多路复用历史](https://www.zhihu.com/question/32163005)

### Demo

```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <poll.h>
#include <string.h>
#include <sys/epoll.h>
#include <sys/select.h>
#include <errno.h>

#define FILE_NUM    (3)

int fd[FILE_NUM];

void read_fd(int fd)
{
    char buffer[100];
    size_t nread;
    printf("> fd %d\n", fd);
    nread = read(fd, buffer, 100);
    buffer[nread] = 0;
    printf("> read %d bytes: %s", (int)nread, buffer);
}

void select_test(void)
{
    fd_set rset;
    struct timeval timeout;

    // 最大值的fd
    int maxfd = fd[2];

    int ret;
    while (1) {
        timeout.tv_sec = 5;
        timeout.tv_usec = 0;
        FD_ZERO(&rset);

        // 每次select之前都要刷新fdset
        for (int i = 0; i < FILE_NUM; i++) {
            printf("Add fd %d in select fd rset\n", fd[i]);
            FD_SET(fd[i], &rset);
        }

        // 若要一直等待，则第五个参数传NULL
        // 若fdset中有fd之前被close，则errno为9（EBADF）
        ret = select(maxfd + 1, &rset, NULL, NULL, &timeout);
        if (ret == -1) {
            printf("select failed with errno %d\n", errno);
            exit(EXIT_FAILURE);
        } else if (ret == 0) {
            printf("timeout\n");
        } else {
            for (int i = 0; i < FILE_NUM; i++) {
                if (FD_ISSET(fd[i], &rset)) {
                    read_fd(fd[i]);
                }
            }
        }
    }
}

void poll_test(void)
{
    struct pollfd event[FILE_NUM];
    for (int i = 0; i < FILE_NUM; i++) {
        memset(&event[i], 0, sizeof(struct pollfd));
        event[i].fd = fd[i];
        event[i].events = POLLIN;
        printf("Add fd %d in poll\n", fd[i]);
    }

    int ret;
    while (1) {
        // 若要一直等待，则第三个参数传-1
        // 若fdset中有fd之前被close，则revents的POLLNVAL置位
        ret = poll(event, FILE_NUM, 5000);
        if (ret == -1) {
            printf("poll failed with errno %d\n", errno);
            exit(EXIT_FAILURE);
        } else if (ret == 0) {
            printf("timeout\n");
        } else {
            for (int i = 0; i < FILE_NUM; i++) {
                if (event[i].revents & POLLIN) {
                    int fd = event[i].fd;
                    read_fd(fd);
                }
            }
        }
    }
}

void epoll_test(void)
{
    int epfd;
    epfd = epoll_create1(0);

    struct epoll_event event[FILE_NUM];
    for (int i = 0; i < FILE_NUM; i++) {
        memset(&event[i], 0, sizeof(struct epoll_event));
        event[i].events = EPOLLIN;
        event[i].data.fd = fd[i];
        printf("Add fd %d in epoll queue\n", fd[i]);
        epoll_ctl(epfd, EPOLL_CTL_ADD, fd[i], &event[i]);
    }

    struct epoll_event events[FILE_NUM];
    int ret;
    while (1) {
        // 若要一直等待，则第四个参数传-1
        ret = epoll_wait(epfd, events, 1, 5000);
        if (ret == -1) {
            printf("epoll_wait failed with errno %d\n", errno);
            exit(EXIT_FAILURE);
        } else if (ret == 0) {
            printf("timeout\n");
        } else {
            printf("fdset event occurs ...\n");
            for (int i = 0; i < ret; i++) {
                if (events[i].events & POLLIN) {
                    int fd = events[i].data.fd;
                    read_fd(fd);
                }
            }
        }
    }
}

void fd_init(void)
{
    fd[0] = open("./f0", O_RDWR);
    fd[1] = open("./f1", O_RDWR);
    fd[2] = open("./f2", O_RDWR);
}

int main(int argc, const char *argv[])
{
    fd_init();

    select_test();
//    poll_test();
//    epoll_test();

    return 0;
}
```

```bash
// 环境准备
> mkfifo f0
> mkfifo f1
> mkfifo f2

// 测试
> echo 12345 > ./f0
> echo 12345 > ./f1
> echo 12345 > ./f2
```