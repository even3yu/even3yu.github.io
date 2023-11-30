---
layout: post
title: rtp 扩展头 解析与创建
date: 2023-11-28 20:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc rtp rtpxh
---


* content
{:toc}

---


## 属性

modules/rtp_rtcp/source/rtp_packet.h

```cpp
  using ExtensionType = RTPExtensionType;
  using ExtensionManager = RtpHeaderExtensionMap;
  
  //map
  ExtensionManager extensions_;
  // 存放扩展相关信息
  std::vector<ExtensionInfo> extension_entries_;
  // 扩展个数，不包括4字节对齐的padding
  size_t extensions_size_ = 0;  // Unaligned.
```





## -------扩展解析------

## RtpPacket::ParseBuffer

modules/rtp_rtcp/source/rtp_packet.cc

 ```cpp
/* rtp 
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
*/

bool RtpPacket::ParseBuffer(const uint8_t* buffer, size_t size) {
  if (size < kFixedHeaderSize) {
    return false;
  }
  // 头两位是 版本
  const uint8_t version = buffer[0] >> 6;
  if (version != kRtpVersion) {
    return false;
  }
  // 第三位是 padding
  const bool has_padding = (buffer[0] & 0x20) != 0;
  // 第四位 扩展
  const bool has_extension = (buffer[0] & 0x10) != 0;
  // 5~8位是 csrc
  const uint8_t number_of_crcs = buffer[0] & 0x0f;
  // 第二个字节， 第一位
  marker_ = (buffer[1] & 0x80) != 0;
  // 第二个字节， 2~8位 是 payload type
  payload_type_ = buffer[1] & 0x7f;

  // 第三，四个字节 序号
  sequence_number_ = ByteReader<uint16_t>::ReadBigEndian(&buffer[2]);
  // 第5~8个字节 是时间戳
  timestamp_ = ByteReader<uint32_t>::ReadBigEndian(&buffer[4]);
  // 第9~12 字节，是 ssrc
  ssrc_ = ByteReader<uint32_t>::ReadBigEndian(&buffer[8]);
  // csrs 如果有，就是 csrs的个数number_of_crcs *4 个字节
  if (size < kFixedHeaderSize + number_of_crcs * 4) {
    return false;
  }
  // kFixedHeaderSize = 12
  // 偏移到csrs之后；
  payload_offset_ = kFixedHeaderSize + number_of_crcs * 4;

/* padding size 最后一个字节
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
|One-byte eXtensions id = 0xbede|       length in 32bits        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Extensions                           |
|                             ....                              |
+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
|                          Payload                              |
|         ....               : padding...                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        padding             | padding size   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
*/
  if (has_padding) {
    //填充解析，填充大小放在最后一个字节
    padding_size_ = buffer[size - 1];
    if (padding_size_ == 0) {
      RTC_LOG(LS_WARNING) << "Padding was set, but padding size is zero";
      return false;
    }
  } else {
    padding_size_ = 0;
  }

  extensions_size_ = 0;
  extension_entries_.clear();
  if (has_extension) {
    /* RTP header extension, RFC 3550.
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |      defined by profile       |           length              |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                        header extension                       |
    |                             ....                              |
    */
     // 检测 rtp header extension 的有效性，取了前4个字节
     // 就是  defined by profile + length 
    size_t extension_offset = payload_offset_ + 4;
    if (extension_offset > size) {
      return false;
    }
    // 读取两个字节， defined by profile
    uint16_t profile =
        ByteReader<uint16_t>::ReadBigEndian(&buffer[payload_offset_]);
    // 两个字节，header extesion 容量，就是占用多少个 4字节
    size_t extensions_capacity =
        ByteReader<uint16_t>::ReadBigEndian(&buffer[payload_offset_ + 2]);
    // 总共占用的字节数
    extensions_capacity *= 4;
    if (extension_offset + extensions_capacity > size) {
      return false;
    }
    /* OneByteExtension
    0 1 2 3 4 5 6 7
    +-+-+-+-+-+-+-+-+
    |  ID   |  len  |
    +-+-+-+-+-+-+-+-+
    */
   
    /*TwoByteExtension
    0                   1
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |       ID      |     length    |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    */
    // uint16_t kOneByteExtensionProfileId = 0xBEDE;
    // uint16_t kTwoByteExtensionProfileId = 0x1000;
    // 两种rtp header extension
    if (profile != kOneByteExtensionProfileId &&
        profile != kTwoByteExtensionProfileId) {
      RTC_LOG(LS_WARNING) << "Unsupported rtp extension " << profile;
    } else {
      
	 //	constexpr size_t kOneByteExtensionHeaderLength = 1;
	 //	constexpr size_t kTwoByteExtensionHeaderLength = 2;
     // extension header 长度 
	 // kOneByteExtensionProfileId,  一个字节
	 // kTwoByteExtensionProfileId,  两个字节
      size_t extension_header_length = profile == kOneByteExtensionProfileId
                                           ? kOneByteExtensionHeaderLength
                                           : kTwoByteExtensionHeaderLength;
      constexpr uint8_t kPaddingByte = 0;
      constexpr uint8_t kPaddingId = 0;
      constexpr uint8_t kOneByteHeaderExtensionReservedId = 15;
      while (extensions_size_ + extension_header_length < extensions_capacity) {
        // kPaddingByte = 0
        // 判断是不是 padding，用于四字节对齐
        if (buffer[extension_offset + extensions_size_] == kPaddingByte) {
          extensions_size_++;
          continue;
        }
        int id;
        uint8_t length;
        if (profile == kOneByteExtensionProfileId) {
          // 取header extnesion 的第extensions_size_个的 第一个字节的前四位
          // 表示extension id
          id = buffer[extension_offset + extensions_size_] >> 4;
          // extension 对应的数据长度，取header extnesion 的第extensions_size_个的 第一个字节的5~8位
          // 注意： length 要+1（rtp报文中的取值范围是0~15）
          length = 1 + (buffer[extension_offset + extensions_size_] & 0xf);
          // kOneByteExtension, 最多可以 16个扩展
          if (id == kOneByteHeaderExtensionReservedId ||
              (id == kPaddingId && length != 1)) {
            break;
          }
        } else {
          // 取header extnesion 的第extensions_size_个的 第一个字节
          // 表示extension id
          id = buffer[extension_offset + extensions_size_];
          // extension 对应的数据长度，取header extnesion 的第extensions_size_个的 第二个字节
          // 注意： length 要+1
          length = buffer[extension_offset + extensions_size_ + 1];
        }

        if (extensions_size_ + extension_header_length + length >
            extensions_capacity) {
          RTC_LOG(LS_WARNING) << "Oversized rtp header extension.";
          break;
        }

        // 检测 rtp header extension的重复性
         // 不重复则创建ExtensionInfo，存放到std::vector<ExtensionInfo> extension_entries_;
        ExtensionInfo& extension_info = FindOrCreateExtensionInfo(id);
        if (extension_info.length != 0) {
          RTC_LOG(LS_VERBOSE)
              << "Duplicate rtp header extension id " << id << ". Overwriting.";
        }
		// extension的数据起始位置
        size_t offset =
            extension_offset + extensions_size_ + extension_header_length;
        if (!rtc::IsValueInRangeForNumericType<uint16_t>(offset)) {
          RTC_DLOG(LS_WARNING) << "Oversized rtp header extension.";
          break;
        }
        extension_info.offset = static_cast<uint16_t>(offset);
        extension_info.length = length;
        extensions_size_ += extension_header_length + length;
      }
    }
    payload_offset_ = extension_offset + extensions_capacity;
  }

  if (payload_offset_ + padding_size_ > size) {
    return false;
  }
  payload_size_ = size - payload_offset_ - padding_size_;
  return true;
}

 ```





