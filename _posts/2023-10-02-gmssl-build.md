---
layout: post
title: gmssl build
date: 2023-10-02 10:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc
---


* content
{:toc}

---


介绍是基于gmssl 2.5.4的版本。gmssl 2.5.4 是基于 OpenSSL 1.1.0d 的版本修改而来；而gmssl 最新版本是和openssl完全独立，不兼容。所欲选择gmssl 2.5.4。



## 1. 前言

GmSSL是一个开源的密码工具箱，支持SM2/SM3/SM4/SM9/ZUC等国密(国家商用密码)算法、SM2国密数字证书及基于SM2证书的SSL/TLS安全通信协议，支持国密硬件密码设备，提供符合国密规范的编程接口与命令行工具，可以用于构建PKI/CA、安全通信、数据加密等符合国密标准的安全应用。GmSSL项目是OpenSSL项目的分支，并与OpenSSL保持接口兼容。因此GmSSL可以替代应用中的OpenSSL组件，并使应用自动具备基于国密的安全能力。GmSSL项目采用对商业应用友好的类BSD开源许可证，开源且可以用于闭源的商业应用。

> GmSSL项目由北京大学关志副研究员的密码学研究组开发维护，项目源码托管于GitHub。自2014年发布以来，GmSSL已经在多个项目和产品中获得部署与应用，并获得2015年度“一铭杯”中国Linux软件大赛二等奖(年度最高奖项)与开源中国密码类推荐项目。GmSSL项目的核心目标是通过开源的密码技术推动国内网络空间安全建设。



## 2. gmssl 和openssl的版本关系

gmssl 2.5.4 是基于 OpenSSL 1.1.0d 的版本修改而来；

```
#  define OPENSSL_VERSION_TEXT    "GmSSL 2.5.4 - OpenSSL 1.1.0d  19 Jun 2019"
```



## 3. windows 编译

### 3.1 编译工具准备

#### 3.1.1 安装VS2017

