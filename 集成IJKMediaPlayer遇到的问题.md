# 集成IJKMediaPlayer遇到的问题

1. 子工程集成问题

   集成IJKMediaplayer到项目上是需要通过子工程的方式实现(Github文档)，但是当我将子工程拖入到自己的项目时，只看到了`IJKMediaPlayer.xcodeproj`文件，没有看到整个工程。

   如下所示

   ![](https://ws1.sinaimg.cn/large/72531cb9gy1g5vxniboalj20f2042q5p.jpg)

   而正确的形式应该是和其官方demo一样

   ![](https://ws1.sinaimg.cn/large/72531cb9gy1g5vxomkwswj20ik0dq4a6.jpg)

   这样才对！

   而为什么会出现这个问题，是因为子工程同时在俩个项目里面打开了，如果想要子工程里面所有的文件都能显示，就只能打开一个包含子工程的项目！！！

2. 编译报错

   按照官方文档提示的添加依赖库，最终还是报错了，报错信息如下

   > ld: symbol(s) not found for architecture x86_64

   这是因为还少了一个依赖库，添加依赖库`libc++.tbd`即可解决这个问题

   完整的依赖库列表如下

   ![](https://ws1.sinaimg.cn/large/72531cb9gy1g5vxt1znowj21bs0mkdk6.jpg)

3. 替换支持`https`的`IJKMediaFrameworkWithSSL.framework`

   IJKMediaPlayer官方的demo默认是使用`IJKMediaFramewor.framework`.默认是不支持https的，所以如果需要支持https需要替换成`IJKMediaFrameworkWithSSL.framework`

   替换步骤如下

   > First
   > 1 ) ./init-ios-openssl.sh
   > 2 ) ./init-ios.sh
   >
   > then cd ios
   >
   > 3.1) ./compile-openssl.sh clean //if u did compiled before
   > 3.2) ./compile-ffmpeg.sh clean //if u did compiled before
   > 4.1) ./compile-openssl.sh all
   > 4.2) ./compile-ffmpeg.sh all
   >
   > Open IJKMediaDemo
   > Select subproject IJKMediaPlayer.xcodeproj
   > Select target IJKMediaFrameworkWithSSL
   >
   > // libcrypto.a and libssl.a这俩个文件会在./compile-openssl.sh all命令执行后出现在
   >
   > // /ios/build/universal/lib/目录下
   >
   > Add the libcrypto.a and libssl.a in 【Build Phases】-【Link Binary With Libraries】
   >
   > Select main project IJKMediaDemo
   > Open IJKVideoViewController.h and replace #import <IJKMediaFramework/IJKMediaFrameworkL.h> with #import <IJKMediaFrameworkWithSSL/IJKMediaFrameworkWithSSL.h>
   >
   > Allow ATS，search "IOS ATS" on search engine
   >
   > Find a video url using Https protocol.
   >
   > Build
   >
   > By the way, fix dyld: Library not loaded: crash
   > In the target's General tab, there is an Embedded Binaries field.
   > Add the IJKMediaFrameworkWithSSL.framework and crash is resolved.

   + 编译openssl

     编辑`compile-openssl.sh`

     删除`FF_ALL_ARCHS_IOS8_SDK="armv7 arm64 i386 x86_64"`中的`armv7`

     必须先编译openssl并且出现`libcrypto.a`和`libssl.a`这俩个文件后，再编译`ffmpeg`否则还是无法支持`https`。会报如下错误

     ```
     https protocol not found, recompile FFmpeg with openssl, gnutls or securetransport enabled.
     ```

     如果编译报如下错误

     `ld: library not found for -lcrypto`

     那么代表工程中没有添加`libcrypto.a`和`libssl.a`，需要我们手动去添加，文件位置在

     ios/build/universal/lib/目录下

   + 编译ffmpeg

     编辑`compile-ffmpeg.sh`

     ```
     // 删掉armv7
     FF_ALL_ARCHS_IOS8_SDK="armv7 arm64 i386 x86_64"
     =>FF_ALL_ARCHS_IOS8_SDK="arm64 i386 x86_64"
     ```

     否则会报如下编译错误

     ```
     AS	libavcodec/arm/aacpsdsp_neon.o
     ./libavutil/arm/asm.S:50:9: error: unknown directive
             .arch armv7-a
             ^
     make: *** [libavcodec/arm/aacpsdsp_neon.o] Error 1
     make: *** Waiting for unfinished jobs....
     ```

   + 编译错误

     + 默认编译的`IJKMediaFrameworkWithSSL.framework`是动态库，添加到自己工程时

       需要在`Embed`选项中选择`Embed & Sign`

     + 通过`Generic iOS Device`编译时，会报错`armv7/avconfig.h` file not found

       这是因为`Target`的`Build Settings`中设置的`Valid Architectures`包含了`armv7 armv7s`，删掉这两个就好

   + 将`IJKMediaFrameworkWithSSL.framework`变为静态库

     `IJKMediaFramework.framework`默认就是静态库了，但是`IJKMediaFrameworkWithSSL.framework`则是动态库，如果将其改为静态库

     需要在`Build Settings`中的`Mach-O Type`修改为`Static Library`

     **使用的时候需要在目标工程添加以下依赖**

     1. `libz.tbd`
     2. `libc++.tbd`

   + `IJKMediaFrameworkWithSSL.framework`的`Enable Bitcode`默认为NO

     而`IJKMediaFramework.framework`的的`Enable Bitcode`默认为YES

     注意这个区别，看我们自己的项目是不是需要`Enable Bitcode`，如果需要，则要修改`IJKMediaFrameworkWithSSL.framework`的`Enable Bitcode`

     **动态库BitCode的设置还得再摸索下，静态库使用如上设置是没问题的**

     动态库并开启BitCode会报如下错误

     ```
     ld: -read_only_relocs and -bitcode_bundle (Xcode setting ENABLE_BITCODE=YES) cannot be used together
     ```

   + 真机与模拟器`framework`合并

     合并前最好在设置`framework`的scheme为`Release`模式

     执行命令

     `lipo -create 真机路径 模拟器路径 -output 真机路径`

     验证结果

     `lipo -info 路径`

     如果输出了`x86_64 arm64`则代表成功

     参考博客[iOS 关于真机和模拟器framework合并](https://www.jianshu.com/p/840badb8a861)

   + 合并后的`IJKMediaFrameworkWithSSL.framework`动态库与静态库大小对比

     + 动态库 framework大小为16.7MB 生成的APP大小为18MB(未开启bitcode)
     + 静态库 framework大小为107.1MB 生成的APP大小为18MB (开启了bitcode)

   出处在这里`https://github.com/Bilibili/ijkplayer/issues/3945`按照如上步骤可以是可以成功编译的

4. 关于视频状态的监听

   视频的状态是通过通知监听的，需要注意以下几点

   1. 添加通知的时机

      ```
      - (void)viewWillAppear:(BOOL)animated
      {
          [super viewWillAppear:animated];
          
          [self installNotifications];
          [self.player prepareToPlay];
      }
      ```

      不知道这里为什么要在`viewWillAppear`这个方法里添加通知才有效。。。

   2. 4个通知方法,来自于官方demo中的`IJKMoviePlayerViewController`

      ```objective-c
      -(void)installMovieNotificationObservers
      {
      	[[NSNotificationCenter defaultCenter] addObserver:self
                                                   selector:@selector(loadStateDidChange:)
                                                       name:IJKMPMoviePlayerLoadStateDidChangeNotification
                                                     object:_player];
      
      	[[NSNotificationCenter defaultCenter] addObserver:self
                                                   selector:@selector(moviePlayBackDidFinish:)
                                                       name:IJKMPMoviePlayerPlaybackDidFinishNotification
                                                     object:_player];
      
      	[[NSNotificationCenter defaultCenter] addObserver:self
                                                   selector:@selector(mediaIsPreparedToPlayDidChange:)
                                                       name:IJKMPMediaPlaybackIsPreparedToPlayDidChangeNotification
                                                     object:_player];
      
      	[[NSNotificationCenter defaultCenter] addObserver:self
                                                   selector:@selector(moviePlayBackStateDidChange:)
                                                       name:IJKMPMoviePlayerPlaybackStateDidChangeNotification
                                                     object:_player];
      }
      ```

      对应实现

      ```objective-c
      - (void)loadStateDidChange:(NSNotification*)notification
      {
          //    MPMovieLoadStateUnknown        = 0,
          //    MPMovieLoadStatePlayable       = 1 << 0,
          //    MPMovieLoadStatePlaythroughOK  = 1 << 1, // Playback will be automatically started in this state when shouldAutoplay is YES
          //    MPMovieLoadStateStalled        = 1 << 2, // Playback will be automatically paused in this state, if started
      
          IJKMPMovieLoadState loadState = _player.loadState;
      
          if ((loadState & IJKMPMovieLoadStatePlaythroughOK) != 0) {
              NSLog(@"loadStateDidChange: IJKMPMovieLoadStatePlaythroughOK: %d\n", (int)loadState);
          } else if ((loadState & IJKMPMovieLoadStateStalled) != 0) {
              NSLog(@"loadStateDidChange: IJKMPMovieLoadStateStalled: %d\n", (int)loadState);
          } else {
              NSLog(@"loadStateDidChange: ???: %d\n", (int)loadState);
          }
      }
      
      - (void)moviePlayBackDidFinish:(NSNotification*)notification
      {
          //    MPMovieFinishReasonPlaybackEnded,
          //    MPMovieFinishReasonPlaybackError,
          //    MPMovieFinishReasonUserExited
          int reason = [[[notification userInfo] valueForKey:IJKMPMoviePlayerPlaybackDidFinishReasonUserInfoKey] intValue];
      
          switch (reason)
          {
              case IJKMPMovieFinishReasonPlaybackEnded:
                  NSLog(@"playbackStateDidChange: IJKMPMovieFinishReasonPlaybackEnded: %d\n", reason);
                  break;
      
              case IJKMPMovieFinishReasonUserExited:
                  NSLog(@"playbackStateDidChange: IJKMPMovieFinishReasonUserExited: %d\n", reason);
                  break;
      
              case IJKMPMovieFinishReasonPlaybackError:
                  NSLog(@"playbackStateDidChange: IJKMPMovieFinishReasonPlaybackError: %d\n", reason);
                  break;
      
              default:
                  NSLog(@"playbackPlayBackDidFinish: ???: %d\n", reason);
                  break;
          }
      }
      
      - (void)mediaIsPreparedToPlayDidChange:(NSNotification*)notification
      {
          NSLog(@"mediaIsPreparedToPlayDidChange\n");
      }
      
      - (void)moviePlayBackStateDidChange:(NSNotification*)notification
      {
          //    MPMoviePlaybackStateStopped,
          //    MPMoviePlaybackStatePlaying,
          //    MPMoviePlaybackStatePaused,
          //    MPMoviePlaybackStateInterrupted,
          //    MPMoviePlaybackStateSeekingForward,
          //    MPMoviePlaybackStateSeekingBackward
      
          switch (_player.playbackState)
          {
              case IJKMPMoviePlaybackStateStopped: {
                  NSLog(@"IJKMPMoviePlayBackStateDidChange %d: stoped", (int)_player.playbackState);
                  break;
              }
              case IJKMPMoviePlaybackStatePlaying: {
                  NSLog(@"IJKMPMoviePlayBackStateDidChange %d: playing", (int)_player.playbackState);
                  break;
              }
              case IJKMPMoviePlaybackStatePaused: {
                  NSLog(@"IJKMPMoviePlayBackStateDidChange %d: paused", (int)_player.playbackState);
                  break;
              }
              case IJKMPMoviePlaybackStateInterrupted: {
                  NSLog(@"IJKMPMoviePlayBackStateDidChange %d: interrupted", (int)_player.playbackState);
                  break;
              }
              case IJKMPMoviePlaybackStateSeekingForward:
              case IJKMPMoviePlaybackStateSeekingBackward: {
                  NSLog(@"IJKMPMoviePlayBackStateDidChange %d: seeking", (int)_player.playbackState);
                  break;
              }
              default: {
                  NSLog(@"IJKMPMoviePlayBackStateDidChange %d: unknown", (int)_player.playbackState);
                  break;
              }
          }
      }
      ```

      