### RtpPacket::ExtensionInfo

modules/rtp_rtcp/source/rtp_packet.cc

```cpp
 struct ExtensionInfo {
    explicit ExtensionInfo(uint8_t id) : ExtensionInfo(id, 0, 0) {}
    ExtensionInfo(uint8_t id, uint8_t length, uint16_t offset)
        : id(id), length(length), offset(offset) {}
    uint8_t id;
    uint8_t length;
    uint16_t offset;
  };
```





## -------扩展封装------

## RtpPacket::AllocateExtension

modules/rtp_rtcp/source/rtp_packet.cc

```cpp
rtc::ArrayView<uint8_t> RtpPacket::AllocateExtension(ExtensionType type,
                                                     size_t length) {
  // TODO(webrtc:7990): Add support for empty extensions (length==0).
   // int kMaxValueSize = 255;
  if (length == 0 || length > RtpExtension::kMaxValueSize ||
    // using ExtensionManager = RtpHeaderExtensionMap;
   // ExtensionManager extensions_;
      // 是否允许，rtp header extension的 onebyte，twobyte 混用
      (!extensions_.ExtmapAllowMixed() &&
        //  kOneByteHeaderExtensionMaxValueSize = 16; oneByte 的header扩展 数据，最大16个字节
       length > RtpExtension::kOneByteHeaderExtensionMaxValueSize)) {
    return nullptr;
  }

  // 是否注册到 ExtensionManager extensions_
  uint8_t id = extensions_.GetId(type);
  if (id == ExtensionManager::kInvalidId) {
    // Extension not registered.
    return nullptr;
  }
  if (!extensions_.ExtmapAllowMixed() &&
      // kOneByteHeaderExtensionMaxId = 14
      id > RtpExtension::kOneByteHeaderExtensionMaxId) {
    return nullptr;
  }
  return AllocateRawExtension(id, length);
}
```







