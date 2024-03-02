

## 0. 前言

1. 可以添加多个track，
2. 一个track，可以添加到多个MediaStream中
3. 本地，只会在AddStream的时候创建，addtrack，addtransceiver 不会创建。
   说白了，本地是没有MediaStream对象。
4. 远端，是通过收到的sdp，每个mline，对应生成一个Transceiver，同时创建了RtpSenderInternal和RtpReceiverInternal；
   - RtpSenderInternal ，里面没有track
   - RtpReceiverInternal创建的时候，同时会创建Track 创建RtpReceiverInternal, 设置receiver_id 就是 track id，是从远端的sdp中解析出来的，就是对应端的track-id



## 1. 继承关系

```less
NotifierInterface
MediaStreamInterface
Notifier<MediaStreamInterface>
MediaStream
```



### 1.1 MediaStreamInterface

api/media_stream_interface.h

```cpp
// C++ version of https://www.w3.org/TR/mediacapture-streams/#mediastream.
//
// A major difference is that remote audio/video tracks (received by a
// PeerConnection/RtpReceiver) are not synchronized simply by adding them to
// the same stream; a session description with the correct "a=msid" attributes
// must be pushed down.
//
// Thus, this interface acts as simply a container for tracks.
class MediaStreamInterface : public rtc::RefCountInterface,
                             public NotifierInterface {
 public:
  virtual std::string id() const = 0;

  virtual AudioTrackVector GetAudioTracks() = 0;
  virtual VideoTrackVector GetVideoTracks() = 0;
  virtual rtc::scoped_refptr<AudioTrackInterface> FindAudioTrack(
      const std::string& track_id) = 0;
  virtual rtc::scoped_refptr<VideoTrackInterface> FindVideoTrack(
      const std::string& track_id) = 0;

  virtual bool AddTrack(AudioTrackInterface* track) = 0;
  virtual bool AddTrack(VideoTrackInterface* track) = 0;
  virtual bool RemoveTrack(AudioTrackInterface* track) = 0;
  virtual bool RemoveTrack(VideoTrackInterface* track) = 0;
};
```





## 2. 属性

###  const std::string id_

MediaStream  的id。

###  AudioTrackVector audio_tracks_

存放audio track。

###  VideoTrackVector video_tracks_

存放video track。



## 3. 创建MediaStream

### 3.1 MediaStream::Create

pc/media_stream.h

```cpp
rtc::scoped_refptr<MediaStream> MediaStream::Create(const std::string& id) {
  rtc::RefCountedObject<MediaStream>* stream =
      new rtc::RefCountedObject<MediaStream>(id);
  return stream;
}
```



### 3.2 MediaStream::MediaStream

```cpp
MediaStream::MediaStream(const std::string& id) : id_(id) {}
```

id_ 就是 stream id。

## 4. addTrack/removeTrack/FindTrack/GetTracks



### 4.1 MediaStream::AddTrack

```cpp
bool MediaStream::AddTrack(AudioTrackInterface* track) {
  return AddTrack<AudioTrackVector, AudioTrackInterface>(&audio_tracks_, track);
}

bool MediaStream::AddTrack(VideoTrackInterface* track) {
  return AddTrack<VideoTrackVector, VideoTrackInterface>(&video_tracks_, track);
}
```



```cpp
template <typename TrackVector, typename Track>
bool MediaStream::AddTrack(TrackVector* tracks, Track* track) {
  typename TrackVector::iterator it = FindTrack(tracks, track->id());
  if (it != tracks->end())
    return false;
  tracks->push_back(track);
  FireOnChanged();
  return true;
}
```



### 4.2 MediaStream::RemoveTrack

```cpp
bool MediaStream::RemoveTrack(AudioTrackInterface* track) {
  return RemoveTrack<AudioTrackVector>(&audio_tracks_, track);
}

bool MediaStream::RemoveTrack(VideoTrackInterface* track) {
  return RemoveTrack<VideoTrackVector>(&video_tracks_, track);
}
```



