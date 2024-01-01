---
layout: post
title: android MediaCodec 简介
date: 2023-12-03 23:10:00 +0800
author: Fisher
pin: True
meta: Post
categories: webrtc android codec video
---


* content
{:toc}

---


## 1. MediaCodec 介绍

MediaCodec 是 从API 16 后引入的处理音视频编解码的类，它可以直接访问 Android 底层的多媒体编解码器，通常与MediaExtractor, MediaSync, MediaMuxer, MediaCrypto, MediaDrm, Image, Surface, 以及AudioTrack一起使用；



## 2. !!! MediaCodec的编解码流程

![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/android-mediacodec.assets/mediacodec-flow.png)


MediaCodec 的数据分为两个部分，从数据的输入到编解码后的数据的输出：

- input ： MediaCodec 会通过getInputBuffer(int bufferId) 去拿到一个空的 ByteBuffer , 用来给客户端去填入数据(比如解码，编码的数据)，MediaCodec 会用这些数据进行解码/编码处理
- output ： MediaCodec 会把解码/编码的数据填充到一个空的 buffer 中，然后把这个填满数据的buffer给到客户端，之后需要释放这个 buffer，MediaCodec 才能继续填充数据。



主要流程如下：

1. 使用者从MediaCodec请求一个空的输入buffer（ByteBuffer），填充满数据后将它传递给MediaCodec处理。
2. MediaCodec处理完这些数据并将处理结果输出至一个空的输出buffer（ByteBuffer）中。 
3. 使用者从MediaCodec获取输出buffer的数据，消耗掉里面的数据，使用完输出buffer的数据之后，将其释放回编解码器。



>**注意**
>
>1. **MediaCodec 内部使用异步的方式对 input 和 output 进行数据处理**，MediaCodec 会把 input 的数据处理好给到 output，供用户去渲染；
>2. **output 的数据必须释放，不然会影响下一次的数据填充**。



## 3. 数据类型

编码器共支持3中数据类型：

压缩数据
原始视频数据
原始音频数据

这三种数据都是可以通过 ByteBuffer 去处理；但是你可以使用 Surface 去解析原始的视频数据，Surface 使用底层的视频缓冲，而不是映射或拷贝到 ByteBuffer，这样会大大提高编码效率。
但使用 Surface 时，无法访问到原始的视频数据，所以，你可以使用 ImageReader 来访问未加密的解码(原始)数据。在 ByteBuffer 的模式下，你也可以使用 Image 或者 getInput/OutputImage(int) 来获取原始视频帧。



## 4. 编解码的生命周期

它的生命周期可以分为三个：Stopped，Executing 和 Released。

而 Stopped 和 Executing 都有各自的生命周期：

- Stopped：Error、Uninitialized 和 Configured
  当调用 MediaCodec 时，此时会处于 Uninitialized 状态，当调用 configure 之后，就会处于 Configured 状态；然后调用 start() 进入 Executing 状态，接着就可以处理数据了。
- Executing：Flushed、Running 和 End of Stream
  当调用 start() 就会进入 Executing 下的 Flushed 状态，此时会拿到所有的 buffers，当第一帧数据从 dequeueInoutBuffer 队列流出时，就会进入 Running 状态，大部分时间都在这个状态处理数据，当队列中有 end-of-stream 标志时，就会进入 End of Stream 状态，此时不再接收 input buffer，但是会继续生成 output buffer，直到 output 也接收到 end-of-stream 标志。你可以使用 flush() 重新回到 Flushed 状态。

周期图如下：

![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/android-mediacodec.assets/mediacodec_states.svg)

 如图可以看到：

1. 当创建编解码器的时候处于未初始化状态。首先你需要调用configure()方法让它处于Configured状态，然后调用start()方法让其处于Executing状态。在Executing状态下，你就可以使用上面提到的缓冲区来处理数据。
2. Executing的状态下也分为三种子状态：Flushed, Running、End-of-Stream。在start() 调用后，编解码器处于Flushed状态，这个状态下它保存着所有的缓冲区。一旦第一个输入buffer出现了，编解码器就会自动运行到Running的状态。当带有end-of-stream标志的buffer进去后，编解码器会进入End-of-Stream状态，这种状态下编解码器不在接受输入buffer，但是仍然在产生输出的buffer。此时你可以调用flush()方法，将编解码器重置于Flushed状态。
3. 调用stop()将编解码器返回到未初始化状态，然后可以重新配置。 完成使用编解码器后，您必须通过调用release()来释放它。
4. 在极少数情况下，编解码器可能会遇到错误并转到错误状态。 这是使用来自排队操作的无效返回值或有时通过异常来传达的。 调用reset()使编解码器再次可用。 您可以从任何状态调用它来将编解码器移回未初始化状态。 否则，调用 release()动到终端释放状态。



