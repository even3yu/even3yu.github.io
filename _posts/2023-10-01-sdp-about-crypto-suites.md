---
layout: post
title: webrtc crypto suites
date: 2023-10-01 10:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---

## 1. CryptoOptions

```cpp
struct RTC_EXPORT CryptoOptions {
  CryptoOptions();
  CryptoOptions(const CryptoOptions& other);
  ~CryptoOptions();

  // Helper method to return an instance of the CryptoOptions with GCM crypto
  // suites disabled. This method should be used instead of depending on current
  // default values set by the constructor.
  static CryptoOptions NoGcm();

  // Returns a list of the supported DTLS-SRTP Crypto suites based on this set
  // of crypto options.
  std::vector<int> GetSupportedDtlsSrtpCryptoSuites() const;

  // SRTP Related Peer Connection options.
  struct Srtp {
    // Enable GCM crypto suites from RFC 7714 for SRTP. GCM will only be used
    // if both sides enable it.
    bool enable_gcm_crypto_suites = false;

    // If set to true, the (potentially insecure) crypto cipher
    // SRTP_AES128_CM_SHA1_32 will be included in the list of supported ciphers
    // during negotiation. It will only be used if both peers support it and no
    // other ciphers get preferred.
    bool enable_aes128_sha1_32_crypto_cipher = false;

    // The most commonly used cipher. Can be disabled, mostly for testing
    // purposes.
    bool enable_aes128_sha1_80_crypto_cipher = true;

    // If set to true, encrypted RTP header extensions as defined in RFC 6904
    // will be negotiated. They will only be used if both peers support them.
    bool enable_encrypted_rtp_header_extensions = false;
  } srtp;

  // Options to be used when the FrameEncryptor / FrameDecryptor APIs are used.
  struct SFrame {
    // If set all RtpSenders must have an FrameEncryptor attached to them before
    // they are allowed to send packets. All RtpReceivers must have a
    // FrameDecryptor attached to them before they are able to receive packets.
    bool require_frame_encryption = false;
  } sframe;
};
```

设置srtp的加密套件是否启动；

### 1.1 设置Cryptions的方式

`api/peer_connection_interface.h`

- `PeerConnectionFactoryInterface::Options` ， 设置CryptoOptions
- `PeerConnectionInterface::RTCConfiguration`



### 1.2 CryptoOptions::GetSupportedDtlsSrtpCryptoSuites

```cpp
std::vector<int> CryptoOptions::GetSupportedDtlsSrtpCryptoSuites() const {
  std::vector<int> crypto_suites;
  // Note: SRTP_AES128_CM_SHA1_80 is what is required to be supported (by
  // draft-ietf-rtcweb-security-arch), but SRTP_AES128_CM_SHA1_32 is allowed as
  // well, and saves a few bytes per packet if it ends up selected.
  // As the cipher suite is potentially insecure, it will only be used if
  // enabled by both peers.
  if (srtp.enable_aes128_sha1_32_crypto_cipher) {
    crypto_suites.push_back(rtc::SRTP_AES128_CM_SHA1_32);
  }
  if (srtp.enable_aes128_sha1_80_crypto_cipher) {
    crypto_suites.push_back(rtc::SRTP_AES128_CM_SHA1_80);
  }

  // Note: GCM cipher suites are not the top choice since they increase the
  // packet size. In order to negotiate them the other side must not support
  // SRTP_AES128_CM_SHA1_80.
  if (srtp.enable_gcm_crypto_suites) {
    crypto_suites.push_back(rtc::SRTP_AEAD_AES_256_GCM);
    crypto_suites.push_back(rtc::SRTP_AEAD_AES_128_GCM);
  }
  RTC_CHECK(!crypto_suites.empty());
  return crypto_suites;
}
```

根据CryptoOptions::Srtp获取加密套件。



## 2. 获取本地的CryptoSuites

### 2.1 -- GetSupportedAudioSdesCryptoSuiteNames

