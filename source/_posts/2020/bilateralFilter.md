---
title: 利用双边滤波处理图片
date: 2020-05-23 
tags: [Metal, 图像处理]
---

双边滤波其实和均值滤波、高斯滤波没有多大的区别  

它只是在这些滤波器的基础上对边缘区域进行加权处理，从而减少模糊对边缘的影响
对于非边缘区域，我们给滤波器一个较大的权重，以提高其模糊效果，在边缘区域则给一个较小的权重来保留细节

双边滤波公式
{% katex %}
\Large I_{D}(i,j) = \frac{\sum_{k,l}{I(k,l)w(i,j,k,l)}}{\sum_{k,l}{w(i,j,k,l)}}
{% endkatex %}

其中
{% katex %}
\Large w(i,j,k,l) = \exp(-\frac{(i-k)^2+(j-l)^2)}{2\sigma_{d}^2}-\frac{\|f(i,j)-f(k,j)\|^2}{2\sigma_{r}^2})
{% endkatex %}

根据指数函数的性质可得
{% katex %}
\Large w(i,j,k,l) = \exp(-\frac{(i-k)^2+(j-l)^2)}{2\sigma_{d}^2})*\exp(-\frac{\|f(i,j)-f(k,j)\|^2}{2\sigma_{r}^2})
{% endkatex %}

这里我们不严格按照双边滤波的定义来实现，稍作化简，其中函数G代表着一种滤波函数
{% katex %}
\Large w(i,j,k,l) = \exp(-\frac{(i-k)^2+(j-l)^2)}{2\sigma_{d}^2})*G(i,j,k,l)
{% endkatex %}

下面是Metal的Shader，实现了双边滤波公式，为了方便计算，滤波函数采用了均值模糊
```
kernel void BilateralFilter(texture2d<float, access::read> inTexture [[texture(0)]],
                            texture2d<float, access::write> outTexture [[texture(1)]],
                            device float *input [[buffer(0)]],
                            uint2 gid [[thread_position_in_grid]])
{
    const short radius = input[0]; // 卷积核半径
    const float boxBlur = 1.0 / (2 * radius + 1) / (2 * radius + 1); // 均值模糊(函数G)
    const float sigma = 0.3; // sigma(σ)
    const float coe = -0.5 / pow(sigma, 2);
    
    float weightSum = 0;
    float3 colorSum = float3(0, 0, 0);
    const float4 colorAtCenter = inTexture.read(gid); // 卷积核中心点颜色(RGBA)
    
    // 对卷积核内的所有像素进行处理并累加
    for (short x = -radius; x <= radius; x++) {
        for (short y = -radius; y <= radius; y++) {
            ushort2 pixel = ushort2(gid.x + x, gid.y + y);
            float4 colorAtPixel = inTexture.read(pixel);
            float closeness = exp(pow(distance(colorAtPixel.xyz, colorAtCenter.xyz), 2) * coe);
            float weight = closeness * boxBlur;
            colorSum += colorAtPixel.rgb * weight;
            weightSum += weight;
        }
    }
    outTexture.write(float4((colorSum / weightSum), 1), gid);
}
```

另一种加权实现，速度更快，但实际效果略差
```
float closeness = 1 - distance(colorAtPixel.xyz, colorAtCenter.xyz) / length(float3(1,1,1));
```

下图为结果  
第一行为均值滤波处理
第二行为双边滤波处理  
从左至右分别为原图，以及卷积核半径等于5、9、13、17的图  

可以看到，第一行图片里，整张图都变得模糊起来，而在第二行的图片中，边缘细节得到了一定程度上的保留
因此，双边滤波可以应用在美颜领域上，配合机器学习，检测人的皮肤区域并对其进行处理，就可以达到磨皮祛斑的美颜效果

可以下载图片查看原图

<img src="/images/2020/bilateralFilter/Beauty.jpg">