## 5. MediaCodec 的主要 API 

1. MediaCodec创建： createDecoderByType/createEncoderByType：根据特定MIME类型(如"video/avc")创建codec。 createByCodecName：知道组件的确切名称(如OMX.google.mp3.decoder)的时候，根据组件名创建codec。使用MediaCodecList可以获取组件的名称。

2. configure：配置解码器或者编码器。

3. start：成功配置组件后调用start。

4. buffer处理的接口： 

   getInputBuffers：获取需要编码数据的输入流队列，返回的是一个ByteBuffer数组 ，已弃用
   dequeueInputBuffer：从输入流队列中取数据进行编码操作。 
   queueInputBuffer：输入流入队列。 

   getOutputBuffers：获取编解码之后的数据输出流队列，返回的是一个ByteBuffer数组 ，已弃用
   dequeueOutputBuffer：从输出队列中取出编码操作之后的数据。 
   releaseOutputBuffer：处理完成，释放ByteBuffer数据。

5. flush：清空的输入和输出端口。 

6. stop：终止decode/encode会话 

7. release：释放编解码器实例使用的资源。



## 6. MediaCodec 流控

### 6.1 流控基本概念

流控就是流量控制。**为什么要控制，因为条件有限！**涉及到了 TCP 和视频编码：

对 TCP 来说就是控制单位时间内发送数据包的数据量，对编码来说就是控制单位时间内输出数据的数据量。

- TCP 的限制条件是网络带宽，流控就是在避免造成或者加剧网络拥塞的前提下，尽可能利用网络带宽。带宽够、网络好，我们就加快速度发送数据包，出现了延迟增大、丢包之后，就放慢发包的速度（因为继续高速发包，可能会加剧网络拥塞，反而发得更慢）。
- 视频编码的限制条件最初是解码器的能力，码率太高就会无法解码，后来随着 codec 的发展，解码能力不再是瓶颈，限制条件变成了传输带宽/文件大小，我们希望在控制数据量的前提下，画面质量尽可能高。

一般编码器都可以设置一个目标码率，但编码器的实际输出码率不会完全符合设置，因为在编码过程中实际可以控制的并不是最终输出的码率，而是编码过程中的一个量化参数（Quantization Parameter，QP），它和码率并没有固定的关系，而是取决于图像内容。

无论是要发送的 TCP 数据包，还是要编码的图像，都可能出现“尖峰”，也就是短时间内出现较大的数据量。TCP 面对尖峰，可以选择不为所动（尤其是网络已经拥塞的时候），这没有太大的问题，但如果视频编码也对尖峰不为所动，那图像质量就会大打折扣了。如果有几帧数据量特别大，但仍要把码率控制在原来的水平，那势必要损失更多的信息，因此图像失真就会更严重。



### 6.2 Android 硬编码流控

MediaCodec 流控相关的接口并不多，一是配置时设置目标码率和码率控制模式，二是动态调整目标码率(Android 19 版本以上)。

配置时指定目标码率和码率控制模式：

```
mediaFormat.setInteger(MediaFormat.KEY_BIT_RATE, bitRate);
mediaFormat.setInteger(MediaFormat.KEY_BITRATE_MODE,
MediaCodecInfo.EncoderCapabilities.BITRATE_MODE_VBR);
mVideoCodec.configure(mediaFormat, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE);
```

码率控制模式有三种：

- CQ  表示完全不控制码率，尽最大可能保证图像质量；
- CBR 表示编码器会尽量把输出码率控制为设定值，即我们前面提到的“不为所动”；
- VBR 表示编码器会根据图像内容的复杂度（实际上是帧间变化量的大小）来动态调整输出码率，图像复杂则码率高，图像简单则码率低；

动态调整目标码率： 

```
Bundle param = new Bundle();
param.putInt(MediaCodec.PARAMETER_KEY_VIDEO_BITRATE, bitrate);
mediaCodec.setParameters(param);
```