```
MediaSessionDescriptionFactory::AddAudioContentForOffer
GetSupportedAudioSdesCryptoSuiteNames
```

【章节4】本地支持的视频加密套件。



### 2.2 -- GetSupportedVideoSdesCryptoSuiteNames

```
MediaSessionDescriptionFactory::AddVideoContentForOffer
GetSupportedVideoSdesCryptoSuiteNames
```

【章节5】本地支持的视频加密套件。



## 3. CryptoSuites 协商

```cpp
MediaSessionDescriptionFactory::AddAudioContentForAnswer
/MediaSessionDescriptionFactory::AddVideoContentForAnswer
CreateMediaContentAnswer
SelectCrypto
```

**协商只有在Answer的时候**，才会进行加密套件的协商确定。



### 3.1 --!!! SelectCrypto

【章节8】，进行加密套件协商，可能有多个加密套件。



## 4. GetSupportedAudioSdesCryptoSuiteNames

pc/media_session.cc

```cpp
void GetSupportedAudioSdesCryptoSuiteNames(
    const webrtc::CryptoOptions& crypto_options,
    std::vector<std::string>* crypto_suite_names) {
  // 得到加密套件的名字
  GetSupportedSdesCryptoSuiteNames(GetSupportedAudioSdesCryptoSuites,
                                   crypto_options, crypto_suite_names);
}
```

获取Audio的SuiteName。

 ```
 MediaSessionDescriptionFactory::AddAudioContentForOffer
 GetSupportedAudioSdesCryptoSuiteNames
 ```

`GetSupportedAudioSdesCryptoSuiteNames` 的说明，【章节5.2】。



## 5. GetSupportedVideoSdesCryptoSuiteNames

pc/media_session.cc

```cpp
void GetSupportedVideoSdesCryptoSuiteNames(
    const webrtc::CryptoOptions& crypto_options,
    std::vector<std::string>* crypto_suite_names) {
  GetSupportedSdesCryptoSuiteNames(GetSupportedVideoSdesCryptoSuites,
                                   crypto_options, crypto_suite_names);
}
```

获取视频的加密套件名称。

```cpp
MediaSessionDescriptionFactory::AddVideoContentForOffer
GetSupportedVideoSdesCryptoSuiteNames
```



### 5.1 GetSupportedSdesCryptoSuiteNames

```cpp
void GetSupportedSdesCryptoSuiteNames(
    void (*func)(const webrtc::CryptoOptions&, std::vector<int>*),
    const webrtc::CryptoOptions& crypto_options,
    std::vector<std::string>* names) {
  std::vector<int> crypto_suites;
  // func = GetSupportedAudioSdesCryptoSuites， 得到支持的加密套件
  func(crypto_options, &crypto_suites);
  // crypto_suites 转换为 names
  for (const auto crypto : crypto_suites) {
    names->push_back(rtc::SrtpCryptoSuiteToName(crypto));
  }
}
```



### 5.2 GetSupportedAudioSdesCryptoSuites

```cpp
// For audio, HMAC 32 (if enabled) is prefered over HMAC 80 because of the
// low overhead.
void GetSupportedAudioSdesCryptoSuites(
    const webrtc::CryptoOptions& crypto_options,
    std::vector<int>* crypto_suites) {
  if (crypto_options.srtp.enable_aes128_sha1_32_crypto_cipher) {
    crypto_suites->push_back(rtc::SRTP_AES128_CM_SHA1_32);
  }
  crypto_suites->push_back(rtc::SRTP_AES128_CM_SHA1_80);
  if (crypto_options.srtp.enable_gcm_crypto_suites) {
    crypto_suites->push_back(rtc::SRTP_AEAD_AES_256_GCM);
    crypto_suites->push_back(rtc::SRTP_AEAD_AES_128_GCM);
  }
}
```

根据CryptoOptions，`std::vector<int>* crypto_suites` 得到加密套件。



### 5.3 GetSupportedVideoSdesCryptoSuites

