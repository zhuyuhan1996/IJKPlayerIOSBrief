# IJKPlayer iOS播放器源码简析

简介：一个基于FFmpeg的播放器。

## 原理

### FFmpeg

- what?
  - 是一套可以用来记录、转换音视频，并将其转化为流的开源计算机项目，提供了录制、转换以及流化音视频的完整解决方案，它包含了非常先进的音视频编解码库libavcodec，是在Linux平台下开发
- FF项目的名称来自MPEG视频编码标准，FF表示的是Fast Forward, 可以使用GPU加速

### IJKPlayer iOS

提供了三个播放器，分别是

- 基于 MPMoviePlayerController的IJKMPMoviePlayerController
- 基于AVPlayer的IJKAVMoviePlayerController
- 基于FFmpeg的IJKFFMoviePlayerController

分别实现了IJKMediaPlayback的协议，外部通过IJKMediaPlayback相关方法，进行视频播放、暂停、倍速、快进、快退等相关操作

### IJKMPMoviePlayerController

- 继承自MPMoviePlayerController

- 实现了MPMediaPlayback协议，具备了一般的播放器控制功能，例如播放、暂停、停止等。

  ```objective-c
  @protocol MPMediaPlayback
  
  // Prepares the current queue for playback, interrupting any active (non-mixible) audio sessions.
  // Automatically invoked when -play is called if the player is not already prepared.
  - (void)prepareToPlay;
  
  // Returns YES if prepared for playback.
  @property(nonatomic, readonly) BOOL isPreparedToPlay;
  
  // Plays items from the current queue, resuming paused playback if possible.
  - (void)play;
  
  // Pauses playback if playing.
  - (void)pause;
  
  // Ends playback. Calling -play again will start from the beginnning of the queue.
  - (void)stop;
  
  // The current playback time of the now playing item in seconds.
  @property(nonatomic) NSTimeInterval currentPlaybackTime;
  
  // The current playback rate of the now playing item. Default is 1.0 (normal speed).
  // Pausing will set the rate to 0.0. Setting the rate to non-zero implies playing.
  @property(nonatomic) float currentPlaybackRate;
  
  // The seeking rate will increase the longer scanning is active.
  - (void)beginSeekingForward;
  - (void)beginSeekingBackward;
  - (void)endSeeking;
  
  @end
  
  // Posted when the prepared state changes of an object conforming to the MPMediaPlayback protocol changes.
  // This supersedes MPMoviePlayerContentPreloadDidFinishNotification.
  MP_EXTERN NSString * const MPMediaPlaybackIsPreparedToPlayDidChangeNotification
  ```

- 不支持设置进度、调整音量、倍速，退后台自动暂停等功能
- 对MPMoviePlayerController中的方法进行了封装，广播进行了转发

MPMoviePlayerController提供的播放器具有高度的封装性，功能也相对较少，使得自定义播放器变的很难，也很难满足开发的需求，

### IJKAVMoviePlayerController

对AVPlayer进行了封装

- 初始化的时候并没有初始化AVPlayer，在外界需要调用prepareToPlay去获取资源