### 6.3 Android 流控策略选择

- 质量要求高、不在乎带宽、解码器支持码率剧烈波动的情况下，可以选择 CQ 码率控制策略。
- VBR 输出码率会在一定范围内波动，对于小幅晃动，方块效应会有所改善，但对剧烈晃动仍无能为力；连续调低码率则会导致码率急剧下降，如果无法接受这个问题，那 VBR 就不是好的选择。
- CBR 的优点是稳定可控，这样对实时性的保证有帮助。所以 WebRTC 开发中一般使用的是CBR。



## 7. 同步解码

### 7.1 流程

- 创建并配置MediaCodec对象。
- 循环直到完成：
  - 如果输入buffer准备好了：
    - 读取一段输入，将其填充到输入buffer中
  - 如果输出buffer准备好了：
    - 从输出buffer中获取数据进行处理。
- 处理完毕后，release MediaCodec 对象。

![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/android-mediacodec.assets/decode.webp)

### 7.2 解码流程

为了方便后续的音频解码，这里我们定义一个基类，用来解析视频和音频，由于是同步解析视频，我们应该在线程中去解析，所以继承 Runnable:

```java
/**
* 解码基类，用于解码音视频
*/
abstract class BaseDecode implements Runnable {
    final static int VIDEO = 1;
    final static int AUDIO = 2;
    //等待时间
    final static int TIME_US = 1000;
    MediaFormat mediaFormat;
    MediaCodec mediaCodec;
    MyExtractor extractor;

    private boolean isDone;
    public BaseDecode() {
      try {
        //获取 MediaExtractor
        extractor = new MyExtractor(Constants.VIDEO_PATH);
        //判断是音频还是视频
        int type = decodeType();
        //拿到音频或视频的 MediaFormat
        mediaFormat = (type == VIDEO ? extractor.getVideoFormat() : extractor.getAudioFormat());
        String mime = mediaFormat.getString(MediaFormat.KEY_MIME);
        //选择要解析的轨道
        extractor.selectTrack(type == VIDEO ? extractor.getVideoTrackId() : extractor.getAudioTrackId());
        //创建 MediaCodec
        mediaCodec = MediaCodec.createDecoderByType(mime);
        //由子类去配置
        configure();
        //开始工作，进入编解码状态
        mediaCodec.start();
      } catch (IOException e) {
        e.printStackTrace();
      }
    }
}
```

可以看到，根据视频或者视频拿到不同的 MediaFormat ，并根据 mime 类型，创建 MediaCodec ;

这样，第一步就创建好了，接着，从上面的状态知道，现在处于Uninitialized 状态； 接着需要调用 configure方法，进入 Configured 状态，这一步由子类完成，比如视频的：

```java
 @Override
 void configure() {
     mediaCodec.configure(mediaFormat, new Surface(mTextureView.getSurfaceTexture()), null, 0);
 }
```

可以看到，通过mediaCodec.configure() 配置了当前的 MediaFormat，和要播放视频的 Surface，这里用的是 TextureView。 接着，调用 MediaCodec 的 start() ，进入 Executing 状态，可以编解码了。



### 7.3 视频解码

解码流程根据这张图 

![在这里插入图片描述]({{ site.url }}{{ site.baseurl }}/images/android-mediacodec.assets/mediacodec-flow.png)

#### **7.3.1 输入**

上面说到 BaseDecode 继承 Runnable ，所以，解码的流程在 run 方法中。