```cpp
void GetSupportedVideoSdesCryptoSuites(
    const webrtc::CryptoOptions& crypto_options,
    std::vector<int>* crypto_suites) {
  crypto_suites->push_back(rtc::SRTP_AES128_CM_SHA1_80);
  if (crypto_options.srtp.enable_gcm_crypto_suites) {
    crypto_suites->push_back(rtc::SRTP_AEAD_AES_256_GCM);
    crypto_suites->push_back(rtc::SRTP_AEAD_AES_128_GCM);
  }
}
```

根据CryptoOptions，`std::vector<int>* crypto_suites` 得到加密套件。



## 6. CreateContentOffer

pc/media_session.cc

```cpp
MediaSessionDescriptionFactory::AddAudioContentForOffer
/MediaSessionDescriptionFactory::AddVideoContentForOffer
CreateMediaContentOffer
CreateContentOffer
```



```cpp
// Create a media content to be offered for the given |sender_options|,
// according to the given options.rtcp_mux, session_options.is_muc, codecs,
// secure_transport, crypto, and current_streams. If we don't currently have
// crypto (in current_cryptos) and it is enabled (in secure_policy), crypto is
// created (according to crypto_suites). The created content is added to the
// offer.
static bool CreateContentOffer(
    const MediaDescriptionOptions& media_description_options,
    const MediaSessionOptions& session_options,
    const SecurePolicy& secure_policy,
    const CryptoParamsVec* current_cryptos, // 上一次offer协商的加密套件
    const std::vector<std::string>& crypto_suites,
    const RtpHeaderExtensions& rtp_extensions,
    UniqueRandomIdGenerator* ssrc_generator,
    StreamParamsVec* current_streams,
    MediaContentDescription* offer) {
  ...
    
	// 需要加密
  if (secure_policy != SEC_DISABLED) {
    // 根据上一次offer的 ContentInfo current_content，得到Cryptos
    // 添加到MediaContentDescription* offer
    if (current_cryptos) {
      AddMediaCryptos(*current_cryptos, offer);
    }
    if (offer->cryptos().empty()) {
      // 如果offer->cryptos()为空，则 需要crypto_suites，
      if (!CreateMediaCryptos(crypto_suites, offer)) {
        return false;
      }
    }
  }

  if (secure_policy == SEC_REQUIRED && offer->cryptos().empty()) {
    return false;
  }
  return true;
}

```

- 如果需要加密传输rtp数据,填充加密套件；`secure_policy != SEC_DISABLED` 满足这个条件;
- 根据上一次offer的 `ContentInfo current_content`，通过`AddMediaCryptos`，添加到`MediaContentDescription* offer`；
- 如果`offer->cryptos()` 是空的，则创建CryptoParams，然后添加；



### 6.1 AddMediaCryptos

pc/media_session.cc

```cpp
void AddMediaCryptos(const CryptoParamsVec& cryptos,
                     MediaContentDescription* media) {
  for (const CryptoParams& crypto : cryptos) {
    media->AddCrypto(crypto);
  }
}
```

- MediaContentDescription 填充CryptoParams



pc/session_description.h

```cpp
  virtual void AddCrypto(const CryptoParams& params) {
    cryptos_.push_back(params);
  }
```





### 6.2 CreateMediaCryptos

pc/media_session.cc

```cpp
bool CreateMediaCryptos(const std::vector<std::string>& crypto_suites,
                        MediaContentDescription* media) {
  // 创建加密套件
  CryptoParamsVec cryptos;
  for (const std::string& crypto_suite : crypto_suites) {
    if (!AddCryptoParams(crypto_suite, &cryptos)) {
      return false;
    }
  }
  // MediaContentDescription 填充CryptoParams
  AddMediaCryptos(cryptos, media);
  return true;
}
```

- 创建加密套件
- 把加密套件加入到MediaContentDescription



#### 6.2.1 AddCryptoParams

pc/media_session.cc