- 出于多媒体文件较大的原因，apple对Asset的属性采用了懒加载的方式。

  - 在创建AVAsset的时候，只生成一个实例
  - 当第一次访问时，才会根据多媒体的数据初始化这个属性
  - 采用了AVAsynchronousKeyValueLoading协议，异步加载相关资源
  - 资源加载完成后初始化AVPlayerItem，并监听其相关属性的变化
  - 初始化AVPlayer,并监听其相关属性

  初始化AVAsset

  ```objective-c
  - (void)prepareToPlay
  {
      AVURLAsset *asset = [AVURLAsset URLAssetWithURL:_playUrl options:nil];
      NSArray *requestedKeys = @[@"playable"];
      
      _playAsset = asset;
      [asset loadValuesAsynchronouslyForKeys:requestedKeys
                           completionHandler:^{
                               dispatch_async( dispatch_get_main_queue(), ^{
                                   [self didPrepareToPlayAsset:asset withKeys:requestedKeys];
                                   [[NSNotificationCenter defaultCenter]
                                    postNotificationName:IJKMPMovieNaturalSizeAvailableNotification
                                    object:self];
  
                                   [self setPlaybackVolume:_playbackVolume];
                               });
                           }];
  }
  ```

  初始化AVPlayerItem，监听其属性变化

  ```objective-c
  - (void)didPrepareToPlayAsset:(AVURLAsset *)asset withKeys:(NSArray *)requestedKeys
  {
      if (_isShutdown)
          return;
      
      /* Make sure that the value of each key has loaded successfully. */
      for (NSString *thisKey in requestedKeys)
      {
          NSError *error = nil;
          AVKeyValueStatus keyStatus = [asset statusOfValueForKey:thisKey error:&error];
          if (keyStatus == AVKeyValueStatusFailed)
          {
              [self assetFailedToPrepareForPlayback:error];
              return;
          } else if (keyStatus == AVKeyValueStatusCancelled) {
              // TODO [AVAsset cancelLoading]
              error = [self createErrorWithCode:kEC_PlayerItemCancelled
                                    description:@"player item cancelled"
                                         reason:nil];
              [self assetFailedToPrepareForPlayback:error];
              return;
          }
      }
      
      /* Use the AVAsset playable property to detect whether the asset can be played. */
      if (!asset.playable)
      {
          NSError *assetCannotBePlayedError = [NSError errorWithDomain:@"AVMoviePlayer"
                                                                  code:0
                                                              userInfo:nil];
          
          [self assetFailedToPrepareForPlayback:assetCannotBePlayedError];
          return;
      }
      
      /* At this point we're ready to set up for playback of the asset. */
      
      /* Stop observing our prior AVPlayerItem, if we have one. */
      [_playerItemKVO safelyRemoveAllObservers];
      [[NSNotificationCenter defaultCenter] removeObserver:self
                                                      name:nil
                                                    object:_playerItem];
      
      /* Create a new instance of AVPlayerItem from the now successfully loaded AVAsset. */
      _playerItem = [AVPlayerItem playerItemWithAsset:asset];
      _playerItemKVO = [[IJKKVOController alloc] initWithTarget:_playerItem];
      [self registerApplicationObservers];
      /* Observe the player item "status" key to determine when it is ready to play. */
      [_playerItemKVO safelyAddObserver:self
                             forKeyPath:@"status"
                                options:NSKeyValueObservingOptionInitial | NSKeyValueObservingOptionNew
                                context:KVO_AVPlayerItem_state];
      
      [_playerItemKVO safelyAddObserver:self
                             forKeyPath:@"loadedTimeRanges"
                                options:NSKeyValueObservingOptionInitial | NSKeyValueObservingOptionNew
                                context:KVO_AVPlayerItem_loadedTimeRanges];
      
      [_playerItemKVO safelyAddObserver:self
                             forKeyPath:@"playbackLikelyToKeepUp"
                                options:NSKeyValueObservingOptionInitial | NSKeyValueObservingOptionNew
                                context:KVO_AVPlayerItem_playbackLikelyToKeepUp];
      
      [_playerItemKVO safelyAddObserver:self
                             forKeyPath:@"playbackBufferEmpty"
                                options:NSKeyValueObservingOptionInitial | NSKeyValueObservingOptionNew
                                context:KVO_AVPlayerItem_playbackBufferEmpty];
      
      [_playerItemKVO safelyAddObserver:self
                             forKeyPath:@"playbackBufferFull"
                                options:NSKeyValueObservingOptionInitial | NSKeyValueObservingOptionNew
                                context:KVO_AVPlayerItem_playbackBufferFull];
      
      [[NSNotificationCenter defaultCenter] addObserver:self
                                               selector:@selector(playerItemDidReachEnd:)
                                                   name:AVPlayerItemDidPlayToEndTimeNotification
                                                 object:_playerItem];
      
      [[NSNotificationCenter defaultCenter] addObserver:self
                                               selector:@selector(playerItemFailedToPlayToEndTime:)
                                                   name:AVPlayerItemFailedToPlayToEndTimeNotification
                                                 object:_playerItem];
      
      _isCompleted = NO;
      
      /* Create new player, if we don't already have one. */
      if (!_player)
      {
          /* Get a new AVPlayer initialized to play the specified player item. */
          _player = [AVPlayer playerWithPlayerItem:_playerItem];
          _playerKVO = [[IJKKVOController alloc] initWithTarget:_player];
          
          /* Observe the AVPlayer "currentItem" property to find out when any
           AVPlayer replaceCurrentItemWithPlayerItem: replacement will/did
           occur.*/
          [_playerKVO safelyAddObserver:self
                             forKeyPath:@"currentItem"
                                options:NSKeyValueObservingOptionInitial | NSKeyValueObservingOptionNew
                                context:KVO_AVPlayer_currentItem];
          
          /* Observe the AVPlayer "rate" property to update the scrubber control. */
          [_playerKVO safelyAddObserver:self
                             forKeyPath:@"rate"
                                options:NSKeyValueObservingOptionInitial | NSKeyValueObservingOptionNew
                                context:KVO_AVPlayer_rate];
          
          [_playerKVO safelyAddObserver:self
                             forKeyPath:@"airPlayVideoActive"
                                options:NSKeyValueObservingOptionInitial | NSKeyValueObservingOptionNew
                                context:KVO_AVPlayer_airplay];
      }
      
      /* Make our new AVPlayerItem the AVPlayer's current item. */
      if (_player.currentItem != _playerItem)
      {
          /* Replace the player item with a new player item. The item replacement occurs
           asynchronously; observe the currentItem property to find out when the
           replacement will/did occur
           
           If needed, configure player item here (example: adding outputs, setting text style rules,
           selecting media options) before associating it with a player
           */
          [_player replaceCurrentItemWithPlayerItem:_playerItem];
          
          // TODO: notify state change
      }
      
      // TODO: set time to 0;
  }
  ```