```cpp
template <typename TrackVector>
bool MediaStream::RemoveTrack(TrackVector* tracks,
                              MediaStreamTrackInterface* track) {
  RTC_DCHECK(tracks != NULL);
  if (!track)
    return false;
  typename TrackVector::iterator it = FindTrack(tracks, track->id());
  if (it == tracks->end())
    return false;
  tracks->erase(it);
  FireOnChanged();
  return true;
}
```



### 4.3 MediaStream::FindTrack

```cpp
rtc::scoped_refptr<AudioTrackInterface> MediaStream::FindAudioTrack(
    const std::string& track_id) {
  AudioTrackVector::iterator it = FindTrack(&audio_tracks_, track_id);
  if (it == audio_tracks_.end())
    return NULL;
  return *it;
}

rtc::scoped_refptr<VideoTrackInterface> MediaStream::FindVideoTrack(
    const std::string& track_id) {
  VideoTrackVector::iterator it = FindTrack(&video_tracks_, track_id);
  if (it == video_tracks_.end())
    return NULL;
  return *it;
}
```



```cpp
rtc::scoped_refptr<VideoTrackInterface> MediaStream::FindVideoTrack(
    const std::string& track_id) {
  VideoTrackVector::iterator it = FindTrack(&video_tracks_, track_id);
  if (it == video_tracks_.end())
    return NULL;
  return *it;
}
```



```cpp
template <class V>
static typename V::iterator FindTrack(V* vector, const std::string& track_id) {
  typename V::iterator it = vector->begin();
  for (; it != vector->end(); ++it) {
    if ((*it)->id() == track_id) {
      break;
    }
  }
  return it;
}
```



### 4.4 MediaStream::GetTracks

```cpp
AudioTrackVector GetAudioTracks() override { return audio_tracks_; }
VideoTrackVector GetVideoTracks() override { return video_tracks_; }
```



## 5. SdpOfferAnswerHandler 存放MediaStream

```cpp
  // 存放本地 MediaStream，通过AddStream 加入的，目前接口弃用
  // Streams added via AddStream.
  const rtc::scoped_refptr<StreamCollection> local_streams_
      RTC_GUARDED_BY(signaling_thread());
  // 存放Remote MediaStream
  // Streams created as a result of SetRemoteDescription.
  const rtc::scoped_refptr<StreamCollection> remote_streams_
      RTC_GUARDED_BY(signaling_thread());
```



### 5.1 StreamCollection

pc\stream_collection.h

```cpp
// Implementation of StreamCollection.
class StreamCollection : public StreamCollectionInterface {
 public:
  static rtc::scoped_refptr<StreamCollection> Create() {
    rtc::RefCountedObject<StreamCollection>* implementation =
        new rtc::RefCountedObject<StreamCollection>();
    return implementation;
  }

  static rtc::scoped_refptr<StreamCollection> Create(
      StreamCollection* streams) {
    rtc::RefCountedObject<StreamCollection>* implementation =
        new rtc::RefCountedObject<StreamCollection>(streams);
    return implementation;
  }

  virtual size_t count() { return media_streams_.size(); }

  virtual MediaStreamInterface* at(size_t index) {
    return media_streams_.at(index);
  }

  virtual MediaStreamInterface* find(const std::string& id) {
    for (StreamVector::iterator it = media_streams_.begin();
         it != media_streams_.end(); ++it) {
      if ((*it)->id().compare(id) == 0) {
        return (*it);
      }
    }
    return NULL;
  }

  virtual MediaStreamTrackInterface* FindAudioTrack(const std::string& id) {
    for (size_t i = 0; i < media_streams_.size(); ++i) {
      MediaStreamTrackInterface* track = media_streams_[i]->FindAudioTrack(id);
      if (track) {
        return track;
      }
    }
    return NULL;
  }

  virtual MediaStreamTrackInterface* FindVideoTrack(const std::string& id) {
    for (size_t i = 0; i < media_streams_.size(); ++i) {
      MediaStreamTrackInterface* track = media_streams_[i]->FindVideoTrack(id);
      if (track) {
        return track;
      }
    }
    return NULL;
  }

  void AddStream(MediaStreamInterface* stream) {
    for (StreamVector::iterator it = media_streams_.begin();
         it != media_streams_.end(); ++it) {
      if ((*it)->id().compare(stream->id()) == 0)
        return;
    }
    media_streams_.push_back(stream);
  }

  void RemoveStream(MediaStreamInterface* remove_stream) {
    for (StreamVector::iterator it = media_streams_.begin();
         it != media_streams_.end(); ++it) {
      if ((*it)->id().compare(remove_stream->id()) == 0) {
        media_streams_.erase(it);
        break;
      }
    }
  }

 protected:
  StreamCollection() {}
  explicit StreamCollection(StreamCollection* original)
      : media_streams_(original->media_streams_) {}
  typedef std::vector<rtc::scoped_refptr<MediaStreamInterface> > StreamVector;
  StreamVector media_streams_;
};
```





