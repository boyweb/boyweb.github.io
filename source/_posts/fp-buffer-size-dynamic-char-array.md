---
title: "文件指针缓冲区大小 & 变长char数组"
date: 2019-09-19 17:44:50
tags:
  - cpp
  - char
comments: false
---

最近在使用管道的时候，因为不想每次都调用 write，想通过文件指针去操作，而且也不想每次写完都 flush，所以需要设置缓冲区来自动触发 flush 事件，先上样例。

```cpp
int main()
{
    int fds[2];
    if (pipe(fds) < 0)
    {
        return -1;
    }

    FILE * in;
    FILE * out;

    in = fdopen(fds[1], "a");
    out = fdopen(fds[0], "r");
}
```

使用`fdopen`可以将文件描述符和文件指针绑定到一起，这样在操作文件指针的时候也会直接操作文件描述符了。

设置缓冲区大小使用的是`setvbuf`函数，[相关文档](http://www.cplusplus.com/reference/cstdio/setvbuf/)在此处。按照文档描述，可以直接新建一个 char 的数组作为缓冲区，模式选择`_IOFBF`便可以在缓冲区写满的时候写入。

```cpp
#define BUFFER_SIZE 16

char in_buf[BUFFER_SIZE];
char out_buf[BUFFER_SIZE];

setvbuf(in, in_buf, _IOFBF, BUFFER_SIZE);
setvbuf(out, out_buf, _IOFBF, BUFFER_SIZE);
```

这样设置之后，在写满 16 个字符之后，会自动触发`fflush`。但是这里是把缓冲区长度定死了，如果想要通过某个参数修改配置，则可以使用变长 char 数组。

```cpp
int size = 32;
char *in_buf;
char *out_buf;

in_buf = new char[size];
out_buf = new char[size];

setvbuf(in, in_buf, _IOFBF, size);
setvbuf(out, out_buf, _IOFBF, size);
```

但是文档里面有这么一句话

> If _buffer_ is a null pointer, the function automatically allocates a buffer (using _size_ as a hint on the size to use). Otherwise, the array pointed by _buffer_ may be used as a buffer of _size_ bytes.

那么按理来说，如果不加 char 数组，直接传一个`size`应该是可以变长的。就是这样

```cpp
int size = 64;

setvbuf(in, nullptr, _IOFBF, size);
setvbuf(out, nullptr, _IOFBF, size);
```

可是我试了没起作用，可能是理解错了吧，有时间再研究研究。