```java
@Override
public void run() {

  try {

    MediaCodec.BufferInfo info = new MediaCodec.BufferInfo();
    //编码
    while (!isDone) {
      /**
                     * 延迟 TIME_US 等待拿到空的 input buffer下标，单位为 us
                     * -1 表示一直等待，知道拿到数据，0 表示立即返回
                     */
      int inputBufferId = mediaCodec.dequeueInputBuffer(TIME_US);

      if (inputBufferId > 0) {
        //拿到 可用的，空的 input buffer
        ByteBuffer inputBuffer = mediaCodec.getInputBuffer(inputBufferId);
        if (inputBuffer != null) {
          /**
                             * 通过 mediaExtractor.readSampleData(buffer, 0) 拿到视频的当前帧的buffer
                             * 通过 mediaExtractor.advance() 拿到下一帧
                             */
          int size = extractor.readBuffer(inputBuffer);
          //解析数据
          if (size >= 0) {
            mediaCodec.queueInputBuffer(
              inputBufferId,
              0,
              size,
              extractor.getSampleTime(),
              extractor.getSampleFlags()
            );
          } else {
            //结束,传递 end-of-stream 标志
            mediaCodec.queueInputBuffer(
              inputBufferId,
              0,
              0,
              0,
              MediaCodec.BUFFER_FLAG_END_OF_STREAM
            );
            isDone = true;

          }
        }
      }
      //解码输出交给子类
      boolean isFinish =  handleOutputData(info);
      if (isFinish){
        break;
      }

    }

    done();

  } catch (Exception e) {
    e.printStackTrace();
  }
}

protected void done(){
  try {
    isDone = true;
    //释放 mediacodec
    mediaCodec.stop();
    mediaCodec.release();

    //释放 MediaExtractor
    extractor.release();
  } catch (Exception e) {
    e.printStackTrace();
  }

}


abstract boolean handleOutputData(MediaCodec.BufferInfo info);
```

在一个 while 循环中，不断的解码，上面的代码做到以下流程：

从 MediaCodec 拿到一个空闲的 buffer 从视频中，拿到视频的当前帧的数据，并填充到 MediaCodec 的buffer中 使用 mediaCodec.queueInputBuffer() 把buffer 的数据交给 MediaCodec 去解码



#### 7.3.2 输出

上面的输出过程，由 handleOutputData() 去实现，去到 VideoDecodeSync ，代码实现如下：

```java
@Override
boolean handleOutputData(MediaCodec.BufferInfo info) {
    //等到拿到输出的buffer下标
    int outputId = mediaCodec.dequeueOutputBuffer(info, TIME_US);

    if (outputId >= 0){
        //释放buffer，并渲染到 Surface 中
        mediaCodec.releaseOutputBuffer(outputId, true);
    }

    // 在所有解码后的帧都被渲染后，就可以停止播放了
    if ((info.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
        Log.e(TAG, "zsr OutputBuffer BUFFER_FLAG_END_OF_STREAM");

        return true;
    }

    return false;
}
```

1. 拿到output buffer
2. 释放 buffer，并渲染视频到 Surface 中，由 releaseOutputBuffer() 第二个参数控制。

但是，发现视频好像是倍速播放一样。



#### 7.3.3 矫正显示时间戳

为什么会出现上面的情况呢？ 正常来说，我们的视频播放的帧率大概为 30，即 30fps，大概每 33.33ms 播放一帧；但解码的出一帧的时间大概就是几 ms 的事情，如果一解码就直接给 Surface 去显示的话，视频看起来就想倍速播放的一样。 那你可能说，直接延时 30ms 播放就可以了啊，那不是标准吗？ 是的，30fps 是标准，但不代表每个视频都是30，这里就需要学习 音视频的基础知识 ，DTS 和 PTS。

DTS (Decoding Time Stamp) ： 即解码时间戳，这个时间戳的意义在于告诉播放器该在什么时候解码这一帧的数据 PTS (Presentation Time Stamp) ：显示时间戳，这个时间戳告诉播放器，什么时候播放这一帧 需要注意的是，虽然 DTS 、PTS 是用于指导播放端的行为，但他们是在编码的时候，由编码器生成的。 在没有B帧的情况下，DTS和 PTS 的输出顺序是一样的，一旦存在 B 帧，则顺序不一样。 这里，我们只需要关心 PTS ，即显示时间戳。通过 MediaCodec.BufferInfo 的 presentationTimeUs 可以拿到当前的 pts 时间戳，单位是微妙，它是相对于0开始播放的时间，所以，我们可以使用系统时间差来模仿两帧的时间差，这样当解码过来的 pts 比这个时间差快，则延时以下再输出到 Surface ，如果不是，则直接显示到 Surface 中。

由于在线程中，所以，我们可以使用 Thread.sleep() 去实现，在渲染到 Surface 之前：

