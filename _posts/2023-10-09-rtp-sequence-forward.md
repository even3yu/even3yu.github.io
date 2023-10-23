---
layout: post
title: rtp sequence id 回绕
date: 2023-10-09 12:30:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc rtp
---


* content
{:toc}

---


```less
   0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P|X|  CC   |M|     PT      |       sequence number         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                           timestamp                           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           synchronization source (SSRC) identifier            |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |            contributing source (CSRC) identifiers             |
   |                             ....                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```



## 1. 问题

在使用RTP协议时，我们需要通过序列号以及时间戳的比较，进行丢包判断。但是有个问题，
比如一个RTP包，序列号为4890，另一个RTP包序列号为59900，可以说59900一定比4890大，是个更新的RTP包吗？
序列号为 0 的包一定比序列号为 65535 的包小，是旧的包吗？
再如，序列号为 65535 的包一定比序列号为 255 的包大，是最新的包吗？

结论，当然不是这样，因为在判断序列号的连续性时要考虑回绕问题，不能直接根据数学意义上的大小进行比较。



## 2. 序列号回绕解释

RTP序列号`sequence number`用两字节表示，时间戳`timestamp`用四字节表示。所以序列号以及时间戳就存在一个取值范围。由于是无符号数，所以序列号范围为：`[0,2^16-1=65535]`，时间戳范围为：`[0,2^32-1]`。当达到最大值时，将发生所谓的**回绕**。例如，序列号到了2^16-1 = 65535，下个包序列号就将是0。所以我们不能直接根据数学意义上的大小进行序列号以及时间戳的比较。

序列号的回绕方式有两种，分别是向前回绕和向后回绕。

### 2.1 向前回绕-ForwardWrap（目前基本应该是这种方式）

向前回绕发生时，有如下特点：

1. 包号呈向前递增趋势。
2. 当前的包号很小，而前一个包号很大。
3. 从上一个包号向前跨越包号 0 到当前的包号。
4. **包号之间的距离小于包号类型能表示的数字个数的一半**。
5. 认为当前的包号是更大的包号，即当前包是更新的包。

下图展现了向前回绕的这些特点

<img src="{{ site.url }}{{ site.baseurl }}/images/rtp-sequence-id-forwar.assets/forwarp.png" alt="图片" style="zoom:50%;" />



### ~~2.2 向后会绕-BackwardWrap~~

向后回绕发生时，有如下特点：

1. 包号呈向后递减趋势。
2. 当前的包号很大，而前一个包号很小。
3. 从上一个包号向后跨越包号 0 到当前的包号。
4. 包号之间的**距离**大于包号类型能表示的数字个数的一半。
5. 一般情况下，认为当前的包号是更小的包号，即当前包是旧的包。

下图展现了向后回绕的这些特点：

<img src="{{ site.url }}{{ site.baseurl }}/images/rtp-sequence-id-forwar.assets/backwrap.png" alt="图片" style="zoom:50%;" />



## 3. webertc 解决方案

在WebRTC中定义了一个大小比较算法，包含数字回绕处理，判断是否是更新的数字。下面说下算法原理：

1. 假设有**两个`U`类型的无符号整数**：`value`, `prev_value`；
2. 定义一个常量`kBreakpoint`，为`U`类型取值范围的一半;
   比如，对于 `uint16_t` 类型，`kBreakpoint` 为 0x8000，对于 `uint32_t` 类型，`kBreakpoint` 为 0x80000000。
3. 当 value == prev_value 时，认为 value 不是更新（更大）的数字。
4. `value - prev_value = kBreakpoint` 且 `value > prev_value`时，`value`大于`prev_value`；
5. `value`与`prev_value`不相等，满足`(U)(valude - prev_value) < kBreakpoint`时，`value`大于`prev_value`。



### 3.1 总结

- `value`与`prev_value`距离小于取值范围的一半且不相等

- 或者`value`与`prev_value`距离等于取值范围的一半，`value`大于`prev_value`，

就可以说明`value`大于`prev_value`。



### 3.2 IsNewer

modules\include\module_common_types_public.h