## 6. MediaStream创建时机

### 6.1 ~~SdpOfferAnswerHandler::UpdateRemoteSendersList~~

pc\sdp_offer_answer.cc

```cpp
void SdpOfferAnswerHandler::UpdateRemoteSendersList(
    const cricket::StreamParamsVec& streams,
    bool default_sender_needed,
    cricket::MediaType media_type,
    StreamCollection* new_streams) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  RTC_DCHECK(!IsUnifiedPlan());

  std::vector<RtpSenderInfo>* current_senders =
      rtp_manager()->GetRemoteSenderInfos(media_type);

  // Find removed senders. I.e., senders where the sender id or ssrc don't match
  // the new StreamParam.
  for (auto sender_it = current_senders->begin();
       sender_it != current_senders->end();
       /* incremented manually */) {
    const RtpSenderInfo& info = *sender_it;
    const cricket::StreamParams* params =
        cricket::GetStreamBySsrc(streams, info.first_ssrc);
    std::string params_stream_id;
    if (params) {
      params_stream_id =
          (!params->first_stream_id().empty() ? params->first_stream_id()
                                              : kDefaultStreamId);
    }
    bool sender_exists = params && params->id == info.sender_id &&
                         params_stream_id == info.stream_id;
    // If this is a default track, and we still need it, don't remove it.
    if ((info.stream_id == kDefaultStreamId && default_sender_needed) ||
        sender_exists) {
      ++sender_it;
    } else {
      rtp_manager()->OnRemoteSenderRemoved(
          info, remote_streams_->find(info.stream_id), media_type);
      sender_it = current_senders->erase(sender_it);
    }
  }

  // Find new and active senders.
  for (const cricket::StreamParams& params : streams) {
    if (!params.has_ssrcs()) {
      // The remote endpoint has streams, but didn't signal ssrcs. For an active
      // sender, this means it is coming from a Unified Plan endpoint,so we just
      // create a default.
      default_sender_needed = true;
      break;
    }

    // |params.id| is the sender id and the stream id uses the first of
    // |params.stream_ids|. The remote description could come from a Unified
    // Plan endpoint, with multiple or no stream_ids() signaled. Since this is
    // not supported in Plan B, we just take the first here and create the
    // default stream ID if none is specified.
    //!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!
    // 只处理第一个stream id，根据stream id， 创建MediaStream
    //!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!
    //!!!!!!!!!!!!!!!!!!!!!!!!!!
    const std::string& stream_id =
        (!params.first_stream_id().empty() ? params.first_stream_id()
                                           : kDefaultStreamId);
    const std::string& sender_id = params.id;
    uint32_t ssrc = params.first_ssrc();

    rtc::scoped_refptr<MediaStreamInterface> stream =
        remote_streams_->find(stream_id);
    if (!stream) {
      // This is a new MediaStream. Create a new remote MediaStream.
      stream = MediaStreamProxy::Create(rtc::Thread::Current(),
                                        MediaStream::Create(stream_id));
      remote_streams_->AddStream(stream);
      new_streams->AddStream(stream);
    }

    const RtpSenderInfo* sender_info =
        rtp_manager()->FindSenderInfo(*current_senders, stream_id, sender_id);
    if (!sender_info) {
      current_senders->push_back(RtpSenderInfo(stream_id, sender_id, ssrc));
      rtp_manager()->OnRemoteSenderAdded(current_senders->back(), stream,
                                         media_type);
    }
  }

  // Add default sender if necessary.
  if (default_sender_needed) {
    rtc::scoped_refptr<MediaStreamInterface> default_stream =
        remote_streams_->find(kDefaultStreamId);
    if (!default_stream) {
      // Create the new default MediaStream.
      default_stream = MediaStreamProxy::Create(
          rtc::Thread::Current(), MediaStream::Create(kDefaultStreamId));
      remote_streams_->AddStream(default_stream);
      new_streams->AddStream(default_stream);
    }
    std::string default_sender_id = (media_type == cricket::MEDIA_TYPE_AUDIO)
                                        ? kDefaultAudioSenderId
                                        : kDefaultVideoSenderId;
    const RtpSenderInfo* default_sender_info = rtp_manager()->FindSenderInfo(
        *current_senders, kDefaultStreamId, default_sender_id);
    if (!default_sender_info) {
      current_senders->push_back(
          RtpSenderInfo(kDefaultStreamId, default_sender_id, /*ssrc=*/0));
      rtp_manager()->OnRemoteSenderAdded(current_senders->back(),
                                         default_stream, media_type);
    }
  }
}
```

