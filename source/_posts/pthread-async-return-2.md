---
title: "pthread异步返回(2)"
date: 2019-09-23 17:36:56
tags:
  - cpp
  - pthread
comments: false
---

有时间继续写了，接上回。

关于`current_rid`加锁的部分就不详细赘述了，先来看下 condition_variable 的消耗是怎么样的。

测试方法如下：

1. 开启一个线程，不断调用 condition_variable 的 notify_one 方法，不用 sleep
2. 主线程不断循环调用 wait 方法，监听事件
3. 查看系统的 cpu 消耗
4. 使用 INT 信号终止进程，并计算 QPS

```cpp
#include <condition_variable>
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <sys/time.h>
#include <sys/types.h>

std::mutex mutex;
std::unique_lock<std::mutex> lck(mutex);
std::condition_variable condition;

bool running = true;
uint64_t i = 0;

void handle_break(int sig)
{
    if (sig == SIGINT)
    {
        printf("STOP!\n");
        running = false;
    }
}

// keep send notification
void *send_request(void *p)
{
    while (running)
    {
        condition.notify_one();
    }
}

int main()
{
    struct sigaction sigbreak;
    sigbreak.sa_handler = &handle_break;
    sigemptyset(&sigbreak.sa_mask);
    sigbreak.sa_flags = 0;
    sigaction(SIGINT, &sigbreak, NULL);

    pthread_t thread_id;
    pthread_create(&thread_id, NULL, send_request, NULL);

    auto start = std::chrono::high_resolution_clock::now();

    // keep recieve notification
    while (running)
    {
        while (condition.wait_for(lck, std::chrono::seconds(1)) == std::cv_status::timeout)
        {
            // TIMEOUT!
        }

        // REQUEST RETURN!
        i++;
    }

    auto elapsed = std::chrono::high_resolution_clock::now() - start;
    long long microseconds = std::chrono::duration_cast<std::chrono::milliseconds>(elapsed).count();
    double seconds = microseconds / 1000.0;

    printf("TIME ELAPSED: %lf QPS: %lf\n", seconds, i / seconds);

    return 0;
}
```

代码和之前的差不多，只不过就是增加了终止信号捕捉和 QPS 的统计，运行一段时间看下结果。

```
STOP!
TIME ELAPSED: 3.120000 QPS: 91419.871795
```

然后再运行一次看看 CPU 的消耗

```
11:30:01 AM       PID    %usr %system  %guest    %CPU   CPU  Command
11:30:03 AM     28207   46.00   61.50    0.00  107.50     0  ./cv
11:30:05 AM     28207   42.50   68.50    0.00  111.00     0  ./cv
11:30:07 AM     28207   48.00   58.50    0.00  106.50     1  ./cv
11:30:09 AM     28207   45.50   62.00    0.00  107.50     0  ./cv
11:30:11 AM     28207   46.00   59.50    0.00  105.50     0  ./cv
```

sys 已经比 usr 都高了，何况还没有做其他的操作，如果拿这个去做高并发肯定完蛋，如果 CPU 没有加上硬限，同一台机器上面的其他实例也会受到影响。

要降低 sys 就是要减少系统操作，一个解决办法就是定时批量触发，这个可能会影响时效性，但是对于提升机器性能是有很大的帮助的。

想要使用 condition_variable 去做批量，需要额外的开发工作，把事件存起来，到达一定数量之后，再去触发请求。

> 这里先暂时不讨论到达一定时间触发的情况，假定请求数量特别大，不会在一定时间内达不到缓冲区的数量大小。

稍微修改一下代码，看看性能情况。

```cpp
#define MAX_BUFFER_SIZE 1024

// keep send notification
void *send_request(void *p)
{
    int i = 0;
    while (running)
    {
        i++;
        if (i >= MAX_BUFFER_SIZE) {
            cv.notify_one();
            i = 0;
        }
    }
}

int main()
{
    // ...

    // keep recieve notification
    while (running)
    {
        // ...

        // REQUEST RETURN!
        for (int j = 0; j < MAX_BUFFER_SIZE; j++) {
            i++;
        }
    }

    // ...
}
```

编译运行一下，看看结果和 CPU

```
STOP!
TIME ELAPSED: 18.752000 QPS: 70748232.081911

11:54:53 AM       PID    %usr %system  %guest    %CPU   CPU  Command
11:54:55 AM     29436   79.50   29.50    0.00  109.00     1  ./cv_batch
11:54:57 AM     29436   89.00   32.50    0.00  121.50     1  ./cv_batch
11:54:59 AM     29436   88.50   35.00    0.00  123.50     0  ./cv_batch
11:55:01 AM     29436   75.00   25.00    0.00  100.00     0  ./cv_batch
```

sys 下降了近 40%，usr 有少量提升，但 QPS 也提升了不少，从代码上也讲得通，当然实战中不可能有理论上这么高的提升，因为请求处理也需要不少时间，不过下降的 sys 则可以拿来做很多其他的事情了，用 usr 来换 sys，是件很划得来的事情。

当然除了 condition_variable 还有其他的一些实现通知的方式，比如 pipe+epoll，而且这种可以利用缓冲区的大小直接实现批量触发的方式，而不需要我们自己再去实现记录事件之类的，都可以拿来试一下，得到最优化的方法。

在实战中，一定要尽量减少系统调用，频繁地切换 cpu 状态是很影响性能的，能批量就别单独使用了。