按照此文进行安装，[《Visual Studio Community 2017安装步骤（只装C++）》](https://blog.csdn.net/zyhse/article/details/105362609)。

主要使用它的编译器，若已安装，则跳过。

注意：编译需要再vs的命令窗口上编译，不同平台对应的cmd也不同

```
C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Visual Studio 2019\Visual Studio Tools\VC
```

**x64_x86 Cross Tools Command Prompt for VS 2019，x86编译**

**x86_x64 Cross Tools Command Prompt for VS 2019, 这个用于 x64 编译**

注意以上两者区别。



#### 3.1.2 安装ActivePerl

64位ActivePerl-5.26下载地址：[perl程序运行下载 ActivePerl v5.26.1.2601 (64bit) for windows 下载-脚本之家](https://www.jb51.net/softs/27286.html#downintro2)

进行安装，安装类型选择“Typical”，其他默认，最后一步安装时间较长。

在cmd中，输入命令查看perl版本号。

```
perl -v
```



#### 3.1.3 安装NASM

nasm-2.15.05下载地址：[Index of /pub/nasm/releasebuilds/2.15.05/win64](https://www.nasm.us/pub/nasm/releasebuilds/2.15.05/win64/)

以管理员身份运行nasm-2.15.05-installer-x64.exe，进行默认安装即可。

并将NASM安装目录添加至Windows系统环境变量Path中。



### 3.2 编译与安装GmSSL

#### 3.2.1 进入GmSSL源码目录

```
cd /d D:\GmSSL-master
```

#### 3.2.2 配置

- 生成32位

```
# 动态库
perl Configure VC-WIN32

#静态库
perl Configure VC-WIN32 no-shared
```

- 生成64位

```
# 动态库
perl Configure VC-WIN64A

#静态库
perl Configure VC-WIN64A no-shared
```

- 如果要生成debug
  
  则再后面 增加 `--debug` 就可以了



#### 3.2.3 编译

```
nmake clean
nmake
```



#### 3.2.4 安装

```
nmake install
```



#### 3.2.5 查看版本

```
gmssl version
```



## 4. android 编译

### 4.1 编译脚本

[wangp8895/gmssl-for-android: Android下编译gmssl静态库 (github.com)](https://github.com/wangp8895/gmssl-for-android)

```bash
#!/bin/bash

TOOLS_ROOT=`pwd`
ARCHS=("android" "android-armeabi" "android64-aarch64" "android-x86" "android64" "android-mips" "android-mips64")
ABIS=("armeabi" "armeabi-v7a" "arm64-v8a" "x86" "x86_64" "mips" "mips64")
# Default to API 21 for it is the minimum requirement for 64 bit archs.
ANDROID_API=${ANDROID_API:-15}
NDK=${ANDROID_NDK_R13}

configure() {
  ARCH=$1; OUT=$2; CLANG=${3:-""};

  TOOLCHAIN_ROOT=${TOOLS_ROOT}/${OUT}-android-toolchain

  if [ "$ARCH" == "android" ]; then
    export ARCH_FLAGS="-mthumb"
    export ARCH_LINK=""
    export TOOL="arm-linux-androideabi"
    NDK_FLAGS="--arch=arm"
  elif [ "$ARCH" == "android-armeabi" ]; then
    export ARCH_FLAGS="-march=armv7-a -mfloat-abi=softfp -mfpu=vfpv3-d16 -mthumb -mfpu=neon"
    export ARCH_LINK="-march=armv7-a -Wl,--fix-cortex-a8"
    export TOOL="arm-linux-androideabi"
    NDK_FLAGS="--arch=arm"
  elif [ "$ARCH" == "android64-aarch64" ]; then
    export ARCH_FLAGS=""
    export ARCH_LINK=""
    export TOOL="aarch64-linux-android"
    NDK_FLAGS="--arch=arm64"
  elif [ "$ARCH" == "android-x86" ]; then
    export ARCH_FLAGS="-march=i686 -mtune=intel -msse3 -mfpmath=sse -m32"
    export ARCH_LINK=""
    export TOOL="i686-linux-android"
    NDK_FLAGS="--arch=x86"
  elif [ "$ARCH" == "android64" ]; then
    export ARCH_FLAGS="-march=x86-64 -msse4.2 -mpopcnt -m64 -mtune=intel"
    export ARCH_LINK=""
    export TOOL="x86_64-linux-android"
    NDK_FLAGS="--arch=x86_64"
  elif [ "$ARCH" == "android-mips" ]; then
    export ARCH_FLAGS=""
    export ARCH_LINK=""
    export TOOL="mipsel-linux-android"
    NDK_FLAGS="--arch=mips"
  elif [ "$ARCH" == "android-mips64" ]; then
    export ARCH="linux64-mips64"
    export ARCH_FLAGS=""
    export ARCH_LINK=""
    export TOOL="mips64el-linux-android"
    NDK_FLAGS="--arch=mips64"
  fi;

  [ -d ${TOOLCHAIN_ROOT} ] || python $NDK/build/tools/make_standalone_toolchain.py \
                                     --api ${ANDROID_API} \
                                     --stl libc++ \
                                     --install-dir=${TOOLCHAIN_ROOT} \
                                     $NDK_FLAGS

  export TOOLCHAIN_PATH=${TOOLCHAIN_ROOT}/bin
  export NDK_TOOLCHAIN_BASENAME=${TOOLCHAIN_PATH}/${TOOL}
  export SYSROOT=${TOOLCHAIN_ROOT}/sysroot
  export CROSS_SYSROOT=$SYSROOT
  if [ -z "${CLANG}" ]; then
    export CC=${NDK_TOOLCHAIN_BASENAME}-gcc
    export CXX=${NDK_TOOLCHAIN_BASENAME}-g++
  else
    export CC=${TOOLCHAIN_PATH}/clang
    export CXX=${TOOLCHAIN_PATH}/clang++
  fi;
  export LINK=${CXX}
  export LD=${NDK_TOOLCHAIN_BASENAME}-ld
  export AR=${NDK_TOOLCHAIN_BASENAME}-ar
  export RANLIB=${NDK_TOOLCHAIN_BASENAME}-ranlib
  export STRIP=${NDK_TOOLCHAIN_BASENAME}-strip
  export CPPFLAGS=${CPPFLAGS:-""}
  export LIBS=${LIBS:-""}
  export CFLAGS="${ARCH_FLAGS} -fpic -ffunction-sections -funwind-tables -fstack-protector -fno-strict-aliasing -finline-limit=64"
  export CXXFLAGS="${CFLAGS} -std=c++11 -frtti -fexceptions"
  export LDFLAGS="${ARCH_LINK}"
  echo "**********************************************"
  echo "use ANDROID_API=${ANDROID_API}"
  echo "use NDK=${NDK}"
  echo "export ARCH=${ARCH}"
  echo "export NDK_TOOLCHAIN_BASENAME=${NDK_TOOLCHAIN_BASENAME}"
  echo "export SYSROOT=${SYSROOT}"
  echo "export CC=${CC}"
  echo "export CXX=${CXX}"
  echo "export LINK=${LINK}"
  echo "export LD=${LD}"
  echo "export AR=${AR}"
  echo "export RANLIB=${RANLIB}"
  echo "export STRIP=${STRIP}"
  echo "export CPPFLAGS=${CPPFLAGS}"
  echo "export CFLAGS=${CFLAGS}"
  echo "export CXXFLAGS=${CXXFLAGS}"
  echo "export LDFLAGS=${LDFLAGS}"
  echo "export LIBS=${LIBS}"
  echo "**********************************************"
}
```

```bash
#!/bin/bash
#
# Copyright 2016 leenjewel
# 
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
# 
#     http://www.apache.org/licenses/LICENSE-2.0
# 
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

set -u

source ./_shared.sh

# Setup architectures, library name and other vars + cleanup from previous runs
LIB_NAME="GmSSL-master"
LIB_DEST_DIR=${TOOLS_ROOT}/libs
[ -d ${LIB_DEST_DIR} ] && rm -rf ${LIB_DEST_DIR}
[ -f "master.zip" ] || wget https://github.com/guanzhi/GmSSL/archive/master.zip;
# Unarchive library, then configure and make for specified architectures
configure_make() {
  ARCH=$1; ABI=$2;
  rm -rf "${LIB_NAME}"
  unzip -o "master.zip"
  pushd "${LIB_NAME}"

  configure $*

  #support openssl-1.0.x
  if [[ $LIB_NAME != "GmSSL-master" ]]; then
    if [[ $ARCH == "android-armeabi" ]]; then
        ARCH="android-armv7"
    elif [[ $ARCH == "android64" ]]; then 
        ARCH="linux-x86_64 shared no-ssl2 no-ssl3 no-hw "
    elif [[ "$ARCH" == "android64-aarch64" ]]; then
        ARCH="android shared no-ssl2 no-ssl3 no-hw "
    fi
  fi

echo "use android api:${ANDROID_API}"

  ./Configure $ARCH \
              --prefix=${LIB_DEST_DIR}/${ABI} \
              --with-zlib-include=$SYSROOT/usr/include \
              --with-zlib-lib=$SYSROOT/usr/lib \
              zlib \
              no-asm \
              no-shared \
              no-unit-test\
              no-serpent

  PATH=$TOOLCHAIN_PATH:$PATH

  if make -j4; then
    make install

    OUTPUT_ROOT=${TOOLS_ROOT}/../output/android/gmssl-${ABI}
    [ -d ${OUTPUT_ROOT}/include ] || mkdir -p ${OUTPUT_ROOT}/include
    cp -r ${LIB_DEST_DIR}/${ABI}/include/openssl ${OUTPUT_ROOT}/include

    [ -d ${OUTPUT_ROOT}/lib ] || mkdir -p ${OUTPUT_ROOT}/lib
    cp ${LIB_DEST_DIR}/${ABI}/lib/libcrypto.a ${OUTPUT_ROOT}/lib
    cp ${LIB_DEST_DIR}/${ABI}/lib/libssl.a ${OUTPUT_ROOT}/lib
  fi;
  popd

}



for ((i=0; i < ${#ARCHS[@]}; i++))
do
  if [[ $# -eq 0 ]] || [[ "$1" == "${ARCHS[i]}" ]]; then
    # Do not build 64 bit arch if ANDROID_API is less than 21 which is
    # the minimum supported API level for 64 bit.
    [[ ${ANDROID_API} < 21 ]] && ( echo "${ABIS[i]}" | grep 64 > /dev/null ) && continue;
    configure_make "${ARCHS[i]}" "${ABIS[i]}"
  fi
done
```

**注意事项：**

1. Configure的修改
   
   ```bash
     ./Configure $ARCH \
                 --prefix=${LIB_DEST_DIR}/${ABI} \
                 --with-zlib-include=$SYSROOT/usr/include \
                 --with-zlib-lib=$SYSROOT/usr/lib \
                 zlib \
                 no-asm \
                 no-shared \
   ```
   
   建议去掉 zlib 相关，这里暂时可以不需要，不然 外层还需要引入zlib

2. `ANDROID_API`  这个的配置， 先把21 改成15把，不然低版本直接不支持了

3. 在Configure 加一个配置项， `-D__ANDROID_API__=15`， **注意， armeabi的 用15， 其他架构用 21**



### 4.2 使用

修改环境变量

```bash
NDK=${ANDROID_NDK}
```



### 4.3 编译

```bash
sh ./build-gmssl4android.sh android # armeabi
sh ./build-gmssl4android.sh android-armeabi #armeabi-v7a
sh ./build-gmssl4android.sh android64-aarch64 #arm64_v8a
sh ./build-gmssl4android.sh android-x86 #x86
sh ./build-gmssl4android.sh android64 #x86-64 
sh ./build-gmssl4android.sh android-mips #mips
sh ./build-gmssl4android.sh android-mips64 #mips64
```



## 5. linux 编译

### 5.1 arm64,__loongarch64 编译修改

```cmake
……Linux/aarch64/libcrypto.a(sha1-armv8.o): relocation R_AARCH64_PREL64 against symbol `OPENSSL_armcap_P' which may bind externally can not be used when making a shared object; recompile with -fPIC
……Linux/aarch64/libcrypto.a(sha1-armv8.o): in function `sha1_block_armv8':
(.text+0x1240): dangerous relocation: 不支持的重定位
……Linux/aarch64/libcrypto.a(chacha-armv8.o): relocation R_AARCH64_PREL64 against symbol `OPENSSL_armcap_P' which may bind externally can not be used when making a shared object; recompile with -fPIC
……Linux/aarch64/libcrypto.a(chacha-armv8.o):(.text+0x20): dangerous relocation: 不支持的重定位
……Linux/aarch64/libcrypto.a(poly1305-armv8.o): relocation R_AARCH64_ADR_PREL_LO21 against symbol `poly1305_blocks' which may bind externally can not be used when making a shared object; recompile with -fPIC
……Linux/aarch64/libcrypto.a(poly1305-armv8.o): in function `poly1305_init':
(.text+0x40): dangerous relocation: 不支持的重定位
……Linux/aarch64/libcrypto.a(poly1305-armv8.o): relocation R_AARCH64_ADR_PREL_LO21 against symbol `poly1305_emit' which may bind externally can not be used when making a shared object; recompile with -fPIC
/usr/bin/ld: (.text+0x48): dangerous relocation: 不支持的重定位
……Linux/aarch64/libcrypto.a(poly1305-armv8.o): relocation R_AARCH64_PREL64 against symbol `OPENSSL_armcap_P' which may bind externally can not be used when making a shared object; recompile with -fPIC
……Linux/aarch64/libcrypto.a(poly1305-armv8.o): in function `poly1305_emit_neon':
(.text+0x9a0): dangerous relocation: 不支持的重定位
……Linux/aarch64/libcrypto.a(sha256-armv8.o): relocation R_AARCH64_PREL64 against symbol `OPENSSL_armcap_P' which may bind externally can not be used when making a shared object; recompile with -fPIC
……Linux/aarch64/libcrypto.a(sha256-armv8.o): in function `sha256_block_data_order':
(.text+0xf88): dangerous relocation: 不支持的重定位
……Linux/aarch64/libcrypto.a(sha512-armv8.o): relocation R_AARCH64_PREL64 against symbol `OPENSSL_armcap_P' which may bind externally can not be used when making a shared object; recompile with -fPIC
……Linux/aarch64/libcrypto.a(sha512-armv8.o): in function `sha512_block_data_order':
(.text+0x1108): dangerous relocation: 不支持的重定位
collect2: error: ld returned 1 exit status
make[2]: *** [src/CMakeFiles/NetSealClientCNG.dir/build.make:474：bin/lib/libNetSealClientCNG.so] 错误 1
make[1]: *** [CMakeFiles/Makefile2:91：src/CMakeFiles/NetSealClientCNG.dir/all] 错误 2
make: *** [Makefile:84：all] 错误 2
```



[Openssl aarch64 静态库使用遇到libcrypto.a(xxxx-armv8.o)……问题解决方案记录_r_aarch64-CSDN博客](https://blog.csdn.net/qq_41797877/article/details/124153046)

- crypto/chacha/asm/chacha-armv8.pl
  
  127 行增加 
  
  ```cmake
  
  125	.text
  126	
  127	.extern	OPENSSL_armcap_P
  + 	.hidden	OPENSSL_armcap_P
  128	.align	5
  129	.Lsigma:
  
  ```
  
- crypto/poly1305/asm/poly1305-armv8.pl
  58行开始修改
  
  ```cmake
  58	// forward "declarations" are required for Apple
  59	.extern	OPENSSL_armcap_P
  +	.hidden	OPENSSL_armcap_P
  +	.globl	poly1305_init
  +	.hidden	poly1305_init
  60	.globl	poly1305_blocks
  +	.hidden	poly1305_blocks
  61.globl	poly1305_emit
  +	.hidden	poly1305_emit
  62	
  -	.globl	poly1305_init
  64	.type	poly1305_init,%function
  64	.align	5
  poly1305_init:
  ```
  
- crypto/sha/asm/sha1-armv8.pl
  178 增加
  
  ```cmake
  176	.text
  177	
  178	.extern	OPENSSL_armcap_P
  +	.hidden OPENSSL_armcap_P
  179	.globl	sha1_block_data_order
  180	.type	sha1_block_data_order,%function
  181	.align	6
  
  ```
  
  去除
  
  ```cmake
  330	#endif
  331	.asciz	"SHA1 block transform for ARMv8, CRYPTOGAMS by <appro\@openssl.org>"
  332	.align	2
  -	.comm	OPENSSL_armcap_P,4,4
  333	___
  334	}}}
  ```

- crypto/sha/asm/sha512-armv8.pl
  
  增加174
  
  
  ```cmake
  193	.text
  194	
  195	.extern	OPENSSL_armcap_P
  +	.hidden	OPENSSL_armcap_P
  197	.globl	$func
  198	.type	$func,%function
  199	.align	6
  
  ```
  
  去除415
  
  ```cmake
  841	___
  842	}
  843	
  -	$code.=<<___;
  -	#ifndef	__KERNEL__
  -	.comm	OPENSSL_armcap_P,4,4
  -	#endif
  -	___
  -	
  849	{   my  %opcode = (
  ```
  
- engines/afalg/e_afalg.c
  111行
  
  ```cpp
  - #ifdef __loongarch64
  + #if defined(__loongarch64) || defined(__aarch64__) 
  ```

### 5.2 脚本

```bash
 $ ./config
 $ make
 $ sudo make install
```



>  **注意：aarch64 ，统信系统编译, 需要增加 “ DGMSSL_NO_TURBO ” **
>
> ```
> ./config -DGMSSL_NO_TURBO
> ```



## 6. 修改crash

crypto/evp/e_chacha20_poly1305.c
319行，ctx-->actx

```
        -OPENSSL_cleanse(ctx->cipher_data, sizeof(*ctx) + Poly1305_ctx_size());
        +OPENSSL_cleanse(ctx->cipher_data, sizeof(*actx) + Poly1305_ctx_size());
```

  这是openssl 的bug，在后续openssl的版本中已经修复。



## 7. ssl 增加srtp 加密套件

- include/openssl/srtp.h
  36行

```cpp
# define SRTP_SM4_CTR_NULL      0x0009
# define SRTP_SM4_CTR_HMAC_SM3_128 0x000a
```

- ssl/d1_srtp.c
  39行

```cpp
    {
     "SRTP_SM4_CTR_NULL",
     SRTP_SM4_CTR_NULL,
     },
    {
     "SRTP_SM4_CTR_HMAC_SM3_128",
     SRTP_SM4_CTR_HMAC_SM3_128,
    },
```





## 参考

1. [GmSSL 在Windows上的使用（编译和使用）_gmssl 命令使用](https://blog.csdn.net/u012156872/article/details/124956060?spm=1001.2101.3001.6650.17&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-17-124956060-blog-112325129.235%5Ev38%5Epc_relevant_sort_base2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-17-124956060-blog-112325129.235%5Ev38%5Epc_relevant_sort_base2&utm_relevant_index=24)

2. [在Windows下安装GmSSL_windows安装gmssl](https://blog.csdn.net/zyhse/article/details/112325129)

3. [wangp8895/gmssl-for-android: Android下编译gmssl静态库 (github.com)](https://github.com/wangp8895/gmssl-for-android/tree/master)

4. [Openssl aarch64 openssl编译问题](https://blog.csdn.net/qq_41797877/article/details/124153046)