- plan b 才会走这里
- ![1]({{ site.url }}{{ site.baseurl }}/images/mediastream.assets/ms-planb.png)





### 6.2 SdpOfferAnswerHandler::SetAssociatedRemoteStreams

```less
SdpOfferAnswerHandler::ApplyRemoteDescription
SdpOfferAnswerHandler::SetAssociatedRemoteStreams
```

- unify plan 才会走这里的代码；
- 收到offer后，根据offer的信息，ContentInfos创建对应的Transceiver对象，这时候会创建RtpReceiverInternal和RtpSenderInternal；
- 根据stream_ids 创建对应的MediaStream；把MediaStream 存放到 RtpReceiverInternal中；

pc\sdp_offer_answer.cc

```cpp
// 收到offer后，根据offer的信息，ContentInfos创建对应的Transceiver对象，
// 这时候会创建Receiver和Sender
//
void SdpOfferAnswerHandler::SetAssociatedRemoteStreams(
    rtc::scoped_refptr<RtpReceiverInternal> receiver,  // 新建的transceiver，对应的RtpReceiverInternal
    const std::vector<std::string>& stream_ids, // 收到对方sdp的 streamId
    std::vector<rtc::scoped_refptr<MediaStreamInterface>>* added_streams,  // 下面两个参数是 记录添加
																    // 和删除的stream，做比对用，
																	// 回调onAddStream，onRemoveStream
    std::vector<rtc::scoped_refptr<MediaStreamInterface>>* removed_streams) {
  RTC_DCHECK_RUN_ON(signaling_thread());
  std::vector<rtc::scoped_refptr<MediaStreamInterface>> media_streams;
  // 处理多个stream id，根据stream id， 创建多个MediaStream
  for (const std::string& stream_id : stream_ids) {
    rtc::scoped_refptr<MediaStreamInterface> stream =
        remote_streams_->find(stream_id);
    if (!stream) {
      stream = MediaStreamProxy::Create(rtc::Thread::Current(),
                                        MediaStream::Create(stream_id));
      remote_streams_->AddStream(stream);
      added_streams->push_back(stream);
    }
    media_streams.push_back(stream);
  }
  // Special case: "a=msid" missing, use random stream ID.
  if (media_streams.empty() &&
      !(remote_description()->description()->msid_signaling() &
        cricket::kMsidSignalingMediaSection)) {
    if (!missing_msid_default_stream_) {
      missing_msid_default_stream_ = MediaStreamProxy::Create(
          rtc::Thread::Current(), MediaStream::Create(rtc::CreateRandomUuid()));
      added_streams->push_back(missing_msid_default_stream_);
    }
    media_streams.push_back(missing_msid_default_stream_);
  }
  std::vector<rtc::scoped_refptr<MediaStreamInterface>> previous_streams =
      receiver->streams();
  // SetStreams() will add/remove the receiver's track to/from the streams. This
  // differs from the spec - the spec uses an "addList" and "removeList" to
  // update the stream-track relationships in a later step. We do this earlier,
  // changing the order of things, but the end-result is the same.
  // TODO(hbos): When we remove remote_streams(), use set_stream_ids()
  // instead. https://crbug.com/webrtc/9480
  receiver->SetStreams(media_streams);
  RemoveRemoteStreamsIfEmpty(previous_streams, removed_streams);
}
```



