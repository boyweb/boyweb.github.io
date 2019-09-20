---
title: "pthread异步返回(1)"
date: 2019-09-10 11:02:32
tags:
  - cpp
  - pthread
comments: false
---

最近莫名其妙的碰到了一个需求，首先要并发发送多个请求，然后在没有请求返回的时候会阻塞，并且该阻塞会有超时时间，然后只要有请求返回的时候就会解除阻塞并且执行后续的操作。

其实挺正常一需求，但是要使用 C++来实现，两眼一抹黑，除了锁想不到别的方式了，但是考虑到高并发的情况，这个锁可能会把 cpu 的 sys 打得特别高，这对程序性能来说，问题很大，不管怎么着先试一下看看确切的情况。

先写一个模拟请求的函数。

```cpp
void *sim_request(void *p)
{
    // 随机等待100~400ms
    long time = rand() % 400000 + 100000;
    usleep(time);
    printf("Request %d Done! Running for %ldus.\n", ((request_data *)p)->id, time);
    return nullptr;
}
```

然后在主线程里面触发 10 个请求

```cpp
#define THREAD_NUM 10

typedef struct request_data_t
{
    int id;
} request_data;

int main()
{
    srand(time(NULL));

    pthread_t tid_list[THREAD_NUM];

    for (int i = 0; i < THREAD_NUM; i++)
    {
        request_data *data = new request_data();
        data->id = i;

        pthread_t tid;
        pthread_create(&tid, NULL, sim_request, data);

        // 记录下tid，异步请求，后面整体join
        tid_list[i] = tid;
    }

    // 等待所有请求结束

    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < THREAD_NUM; i++)
    {
        pthread_join(tid_list[i], nullptr);
    }
    auto elapsed = std::chrono::high_resolution_clock::now() - start;
    long long microseconds = std::chrono::duration_cast<std::chrono::microseconds>(elapsed).count();

    printf("All Request Done! Time elapsed: %lldus.\n", microseconds);

    return 0;
}
```

等待所有请求结束的我们常见的方法就是使用`pthread_join`来处理，但是其实也是互相会有影响，返回的时间依赖于耗时最长的那个请求。

结果如下

```
Request 2 Done! Running for 143260us.
Request 0 Done! Running for 216105us.
Request 9 Done! Running for 320221us.
Request 4 Done! Running for 350207us.
Request 6 Done! Running for 354879us.
Request 7 Done! Running for 360655us.
Request 5 Done! Running for 372377us.
Request 8 Done! Running for 389432us.
Request 1 Done! Running for 426694us.
Request 3 Done! Running for 486680us.
All Request Done! Time elapsed: 488597us.
```

符合我们的预期，要等到耗时最长的一个请求处理完了才会继续执行。现在把 join 的地方修改一下，每次 join 都输出一下当前请求的结果和耗时，看看如何。

```cpp
int main()
{
    // ...

    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < THREAD_NUM; i++)
    {
        pthread_join(tid_list[i], nullptr);
        auto request_elapsed = std::chrono::high_resolution_clock::now() - start;
        long long request_microseconds = std::chrono::duration_cast<std::chrono::microseconds>(request_elapsed).count();

        printf("Join Request %d Done! Time elapsed: %lldus.\n", i, request_microseconds);
    }
    auto elapsed = std::chrono::high_resolution_clock::now() - start;
    long long microseconds = std::chrono::duration_cast<std::chrono::microseconds>(elapsed).count();

    // ...
}
```

结果如下

