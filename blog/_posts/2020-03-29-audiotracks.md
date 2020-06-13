---
title: WebRTC中合并多路音频源
# image: /assets/img/blog/...
description: >
  chrome中获取系统和麦克风声音并将他们合并到同一个媒体源中。
# cofigure what you want to add in the end of the post, [about, newsletter, related, random, license]
addons: [license]
# the tag of post.
tags: [webrtc]
---

最近在做webRTC交互时，遇到在屏幕共享中添加声音的需求——需要同时添加系统声音以及麦克风的声音。本文记录一下是如何解决这个问题的。<br>

# 获取chrome中的媒体

获取摄像头和麦克风：
```js
navigator.mediaDevices.getUserMedia({
    video: true,
    audio: true
}).then(function(stream){
    // ...
})
```
看一下结果：<br>
![]({{site.data.strings.blog_url}}webcam.png)


获取抓取屏幕(2018年底开始支持):
```js
// chrome中，这里的声音指的是系统声音
navigator.mediaDevices.getDisplayMedia({
    video: true,
    audio: true
}).then(function(stream){
    // ...
})
```
以下是浏览器获取的结果：<br>
![]({{site.data.strings.blog_url}}screen.png)


# 将系统声音和麦克风声音添加到视频中并通过WebRTC传输

一开始想的是想很暴力的直接添加audioTrack来解决：
```javascript
screen.addTrack(webcam.getAudioTracks()[0]);
```
添加很成功，然后浏览器获取一个包含两个audiotrack和一个videoTrack的MediaStream对象，浏览器也可以播放：
```javascript
  var video = document.querySelector('video');
  // 旧的浏览器可能没有srcObject
  if ("srcObject" in video) {
    video.srcObject = stream;
  } else {
    // 防止在新的浏览器里使用它，应为它已经不再支持了
    video.src = window.URL.createObjectURL(stream);
  }
  video.onloadedmetadata = function(e) {
    video.play();
  };
```
但是通过WebRTC传输时就报错了。<br>

然后就想到能不能将两个audioTrack合并，看了下浏览器的API，于是就有了第二种方案：
```js
const mergeTracks = (baseStrem, extraStream) => {
    if (!baseStrem.getAudioTracks().length){
        baseStrem.addTrack(extraStream.getAudioTracks()[0])
        return baseStrem;
    }
    var context = new AudioContext();
    var baseSource = context.createMediaStreamSource(baseStrem);
    var extraSource = context.createMediaStreamSource(extraStream);
    var dest = context.createMediaStreamDestination();

    var baseGain = context.createGain();
    var extraGain = context.createGain();
    baseGain.gain.value = 0.8;
    extraGain.gain.value = 0.8;

    baseSource.connect(baseGain).connect(dest);
    extraSource.connect(extraGain).connect(dest);

    return new MediaStream([baseStrem.getVideoTracks()[0], dest.stream.getAudioTracks()[0]]);
}
```
主要就是[`createMediaStreamSource()`](https://developer.mozilla.org/zh-CN/docs/Web/API/AudioContext/createMediaStreamSource)和[`createMediaStreamDestination()`](https://developer.mozilla.org/zh-CN/docs/Web/API/AudioContext/createMediaStreamDestination)两个函数，具体介绍可以看啊可能官网，简单的看一下合流的结果：<br>
![]({{site.data.strings.blog_url}}merge.png)

然后就可以通过WebRTC传输了。<br>
[a simple demo](https://github.com/Soo-Q6/mergeAudioTracks)