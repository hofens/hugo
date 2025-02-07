---
share: true
---

# 一、插件开发目的

方便开发时查找项目中可复用图片，尽量避免引入重复图片资源  
![](http://km.fp.ps.netease.com/file/65322a1e94064f5d24433301W3eV4y0m05)

# 二、插件功能

**以图搜图**：上传图片后可找到项目中存在的相似图片，根据相似度设置，可返回不同相似程度图片。相似度为1则为检索相同图片，如果没有相同图片，可适当降低相似度查找，会搜寻指定相似度以上的图片（如0.8，会查找>=0.8相似程度图片）。

**图片缓存**：为了快速查找到图片，会将图片相关数据缓存下来，首次缓存将在项目安装插件重启AS后自动构建缓存，预计后台2分钟完成。也可以点击重建缓存按钮，可重新生成图片数据的缓存；一般不需要主动去重建缓存，因为插件会在图片检索、删除等时机更新缓存数据。

**重置预览**：点击重置预览，可清除正在预览的图片。

**导入重复图片时弹窗提醒**：勾选后，可在项目启动时开启监听，当拷贝或移入新图片时，会自动检测项目中是否存在相同图片，若存在会弹窗提示。不依赖于插件，即使插件没打开，也可以使用。

**损毁文件检测**：当重建缓存完成后，如果项目中存在问题图片，会发送消息，可在Event Log查看。

# 三、插件使用

IDE需安装Alice插件，需下载>=1.0.191 beta版，低版本AS可能不支持需升级；重启后会先用2分钟时间构建缓存。

下载地址：[https://plugins.jetbrains.com/plugin/20890-alice/versions/beta](https://plugins.jetbrains.com/plugin/20890-alice/versions/beta) 或 下载底下附件

Figma插件使用需安装Wormhole，可更新最新版。

## 3.1 使用插件查找重复图片

1、将图片拖拽或打开目录选取图片  
2、图片打开后会显示预览图，并自动开始检测  
3、如果不存在相同图片，可重命名刚选择的图片，并复制该文件到剪切板，可直接在项目目录下粘贴  
4、如果存在相同图片，可在搜索结果中单击预览选中图片，双击复制文件名，三击可在项目中打开文件  
5、可修改相似度，设置完成点击Research可再次查找

## 3.2 无需主动使用插件，粘贴图片自动提醒

无需打开插件，当向项目中导入新图片时，会自动检查是否有重复图片，如果有重复图片会弹窗进行提醒

## 3.3 Figma插件使用

无需打开插件，在Wormhole插件选中UI图后，会自动查找相似图片

# 四、说明

**1.相似度如何取值？**

0-1之间，越大相似度越高。默认设定了0.95（经验值），会查找相似程度在0.95及以上的结果，如果未找到相似图片可降低相似值；如果只想找相同图片，可设为1。

**2.自动检测会弹窗提醒**

自动检测的相似度是1也就是拷贝了一张一样的图片才会触发，预计不会很频繁；这个功能也支持关闭。在插件上有个选项可勾选取消。

# 五、原理

任何一种颜色都是由红绿蓝三原色（RGB）构成，图片的每个像素点都有一个RGB（如：0,255,255），统计每种颜色（RGB）在图片所有像素上出现的次数，可得出该图片的颜色特征。

> 例子：  
> (0,0,0) 出现次数10000次  
> (0,0,1) 出现次数25次  
> ...  
> (0,0,255) 出现次数90次  
> ...  
> (255,255,255) 出现次数1000次  
> 把这些用次数收集起来，得到集合{10000,25,...,90,...,1000}，这个就可以代表一张图片上分布RGB的情况。  
> 但如果R,G,B统计都是0-255范围的话，有255^3=1600多万种组合。所以实际上为了简化计算，将0～255分成四个区：0～63为第0区，64～127为第1区，128～191为第2区，192～255为第3区，可以理解为把R/G/B都简化为了4种而不是255种，这样RGB的组合就只有4^3=64个。

分别对两个图片都这样操作，然后得到这个数据集合，再通过皮尔逊相关系数（一种计算两组数据是否存在相关性的算法）进行对比，就可以得出这两张图片的相似度。

> 算法来源：

[相似图片搜索的原理](https://www.ruanyifeng.com/blog/2013/03/similar_image_search_part_ii.html)

# 六、几个例子

![](https://cc.fp.ps.netease.com/file/653223816b453a58be36fda6zNXBRXdy05)