```
Request 5 Done! Running for 181926us.
Request 4 Done! Running for 258100us.
Request 1 Done! Running for 265781us.
Request 9 Done! Running for 314483us.
Request 0 Done! Running for 320381us.
Join Request 0 Done! Time elapsed: 325151us.
Join Request 1 Done! Time elapsed: 325165us.
Request 6 Done! Running for 332907us.
Request 7 Done! Running for 372205us.
Request 2 Done! Running for 382483us.
Join Request 2 Done! Time elapsed: 383296us.
Request 8 Done! Running for 449042us.
Request 3 Done! Running for 498443us.
Join Request 3 Done! Time elapsed: 499584us.
Join Request 4 Done! Time elapsed: 499622us.
Join Request 5 Done! Time elapsed: 499642us.
Join Request 6 Done! Time elapsed: 499661us.
Join Request 7 Done! Time elapsed: 499679us.
Join Request 8 Done! Time elapsed: 499699us.
Join Request 9 Done! Time elapsed: 499737us.
All Request Done! Time elapsed: 499751us.
```

其中 1/4/5/9 这几个请求的耗时都比实际上的处理慢了不少时间，因此才有了异步返回的这个需求。

这里可以直接使用`condition_variable`和`mutex`实现。步骤如下：

> 1. 在全局声明相关 CV 变量
> 2. 在请求中增加通知行为
> 3. 在`main`中增加接受通知行为

```cpp
std::mutex mutex;
std::unique_lock<std::mutex> lock(mutex);
std::condition_variable condition;
int current_rid; // 记录当前完成的请求

void *sim_request(void *p)
{
    // 随机等待100~400ms
    long time = rand() % 400000 + 100000;
    usleep(time);

    request_data *data = (request_data *)p;
    printf("Request %d Done! Running for %ldus.\n", data->id, time);

    current_rid = data->id;
    condition.notify_one();

    return nullptr;
}

int main()
{
    // ...

    // 等待所有请求结束
    auto start = std::chrono::high_resolution_clock::now();
    int counter = THREAD_NUM;
    while (counter > 0)
    {
        condition.wait(lock);

        auto request_elapsed = std::chrono::high_resolution_clock::now() - start;
        long long request_microseconds = std::chrono::duration_cast<std::chrono::microseconds>(request_elapsed).count();

        printf("Main Request %d Done! Time elapsed: %lldus.\n", current_rid, request_microseconds);

        counter--;
    }

    // ...
}
```

运行结果如下

```
Request 2 Done! Running for 213561us.
Main Request 2 Done! Time elapsed: 216094us.
Request 4 Done! Running for 245467us.
Main Request 4 Done! Time elapsed: 248022us.
Request 7 Done! Running for 259471us.
Main Request 7 Done! Time elapsed: 259731us.
Request 0 Done! Running for 260219us.
Main Request 0 Done! Time elapsed: 260654us.
Request 8 Done! Running for 309838us.
Main Request 8 Done! Time elapsed: 310834us.
Request 9 Done! Running for 318751us.
Main Request 9 Done! Time elapsed: 321981us.
Request 1 Done! Running for 372233us.
Main Request 1 Done! Time elapsed: 377825us.
Request 5 Done! Running for 383058us.
Main Request 5 Done! Time elapsed: 383297us.
Request 6 Done! Running for 387030us.
Main Request 6 Done! Time elapsed: 398777us.
Request 3 Done! Running for 428213us.
Main Request 3 Done! Time elapsed: 431582us.
Join Request 0 Done! Time elapsed: 431645us.
Join Request 1 Done! Time elapsed: 431657us.
Join Request 2 Done! Time elapsed: 431664us.
Join Request 3 Done! Time elapsed: 431669us.
Join Request 4 Done! Time elapsed: 431675us.
Join Request 5 Done! Time elapsed: 431680us.
Join Request 6 Done! Time elapsed: 431685us.
Join Request 7 Done! Time elapsed: 431690us.
Join Request 8 Done! Time elapsed: 431695us.
Join Request 9 Done! Time elapsed: 431701us.
All Request Done! Time elapsed: 431703us.
```

可以看到每个请求处理耗时已经和真实的情况比较接近了，但是这里其实还是有点问题的，`current_rid`应该需要加锁，否则在两个请求耗时十分接近的时候，会出现变量线程不安全的问题。

下一次继续写，包括这种方式的 cpu 消耗测试、批量通知方式等。
