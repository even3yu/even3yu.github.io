## 1. NTP协议简介

**网络时间协议**NTP(Network Time Protocol)的主要开发者是美国特拉华大学的MILLS David L教授设计实现的，由时间协议、ICMP时间戳消息及IP时间戳选项发展而来。NTP用于将计算机客户或服务器的时间与另一服务器同步，使用层次式时间分布模型。在配置时，NTP可以利用冗余服务器和多条网络路径来获得时间的高准确性和高可靠性。即使客户机在长时间无法与某一时间服务器相联系的情况下,仍可提供高准确度时间。

## 2. NTP时间戳

- NTP时间戳使用一个64位的无符号整数来表示时间值，
- 其中前32位（MSW）表示自1900年1月1日以来的秒数，
- 后32位（LSW）表示秒数的小数部分，单位是232皮秒（picosecond=10的12次幂）（通常以固定小数点格式表示）这样的表示方式可以提供高精度的时间值。

> 关于232皮秒， 就是把1秒分为2的32次，  1/ 2的32次 * 10的12次 = 232.83...
> 1 second = 1,000,000,000,000 picoseconds, 这个值貌似有点大啊，而2的32次=4294967296，很明显用32bits无法精确到1 picoseconds， 于是我就想到不能精确到1 picoseconds， 那也应该尽力而为之吧，于是自然就有了把1,000,000,000,000 picoseconds劈成2的32次份：
>
> 1，000，000，000，000/4294967296 = 232.83064365386962890625
>
> That's it！！
>
> 
>
> [关于232皮秒的内容](https://blog.csdn.net/chinabinlang/article/details/39582977?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-39582977-blog-26478209.235^v38^pc_relevant_sort_base2&spm=1001.2101.3001.4242.1&utm_relevant_index=3)



> UTC 转 NTP时间戳：
>
> MSW = (70LL * 365 + 17) * 24 * 60 * 60 + tv.tv_sec （单位s），
> UTC是从1970开始计算，NTP是从1900开始计算，所以需要加上70年
>
> LSW = (tv.tv_usec << 32) / 1000000 (单位232ps)
> 其中，1s=10^12ps，tv_usec 是微妙 10的6次，左移32位就是除2的32次，转换为232，然后除1000000，就是转换为皮秒

### 2.1 NTP时间结构体

```cpp
typedef struct {
    uint32_t seconds;    // 自1900年1月1日以来的秒数
    uint32_t fraction;   // 秒数的小数部分
} NtpTimestamp;
```



### 2.2 VLC的source code 的NTP计算

![img]({{ site.url }}{{ site.baseurl }}/images/ntp.assets/rtsp_ntp_wireshark.gif)

```cpp
/**
 * @return NTP 64-bits timestamp in host byte order.
 */
uint64_t NTPtime64 (void)
{
    struct timespec ts;
#if defined (CLOCK_REALTIME)
    clock_gettime (CLOCK_REALTIME, &ts);
#else
    {
        struct timeval tv;
        gettimeofday (&tv, NULL);
        ts.tv_sec = tv.tv_sec;
        ts.tv_nsec = tv.tv_usec * 1000;
    }
#endif

    /* Convert nanoseconds to 32-bits fraction (232 picosecond units) */
    uint64_t t = (uint64_t)(ts.tv_nsec) << 32;
    t /= 1000000000;


    /* There is 70 years (incl. 17 leap ones) offset to the Unix Epoch.
     * No leap seconds during that period since they were not invented yet.
     */
    assert (t < 0x100000000);
    t |= ((70LL * 365 + 17) * 24 * 60 * 60 + ts.tv_sec) << 32;
    return t;
}
```



## 3. UTC时间戳

```cpp
#include <chrono>
// 替换std::chrono::seconds
auto seconds = std::chrono::duration_cast<std::chrono::seconds>(
  std::chrono::system_clock::now().time_since_epoch()
).count();
```



| 单位       |                           |         |
| ---------- | ------------------------- | ------- |
| 秒（s）    | std::chrono::seconds      |         |
| 毫秒（ms） | std::chrono::milliseconds | 10的3次 |
| 微秒（us） | std::chrono::microseconds | 10的6次 |
| 纳秒（ns） | std::chrono::nanoseconds  | 10的9次 |





## 参考

https://blog.csdn.net/davidsguo008/article/details/130708712

[RTCP包中的NTP Time](https://blog.csdn.net/ccskyer/article/details/26478209)

[RTCP中的NTP的时间计算方法](https://blog.csdn.net/chinabinlang/article/details/39582977?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-0-39582977-blog-26478209.235^v38^pc_relevant_sort_base2&spm=1001.2101.3001.4242.1&utm_relevant_index=3)

[NTP网络时间协议](https://blog.csdn.net/wqfhenanxc/article/details/81196462)

[NTP授时精度相关内容](https://blog.csdn.net/wqfhenanxc/article/details/81196462)

[NTP时间戳转换成UTC时间的过程](https://blog.csdn.net/weixin_45873923/article/details/120119622)

