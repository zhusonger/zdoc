# 一. 场景

有一张10000x10000的图片, 需要取其中的人像头部区域, 并缩放到512x512的大小

首先需要识别到头像, 借助识别SDK识别, 传入整体图【图1】, 识别出头像区域得到头像图【图2】

简单的一个计算图片的内存占用, 假如图片是ARGB_8888的, 那这张图占用的内存是 10000 x 10000 x 4 / 1024 / 1024 = 381.5MB

以下是ChatGPT的回答

* Question

	> Bitmap在不同的Android系统版本的内存限制有什么区别

* Answer

	> 在不同的 Android 系统版本中，Bitmap 内存的限制是不同的，主要有以下几种区别：

	> Dalvik VM：在 Android 4.4 及以下版本中，Dalvik 虚拟机使用的是堆内存，而堆内存的大小是由 Dalvik Heap Size 参数控制的。在默认情况下，Dalvik Heap Size 的大小通常是与设备的 RAM 大小相关的，但是开发者可以在应用程序的 manifest 文件中设置 android:largeHeap="true" 属性来增加 Dalvik Heap Size 的大小，从而允许应用程序使用更多的堆内存。

	> ART：在 Android 5.0 及以上版本中，ART 虚拟机引入了新的内存分配方式，即使用了可分配的非堆内存空间来存储 Bitmap 对象，这种内存空间被称为“native memory”（本地内存）。与 Dalvik VM 不同的是，ART 中 Bitmap 对象的内存限制是与设备的 RAM 大小无关的，而是由系统分配的本地内存空间大小决定的。在 Android 5.0 及以上版本中，每个应用程序能够使用的本地内存空间大小通常为 1/4 到 1/2 的可用 RAM 大小，但是系统也会根据当前内存使用情况进行调整。

	> Android Go：在 Android Go 版本中，为了适应低端设备的内存限制，Google 对 Android 系统进行了一系列的优化，其中之一就是对 Bitmap 对象的内存限制进行了调整。在 Android Go 版本中，每个应用程序能够使用的最大 Bitmap 内存大小通常为 16 MB，这是相对于普通 Android 系统来说更小的内存限制。

	> 因此，不同版本的 Android 系统中 Bitmap 对象的内存限制存在较大的差异，开发者需要针对不同的系统版本进行优化和适配。如果 Bitmap 对象占用过多的内存，可能会导致 OOM 错误，因此在开发过程中应该尽量避免使用过大的 Bitmap 对象，同时及时释放不需要的 Bitmap 对象所占用的内存空间。

---

简单来说, 如果要原图加载【**一张**】图, 不考虑其他的内存占用, 就需要至少 1G左右的手机硬件内存, 注意只是【**一张**】

## 单个像素的数据格式

单个像素的数据格式是一种原始的、未经过压缩的数据格式

* 1 位（单色）：每个像素只有黑和白两种颜色，占据 1 个 bit。用于黑白图像和文本等简单的图像处理。
* 2 位（4色）：每个像素有 4 种颜色选择，占据 2 个 bit。用于一些简单的图像处理，如图标和游戏界面等。
* 4 位（16色）：每个像素有 16 种颜色选择，占据 4 个 bit。常用于低端移动设备或嵌入式系统的图形显示。
* 8 位（256色）：每个像素有 256 种颜色选择，占据 8 个 bit。常用于一些较为简单的图像处理，如简单的图片编辑器和 Web 设计中的 GIF 图像等。索引颜色。
* 16 位（高彩色）：每个像素有 2 的 16 次方（约 6.5 万）种颜色选择，通常采用 RGB565、ARGB4444 或 RGBA5551 格式，占据 2 个字节。用于显示图片或视频等的中端设备。
* 24 位（真彩色）：每个像素有 2 的 24 次方（约 1600 万）种颜色选择，通常采用 RGB888 格式，占据 3 个字节。用于显示照片等高品质的图像。
* 32 位（真彩色 + Alpha 通道）：每个像素有 2 的 32 次方（约 42 亿）种颜色选择，通常采用 ARGB8888 或 RGBA8888 格式，其中 Alpha 通道占据高 8 位，Red、Green、Blue 三个通道各占据 8 位，共占据 4  个字节。用于需要透明效果的高品质图像处理，如图像编辑器和游戏开发中的粒子效果等。
* 48 位（16 位色深 RGB）：每个像素由 16 位红、16 位绿、16 位蓝三个通道组成，共占据 6 个字节。用于高品质图像的处理和传输，如数字摄影和高端电视等。
* 64 位（16 位色深 RGBA）：每个像素由 16 位红、16 位绿、16 位蓝和 16 位 Alpha 通道组成，共占据 8 个字节。用于高品质图像的处理和传输，如数字摄影和高端电视等。

