---
layout: post
title: StreamingKit保存在线音频流到本地
subtitle: 如何修改StreamingKit实现音频缓存功能
tags: [iOS]
comments: false
gh-repo: liyaozhong/StreamingKit
---

>StreamingKit是一个强大的IOS音频播放工具，链接在此。具体用法和功能不再赘述，网上资料很多。本文主要记录笔者在项目中遇到的问题－－保存在线音频流到本地。

>具体需求如下：在线播放音频流到时候，需要将播放完的音频流保存到本地，这样下次播放的时候就不需要从网络加载。

首先看下[StreamingKit源码](https://github.com/tumtumtum/StreamingKit/)，在STKAudioPlayer.m中可以找到``+(STKDataSource*) dataSourceFromURL:(NSURL*)url``方法。

```
+(STKDataSource*) dataSourceFromURL:(NSURL*)url
{
  STKDataSource* retval = nil;

  if ([url.scheme isEqualToString:@"file"])
  {
    retval = [[STKLocalFileDataSource alloc] initWithFilePath:url.path];
  }
  else if ([url.scheme caseInsensitiveCompare:@"http"] == NSOrderedSame || [url.scheme caseInsensitiveCompare:@"https"] == NSOrderedSame)
  {
    retval = [[STKAutoRecoveringHTTPDataSource alloc] initWithHTTPDataSource:[[STKHTTPDataSource alloc] initWithURL:url];
  }
  return retval;
}
```

可见在线音频会通过STKHTTPDataSource对象封装。查看``STKHTTPDataSource.m``源码，可以看到有一个``-(int) privateReadIntoBuffer:(UInt8*)buffer withSize:(int)size``方法。OK，既然能读就能写，就在这儿下手。

首先给STKHTTPDataSource添加两个变量
```
NSString * audioCacheFilePath;
AudioFileID audioFile;
```
修改下init方法：
```
-(instancetype) initWithURL:(NSURL*)url andCacheFilePath : (NSString *) filePath;

-(instancetype) initWithURL:(NSURL*)url httpRequestHeaders:(NSDictionary*)httpRequestHeaders andCacheFilePath : (NSString *) filePath;

-(instancetype) initWithURLProvider:(STKURLProvider)urlProvider andCacheFilePath : (NSString *) filePath;

-(instancetype) initWithAsyncURLProvider:(STKAsyncURLProvider)asyncUrlProvider andCacheFilePath : (NSString *) filePath;
```
我们在``-(instancetype) initWithAsyncURLProvider:(STKAsyncURLProvider)asyncUrlProviderIn andCacheFilePath : (NSString *) filePath``方法里，对audioFile就行初始化。
```
if(filePath){

  audioCacheFilePath = filePath;

  AudioStreamBasicDescription asbd = (AudioStreamBasicDescription)

  {

    .mSampleRate = 44100.00,

    .mFormatID = kAudioFormatMPEGLayer3,

    .mFramesPerPacket = 1,

    .mChannelsPerFrame = 2,

    .mBytesPerFrame = sizeof(SInt16) * 2 /*channelsPerFrame*/,

    .mBitsPerChannel = 8 * sizeof(SInt16),

    .mBytesPerPacket = (sizeof(SInt16) * 2)

  };

  OSStatus audioErr = noErr;

  audioErr = AudioFileCreateWithURL((__bridge CFURLRef)[NSURL fileURLWithPath:[filePath stringByAppendingString:@"_tmp"]], kAudioFileMP3Type, &asbd, kAudioFileFlags_EraseFile, &audioFile);
}
```
注意：这里我们的audioFile并不是指向filePath，而是一个临时文件。这是因为如果在写入文件的过程中出现网络问题，或者应用挂掉，那么我们的缓存文件就是一个不完整的音频文件，所以先将它保存在一个临时文件中。此处笔者保存文件格式是mp3，读者可以根据自己的需要自行调整文件格式，但是要保证AudioFileCreateWithURL方法和AudioFileCreateWithURL中的格式和参数相符，不然就会出现音频文件无法播放的问题。

然后修改``-(int) privateReadIntoBuffer:(UInt8*)buffer withSize:(int)size``方法：
```
-(int) privateReadIntoBuffer:(UInt8*)buffer withSize:(int)size
{
  if (size == 0){
    return 0;
  }

  if (prefixBytes != nil){

    int count = MIN(size, (int)prefixBytes.length - prefixBytesRead);

    [prefixBytes getBytes:buffer length:count];

    prefixBytesRead += count;

    if (prefixBytesRead >= prefixBytes.length){

      prefixBytes = nil;

    }

    return count;

  }

  int read = [super readIntoBuffer:buffer withSize:size];

  if(audioFile && audioCacheFilePath && read >= 0){

    UInt32 bytesToWrite = read;

    AudioFileWriteBytes(audioFile, false, relativePosition,     &bytesToWrite, buffer);

  }

  if (read < 0){

    return read;

  }

  relativePosition += read;

  return read;

}
```
最后，我们需要在eof的时候将临时文件改成目标文件，添加方法：

在``STKAutoRecoveringHTTPDataSource.m``的``-(void) dataSourceEof:(STKDataSource*)dataSource``方法添加如下逻辑：
```
if([dataSource isKindOfClass:[STKHTTPDataSource class]]){

  [(STKHTTPDataSource*)dataSource closeCacheFile];

}
```
当然，你还的给``+(STKDataSource*) dataSourceFromURL:(NSURL*)url``方法添加参数，以使用我们新的init方法。

第一次使用StreamingKit，如有纰漏还望见。欢迎讨论！

>源码&安装
github: git@github.com:liyaozhong/StreamingKit.git
cocoapods: pod 'AdvancedStreamingKit' 