监听相应的状态变化

```objective-c
// 监听播放状态的变化
- (void)didPlaybackStateChange
{
    if (_playbackState != self.playbackState) {
        _playbackState = self.playbackState;
        [[NSNotificationCenter defaultCenter]
         postNotificationName:IJKMPMoviePlayerPlaybackStateDidChangeNotification
         object:self];
    }
    
}

// 监听加载状态的变化
- (void)didLoadStateChange
{
    // NOTE: do not force play after stall,
    // which may cause AVPlayer get into wrong state
    //
    // Rely on AVPlayer's auto resume.
    
    [[NSNotificationCenter defaultCenter]
     postNotificationName:IJKMPMoviePlayerLoadStateDidChangeNotification
     object:self];
}
```

## IJKFFMoviePlayerController

基于FFmpeg的播放器

### IJKFFOptions

- 帧的概念：

  每一帧代表一幅静止的图像，而在实际压缩中，会采用各种算法减少数据的容量，常见的为IPB

  ​	- I帧表示关键帧，属于帧内压缩，和AVI的压缩是一样的，P是向前搜索的意思，B是双向搜索，他们都是基于I帧来压缩数据

  ​	- I帧表示关键帧，可以理解为这一帧的完整画面数据；解码时只需要本帧数据就可以完成（因为包含完整画面，以前的播放软件暂	- 停的时候的图片是模糊的，这就是非关键帧，现在的播放软件暂停的时候显示的是关键帧，是一张清晰的图片）。

   	- P帧表示的是这一帧跟之前的一个关键帧（或P帧）的差别，解码时需要用之前缓存的画面叠加上本帧定义的差别，生成最终画面。（也就是差别帧，P帧没有完整画面数据，只有与前一帧的画面差别的数据）

  ​	- B帧是双向差别帧，也就是B帧记录的是本帧与前后帧的差别（具体比较复杂，有4种情况），换言之，要解码B帧，不仅要取得之前的缓存画面，还要解码之后的画面，通过前后画面的与本帧数据的叠加取得最终的画面。B帧压缩率高，但是解码时CPU会比较累~。

