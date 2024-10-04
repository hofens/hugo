---
share: true
---

## 一、目的

目前CC使用第三方库android-gif-drawable来播放直播列表封面的动图，这个库在性能以及内存占用上都有较好的表现，为了进一步优化gif内存占用，在内存紧张时能够及时回收以及需要时自动加载，修改android-gif-drawable源码实现该目标。

## 二、GIF播放逻辑

创建：GIFDrawable支持多种方式构造，如InputStream、ByteBuffer、Resources、File等。每次都会创建GifInfoHandle 对象，保存gif文件在Native层对应的GifInfo的地址，通过GIFInfo地址我们可以获得当前的Gif文件信息和读取方法。

绘制：设定定时任务，每次传给Natice层 GifInfo对象地址与一个Bitmap， Native层通过传入GifInfo对象地址不断从GIF文件中获取下一帧的数据，Natice层将这一帧数据生成到Bitmap上，在下一次GifDrawable#draw()的时候进行绘制。

 ![](assets/Pasted%20image%2020240530141943.png)
因此使用文件或者InputStream可以保证内存里只存在一帧和共享区域的数据，如果使用ByteBuffer或者byte[]的话，整个Gif文件必须存在在内存里，会占用较大的内存。

## 三、可能的优化方向

### 1、在activity/fragment onPause的时候停止GifDrawable#draw()

onPuase情况下因为主动调用stop所以不会继续绘制。

![](assets/Pasted%20image%2020240530141951.png)
### 2、内存不足时释放大内存对象

大内存对象主要是用于绘制当前帧的Bitmap与gif数据。

#### 2.1 获取GIF数据占用内存

以18.3M的gif文件为例，如果以byte[]初始化GIFDrawable，需要将gif完整加载到内存中，会多增加19M左右的内存；由于CC是通过File的方式构造Drawable的，不会把整个文件加载到内存中，所以并不会占用太多内存，用于保存gif加载方式及gif数据指针的对象占据内存如下

 ![](assets/Pasted%20image%2020240530142005.png)

内存截图如下：

未初始化GIFDrawable  
 ![](assets/Pasted%20image%2020240530142011.png)

通过File初始化GIFDrawable

![](assets/Pasted%20image%2020240530142019.png)

通过byte初始化GIFDrawable，相比File多了byte[]的内存

![](assets/Pasted%20image%2020240530142026.png)

#### 2.2 绘制当前帧的Bitmap占用内存

CC中常用的动图场景是直播列表封面，随机选取一个gif文件size是639x360，则bitmap占用内存在0.9M 左右。

要省去这个开销可以在内存不足时回收Bitmap，重新播放动画时创建Bitmap。

### 结论

CC目前会在onPause、onDetachedFromWindow()的时候会停止Gif的播放，基本覆盖了CC使用gif的场景（进房停止播放列表封面动图、滑动列表看不见的item停止封面动图），因此可以补充下在内存不足时进行回收的逻辑

## 四、修改方案

**cc-gif-drawable**

回收bitmap：GIFDrawable增加clearFrameBitmap()，调用该方法时判断如果当前gif不在播放，则回收Bitmap。

恢复bitmap：保存创建GIFDrawable的方式以及相关数据，在再次调用start()时根据保存的gif数据重新创建Bitmap。

**CC**

修改LifecycleGifImageView，监听内存不足TrimMemoryEvent事件，当内存不足时，调用clearFrameBitmap()对不在播放的gif进行回收Bitmap。

可回收以下场景gif Bitmap

1.onPause时会执行GifDrawable#stop()方法停止播放gif，这时候如果收到内存不足事件Bitmap就会回收；onResume时会调用GifDrawable#start()重新播放gif。适用于进出房间、home退出app等场景。

2.直播列表滑动，当onDetachedFromWindow()时会停止播放gif，此时遇上内存不足也会回收Bitmap

##   五、优化效果

从Profile中不好看出CC在内存不足时总共能回收列表GIF Bitmap多少内存，理论上通过计算一个size在639x360的gif，用于绘制的bitmap占用内存在0.9M。