> 以上是常见的像素数据格式及其特点。需要注意的是，同一种像素数据格式的存储方式可能不同，例如像素排列顺序（如大端序、小端序），行对齐方式等。
> 
> Android中Java/Kotlin默认使用大端字节序，所见即所得
> 
> NDK(Native) 中C/C++默认使用小端字节序
> 
> ```java
> // 在Android里使用OpenGL时, 设置坐标点的就是使用Native的字节序
> ByteBuffer bb = ByteBuffer.allocateDirect(WORLD_COORDS.length * 4);
> bb.order(ByteOrder.nativeOrder());
> mVertexBuffer = bb.asFloatBuffer();
> mVertexBuffer.put(WORLD_COORDS);
> mVertexBuffer.position(0);
> ```

---

## 常见的图像格式

常用的图像格式（如 PNG、JPG、BMP、WebP、GIF、HEIC 等）是经过编码和压缩的数据格式

* JPEG（Joint Photographic Experts Group）：数据格式为RGB、YCbCr等；压缩方式为有损压缩，使用离散余弦变换（DCT）和量化技术，可控制压缩比和图像质量；编码方式为基于二进制的编码方式。
* PNG（Portable Network Graphics）：数据格式为RGB、RGBA等；压缩方式为无损压缩，使用DEFLATE算法，可保存图像的透明通道；编码方式为基于二进制的编码方式。
* BMP（Bitmap）：数据格式为RGB、RGBA等；压缩方式有无压缩和RLE压缩两种方式，其中RLE压缩是一种无损压缩方式，但只适用于图像中有大量连续相同像素的情况；编码方式为基于二进制的编码方式。
* WebP：数据格式为RGB、RGBA等；压缩方式为有损和无损压缩，有损压缩使用VP8视频编码技术，可实现高压缩比和高图像质量，无损压缩使用预测编码和熵编码技术，适用于有透明通道的图像；编码方式为基于二进制的编码方式。
* GIF（Graphics Interchange Format）：数据格式为索引颜色（Indexed Color），即使用颜色表中的索引值来表示每个像素的颜色；压缩方式为无损压缩，使用LZW算法；编码方式为基于二进制的编码方式。
* HEIC（High Efficiency Image Format）：数据格式为RGB、YCbCr等；压缩方式为有损压缩，使用HEVC视频编码技术，可实现高压缩比和高图像质量；编码方式为基于二进制的编码方式。

---


# 二. 实现方案对比

假设原图的头像区域识别为1000 x 1000的大小

内存 = 1000 x 1000 x 4 / 1024 / 1024 = 3MB


## 1. 粗暴做法

直接读入原图【图1】, 校正Exif的旋转与镜像获得【图2】, 识别【图2】, 得到头像区域, 裁切头像区域【图3】


> 内存计算
> 
> 【图1】380MB + 【图2】380MB + 【图3】3MB = 763MB
> 

缺点: 开发SDK能识别这么大的图, 且识别还是会有内存使用, 耗时相对的增长

## 2. 优化版本1

缩放到1000x1000读取缩放图【图1】, 校正Exif的旋转与镜像获得【图2】, 识别【图2】, 得到头像区域, 读入原图【图3】, 校正Exif的旋转与镜像获得【图4】, 裁切头像区域【图4】

> 内存计算
> 
> 【图1】3MB + 【图2】3MB + 【图3】380MB + 【图4】380MB + 【图5】3MB = 766MB
> 

我去, 优化完咋内存还更大了呢, 主要是大部分识别SDK还是不能使用太大的图识别, 项目中一般使用的该方案

## 3. 最终优化版

缩放到1000x1000读取缩放图【图1】, 校正Exif的旋转与镜像获得【图2】, 识别【图2】, 得到头像区域, 校正Exif的旋转与镜像【映射】到原图的区域, 解码换算原图的头像区域图片【图3】, 校正Exif的旋转与镜像获得【图4】

> 内存计算
> 
> 【图1】3MB + 【图2】3MB + 【图3】3MB + 【图4】3MB = 12MB
> 
 