- 参数：所有的参数可以在：（ffp_context_options）找到

  ```objective-c
  + (IJKFFOptions *)optionsByDefault
  {
      IJKFFOptions *options = [[IJKFFOptions alloc] init];
  
      [options setPlayerOptionIntValue:30     forKey:@"max-fps"];
      [options setPlayerOptionIntValue:0      forKey:@"framedrop"];
      [options setPlayerOptionIntValue:3      forKey:@"video-pictq-size"];
      [options setPlayerOptionIntValue:0      forKey:@"videotoolbox"];
      [options setPlayerOptionIntValue:960    forKey:@"videotoolbox-max-frame-width"];
  
      [options setFormatOptionIntValue:0                  forKey:@"auto_convert"];
      [options setFormatOptionIntValue:1                  forKey:@"reconnect"];
      [options setFormatOptionIntValue:30 * 1000 * 1000   forKey:@"timeout"];
      [options setFormatOptionValue:@"ijkplayer"          forKey:@"user-agent"];
  
      options.showHudView   = NO;
  
      return options;
  }
  
  player-opts : start-on-prepared            = 1						// 开始准备
  player-opts : overlay-format               = fcc-i420
  player-opts : max-fps                      = 60						// 最大fps
  player-opts : framedrop                    = 0						// 跳帧开关：当cpu过慢时进行帧降低处理
  player-opts : videotoolbox-max-frame-width = 960					// 指定最大宽度
  player-opts : videotoolbox                 = 1						// 解码模式 0:软解、1:硬解码
  player-opts : video-pictq-size             = 3
  format-opts : ijkinject-opaque             = 140449007406288
  format-opts : user-agent                   = ijkplayer
  format-opts : auto_convert                 = 0						// 自动转屏开关
  format-opts : timeout                      = 30000000			// 超时时间
  format-opts : reconnect                    = 1						// 重连次数
  format-opts : safe                         = 0
  codec-opts  : skip_frame                   = 0
  codec-opts  : skip_loop_filter             = 0
  ```

  - ```objective-c
    //1为开启硬件解码，用特定方法把数字编码还原成它所代表的内容或将电脉冲信号转换成它所代表的信息、数据等;
    //硬件解码其实就是用GPU的专门模块编码来解（建议使用硬解码）。
    //0为软件解码，更稳定，是cpu进行解码(如果使用软解码可能导致cpu占用率高)
    [options setPlayerOptionIntValue:1 forKey:@"videotoolbox"];
    ```

  - ```objective-c
    // 设置音量大小，256为标准音量。（要设置成两倍音量时则输入512，依此类推）
    [options setPlayerOptionIntValue:512 forKey:@"vol"];
    ```

  - ```objective-c
    // 最大fps
    [options setPlayerOptionIntValue:30 forKey:@"max-fps"];
    
    // 跳帧开关，如果cpu解码能力不足，可以设置成5，否则
    // 会引起音视频不同步，也可以通过设置它来跳帧达到倍速播放
    [options setPlayerOptionIntValue:0 forKey:@"framedrop"];
    
    // 指定最大宽度
    [options setPlayerOptionIntValue:960 forKey:@"videotoolbox-max-frame-width"];
    
    // 自动转屏开关
    [options setFormatOptionIntValue:0 forKey:@"auto_convert"];
    
    // 重连次数
    [options setFormatOptionIntValue:1 forKey:@"reconnect"];
    
    // 超时时间，timeout参数只对http设置有效，若果你用rtmp设置timeout，ijkplayer内部会忽略timeout参数。rtmp的timeout参数含义和http的不一样。
    [options setFormatOptionIntValue:30 * 1000 * 1000 forKey:@"timeout"];
    ```

  - ```objective-c
    // 帧速率(fps) （可以改，确认非标准桢率会导致音画不同步，所以只能设定为15或者29.97）帧速率越大,画质越好,但太大了,有些机器版本不支持,反而有些卡。
    [options setPlayerOptionIntValue:29.97 forKey:@"r"];
    //如果是rtsp协议，可以优先用tcp(默认是用udp)
    [options setFormatOptionValue:@"tcp" forKey:@"rtsp_transport"];
    //播放前的探测Size，默认是1M, 改小一点会出画面更快
    [options setFormatOptionIntValue:1024 * 16 forKey:@"probesize"];
    //播放前的探测时间
    [options setFormatOptionIntValue:50000 forKey:@"analyzeduration"];
    
    [options setCodecOptionIntValue:IJK_AVDISCARD_DEFAULT forKey:@"skip_loop_filter"];
    [options setCodecOptionIntValue:IJK_AVDISCARD_DEFAULT forKey:@"skip_frame"];
    if (_isLive) {
           	// 直播参数
           [options setPlayerOptionIntValue:3000 forKey:@"max_cached_duration"];   
      			// 最大缓存大小是3秒，可以依据自己的需求修改
    				//设置无极限的播放器buffer，这个选项常见于实时流媒体播放场景
           [options setPlayerOptionIntValue:1 forKey:@"infbuf"];  // 无限读
          	//播放器缓冲可以避免因为丢帧引入花屏的，因为丢帧都是丢到I帧之前的P/B帧为止。我之前也写过一个类似的，思路都是一样，但这个代码更精简。
         	 // A：如果你想要实时性，可以去掉缓冲区，一句代码：
         	 // B: 如果你这样试过，发现你的项目中播放频繁卡顿，
         	 // 你想留1-2秒缓冲区，让数据更平缓一些，
           // 那你可以选择保留缓冲区，不设置上面那个就行。
            [options setPlayerOptionIntValue:0 forKey:@"packet-buffering"];  //  关闭播放器缓冲
    } else {
            // 如果不判断是否是关键帧会导致视频画面花屏。但是这样会导致全部清空的可能也会出现花屏
            // 所以这里推流端设置好 GOP（画面组，一个GOP就是一组连续的画面。MPEG编码将画面（即帧）分为I、P、B三种） 的大小，如果 max_cached_duration > 2 * GOP，可以尽可能规避全部清空
            // 也可以在调用control_queue_duration之前判断新进来的视频pkt是否是关键帧，这样即使全部清空了也不会花屏。
            [options setPlayerOptionIntValue:0 forKey:@"max_cached_duration"];
            [options setPlayerOptionIntValue:0 forKey:@"infbuf"];
    				// 播放时开启，推流时关闭
            [options setPlayerOptionIntValue:1 forKey:@"packet-buffering"];
    }
    ```

  - ```objective-c
    //  'skip_loop_filter' 的 'skip_frame'编码
    typedef enum IJKAVDiscard {
        /* We leave some space between them for extensions (drop some
         * keyframes for intra-only or drop just some bidir frames). */
        IJK_AVDISCARD_NONE    =-16, ///< 什么也不丢弃
        IJK_AVDISCARD_DEFAULT =  0, ///< 丢弃无用的数据包，像0大小的数据包在avi
        IJK_AVDISCARD_NONREF  =  8, ///< 抛弃非参考帧（P帧）
        IJK_AVDISCARD_BIDIR   = 16, ///< 抛弃所有的双向帧
        IJK_AVDISCARD_NONKEY  = 32, ///< 抛弃除关键帧以外的帧，比如B，P帧
        IJK_AVDISCARD_ALL     = 48, ///< 抛弃所有的帧
    } IJKAVDiscard;
    ```

    - skip_loop_filter和skip_frame的对象要过滤哪些帧类型。
    - skip_loop_filter这个是解码的一个参数，叫环路滤波，环路滤波主要用于滤除方块效应（块内模糊， 图像信号的低频区是用来反应一个图像的细节程度的，要是去掉低频部分，只用高频来描述图像的轮廓肯定很明显的出现方块效应 。）decode_slice()在解码完一行宏块之后，会调用loop_filter()函数完成环路滤波功能。
       设置成48和0，图像清晰度对比，0比48清楚，理解起来就是，
       0是开启了环路滤波，过滤的是大部分，
       而48基本没启用环路滤波，所以清晰度更低，但是解码性能开销小
    - skip_loop_filter（环路滤波）简言之：
      - 环路滤波器可以保证不同水平的图像质量
      - 环路滤波器更能增加视频流的主客观质量，同时降低解码器的复杂度

  ### playerController

  - 初始化传入地址， IJKFFOptions，就可以使用了，主要的设置都在IJKFFOptions中。

  - 设置缓存大小和是否自动播放

  - ```objective-c
    IJKMPMediaPlaybackIsPreparedToPlayDidChangeNotification
    // 收到该通知后，说明视频准备好可以开始播放了，这个时候可以取到当前视频的总时长，当前播放时间信息。
    // 相比于IJKAVMoviePlayerController和IJKMPMoviePlayerController，多了以下的通知：
    IJKMPMoviePlayerDidSeekCompleteNotification;
    IJKMPMoviePlayerDidSeekCompleteTargetKey;
    IJKMPMoviePlayerDidSeekCompleteErrorKey;
    IJKMPMoviePlayerDidAccurateSeekCompleteCurPos;
    IJKMPMoviePlayerAccurateSeekCompleteNotification;
    IJKMPMoviePlayerSeekAudioStartNotification;
    IJKMPMoviePlayerSeekVideoStartNotification
    // 可以根据具体的需求来查看相关的通知的作用。
    ```