```cpp
static bool AddCryptoParams(const std::string& cipher_suite,
                            CryptoParamsVec* cryptos_out) {
  // CryptoParamsVec 元素个数
  int size = static_cast<int>(cryptos_out->size());

  cryptos_out->resize(size + 1);
  return CreateCryptoParams(size, cipher_suite, &cryptos_out->at(size));
}
```



##### -- CreateCryptoParams

pc/media_session.cc

【章节7】。

#### 6.2.2 --AddMediaCryptos==6.1

pc/media_session.cc

【章节6.1】

## 7. !!! CreateCryptoParams

pc/media_session.cc

```cpp
// tag 就是CryptoParamsVec 的index
static bool CreateCryptoParams(int tag,
                               const std::string& cipher,
                               CryptoParams* crypto_out) {
  int key_len;
  int salt_len;
  // 根据加密套件，计算得到key，salt的长度
  if (!rtc::GetSrtpKeyAndSaltLengths(rtc::SrtpCryptoSuiteFromName(cipher),
                                     &key_len, &salt_len)) {
    return false;
  }

  // master key的长度，根据长度得到master_key
  int master_key_len = key_len + salt_len;
  std::string master_key;
  if (!rtc::CreateRandomData(master_key_len, &master_key)) {
    return false;
  }

  // base64 编码
  std::string key = rtc::Base64::Encode(master_key);

  crypto_out->tag = tag;
  crypto_out->cipher_suite = cipher;
  // const char kInline[] = "inline:";
  crypto_out->key_params = kInline;
  crypto_out->key_params += key;
  return true;
}
```

填充CryptoParams。计算key；



### 7.0 !!! CryptoParams

api/crypto_params.h

```cpp
struct CryptoParams {
  
  bool Matches(const CryptoParams& params) const {
    return (tag == params.tag && cipher_suite == params.cipher_suite);
  }
	// 在CryptoParamsVec的index，序列号
  int tag;
  std::string cipher_suite; // 加密套件的名称
  std::string key_params;// inline: base64
  std::string session_params;???
};
```



### 7.1 SrtpCryptoSuiteFromName

rtc_base/ssl_stream_adapter.cc

```cpp
int SrtpCryptoSuiteFromName(const std::string& crypto_suite) {
  if (crypto_suite == CS_AES_CM_128_HMAC_SHA1_32)
    return SRTP_AES128_CM_SHA1_32;
  if (crypto_suite == CS_AES_CM_128_HMAC_SHA1_80)
    return SRTP_AES128_CM_SHA1_80;
  if (crypto_suite == CS_AEAD_AES_128_GCM)
    return SRTP_AEAD_AES_128_GCM;
  if (crypto_suite == CS_AEAD_AES_256_GCM)
    return SRTP_AEAD_AES_256_GCM;
  return SRTP_INVALID_CRYPTO_SUITE;
}
```

- 根据加密套件，转换为srtp的加密套件

```cpp
const char CS_AES_CM_128_HMAC_SHA1_80[] = "AES_CM_128_HMAC_SHA1_80";
const char CS_AES_CM_128_HMAC_SHA1_32[] = "AES_CM_128_HMAC_SHA1_32";
const char CS_AEAD_AES_128_GCM[] = "AEAD_AES_128_GCM";
const char CS_AEAD_AES_256_GCM[] = "AEAD_AES_256_GCM";

//----------------------------------
const int SRTP_INVALID_CRYPTO_SUITE = 0;
#ifndef SRTP_AES128_CM_SHA1_80
const int SRTP_AES128_CM_SHA1_80 = 0x0001;
#endif
#ifndef SRTP_AES128_CM_SHA1_32
const int SRTP_AES128_CM_SHA1_32 = 0x0002;
#endif
#ifndef SRTP_AEAD_AES_128_GCM
const int SRTP_AEAD_AES_128_GCM = 0x0007;
#endif
#ifndef SRTP_AEAD_AES_256_GCM
const int SRTP_AEAD_AES_256_GCM = 0x0008;
#endif
const int SRTP_CRYPTO_SUITE_MAX_VALUE = 0xFFFF;
```