```java
 // 用于对准视频的时间戳
 private long startMs = -1;
if (outputId >= 0){
     if (mStartMs == -1) {
            mStartMs = System.currentTimeMillis();
      }
    //矫正pts
     sleepRender(info, startMs);
     //释放buffer，并渲染到 Surface 中
     mediaCodec.releaseOutputBuffer(outputId, true);
 }
#sleepRender
    /**
     * 数据的时间戳对齐
     **/
    private void sleepRender(MediaCodec.BufferInfo info, long startMs) {
        /**
         * 注意这里是以 0 为出事目标的，info.presenttationTimes 的单位为微秒
         * 这里用系统时间来模拟两帧的时间差
         */
        long ptsTimes = info.presentationTimeUs / 1000;
        long systemTimes = System.currentTimeMillis() - startMs;
        long timeDifference = ptsTimes - systemTimes;
        // 如果当前帧比系统时间差快了，则延时以下
        if (timeDifference > 0) {
            try {
                Thread.sleep(timeDifference);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }
    }
```

现在时间播放就正常了。



### 7.4 解码音频

上面已经理解了视频的解码，那么音频的解码就比较简单了，新建一个 AudioDecodeSync 类，继承BaseDecode ,在 configure 方法中去配置 MediaCodec ，由于不需要 Surface ，传递 null 即可。

```java
 @Override
 void configure() {
     mediaCodec.configure(mediaFormat, null, null, 0);
 }
```

虽然我们不需要使用 Surface ，但是需要播放视频，那么则需要使用 AudioTrack，如果对它不需要，可以参考 Android 音视频开发(一) – 使用AudioRecord 录制PCM(录音)；AudioTrack播放音频.

所以，在 AudioDecodeSync 的构造方法中，需要配置以下 AudioTrack ：

```java
class AudioDecodeSync extends BaseDecode {

        private int mPcmEncode;
        //一帧的最小buffer大小
        private final int minBufferSize;
        private AudioTrack audioTrack;


        public AudioDecodeSync() {
            //拿到采样率
            if (mediaFormat.containsKey(MediaFormat.KEY_PCM_ENCODING)) {
                mPcmEncode = mediaFormat.getInteger(MediaFormat.KEY_PCM_ENCODING);
            } else {
                //默认采样率为 16bit
                mPcmEncode = AudioFormat.ENCODING_PCM_16BIT;
            }

            //音频采样率
            int sampleRate = mediaFormat.getInteger(MediaFormat.KEY_SAMPLE_RATE);
            //获取视频通道数
            int channelCount = mediaFormat.getInteger(MediaFormat.KEY_CHANNEL_COUNT);

            //拿到声道
            int channelConfig = channelCount == 1 ? AudioFormat.CHANNEL_IN_MONO : AudioFormat.CHANNEL_IN_STEREO;
            minBufferSize = AudioTrack.getMinBufferSize(sampleRate, channelConfig, mPcmEncode);


            /**
             * 设置音频信息属性
             * 1.设置支持多媒体属性，比如audio，video
             * 2.设置音频格式，比如 music
             */
            AudioAttributes attributes = new AudioAttributes.Builder()
                    .setUsage(AudioAttributes.USAGE_MEDIA)
                    .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
                    .build();
            /**
             * 设置音频数据
             * 1. 设置采样率
             * 2. 设置采样位数
             * 3. 设置声道
             */
            AudioFormat format = new AudioFormat.Builder()
                    .setSampleRate(sampleRate)
                    .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
                    .setChannelMask(channelConfig)
                    .build();


            //配置 audioTrack
            audioTrack = new AudioTrack(
                    attributes,
                    format,
                    minBufferSize,
                    AudioTrack.MODE_STREAM, //采用流模式
                    AudioManager.AUDIO_SESSION_ID_GENERATE
            );
            //监听播放
            audioTrack.play();
        }
 }
```

拿到 AudioTrack 之后，就可以使用 play() 方法，去监听是否有数据写入，开始播放音频了。

在 handleOutputData：

```java
@Override
boolean handleOutputData(MediaCodec.BufferInfo info) {
    //拿到output buffer
    int outputIndex = mediaCodec.dequeueOutputBuffer(info, TIME_US);
    ByteBuffer outputBuffer;
    if (outputIndex >= 0) {
      outputBuffer = mediaCodec.getOutputBuffer(outputIndex);
      //写数据到 AudioTrack 只，实现音频播放
      audioTrack.write(outputBuffer, info.size, AudioTrack.WRITE_BLOCKING);
      mediaCodec.releaseOutputBuffer(outputIndex, false);
    }
    // 在所有解码后的帧都被渲染后，就可以停止播放了
    if ((info.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0) {
      Log.e(TAG, "zsr OutputBuffer BUFFER_FLAG_END_OF_STREAM");

      return true;
    }
    return false;
}
```