api/rtp_parameters.h

```
// Inclusive min and max IDs for two-byte header extensions and one-byte
  // header extensions, per RFC8285 Section 4.2-4.3.
  static constexpr int kMinId = 1;
  static constexpr int kMaxId = 255;
  static constexpr int kMaxValueSize = 255;
  static constexpr int kOneByteHeaderExtensionMaxId = 14;
  static constexpr int kOneByteHeaderExtensionMaxValueSize = 16;
```



### ExtensionType

modules/rtp_rtcp/source/rtp_packet.cc

```
using ExtensionType = RTPExtensionType;
```

api/rtp_parameters.h

```cpp

// This enum must not have any gaps, i.e., all integers between
// kRtpExtensionNone and kRtpExtensionNumberOfExtensions must be valid enum
// entries.
enum RTPExtensionType : int {
  kRtpExtensionNone,
  kRtpExtensionTransmissionTimeOffset,
  kRtpExtensionAudioLevel,
  kRtpExtensionInbandComfortNoise,
  kRtpExtensionAbsoluteSendTime,
  kRtpExtensionAbsoluteCaptureTime,
  kRtpExtensionVideoRotation,
  kRtpExtensionTransportSequenceNumber,
  kRtpExtensionTransportSequenceNumber02,
  kRtpExtensionPlayoutDelay,
  kRtpExtensionVideoContentType,
  kRtpExtensionVideoLayersAllocation,
  kRtpExtensionVideoTiming,
  kRtpExtensionRtpStreamId,
  kRtpExtensionRepairedRtpStreamId,
  kRtpExtensionMid,
  kRtpExtensionGenericFrameDescriptor00,
  kRtpExtensionGenericFrameDescriptor = kRtpExtensionGenericFrameDescriptor00,
  kRtpExtensionGenericFrameDescriptor02,
  kRtpExtensionColorSpace,
  kRtpExtensionNumberOfExtensions  // Must be the last entity in the enum.
};
```





## RtpPacket::AllocateRawExtension

modules/rtp_rtcp/source/rtp_packet.cc