### 7.2 GetSrtpKeyAndSaltLengths

rtc_base/ssl_stream_adapter.cc

```cpp
bool GetSrtpKeyAndSaltLengths(int crypto_suite,
                              int* key_length,
                              int* salt_length) {
  switch (crypto_suite) {
    case SRTP_AES128_CM_SHA1_32:
    case SRTP_AES128_CM_SHA1_80:
      // SRTP_AES128_CM_HMAC_SHA1_32 and SRTP_AES128_CM_HMAC_SHA1_80 are defined
      // in RFC 5764 to use a 128 bits key and 112 bits salt for the cipher.
      *key_length = 16;
      *salt_length = 14;
      break;
    case SRTP_AEAD_AES_128_GCM:
      // SRTP_AEAD_AES_128_GCM is defined in RFC 7714 to use a 128 bits key and
      // a 96 bits salt for the cipher.
      *key_length = 16;
      *salt_length = 12;
      break;
    case SRTP_AEAD_AES_256_GCM:
      // SRTP_AEAD_AES_256_GCM is defined in RFC 7714 to use a 256 bits key and
      // a 96 bits salt for the cipher.
      *key_length = 32;
      *salt_length = 12;
      break;
    default:
      return false;
  }
  return true;
}
```

获取key和salt的长度。



## 8. !!! SelectCrypto???

pc/media_session.cc

```cpp
// Support any GCM cipher (if enabled through options). For video support only
// 80-bit SHA1 HMAC. For audio 32-bit HMAC is tolerated (if enabled) unless
// bundle is enabled because it is low overhead.
// Pick the crypto in the list that is supported.
static bool SelectCrypto(const MediaContentDescription* offer, // 远端支持的加密套件？？？？
                         bool bundle,
                         const webrtc::CryptoOptions& crypto_options, // 本地支持的加密套件配置
                         CryptoParams* crypto_out) { // 满足条件的加密套件
  bool audio = offer->type() == MEDIA_TYPE_AUDIO;
  const CryptoParamsVec& cryptos = offer->cryptos();

  for (const CryptoParams& crypto : cryptos) {
    if ((crypto_options.srtp.enable_gcm_crypto_suites &&
         rtc::IsGcmCryptoSuiteName(crypto.cipher_suite)) ||
        rtc::CS_AES_CM_128_HMAC_SHA1_80 == crypto.cipher_suite ||
        (rtc::CS_AES_CM_128_HMAC_SHA1_32 == crypto.cipher_suite && audio &&
         !bundle && crypto_options.srtp.enable_aes128_sha1_32_crypto_cipher)) {
      return CreateCryptoParams(crypto.tag, crypto.cipher_suite, crypto_out);
    }
  }
  return false;
}
```

加密套件的协商，可能有多个加密套件满足。



```cpp
MediaSessionDescriptionFactory::AddAudioContentForAnswer
/MediaSessionDescriptionFactory::AddVideoContentForAnswer
CreateMediaContentAnswer
SelectCrypto
FindMatchingCrypto
```



## 9. FindMatchingCrypto

```cpp
bool FindMatchingCrypto(const CryptoParamsVec& cryptos,
                        const CryptoParams& crypto,
                        CryptoParams* crypto_out) {
  auto it = absl::c_find_if(
      cryptos, [&crypto](const CryptoParams& c) { return crypto.Matches(c); });
  if (it == cryptos.end()) {
    return false;
  }
  *crypto_out = *it;
  return true;
}
```



## 10. !!! CryptoOptions && CryptoParams

- CryptoOptions::Srtp 表示的支持的srtp的加密套件， 【章节1】
- CryptoParams， 存放一个加密套件的相关信息，所有的加密套件是存放在CryptoParamsVec中， 【章节7.0】