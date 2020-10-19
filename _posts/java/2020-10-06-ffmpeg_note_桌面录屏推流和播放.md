# ffmpeg 简记 - 桌面录屏推流和播放

简单用 gdigrab 转发当前屏幕到 udp 上

```perl
ffmpeg -f gdigrab -framerate 10 -i desktop -f mpegts udp://127.0.0.1:9090
```

参数解释：

* `-f` 指定 format; gdigrab 是 win32 提供的图形设备接口, windows 平台下还能使用 dshow
* `framerate` 帧率
* `-i` input filename, 指定的录制对象, gdi 支持 desktop (当前桌面) 和 title (指定某个窗口名字)

用 ffplay 来播放

```perl
ffplay udp://127.0.0.1:9090
```

划定一个区域的录屏

```perl
ffmpeg -f gdigrab -framerate ntsc -offset_x 10 -offset_y 20 -video_size 640x480 \
-show_region 1 -i desktop [output]
```


* see [stackoverflow 的windows 录屏问题讨论](https://stackoverflow.com/questions/6766333/capture-windows-screen-with-ffmpeg)


__官方文档__: 

* [ffmpeg 推流](https://trac.ffmpeg.org/wiki/StreamingGuide)
* [ffmpeg 录屏](https://trac.ffmpeg.org/wiki/Capture/Desktop)