## 状态转移

接入视频播放器的过程就是状态转移的过程，从下载资源、准备播放、开始播放、暂停、快进快退、播放结束、播放出错等等，根据不同的状态展示不同的样式。

IJKMPMoviePlayerController的状态相对较少，这里我们不做讨论，接下来分别通过阅读IJKAVMoviePlayerController和IJKFFMoviePlayerController的源码，总结一下他们不同状态之间的转换

在此之前，需要先了解两个state，一个reason

```objective-c
typedef NS_OPTIONS(NSUInteger, IJKMPMovieLoadState) {	// 加载状态
    IJKMPMovieLoadStateUnknown        = 0,			// 未知状态，初始状态
    IJKMPMovieLoadStatePlayable       = 1 << 0, // 可以播放状态
    IJKMPMovieLoadStatePlaythroughOK  = 1 << 1, // 如果设置了自动播放，会直接进入该状态
    IJKMPMovieLoadStateStalled        = 1 << 2, // 播放器暂停
};

typedef NS_ENUM(NSInteger, IJKMPMoviePlaybackState) {	// 播放状态
    IJKMPMoviePlaybackStateStopped,							// 播放结束
    IJKMPMoviePlaybackStatePlaying,							// 播放中
    IJKMPMoviePlaybackStatePaused,							// 暂停
    IJKMPMoviePlaybackStateInterrupted,					// 中断 （目前没有播放器使用该状态）
    IJKMPMoviePlaybackStateSeekingForward,			// 快进
    IJKMPMoviePlaybackStateSeekingBackward			// 快退 （没有使用该状态，不管前进还是后退都会返回IJKMPMoviePlaybackStateSeekingForward）
};

typedef NS_ENUM(NSInteger, IJKMPMovieFinishReason) {	// 播放结束的原因
    IJKMPMovieFinishReasonPlaybackEnded,				// 结束
    IJKMPMovieFinishReasonPlaybackError,				// 出错
    IJKMPMovieFinishReasonUserExited						// 出现用户行为的退出 （没有使用该状态）
};

```