哎嗨, 这个好!

# 三、Android如何实现

对比了上面的3个方案, **最终优化版**明显更好, 接下来开始Code

其他步骤懂得都懂～

但是为啥要映射呢, 因为原图可能带Exif的旋转与镜像, Android中大图局部解码的是不会考虑这些的(我觉得大部分实现都不会去考虑)

那在Android中如何实现呢?

其实就是如何从校正过的头像区域, 换算到原图的区域, 如果自己去计算会比较麻烦, 利用Matrix【ˈmeɪ.trɪks】进行换算

本处提供了1~8对应的图片供参考, origin文件夹下对应同名就是去除meta信息的原图

<img src='./values.png'>

## 1. 映射计算

<img src='./transform_2_origin.png'>

上图表示ORIENTATION_FLIP_HORIZONTAL, 左右镜像的图片映射过程

这里工具原因Y轴负坐标无法修改为正坐标, 这里理解成Y轴正坐标

(0,0)表示图片左上角, X正坐标表示图片左上角往右, Y正坐标表示图片左上角往下

【右侧大框】(0, 0, 10, 10) 表示缩放图的整体图

【框1】(2, 2, 4, 4) 表示头像在整体图的位置

【框2】(-4, 2, -2, 4) 表示头像镜像还原到原图的位置

【框3】(6, 2, 8, 4) 表示头像还原到原图的位置

其他旋转与镜像关系通过类似的方式去映射，即可得到头像框在原图(未校正)的区域

## 2. Show Me Code

```java

val matrix = Matrix()
val rotate = orientation != ExifInterface.ORIENTATION_NORMAL 
		&& orientation != ExifInterface.ORIENTATION_UNDEFINED
if (rotate) {
    // Rotate the image back to origin.
    // 缩放图为【图2】
    when (orientation) {
        ExifInterface.ORIENTATION_FLIP_HORIZONTAL -> {
            matrix.postScale(-1f, 1f)
            matrix.postTranslate(缩放图宽, 0f)
        }
        ExifInterface.ORIENTATION_ROTATE_180 -> {
            matrix.postRotate(180f)
            matrix.postTranslate(缩放图宽, 缩放图高)
        }
        ExifInterface.ORIENTATION_FLIP_VERTICAL -> {
            matrix.postScale(1f, -1f)
            matrix.postTranslate(0f, 缩放图高)
        }
        ExifInterface.ORIENTATION_TRANSPOSE -> {
            matrix.postRotate(90f)
            matrix.postTranslate(缩放图高, 0f)
            matrix.postScale(-1f, 1f)
            matrix.postTranslate(缩放图高, 0f)
        }
        ExifInterface.ORIENTATION_ROTATE_90 -> {
            matrix.postRotate(-90f)
            matrix.postTranslate(0f, 缩放图宽)
        }
        ExifInterface.ORIENTATION_TRANSVERSE -> {
            matrix.postRotate(90f)
            matrix.postTranslate(缩放图高, 0f)
            matrix.postScale(1f, -1f)
            matrix.postTranslate(0f, 缩放图宽)
        }
        ExifInterface.ORIENTATION_ROTATE_270 -> {
            matrix.postRotate(90f)
            matrix.postTranslate(缩放图高, 0f)
        }
    }
}
val scale = 原图大小. / 识别的缩放大小
matrix.postScale(scale, scale)
// 头像区域就更新为转换后在原图的区域
matrix.mapRect(头像区域)

val decoder = BitmapRegionDecoder.newInstance(path, true)
// 为了避免出现原图内头像区域同样非常大, 比如有8000x8000, 不需要这么大的图时, 设置下最大目标尺寸
// 在解码时进行缩放即可
val opts = BitmapFactory.Options().apply {
    // 校正的头像区域是目标大小2倍以上时, 进行缩放, 目标大小 ~ 2x目标大小, 保持原图大小
    var inSampleSize = 1
    if (原图内头像区域尺寸 >= 目标大小 * 2) {
        while (原图内头像区域尺寸 / inSampleSize >= 目标大小 * 2) {
            inSampleSize *= 2
        }
    }
    this.inSampleSize = inSampleSize
}
opts.inPreferredConfig = Bitmap.Config.RGB_565
decoder.decodeRegion(头像区域, opts)
// 记得回收哟
decoder.recycle()
```




