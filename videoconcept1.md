

说到安卓的视频开发，大多数朋友们都是用着开源的播放器，或者安卓自带的native mediaplayer，拿来主义居多，我曾经也是。。。
最近这半年因为开始着手重构公司的播放器,也开始学习了很多视频音频开发的相关知识，抱着独乐乐不如众乐乐的想法，开始写一些值得分享的东西。这次的连载和之前的RxJava分享一样，会分开不容的章节。

第一次我打算分享一下视频开发中常见的一些知识点，概念和术语，给不熟悉的朋友们先"扫扫盲"。在之后的章节我会慢慢的介绍除了基本的在线视频播放技术之外，一些更加“高级”的技术 :smirk:,包括安卓平台在4.4之后发布的全新的[Codec API](https://developer.android.com/reference/android/media/MediaCodec.html), 还有怎么处理自适应视频播放([Adaptive Streaming](https://www.google.com.sg/webhp?sourceid=chrome-instant&ion=1&espv=2&ie=UTF-8#q=adaptive+streaming)),版权管理内容([DRM Content](http://searchcio.techtarget.com/definition/digital-rights-management))，最后几章会使用谷歌开源的[ExoPlayer](https://github.com/google/ExoPlayer)作为例子，从源码的角度分析一个完整的播放器需要哪些构件。

每一个术语我都会尽量用中文写一遍，再写一遍英文，因为说实话。。。不用英文查资料，很多东西都搜不出来。

![](https://github.com/richardissuperman/videoconcepts/blob/master/images/download.png?raw=true)


>1. 什么是Codec
>2. 什么是Container format file
>3. 视频处理的流程-从后台到前端


(华丽丽的分割线)
***********


### 1.什么是Codec

一说到视频，音频，大家肯定都听说，至少有所耳闻这两个词 - **编码**（encode） 和 **解码**(decode)。
我这里提到的Codec就是一种程序，这种程序可以对视频文件进行编码和解码。在维基百科上对Codec是这样定义的：
>**A video codec is an electronic circuit or software that compresses or decompresses digital video. It converts raw (uncompressed) digital video to a compressed format or vice versa. In the context of video compression, "codec" is a concatenation of "encoder" and "decoder"—a device that only compresses is typically called an encoder, and one that only decompresses is a decoder.**

那么问题来了，视频不就是视频吗，MP4，avi，rmvb，我们看的很多小电影不就是视频嘛。。。下载下来就可以看了啊。。。。为何需要编码解码。。。都是什么鬼。


...

<img src="https://github.com/richardissuperman/videoconcepts/blob/master/images/Screen%20Shot%202017-04-21%20at%2012.26.43%20am.png?raw=true" alt="Drawing" style="width: 400px;"/>


首先，我们常说**编码** 就是压缩，**解码** 就是解压缩。视频文件的本质其实就是图片的集合而已，当一段连续的图片不断的出现在人眼前(一般一个连贯的电影或者动画至少要求一秒24帧，也就是一秒内连续出现24张图片)，肉眼就会“欺骗性”的告诉大脑我们在看一个视频，而不是幻灯片。

那我们可以开始做点算术题了，假设一张像素为1280X720(清晰度，宽1280个像素点，高720个像素点)的图片，大小为约为1280X720X3 bytes,就是2.7MB。大家可以猜想一下为何我这里还需要乘以一个数字3.那么一段60秒钟的小电影，就需要60X24（24张图片）X2.7MB ，约为3.9GB了！

>之所以图片大小是像素宽高相乘还要乘3 是因为一个像素点需要至少三原色(**RGB**)来显示像素点本身的颜色，做过安卓开发的同学都知道在xml里面定义颜色的格式吧？#ffffff - 代表白色 f是十六进制数，也就是4位二进制数，三原色需要3X4X2位二进制数，也就是3个八位，一个八位是一个字节，所以我们需要3个字节来显示一个像素点

<img src="https://github.com/richardissuperman/videoconcepts/blob/master/images/Screen%20Shot%202017-04-21%20at%2012.40.15%20am.png?raw=true" alt="Drawing" style="width: 400px;"/>


这tm必然是不能接受的啊！这样我用我3TB的移动硬盘，也不能把苍老师的全部小电影保存起来，宝宝心里苦啊！

所以Codec这种程序就出现了，它会把这些连续的图片们通过一定的算法压缩成体积更小的文件格式，这就是我们所谓的**编码**，压缩。但是在播放器的客户端，不管是PC，手机也好，他们要显示在屏幕上的，必须是实实在在的图片啊，所以这些被压缩过的文件最终又必须被还原成图片格式，这就是**解码**，解压缩。

视频编码，压缩是一个非常复杂的过程，万幸的是，现在市面上已经有很多工具，还有现有规范来指导开发者进行编码解码了。其中最常用的一些规范是:


<img src="https://github.com/richardissuperman/videoconcepts/blob/master/images/Screen%20Shot%202017-04-21%20at%2012.49.19%20am.png?raw=true" alt="Drawing" style="width: 400px;"/>


可能大家对压缩解压缩还是不太理解。。。到底有哪些地方可以压缩呢？
那我们举个栗子！

![](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSuChlyUk5r4NblScx6I0TZupZyks5dU26CD-e-6YmtgSrMXFED_g8aD6I)


咱们想象一下一个场景，比如说在某些电影中，主人公在安静的公园中因为失恋悲伤不已，全世界都仿佛静止一般。。。。就这么呆坐了整整30s。那么对于这种“静态的场景”，视频压缩算法会只取这三十秒的前几帧作为基准帧图片，对其余的29s的帧，采取只保存“不同的部分”的策略，这样就不用保存这些差不多相同的图片，这种做法叫“去冗余”。大大减少了视频的体积。

![](https://github.com/richardissuperman/videoconcepts/blob/master/images/Screen%20Shot%202017-04-21%20at%2012.45.14%20am.png?raw=true)


当然，这只是视频压缩算法的冰山一角，我们不多研究。

另外需要注意的是，Codec的编码与解码包含对**视频数据**的编码解码和**音频数据**的编码解码,因为音频的本质是声波信息，视频是图片处理，他们本质上是不同的，我这里主要是介绍视频数据的处理。

回到我们说的Codec，所以说Codec是一套程序，它遵循不同的规范，根据规范的不同提供不同的压缩解压缩策略。
既然是一套规范，那么就肯定需要实现啊！ 在安卓平台里面，谷歌提供了视频编码解码的API，对一些基础的编码解码规范做了API的封装，在接下里的章节我会慢慢介绍，其他移动平台也都差不多，多多少少都会提供API的支持。


### 2. 什么是Container format file（视频容器文件）

之前说到，咱们在看小电影的时候都会看到很多文件的后缀名，例如mp4,rmvb,avi，喜欢看高清美剧的同学应该还会经常看到所谓的蓝光mkv格式等等。我们习惯叫他们视频文件，但是这样说显得不够专业。。。

严格的来讲，他们应该被叫做容器文件。。。。因为一个容器里，不仅仅包括了视频(video)数据，还包括了(audio)音频数据，有的容器还内嵌字幕，那么就还有文字(Text)数据。不过容器文件虽然听起来吓人，但是它说到底也就是一个结构化的文件而已。之所以说它结构化，就是它包含的视频，音频，文字数据都必须按照一定的规范，放在文件指定的位置(方便播放器解析)。

容器文件就是上面说到的Codec程序对图片集进行编码之后的产物，被Codec编码之后，除了必要的视频音频信息之外，它还有一些其他的信息。

我画了一个草图，解释了一个经典的MP4容器结构是啥样。。。

![](https://github.com/richardissuperman/videoconcepts/blob/master/images/Screen%20Shot%202017-04-21%20at%201.07.08%20am.png?raw=true)

里面提到了Track(轨道)，这是一个专业术语，用来区分不同的音视频/文字数据
但是MP4文件里面最重要的却是这个MetaData，它包含了很多关于视频的原始数据，比如视频的大小，视频的时长，还有**一个索引表**，这个索引表包含了不同轨道的起始位置(以字节为单位)，又因为每个轨道会被分成若干块sample(采样，每一块采样都是可以单独被播放器播放的一段数据，以微妙为单位)，metadata也会维护一个细粒度更小的索引表，记录了每一块sample的大小，起始位置，对应视频的时间是多少(以字节为单位)等等的信息。

举个简单的例子，有些电影包含粤语，国语两个声道。我们想换声道的时候会告诉播放器，我想听粤语，那么播放器会去索引表查找粤语的轨道起始位置，并且源源不断的读取粤语音轨的数据并播放出来。这也解释了为何上图会有两个audio track。

在接下来的章节我会详细介绍播放器是怎么解析容器文件，这里大家只需要知道大概就好。


### 3. 视频处理的流程-从后台到前端

从一个实际的流程出发，

![](https://github.com/richardissuperman/videoconcepts/blob/master/images/Screen%20Shot%202017-04-21%20at%201.21.16%20am.png?raw=true)

导演用胶片拍摄了原片(Raw Data)，胶片就代表着原始文件，也就是图片集(因为胶片就是一帧一帧的连续图片)，使用软件把源文件编码(Encode)成容器文件(Container),之后可能为了不容分辨率的原因，还需要将原始的高清容器，转换成不同的分辨率的容器文件，对应图中的process这一步。最后在放在服务器或者CDN上，又播放器将其下载播放。

![](https://github.com/richardissuperman/videoconcepts/blob/master/images/Screen%20Shot%202017-04-21%20at%201.21.28%20am.png?raw=true)

补充一张图。


*************
华丽丽的分割线

ok，这次分享就结束了，我会在下次分享详细介绍播放器是怎么解析，读取容器文件，同时也会深入的介绍一下MP4容器的一些格式规范，有助于大家去分析开源播放器的源码。

最后给不知道为何要乘3的同学答案。
>之所以图片大小是像素宽高相乘还要乘3 是因为一个像素点需要至少三原色(**RGB**)来显示像素点本身的颜色，做过安卓开发的同学都知道在xml里面定义颜色的格式吧？#ffffff - 代表白色 f是十六进制数，也就是4位二进制数，三原色需要3*2*4位二进制数，也就是3个八位，一个八位是一个字节，所以我们需要3个字节来显示一个像素点