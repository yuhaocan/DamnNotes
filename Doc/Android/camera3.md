# Android相机开发之图像格式（三）
第三篇来了解相机输出的数据格式。除了我们平常常见的JPEG、PNG、RAW等图片格式以及视频格式MP4、AVI等。对于Android设备的输出格式可能会比较陌生，例如YUV_420_888、NV21、NV12等。在旧版API中所有设备都支持NV21和YV12两种格式、新API设备支持YUV_420_888。
>All devices will support NV21 and YV12 formats for the old camera API (since API 12), and for camera2, all devices will support YUV_420_888
### Camera1
查看源码会发现Camera1的setPreviewFormat设置预览格式默认支持NV21、YV12
>         * <p>It is strongly recommended that either
>         * {@link android.graphics.ImageFormat#NV21} or
>         * {@link android.graphics.ImageFormat#YV12} is used, since
>         * they are supported by all camera devices.</p>

setPictureFormat设置图片格式支持NV21、RGB_565、JPEG
>          * @param pixel_format the desired picture format
>         *                     (<var>ImageFormat.NV21</var>,
>         *                      <var>ImageFormat.RGB_565</var>, or
>        *                      <var>ImageFormat.JPEG</var>)

同样可以通过Camera参数获取到当前设备Camera1所支持的格式集合。
```
List<Integer> picFormats = camera.getParameters().getSupportedPictureFormats();
List<Integer> preFormats = camera.getParameters().getSupportedPreviewFormats();

public void setPictureFormat(int pixel_format) {
            String s = cameraFormatForPixelFormat(pixel_format);
            if (s == null) {
                throw new IllegalArgumentException(
                        "Invalid pixel_format=" + pixel_format);
            }

            set(KEY_PICTURE_FORMAT, s);
        }

public void setPreviewFormat(int pixel_format) {
            String s = cameraFormatForPixelFormat(pixel_format);
            if (s == null) {
                throw new IllegalArgumentException(
                        "Invalid pixel_format=" + pixel_format);
            }

            set(KEY_PREVIEW_FORMAT, s);
        }
```

### Camera2
Camera2图像格式是通过ImageReader对象进行设置，同时还支持设置输出图像的宽高。图像格式可以是ImageFormat或是PixelFormat但并不是所有格式都支持例如下面在源码中发现的问题。
```
ImageReader.newInstance(mPictureSize.getWidth(), mPictureSize.getHeight(),
                ImageFormat.JPEG, /* maxImages */ 2);
```
有趣的是当你设置ImageFormat.NV21作为ImageReader的输出格式代码内部检查会抛出异常。但我测试发现保存的每帧图像格式却是NV21的，不知道是个别Android设备问题还是都是如此。
```
        if (format == ImageFormat.NV21) {
            throw new IllegalArgumentException(
                    "NV21 format is not supported");
        }
```

### 常见格式YUV
首先介绍YUV颜色编码方法
>YUV是编译true-color颜色空间（colorspace）的种类，Y'UV,YUV,YCbCr，YPbPr等专有名词都可以称为YUV，彼此有重叠。“Y”表示明亮度（Luminance、Luma），“U”和“V”则是色度、浓度（Chrominance、Chroma）。Y′UV,YUV,YCbCr,YPbPr所指涉的范围，常有混淆或重叠的情况。从历史的演变来说，其中YUV和Y'UV通常用来编码电视的模拟信号，而YCbCr则是用来描述数字的视频信号，适合视频与图片压缩以及传输，例如MPEG、JPEG。但在现今，YUV通常已经在电脑系统上广泛使用。
YUV的格式分为两类：
- 紧缩格式（packed formats)
将Y、U、V值存储成Macro Pixels数组,紧缩格式（packed format）中的YUV是混合在一起的，对于YUV4:4:4格式而言，用紧缩格式很合适的，因此就有了UYVY、YUYV等。

- 平面格式（planar formats）
将Y、U、V的三个分量分别存放在不同的矩阵中。每Y分量，U分量和V分量都是以独立的平面组织的，也就是说所有的U分量必须在Y分量后面，而V分量在所有的U分量后面，例如有I420（4:2:0）、YV12、IYUV等。

不同于RGB中每个像素点都有独立的R、G和B三个颜色分量值，YUV根据U和V采样数目的不同，分为如YUV444、YUV422和YUV420等，而YUV420表示的就是每个像素点有一个独立的亮度表示，即Y分量；而色度，即U和V分量则由每4个像素点共享一个。举例来说，对于4x4的图片，在YUV420下，有16个Y值，4个U值和4个V值。
YUV420根据颜色数据的存储顺序不同，又分为了多种不同的格式。可以说YUV420是一类格式的集合并不能完全确定颜色数据的存储顺序，例如YUV420Planar、YUV420PackedPlanar、YUV420SemiPlanar和YUV420PackedSemiPlanar，这些格式实际存储的信息还是完全一致的。I420（YUV420Planar的一种）则为YYYYYYYYYYYYYYYYUUUUVVVV，NV21（YUV420SemiPlanar）则为YYYYYYYYYYYYYYYYUVUVUVUV。

提取每个像素的YUV分量会用到。
YUV 4:4:4采样，每一个Y对应一组UV分量8+8+8 = 24bits,3个字节。
YUV 4:2:2采样，每两个Y共用一组UV分量,一个YUV占8+4+4 = 16bits 2个字节。
YUV 4:2:0采样，每四个Y共用一组UV分量一个YUV占8+2+2 = 12bits  1.5个字节。


> 有趣的是在实际开发中发现当使用Camera2设置YUV420_888作为数据流输出格式时检查数据格式却是NV21的格式，这也就验证官方注释中介绍的AndroidCamera输出格式最少支持的格式是NV21和YV12。当设备不支持高级格式会降级到父类格式，即使Camera2不支持设置为NV21格式输出最终却是NV21格式是不是很奇怪。
#### YUV422 Planar
Y\U\V数据是分开存放的,每两个水平Y采样点，有一个Cb和一个Cr采样点
![](../art/yuv422.jpg)
#### YUV420 Planar
格式跟YUV422 Planar 类似，但对于Cb和Cr的采样在水平和垂直方向都减少为2:1
![](../art/yuv420.jpg)
#### YUV422 Semi-Planar 
Semi 是’半‘的意思 我的理解这个半平面模式，这个格式的数据量跟YUV422 Planar的一样，但是U、V是交叉存放
![](../art/yuv422s.jpg)
#### YUV422 Interleaved  (packed formats)
![](../art/yuv422i.jpg)


## 参考
- https://developer.android.com/reference/android/graphics/ImageFormat
- https://www.polarxiong.com/archives/Android-Image%E7%B1%BB%E6%B5%85%E6%9E%90-%E7%BB%93%E5%90%88YUV_420_888.html
- https://msdn.microsoft.com/en-us/library/aa904813(VS.80).aspx
- https://www.cnblogs.com/watson/p/3788257.html