```cpp
template <typename U>
inline bool IsNewer(U value, U prev_value) {
  static_assert(!std::numeric_limits<U>::is_signed, "U must be unsigned");
  // 1. kBreakpoint is the half-way mark for the type U. For instance, for a
  // uint16_t it will be 0x8000, and for a uint32_t, it will be 0x8000000.
  // std::numeric_limits<uint16_t>::max() = 65535
  // 取一半，就是中间值 32768
  constexpr U kBreakpoint = (std::numeric_limits<U>::max() >> 1) + 1;
  // 2. Distinguish between elements that are exactly kBreakpoint apart.
  // If t1>t2 and |t1-t2| = kBreakpoint: IsNewer(t1,t2)=true,
  // IsNewer(t2,t1)=false
  // rather than having IsNewer(t1,t2) = IsNewer(t2,t1) = false.
  // 0-32768 false
  // 32768-0 true
  if (value - prev_value == kBreakpoint) {
    return value > prev_value;
  }
  //0-255 = 65281 false
  //0-65535 = 1 true
  return value != prev_value &&
         static_cast<U>(value - prev_value) < kBreakpoint;
}
// NB: Doesn't fulfill strict weak ordering requirements.
//     Mustn't be used as std::map Compare function.
inline bool IsNewerSequenceNumber(uint16_t sequence_number,
                                  uint16_t prev_sequence_number) {
  return IsNewer(sequence_number, prev_sequence_number);
}
 
// NB: Doesn't fulfill strict weak ordering requirements.
//     Mustn't be used as std::map Compare function.
inline bool IsNewerTimestamp(uint32_t timestamp, uint32_t prev_timestamp) {
  return IsNewer(timestamp, prev_timestamp);
}
```

该函数实现了数字（序列号、时间戳）的比较算法。输入当前数字和之前的数字，如果当前数字是更新的数字则返回 `true`，否则返回 `false`。

> **注意，执行该算法的数字的类型必须是无符号类型。**



### 3.3 Unwrap 函数

```cpp
template <typename U>
class Unwrapper {
public:
  int64_t Unwrap(U value);
  int64_t UnwrapWithoutUpdate(U value) const;
private:
  absl::optional<int64_t> last_value_;
};
```

该函数用于展开回绕的数字，得到更大类型的真正的数字，其核心逻辑通过调用 `UnwrapWithoutUpdate` 函数实现。

`Unwrap` 函数相较于 `UnwrapWithoutUpdate` 函数的唯一区别是：`Unwrap` 函数会更新 `last_value_` 值，这个变量记录了上一次展开回绕后的数字的值。

在实际的流媒体场景中，包号或者时间戳所代表的的数字都是连续的，而且会发生回绕。如果有特殊的应用场景，需要得到回绕展开后的包号或者时间戳，则可以使用该函数。





## 参考

[webRTC研究：RTP中的序列号以及时间戳比较](https://blog.jianchihu.net/webrtc-research-rtp-number-timestamp-compare.html)

[WebRTC 基础技术 | RTP 包序列号的回绕处理](https://mp.weixin.qq.com/s?__biz=MzU3MTUyNDUzMA==&mid=2247483912&idx=1&sn=6e428962c804a2eb369cd8520fcdbfbb&chksm=fcdf96d5cba81fc3a8e6b53f7034b4ae1a857e81ca76a5a68c1d69b5a168a3ecb2d22c5d2456&mpshare=1&scene=24&srcid=&sharer_sharetime=1588812503210&sharer_shareid=ad5fbc7fb9fd369d489731f017e7c584&key=0e13cc338316a1ef731ef4b5a8f7732672617b53a46ff9213a22d55cb6e168f18e626d573dd316d4218a6cc5a4ca8df7cc366a554b9834d41a6fc52cf2e4d603b5f0c0468728b85287f2498e5633911d&ascene=14&uin=NTQyMTQ5MDAw&devicetype=Windows+10+x64&version=62090070&lang=zh_CN&exportkey=A5UNLVQNX6kfrcTYBa7tMpY%3D&pass_ticket=9Y9v7lYThc7kASRXSCY07pCIUAnBJ9Cltp46XQ4x57SKQDXv03vqlbDbqz%2BXTfPQ)