```cpp
rtc::ArrayView<uint8_t> RtpPacket::AllocateRawExtension(int id, size_t length) {
  RTC_DCHECK_GE(id, RtpExtension::kMinId);
  RTC_DCHECK_LE(id, RtpExtension::kMaxId);
  RTC_DCHECK_GE(length, 1);
  RTC_DCHECK_LE(length, RtpExtension::kMaxValueSize);
  
   // 根据id 找到ExtensionInfo
  const ExtensionInfo* extension_entry = FindExtensionInfo(id);
  if (extension_entry != nullptr) {
    // Extension already reserved. Check if same length is used.
    if (extension_entry->length == length)
      return rtc::MakeArrayView(WriteAt(extension_entry->offset), length);

    RTC_LOG(LS_ERROR) << "Length mismatch for extension id " << id
                      << ": expected "
                      << static_cast<int>(extension_entry->length)
                      << ". received " << length;
    return nullptr;
  }
  if (payload_size_ > 0) {
    RTC_LOG(LS_ERROR) << "Can't add new extension id " << id
                      << " after payload was set.";
    return nullptr;
  }
  if (padding_size_ > 0) {
    RTC_LOG(LS_ERROR) << "Can't add new extension id " << id
                      << " after padding was set.";
    return nullptr;
  }

  // 获取 csrs的个数
  const size_t num_csrc = data()[0] & 0x0F;
  // 扩展开始的偏移量， 第一个4 是 csrs的 每个占用4个字节；后一个4 是扩展总的头（就是是one/twobyte）
  const size_t extensions_offset = kFixedHeaderSize + (num_csrc * 4) + 4;
  // Determine if two-byte header is required for the extension based on id and
  // length. Please note that a length of 0 also requires two-byte header
  // extension. See RFC8285 Section 4.2-4.3.
  // 是否需要两字节 header extension
  const bool two_byte_header_required =
      id > RtpExtension::kOneByteHeaderExtensionMaxId ||
      length > RtpExtension::kOneByteHeaderExtensionMaxValueSize || length == 0;
  RTC_CHECK(!two_byte_header_required || extensions_.ExtmapAllowMixed());

  //profile_id =  kOneByteExtensionProfileId / kTwoByteExtensionProfileId
  uint16_t profile_id;
  if (extensions_size_ > 0) {
    // 如果已经有extension，则读取原先的值
    profile_id =
        ByteReader<uint16_t>::ReadBigEndian(data() + extensions_offset - 4);
     // profile_id 从kOneByteExtensionProfileId 升级到two_byte_header_required
    if (profile_id == kOneByteExtensionProfileId && two_byte_header_required) {
      // Is buffer size big enough to fit promotion and new data field?
      // The header extension will grow with one byte per already allocated
      // extension + the size of the extension that is about to be allocated.
      size_t expected_new_extensions_size =
          extensions_size_ + extension_entries_.size() +
          kTwoByteExtensionHeaderLength + length;
      if (extensions_offset + expected_new_extensions_size > capacity()) {
        RTC_LOG(LS_ERROR)
            << "Extension cannot be registered: Not enough space left in "
               "buffer to change to two-byte header extension and add new "
               "extension.";
        return nullptr;
      }
      // Promote already written data to two-byte header format.
      PromoteToTwoByteHeaderExtension();
      profile_id = kTwoByteExtensionProfileId;
    }
     
  } else { // !end  if (extensions_size_ > 0)
	// 还没有extensino，首先确认profile
    // Profile specific ID, set to OneByteExtensionHeader unless
    // TwoByteExtensionHeader is required.
    profile_id = two_byte_header_required ? kTwoByteExtensionProfileId
                                          : kOneByteExtensionProfileId;
  }

  const size_t extension_header_size = profile_id == kOneByteExtensionProfileId
                                           ? kOneByteExtensionHeaderLength
                                           : kTwoByteExtensionHeaderLength;
  size_t new_extensions_size =
      extensions_size_ + extension_header_size + length;
  if (extensions_offset + new_extensions_size > capacity()) {
    RTC_LOG(LS_ERROR)
        << "Extension cannot be registered: Not enough space left in buffer.";
    return nullptr;
  }

  // All checks passed, write down the extension headers.
  // 填入第一个extension，则先写入profile_id
  if (extensions_size_ == 0) {
    RTC_DCHECK_EQ(payload_offset_, kFixedHeaderSize + (num_csrc * 4));
    WriteAt(0, data()[0] | 0x10);  // Set extension bit.
    ByteWriter<uint16_t>::WriteBigEndian(WriteAt(extensions_offset - 4),
                                         profile_id);
  }

   // 写入 每个extesion的id和长度
  if (profile_id == kOneByteExtensionProfileId) {
     // id
    uint8_t one_byte_header = rtc::dchecked_cast<uint8_t>(id) << 4;
    // 对应len
    one_byte_header |= rtc::dchecked_cast<uint8_t>(length - 1);
    WriteAt(extensions_offset + extensions_size_, one_byte_header);
  } else {
    // TwoByteHeaderExtension.
    // id
    uint8_t extension_id = rtc::dchecked_cast<uint8_t>(id);
    WriteAt(extensions_offset + extensions_size_, extension_id);
    // len
    uint8_t extension_length = rtc::dchecked_cast<uint8_t>(length);
    WriteAt(extensions_offset + extensions_size_ + 1, extension_length);
  }

    
  const uint16_t extension_info_offset = rtc::dchecked_cast<uint16_t>(
      extensions_offset + extensions_size_ + extension_header_size);
  const uint8_t extension_info_length = rtc::dchecked_cast<uint8_t>(length);
  // std::vector<ExtensionInfo> extension_entries_
  extension_entries_.emplace_back(id, extension_info_length,
                                  extension_info_offset);

  // 更新扩展的个数
  extensions_size_ = new_extensions_size;

  // padding
  uint16_t extensions_size_padded =
      SetExtensionLengthMaybeAddZeroPadding(extensions_offset);
  payload_offset_ = extensions_offset + extensions_size_padded;
  buffer_.SetSize(payload_offset_);
  return rtc::MakeArrayView(WriteAt(extension_info_offset),
                            extension_info_length);
}
```

## 参考
[padding size](https://even3yu.github.io/2023/10/08/rtp/#6-rtp-扩展)

[WebRTC之RTP封装与解封装](https://blog.csdn.net/yinshipin007/article/details/129299612)