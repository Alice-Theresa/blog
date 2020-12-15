---
title: Apple ProRAW，手机摄影创作者的狂欢
date: 2021-01-26 
tags: [图像处理]
---

2020年10月，苹果一口气发布了四款手机，iPhone12 mini、iPhone 12、iPhone 12 Pro、iPhone 12 Pro Max，其中后两款在iOS14.3中支持Apple ProRAW拍摄。

### 什么是RAW

现在的图像传感器主要工艺为CMOS，RAW文件包含它所输出的原始数据以及相关的相机设置参数。
图像传感器本身只对光的亮度做出响应，并不能区分光的颜色，所以它所获取到的原片都是黑白的，为了让传感器能够响应颜色，需要在上面铺设一个彩色滤光片阵列，如比较常见的拜耳阵列。

<img src="/images/2021/appleProResRaw/filterArray.jpg">

它会在每一个像素点上面铺上一种颜色的滤镜，以响应特定颜色的光（只通过特定波长的光，过滤其他颜色，红色滤镜只通过红色的光），比较常见的阵列为RGGB，即在每2x2的大小的网格中有1个红色像素、1个蓝色像素和2个绿色像素，除此之外，也有RYYB（Y为Yellow，华为P30 Pro上使用了这种CMOS）、RCCC、RCCB等（C为Clear，即透传，拥有较好的光灵敏度，适合弱光场景）。

<img src="/images/2021/appleProResRaw/colorFilter.jpg">

每一个像素点的最终输出实际上是通过滤镜后光的亮度，传感器的原始图像仍然是黑白，输出后会为其标记该像素是什么颜色
由于每个像素的原始数据只有一种颜色，后续需要通过插值算法为其补上剩余的两种颜色

### RAW的好处

<img src="/images/2021/appleProResRaw/origin.jpg">

这是一张Sony A7M3所拍摄的图，jpeg直接输出，很明显，这张图拍烂了，背景过暗，人物过曝，我们需要对其调整

<img src="/images/2021/appleProResRaw/jpg_girl.jpg">

遗憾的是，即便调整过后，图像中的大量细节仍然丢失了

<img src="/images/2021/appleProResRaw/jpg_detail.jpg">

相机直接输出时使用的jpeg编码本身属于有损压缩，简单来说，该算法为了缩小体积，利用人眼对计算机色彩中的高频信息部分不敏感的特点，去除肉眼视觉不容易察觉到的部分，来保证最终的输出体积缩小但看上去差异不大

但幸运的是，拍摄的时候保留了其RAW格式的原图，A7M3拥有一块2400万像素的CMOS，可输出14bit深，存储时占16bit，传感器的像素数据就为 24_000_000 * 16 / 8 / 1024 / 1024，大约45.78MB，加上其他数据大概50MB左右

使用相同的参数调整
<img src="/images/2021/appleProResRaw/raw_girl.jpg">

可以看见在RAW上的调整，细节得到的不错的保留
<img src="/images/2021/appleProResRaw/raw_detail.jpg">

###3 价格很Pro

iPhone 12 Pro Max 512GB 官网售价高达11899，这个价格已经可以买一台不错的全画幅相机了，如果没有Pro一点的功能，真的对不起这高昂的价格
而Apple ProRAW也让那些Pro的创作者有更多的选择，当然了，你也别指望iPhone那么小的CMOS以及那么廉价的镜头能够让你拍出什么大片就是了
