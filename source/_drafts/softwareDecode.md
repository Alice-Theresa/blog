---
title: 软解码，末日已到
date: 2020-11-21 
tags: [FFmpeg, Metal, 音视频, 图像处理]
---

前阵子播放一个视频时发现是这样子的

<img src="/images/2021/softwareDecode/error_decode.png">

在电脑播放器里播放，是正常的

<img src="/images/2021/softwareDecode/normal_decode.png">

电脑采用硬解码模式播放，手机播放器采用FFmpeg软解码（CPU），将其切换至硬解码（SoC内置模块），播放正常。
在软解码模式下播放别的视频，播放正常
仔细对比两个视频发现，播放异常的视频是采用10bit压制的

问题找到了

该视频主要参数为1080p yuv420 10bit，FFmpeg解码后的帧以AVFrame的形式存在，其中data数组中的头三位分别存放着yuv数据，另外可以看到10bit视频linesize为分辨率宽度的2倍

<img src="/images/2021/softwareDecode/frame_info.jpg">

播放器使用了Metal作为渲染层，其读取的纹理格式为MTLPixelFormatR8Unorm，即每个像素占用1个Byte，R8代表单像素8bit，U为unsigned，norm为归一化，即用[0.0, 1.0]代表着0-255色阶

这个10bit视频软解码后的帧格式为AV_PIX_FMT_YUV420P10LE，每个像素占用2个Byte，直接从2个Byte的数据读取一个Byte来渲染当然会有问题

使用硬解码时，获得的帧已经经过转换，格式为AV_PIX_FMT_NV12，每个像素占用1个Byte，虽然使用的shader不同，但渲染层能够直接渲染成功

渲染层不想改动，想要正常播放只能改动数据源，以下代码实现了10bit转8bit

```
for (int i = 0; i < 8; i++) {
    int width = frame->linesize[i];
    int height = frame->height;
    uint8_t *src = frame->data[i];
    uint8_t *dst = frame->data[i];
    uint8_t *p = frame->data[i];

    for (int h = 0; h < height; h++) {
        for (int w = 0; w < width; w+=2) {
            uint16_t value = *(uint16_t *)p;
            memset(dst, value>>2, 1);
            dst += 1;
            p += 2;
        }
    }
    self->_data[i] = src;
}
```

但这带来了新的问题，软解码本身消耗大量CPU算力，解码后还要对其图像进行二次处理，进一步加大CPU的压力，对于高分辨率或者高帧数的视频来说将无法正常播放

直接改动数据源是不明智的选择，于是着手渲染层，将读取的纹理格式变为MTLPixelFormatR16Unorm，即每像素2byte，同时纹理的bytesPerRow设定为原来像素的2倍

```objc
size_t width = frame.width;
size_t height = frame.height;
MTLRegion yRegion = { { 0, 0, 0 }, { width, height, 1 } };
MTLRegion uvRegion = { { 0, 0, 0 }, { width / 2, height / 2, 1 } };

MTLTextureDescriptor *yDesc = [MTLTextureDescriptor texture2DDescriptorWithPixelFormat:MTLPixelFormatR16Unorm
                                                                                  width:width
                                                                                height:height
                                                                              mipmapped:YES];
MTLTextureDescriptor *uvDesc = [MTLTextureDescriptor texture2DDescriptorWithPixelFormat:MTLPixelFormatR16Unorm
                                                                                  width:width / 2
                                                                                  height:height / 2
                                                                              mipmapped:YES];
id<MTLTexture> yTexture = [self.device newTextureWithDescriptor:yDesc];
id<MTLTexture> uTexture = [self.device newTextureWithDescriptor:uvDesc];
id<MTLTexture> vTexture = [self.device newTextureWithDescriptor:uvDesc];
[yTexture replaceRegion:yRegion mipmapLevel:0 withBytes:frame.data[0] bytesPerRow:width * 2];
[uTexture replaceRegion:uvRegion mipmapLevel:0 withBytes:frame.data[1] bytesPerRow:width];
[vTexture replaceRegion:uvRegion mipmapLevel:0 withBytes:frame.data[2] bytesPerRow:width];
```

fragment shader也需要做相对应的改动，10bit转成16bit，读取的yuv数据分别乘上一个比例系数 2^16/2^10（即65535/1024）

这里色深没有像转8bit那样有所损失，画质完美无损

```c++
half coe = 65535.0 / 1024.0;
half y, u, v;

y = half4(ytexture.sample(s, mappingVertex.textureCoordinate)).r * coe;
u = half4(utexture.sample(s, mappingVertex.textureCoordinate)).r * coe - 0.5;
v = half4(vtexture.sample(s, mappingVertex.textureCoordinate)).r * coe - 0.5;
```

重新运行，现在能够正常播放了

2020年的今天，新上市的手机都能够支持H264、H265视频的硬解，且速度快、性能高，非常省电，已经是播放视频的首要选择，软解码充其量作为后备方案，但遗憾的是，这一后备的选择可能永远都用不上了