你会发现音频正常播放了，且没有什么快进的意思，因为音频的时间戳是比较连续的，因此不用矫正。



### 7.5 音视频同步

那么，如何让音视频同步呢？其实也不算难，就是开辟两个线程，让它同时播放即可：

```java
if (mExecutorService.isShutdown()) {
            mExecutorService = Executors.newFixedThreadPool(2);
        }
        mVideoSync = new VideoDecodeSync();
        mAudioDecodeSync = new AudioDecodeSync();
        mExecutorService.execute(mVideoSync);
        mExecutorService.execute(mAudioDecodeSync);
  }
```



## 8. 异步解码

在 5.0 之后，google 建议使用异步解码的方式去使用 MediaCodec，使用也非常简单，只需要使用 setCallback 方法即可。 比如解析上面的视频，步骤可以如下：

- 创建并配置MediaCodec对象。
- 给MediaCodec对象设置回调MediaCodec.Callback
- 在onInputBufferAvailable回调中：
    - 读取一段输入，将其填充到输入buffer中
- 在onOutputBufferAvailable回调中：
    - 从输出buffer中获取数据进行处理。
- 处理完毕后，release MediaCodec 对象。



使用 MediaExtractor 解析视频，拿到 MediaFormat 使用 MediaCodec.setCallback() 方法 调用 mediaCodec.configure() 和 mediaCodec.start() 开始解码 所以，代码如下：

```java
    class AsyncDecode {
        MediaFormat mediaFormat;
        MediaCodec mediaCodec;
        MyExtractor extractor;

        public AsyncDecode() {
            try {
                //解析视频，拿到 mediaformat
                extractor = new MyExtractor(Constants.VIDEO_PATH);
                mediaFormat = (extractor.getVideoFormat());
                String mime = mediaFormat.getString(MediaFormat.KEY_MIME);
                extractor.selectTrack(extractor.getVideoTrackId());
                mediaCodec = MediaCodec.createDecoderByType(mime);

            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        private void start() {
            //异步解码
            mediaCodec.setCallback(new MediaCodec.Callback() {
                @Override
                public void onInputBufferAvailable(@NonNull MediaCodec codec, int index) {
                        ByteBuffer inputBuffer = codec.getInputBuffer(index);
                    int size = extractor.readBuffer(inputBuffer);
                    if (size >= 0) {
                        codec.queueInputBuffer(
                                index,
                                0,
                                size,
                                extractor.getSampleTime(),
                                extractor.getSampleFlags()
                        );
                        handler.sendEmptyMessage(1);
                    } else {
                        //结束
                        codec.queueInputBuffer(
                                index,
                                0,
                                0,
                                0,
                                MediaCodec.BUFFER_FLAG_END_OF_STREAM
                        );
                    }
                }

                @Override
                public void onOutputBufferAvailable(@NonNull MediaCodec codec, int index, @NonNull MediaCodec.BufferInfo info) {
                        mediaCodec.releaseOutputBuffer(index, true);
                }

                @Override
                public void onError(@NonNull MediaCodec codec, @NonNull MediaCodec.CodecException e) {
                    codec.reset();
                }

                @Override
                public void onOutputFormatChanged(@NonNull MediaCodec codec, @NonNull MediaFormat format) {

                }
            });
            //需要在 setCallback 之后，配置 configure
            mediaCodec.configure(mediaFormat, new Surface(mTextureView.getSurfaceTexture()), null, 0);
            //开始解码
            mediaCodec.start();
        }

    }
```

使用异步解码，与同步解码的流程基本一致；只不过，在同步的代码中，我们通过

```java
int inputBufferId = mediaCodec.dequeueInputBuffer(TIME_US);
```

去等待拿到空闲的 input buffer 下标，而在异步中，则是通过回调

```java
void onInputBufferAvailable(@NonNull MediaCodec codec,int index) 
```





## 参考

[Android 音视频开发(六)： MediaCodec API 详解](https://www.cnblogs.com/renhui/p/7478527.html)

[Android 音视频编解码(二) -- MediaCodec 解码(同步和异步)](https://blog.csdn.net/u011418943/article/details/107561111)

[官网](https://developer.android.google.cn/reference/android/media/MediaCodec?hl=en)

[弄懂MediaCodec](https://zhuanlan.zhihu.com/p/567591816)