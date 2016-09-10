### ffmpeg使用tee实现单次编码多路输出


##### tee简介

首先贴一下官方手册的链接：https://ffmpeg.org/ffmpeg-formats.html#tee

tee muxer可以将相同的数据写到多个文件或者其它的muxer。例如：它可以同时将一路视频流输出到网络和本地磁盘。与ffmpeg的默认多路输出不同的是，用tee只编码一次。

![tee框图](https://trac.ffmpeg.org/raw-attachment/wiki/Creating%20multiple%20outputs/creating_multiple_outputs1.png)

##### 语法

和普通的ffmpeg命令行相比，使用tee主要是两点区别：

1.主干的'-f'指定'tee'; 

2.输出路径是由'|'分隔的各个路径集合。


```shell
#例如：
ffmpeg -i input.file -acodec aac -vcodec h264 -f tee -map 0:v -map 0:a "tee1.mp4|tee2.flv"
```

具体的，tee还支持一些参数：
```
f
直接指定封装格式。有的时候靠ffmpeg根据输出路径猜封装格式是不牢靠的，直接指定格式，简单暴力。
    
bsfs[/spec]
设置比特率过滤器。

select
选择指定的流输出，默认是使用全部流（主干）。


```

##### 实例
```shell
# 1. 单路输入，输出一路mp4本地，一路TS over UDP (其中TS over UDP需要指定格式)
ffmpeg -re -i Meerkats.mp4 -acodec aac -vcodec h264 -f tee -map 0:v -map 0:a "tee1.mp4|[f=mpegts]udp://10.33.2.27:9999"

# 2. 使用ffmpeg进行编码，实现单路输入，四路输出（一路rtmp,一路ts，一路mp4,一路aac）。
ffmpeg -re -i Meerkats.mp4 -acodec aac -vcodec h264 -flags +global_header -strict experimental -f tee -map 0:v -map 0:a "[f=flv]rtmp://10.33.1.48/live1/tee1|[bsfs/v=dump_extra]out.ts|[movflags=+faststart]out.mp4|[select=a]out.aac"



```