相关通知：

```objective-c
IJKMPMediaPlaybackIsPreparedToPlayDidChangeNotification; // 播放状态的改变 代替MPMoviePlayerContentPreloadDidFinishNotification

 IJKMPMoviePlayerScalingModeDidChangeNotification; // 缩放比例的改变

 IJKMPMoviePlayerPlaybackDidFinishNotification;	// 视频播放结束的通知
 IJKMPMoviePlayerPlaybackDidFinishReasonUserInfoKey; // NSNumber (IJKMPMovieFinishReason)
 当电影播放结束或用户退出播放时调用。

 IJKMPMoviePlayerPlaybackStateDidChangeNotification; // 用户改变播放状态改变时调用
 IJKMPMoviePlayerLoadStateDidChangeNotification; // 当网络加载状态发生变化时。
 IJKMPMoviePlayerIsAirPlayVideoActiveDidChangeNotification; // 当视频通过 AirPlay 开始播放视频或结束时调用

 Movie Property Notifications
 属性相关的同时声明
 IJKMPMovieNaturalSizeAvailableNotification; // 在执行 prepareToPlay 时开始异步确定影片属性，当相关属性变为有效可用时调用该通知
 IJKMPMoviePlayerVideoDecoderOpenNotification; // 视频 编译器打开通知
 IJKMPMoviePlayerFirstVideoFrameRenderedNotification; // 视频 视频第一帧时通知
 IJKMPMoviePlayerFirstAudioFrameRenderedNotification; // 视频 音频第一段时通知
```