#### 6.2.1 SdpOfferAnswerHandler::AssociateTransceiver

- 收到对方的Offer数据，发现没有创建Transceiver，则创建Transceiver;
-  同时创建RtpSenderInternal， 里面没有track;
-  创建RtpReceiverInternal, 设置receiver_id 就是 对方的 track id;
- 在CreateReceiver的时候，里面默认还会创建Track，存在receiver中，receiver_id 就是track id

```cpp
RTCErrorOr<rtc::scoped_refptr<RtpTransceiverProxyWithInternal<RtpTransceiver>>>
SdpOfferAnswerHandler::AssociateTransceiver(
    cricket::ContentSource source,
    SdpType type,
    size_t mline_index,
    const ContentInfo& content,
    const ContentInfo* old_local_content,
    const ContentInfo* old_remote_content) {
 ...

  const MediaContentDescription* media_desc = content.media_description();
  auto transceiver = transceivers()->FindByMid(content.name);
  if (source == cricket::CS_LOCAL) {
    // Find the RtpTransceiver that corresponds to this m= section, using the
    // mapping between transceivers and m= section indices established when
    // creating the offer.
    if (!transceiver) {
      transceiver = transceivers()->FindByMLineIndex(mline_index);
    }
    if (!transceiver) {
      // This may happen normally when media sections are rejected.
      LOG_AND_RETURN_ERROR(RTCErrorType::INVALID_PARAMETER,
                           "Transceiver not found based on m-line index");
    }
  } else {
    RTC_DCHECK_EQ(source, cricket::CS_REMOTE);
    // If the m= section is sendrecv or recvonly, and there are RtpTransceivers
    // of the same type...
    // When simulcast is requested, a transceiver cannot be associated because
    // AddTrack cannot be called to initialize it.
    if (!transceiver &&
        RtpTransceiverDirectionHasRecv(media_desc->direction()) &&
        !media_desc->HasSimulcast()) {
      transceiver = FindAvailableTransceiverToReceive(media_desc->type());
    }
    // If no RtpTransceiver was found in the previous step, create one with a
    // recvonly direction.
     // 收到对方的Offer数据，发现没有创建Transceiver，则创建Transceiver
     // 同时创建RtpSenderInternal， 里面没有track
     // 创建RtpReceiverInternal, 设置receiver_id 就是 对方的 track id
    if (!transceiver) {
      RTC_LOG(LS_INFO) << "Adding "
                       << cricket::MediaTypeToString(media_desc->type())
                       << " transceiver for MID=" << content.name
                       << " at i=" << mline_index
                       << " in response to the remote description.";
      std::string sender_id = rtc::CreateRandomUuid();
      std::vector<RtpEncodingParameters> send_encodings =
          GetSendEncodingsFromRemoteDescription(*media_desc);
        // !!!!!!!!!!!!!!!!!
        // !!!!!!!!!!!!!!!!!
        // !!!!!!!!!!!!!!!!!
      auto sender = rtp_manager()->CreateSender(media_desc->type(), sender_id,
                                                nullptr, {}, send_encodings);
      std::string receiver_id;
      if (!media_desc->streams().empty()) {
        // !!!!!!!!!!!!!!!!!
        // !!!!!!!!!!!!!!!!!
        // !!!!!!!!!!!!!!!!!
        // receiver_id 就是 track id
        receiver_id = media_desc->streams()[0].id;
        // !!!!!!!!!!!!!!!!!
        // !!!!!!!!!!!!!!!!!
        // !!!!!!!!!!!!!!!!!
      } else {
        receiver_id = rtc::CreateRandomUuid();
      }
        // !!!!!!!!!!!!!!!!!
        // !!!!!!!!!!!!!!!!!
        // !!!!!!!!!!!!!!!!!
        // 在CreateReceiver的时候，里面默认还会创建Track，存在Receiver中，receiver_id 就是track id
      auto receiver =
          rtp_manager()->CreateReceiver(media_desc->type(), receiver_id);
      transceiver = rtp_manager()->CreateAndAddTransceiver(sender, receiver);
      transceiver->internal()->set_direction(
          RtpTransceiverDirection::kRecvOnly);
        // !!!!!!!!!!!!!!!!!
        // !!!!!!!!!!!!!!!!!
        // !!!!!!!!!!!!!!!!!
      if (type == SdpType::kOffer) {
        transceivers()->StableState(transceiver)->set_newly_created();
      }
    }

    ...
  // Associate the found or created RtpTransceiver with the m= section by
  // setting the value of the RtpTransceiver's mid property to the MID of the m=
  // section, and establish a mapping between the transceiver and the index of
  // the m= section.
  transceiver->internal()->set_mid(content.name);
  transceiver->internal()->set_mline_index(mline_index);
  return std::move(transceiver);
}
```



