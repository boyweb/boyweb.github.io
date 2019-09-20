---
title: "epoll的ET/LT模式"
date: 2019-09-20 14:55:15
tags:
  - cpp
  - epoll
comments: false
---

最近在使用 epoll 的时候，看到了一个样例里面使用了 EPOLLET，之前从来没有用过，于是去研究了一下是干什么的。

epoll 的工作模式分为了两种，ET 和 LT，这两种的区别就是，ET 模式在触发事件之后，如果没有操作其中的 fd，下次也不会再通知了，而 LT 模式则会要求对 fd 进行操作，否则下次会再次通知。

写个代码试一下就能看出效果。

```cpp
int fds[2];

void *write_thread(void *p)
{
    usleep(500000);
    printf("write!\n");
    write(fds[1], "1", 1);

    return nullptr;
}

int main()
{
    pipe(fds);

    struct epoll_event ev;
    struct epoll_event events[4];
    int epfd;

    ev.data.fd = fds[0];
    ev.events = EPOLLIN;

    epfd = epoll_create(4);
    epoll_ctl(epfd, EPOLL_CTL_ADD, fds[0], &ev);

    pthread_t tid;
    pthread_create(&tid, nullptr, write_thread, nullptr);

    while (true)
    {
        int epoll_count = epoll_wait(epfd, events, 4, 5000);
        if (epoll_count > 0)
        {
            // 拿到通知之后，不做任何事情
            printf("event count: %d\n", epoll_count);
        }
        else
        {
            break;
        }
    }

    return 0;
}
```

编译执行之后，会发现死循环了，不处理就会一直触发，这个很符合预期。

```
write!
event count: 1
event count: 1
event count: 1
event count: 1
...
```

现在做两种修改：1. 切换成 ET 模式，2. 在接到通知后调用 read 读取

```cpp
// 切换ET模式
int main()
{
    // ...
    ev.events = EPOLLIN | EPOLLET;
    // ...
}

// read读取
int main()
{
    // ...
    while (true)
    {
        int epoll_count = epoll_wait(epfd, events, 4, 5000);
        if (epoll_count > 0)
        {
            char tmp[10];
            read(fds[0], tmp, 1);
            printf("event count: %d\n", epoll_count);
        }
        else
        {
            break;
        }
    }
    // ...
}
```

这两种编译后结果都是一样的，触发一次之后，就不会再触发了。

```
write!
event count: 1
```

从 kernel 代码来看，ET/LT 模式的处理逻辑几乎完全相同，差别仅在于 LT 模式在 event 发生时不会将其从 ready list 中移除，略为增大了 event 处理过程中 kernel space 中记录数据的大小。

如何选择就完全看需求了，一般都还是使用LT模式更为安全，ET模式对代码逻辑性要求较高。