### IJKAVMoviePlayerController

- 上面说过，IJKAVMoviePlayerController初始化是会先初始化AVAsset，在资源加载成功后，会广播IJKMPMediaPlaybackIsPreparedToPlayDidChangeNotification，该广播只会在调用这一次
- 在用户进行操作的过程中，会不断的广播IJKMPMoviePlayerPlaybackStateDidChangeNotification 和 IJKMPMoviePlayerLoadStateDidChangeNotification，用户可以通过取当前的loadState和playbackState，来进行播放到缓冲，缓冲到播放，暂停到缓冲，缓冲到播放的操作，具体的状态转移需要在代码实际的实现过程中进行处理。
- 播放结束后，会调用IJKMPMoviePlayerPlaybackDidFinishNotification，返回IJKMPMovieFinishReasonPlaybackEnded。
- 在上述所有的过程中，如果出现错误，或者播放失败，都会调用IJKMPMoviePlayerPlaybackDidFinishNotification，返回IJKMPMovieFinishReasonPlaybackError。
- 播放结束后，再次调用play方法，可以自动从头开始播放
- 通过设置currentPlaybackTime可以达到快进快退的效果
- 通过playbackRate可以设置播放速度，倍速可以设置为1.2， 2等等
- 通过playbackVolume可以设置播放音量的大小

### IJKFFMoviePlayerController

状态转移基本与IJKAVMoviePlayerController相同，上文提到，相比于IJKAVMoviePlayerController和IJKMPMoviePlayerController，IJKFFMoviePlayerController有更多的通知，如果只是想用IJKFFMoviePlayerController实现播放器的简单功能，或者是从原生播放器切换到IJKFFMoviePlayerController，IJKAVMoviePlayerController里面的状态转移方式在IJKFFMoviePlayerController里已经够用，但是如果需要更多的定制化需求，可以进一步研究一下IJKFFMoviePlayerController的额外通知的广播时机和调用时机，可以针对播放器做出更多的精细化操作

### 总结

- 因为本次个人需求为IJKPlayer和原生播放器之间的切换，简单的状态转移已经够用，之后有更多定制化的播放器需求，会进一步研究IJKFFMoviePlayerController。

- 如果对FFmpeg更感兴趣可以阅读一下相关源码，这样有利于理解IJKFFMoviePlayerController的整个资源加载、播放的流程

## 参考文档：

- [IJKPlayer](https://github.com/bilibili/ijkplayer)

- [ijkplayer 参数说明文档](https://www.jianshu.com/p/80c56f47a870)

- [iOS集成IJKPlayer播放器](https://www.jianshu.com/p/818a7ac2639d)

- [FFmpeg](http://ffmpeg.org/)

  