---
layout: post
title:  Capture Code Porting
date:   2016-06-24
categories: Drivers
tags: Drivers
---

### 后续追加
**2015**<br>
后续有修改过几次，从ti板子上移植到imx6板子上的capture完全跑通，并正常了，其中核心的问题是需要设置系统中关于图像采集的几个参数，包括每行内存大小，最大行Size，最大mem大小，以及还有一个参数是涉及图像是原始数据还是ISP scale的数据，都会对图像分辨率，显示产生影响。最开始调试，图像总是不对，还是没搞清这些参数所致。

### 参考
http://bbs.csdn.net/topics/390362302   imx6q上调cmos的帖子

### NFS不能挂载问题
**2014-06-05**<br>
今天打算在imx6板子上跑nfs，计划是在windows下coding，然后再vm下跑ubuntu编译，然后imx6通过nfs直接挂目标所在目录运行。按照网上的教程在ubuntu上安装nfs server，把exports目录指向vm共享的windows目录，结果客户端不能加载，在ubuntu下自己加载nfs也挂不上。更换其他目录之后，可以挂载上，看来不能使用vm的windows共享目录。

在imx6下挂载nfs时，也遇到问题。  
```
~ # mount -t nfs 192.168.0.164:/opt/Embedsky/Tools /mnt   
svc: failed to register lockdv1 RPC service (errno 111).  
mount: mounting 192.168.0.164:/opt/Embedsky/Tools on /mnt failed: Connection refused
```

后来按照网上搜索的提示，mount命令增加“-o nolock”参数，问题解决 
```
~ # mount -t nfs -o nolock 192.168.0.164:/opt/Embedsky/Tools /mnt
```

### 输出图像分辨率问题和调试
**2014-06-24**<br>
这几天在imx6q板子（E9）上调试cmos采集和显示，抽间隙时间调了2周多，终于完成。

在linux中，使用v4l2这一层调试图像采集和显示，不用考虑底层具体硬件，只需要了解v4l2相关的接口即可。手头的imx6q只有android下的摄像头demo，在linux下没有相应的demo。通过文件比对，和menuconfig浏览配置，确认在linux下也是有ov3640驱动程序的。于是开始进行。

找了两三个其他版本其他系统上cmos采集显示的程序，后来使用的是DEVKIT8500（DM3730）开发板上的一个demo程序。先花了几天研究demo程序，学习了v4l2的接口和流程。然后将demo程序挪到imx6q的开发环境中，程序中有几个设备文件的绝对路径，但是是#undefine掉的，所以直接删除掉，程序没有大改，即可编译运行，然后报错，ok，第一步完成。

搜索linux文件目录，/driver下有 video0，video1，video16，video17 4个文件，估计camara应该是0,1，display应该是16，17。然后屏蔽掉capture代码，修改了宽高等display的参数，display功能就循环跑起来了，读了一个图片进行显示，一切正常。display基本没有多大改动就运行了。

开始打开capture部分代码，最开始在ioctl阶段，有很多命令都过去不，DQBUF, G_INPUT, EMU_STD……。查看了一下v4l2的文档，适当调整了代码顺序，修改了配置参数，逐个测试，后来所有的ioctl都能正确执行。这个阶段很混乱，到最后我也搞不清楚是怎么就跑通了。有点晕

问题1：如图所示（图中图标的文件名里，6~8位数字是分辨率，宽高挨着写的）

<div align="center">
<img src="/images/capture-ov3640-bug1.jpg" width="600" />
</div>

在采图时，只要设置参数fmt的宽或者高超过1024，就会出现格子状的效果，而且确定，格子其实是由n帧图像构成的，同一行，可能是一帧图像的数据交错乱序了，但是从上往下，有多少行，就有多少帧图像。只要将宽高设置小于1024，图像采集就完全没问题，而且缩放也正常。但是需要注意，图像的宽度设置小于1024时，图像buffer的每一行，仍然是1024的宽度。如上图所示，当宽度小于1024时，用1024的宽度保存图像就能看到正常的图像，但是多出的宽度就是灰色的，如果按实际fmt设置的宽度保存图像，图像就会错乱了。

当行列数设置超过1024时，但是跟1024是整数比例关系，例如1280x720，1536x1152，2048x1536，就能看到格子状的采集图像，如果是其他比例的宽高，可能就看不到格子状，就是完全乱了，看不到轮廓和规则的条纹。

网上找到一个哥们在imx6q上调ov3640的例子[http://bbs.csdn.net/topics/390362302](http://bbs.csdn.net/topics/390362302)。  

他说是full 分辨率的时候，得设置 S_INPUT 参数为1，按这个修改后无效，图像全乱了。后来我又主动去修改fmt中的bytesperline参数，扩大为两倍，从4096改为8192，没有效果。又尝试将sizeimage参数改大，扩大一倍，从6291456（4K x 1536）改为12582912，但是这时就提示IOCTL分配不了这么大的缓冲区给视频capture，参照上面的贴子，应该是需要修改一个8M限制的地方。目前还没有继续去修改跟进。

所以目前我调通的3640，只能采集1024x1024分辨率以下的图像。

但是应该也问题不大，3640的最大分辨率是2048x1536，但是实际上embedsky提供的驱动，不论在linux还是android下面采集图像，都明显只能采集到右半边（摄像头正对墙壁，拍摄的图像明显是偏右的半边墙），所以有可能其实embedsky提供的驱动程序配置的3640就是输出1024x768的（如果full输出，还得改采集端的驱动，所以embedsky图省事，直接开窗输出，设置ov3640寄存器直接输出半拉图像，并且开窗连startx，starty寄存器也都没考虑修改过，所以就直接输出偏右的半边图像）。

        








![picture](/images/matou.jpg)

<h3>Original Jekyll instruction</h3>

You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:


    def print_hi(name)
      puts "Hi, #{name}"
    end
    print_hi('Tom')
    #=> prints 'Hi, Tom' to STDOUT.```


Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
