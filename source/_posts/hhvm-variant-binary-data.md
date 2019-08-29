---
title: "HHVM扩展中使用二进制字符串"
date: 2019-08-29 17:30:49
tags:
  - HHVM
comments: false
---

最近都忙成狗，连写点东西的时间都没有，不过最近确实也学了不少东西。

工作中用到了HHVM，不知道是怎么想的，项目里面要把一部分C++的功能移到HHVM中，结果我一看源码就懵逼了，各种指针操作，位运算，引用啥啥的，完全不想去想怎么在PHP中去实现这一坨逻辑，所以干脆就直接暴力一点，使用C++扩展解决算了。

在迁移的过程中，也碰到了一个坑，就是关于二进制字符串的处理，C++的代码类似下面这段。

```cpp
// _m_buf - char * - 编码串
// _m_csr - char * - 当前尾指针位置
// _m_total_len - int - 缓冲区总长度

int Encoder::put_binary(const char * v, int len)
{
    if (0 != put_int(len)) {
        return -1;
    }
    if (_m_csr - _m_buf >= _m_total_len - len) {
        return -1;
    }

    memcpy(_m_csr, v, len);
    _m_csr += len;

    return 0;
}

int Decoder::get_binary(char* v, int& len)
{
    if (0 != get_int(len)) {
        return -1;
    }
    if (_m_csr - _m_buf >= _m_total_len - len) {
        return -1;
    }

    memcpy(v, _m_csr, len);
    _m_csr += len;

    return 0;
}
```

看着其实蛮简单的，就是把二进制的内容放进去，所以我在HHVM的扩展里面也就直接写了

```cpp
// 用来拿HHVM扩展Class的注册的数据结构体的
#define GET_NATIVE(cn) Native::data<cn>(this_.get())

int64_t HHVM_METHOD(Encoder, put_binary, CVarRef v)
{
    auto data = GET_NATIVE(EncoderData);
    return data->encoder->put_binary(v.c_str(), v.length());
}

int64_t HHVM_METHOD(Decoder, get_binary, VRefParam v, VRefParam len)
{
    auto data = GET_NATIVE(DecoderData);
    int length = 0;
    int ret = data->decoder->get_binary(data->tmp, length);
    data->tmp[length] = 0;
    v = Variant(String(data->tmp));
    len = Variant(length);
    return ret;
}
```

测了会儿没发现问题，结果拿线上流量一测，立刻崩了，包括心态。看了半天说是数据不完整，打印出来一看，里面全是`\0`。怪自己没考虑周全，硬着头皮修了，其实改起来也蛮简单的。

```cpp
int64_t HHVM_METHOD(Encoder, put_binary, CVarRef v)
{
    auto data = GET_NATIVE(EncoderData);
    String pack_str = v.toString();
    const char* pack_data = pack_str.data();
    size_t pack_size = pack_str.size();
    return data->encoder->put_binary(pack_data, pack_size);
}

int64_t HHVM_METHOD(Decoder, get_binary, VRefParam v, VRefParam len)
{
    auto data = GET_NATIVE(DecoderData);
    int length = 0;
    int ret = data->decoder->get_binary(data->tmp, length);
    v = Variant(String(data->tmp, length, CopyString));
    len = Variant(length);
    return ret;
}
```

二进制一定要用`.data()`，然后长度这种东西在C++里面一定要注意使用，能带上尽量带上，毕竟不像PHP这种“最好”的语言。