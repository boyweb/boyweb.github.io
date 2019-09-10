---
title: "pthread异步返回"
date: 2019-09-10 11:02:32
tags:
  - C++
  - pthread
comments: false
---

最近莫名其妙的碰到了一个需求，首先要并发发送多个请求，然后在没有请求返回的时候会阻塞，并且该阻塞会有超时时间，然后只要有请求返回的时候就会解除阻塞并且执行后续的操作。

其实挺正常一需求，但是要使用C++来实现，两眼一抹黑，除了锁想不到别的方式了，但是考虑到高并发的情况，这个锁可能会把cpu的sys打得特别高，这对程序性能来说，问题很大，不管怎么着先试一下看看确切的情况。

先写一个模拟请求的函数。

```cpp
void *sim_request(void *p)
{
    // 随机等待100~400ms
    usleep(rand() % 400000 + 100000);
}
```

然后在主线程里面触发10个请求

```cpp
#define THREAD_NUM 10

int main()
{
    pthread_t tid_list[THREAD_NUM];

    for (int i = 0; i < THREAD_NUM; i++)
    {
        pthread_t tid;
        pthread_create(&tid, NULL, sim_request, NULL);

        // 记录下tid，异步请求，后面整体join
        tid_list[i] = tid;
    }

    for (int i = 0; i < THREAD_NUM; i++)
    {
        pthread_join(tid_list[i]);
    }

    return 0;
}
```