#### 6.2.2 VideoRtpReceiver::VideoRtpReceiver

创建track，receiver_id 就是track_id，是从对方传递的offer中解析，出来的。

pc/video_rtp_receiver.cc

```cpp
VideoRtpReceiver::VideoRtpReceiver(
    rtc::Thread* worker_thread,
    const std::string& receiver_id,
    const std::vector<rtc::scoped_refptr<MediaStreamInterface>>& streams)
    : worker_thread_(worker_thread),// track id
   	  // track id
      id_(receiver_id),
	  // 创建source
      source_(new RefCountedObject<VideoRtpTrackSource>(this)),
	  // track
      track_(VideoTrackProxyWithInternal<VideoTrack>::Create(
          rtc::Thread::Current(),
          worker_thread,
          VideoTrack::Create(
              receiver_id,
              VideoTrackSourceProxy::Create(rtc::Thread::Current(),
                                            worker_thread,
                                            source_),
              worker_thread))),
      attachment_id_(GenerateUniqueId()),
      delay_(JitterBufferDelayProxy::Create(
          rtc::Thread::Current(),
          worker_thread,
          new rtc::RefCountedObject<JitterBufferDelay>(worker_thread))) {
          ...
}
```





## 7. unify plan 和plan b的区别

| 不同点                | plan b                                         | unify plan                                        |
| --------------------- | ---------------------------------------------- | ------------------------------------------------- |
| addTrack              | stream_id 只能添加一个                         | 可以添加多个                                      |
| 创建MediaStream的位置 | SdpOfferAnswerHandler::UpdateRemoteSendersList | SdpOfferAnswerHandler::SetAssociatedRemoteStreams |
|                       |                                                |                                                   |



## 8. MediaStream 本地不会创建，只会创建remote

本地，只会在AddStream的时候创建，addtrack，addtransceiver 不会创建。
说白了，本地是没有MediaStream对象。




