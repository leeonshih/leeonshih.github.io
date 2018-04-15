---
layout: post
title:  xu3 开发环境相关
date:   2017-02-09 
categories: Devel
tags: Devel Xu3
---
[pkg-config](http://blog.csdn.net/linux7985/article/details/6005267)<br>
[wayland build](https://wayland.freedesktop.org/building.html)<br>
[wayland driver for xu4](http://forum.odroid.com/viewtopic.php?f=96&t=20026)<br>
[xu3主页](http://odroid.com/dokuwiki/doku.php?id=en:odroid-xu3) <br>
[猪哥工房菜](http://www.embeddedlinux.org.cn/emb-linux/entry-level/201009/08-896.html)<br>
[mali drivers](https://developer.arm.com/products/software/mali-drivers/user-space)

### 参考资料
Xu4开发环境搭建  [http://www.zhimengzhe.com/linux/50121.html](http://www.zhimengzhe.com/linux/50121.html)<br> 
Xu3 Uboot编译 [http://odroid.com/dokuwiki/doku.php?id=en:xu3_building_u-boot](http://odroid.com/dokuwiki/doku.php?id=en:xu3_building_u-boot)<br>
Xu3编译环境搭建 [http://blog.csdn.net/aganlengzi/article/details/50036951](http://blog.csdn.net/aganlengzi/article/details/50036951)<br>
QtPro文件说明 [http://blog.csdn.net/woniuye/article/details/54927792](http://blog.csdn.net/woniuye/article/details/54927792)

### Image burning
从 www.odroid.com , www.hardkernel.com 都可以下载到完整的image 文件，lubuntu14.04, ubuntu16.04-mate 都有，windows下使用 **Win32DiskImager_odroid_v13** 写入到 tf 卡或者 emmc 中，然后可以直接从 tf 或者 emmc 中启动。

### Understanding Mesa & Mali
[http://forum.odroid.com/viewtopic.php?f=83&t=6933](http://forum.odroid.com/viewtopic.php?f=83&t=6933)<br>
这个帖子说了以下4点内容
1. xu3平台上，硬件加速依赖的是 mali driver，而不是 mesa。mali driver包括<br> 
- 开源内核驱动<br>
- 开源的EXA for X11<br>
- 二进制 libMali.so, 替代 libEGL（libEGL 是一个指向libMali 的符号链接）<br>
参考：[http://malideveloper.arm.com/develop-for-mali/drivers/](http://malideveloper.arm.com/develop-for-mali/drivers/)<br>
另外还有一个 fbdev 版本的 libMali.so , 这个版本仅需要内核驱动。可以下载并安装mali SDK，包含EGL的例子。<br>
2. libMali 在 /usr/lib/arm-linux-gnueabihf/mali-egl. /usr/lib/arm-linux-gnueabihf/libGEL.so 是一个指向前者的符号链接，可以手动修改这个符号链接，指向 /usr/lib/arm-linux-gnueabihf/mesa-egl/libEGL.so ,这时就是使用mesa的opengl-es软件了。mali 是作为alterantive 安装的，所以可以使用 update-alternatives 来切换 mesa 跟 mali。<br>
3. hardkernel 的 Odroid 支持 GLES2.0（3.0 for XU3），EGL under X11； and fbdev版本需要自行下载替换。<br>
4. 可以在odroid上边直接编译，无需交叉编译，慢不了太多。

### [Qt for embeded linux](https://doc-snapshots.qt.io/qt5-dev/embedded-linux.html)
<font color="#44bb44">这个帖子中间再次解释了configure时，prefix，hostprefix，extprefix的意思，基本说清楚了。 </font>
这个帖子介绍了在嵌入式平台上，QT的一些系统性原理
#### configure
 编译Qt需要 toolchain，sysroot，额外可能还需要第三方的 EGL，OpenGL ES2.0 库。
**qtbase/mkspecs/devices** 包含了数个设备的配置和显卡适配代码，比如 linux-rasp-pi2-g++ mkspec 包含了树莓派2设备的各项设置，同时也包含了相关设备的eglfs hooks实现或者eglfs 集成插件。通过 **\-device** 参数可以选择设备型号。下边是树莓派编译Qt的参考配置代码


```
./configure -release -opengl es2 -device linux-rasp-pi2-g++ -device-option CROSS_COMPILE=$TOOLCHAIN/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian/bin/arm-linux-gnueabihf- -sysroot $ROOTFS -prefix /usr/local/qt5
```


有两个重要参数，**\-device** , **\-sysroot**. Configure 的属性探测，会到 **\-sysroot** 指定的目录，而不是使用PC机上的标准位置，这种场景意味着主机上有什么跟我们的编译并不相关，我们是为一个目标系统进行编译，所以我们需要到目标系统的 根目录，也就是**\-sysroot**下边，去寻找libs 和headers。  

在交叉编译情况下，configure自动设置 **PKG_CONFIG_LIBDIR**,让**pkg-configs**报告编译链接设置，基于**\-sysroot**而不是主机。需要注意的是，主机不能设置 PKG_CONFIG_PATH 变量，否则会指向主机的相关目录  
指定**\-sysroot**会自动设置**\-\-ysroot**参数，一些情况下，可以使用 **\-no-gcc-sysroot**.  

**\-prefix** , 目标设备上想要部署的路径，**实际上 make install 并不往 prefix 上部署任何东西，而是往 extprefix 上部署**。

**\-extprefix**, 如果不指定，那么默认是 **sysroot + prefix**，但是sysroot经常可能会被 “污染”, 因此可以专门指定一下 extprefix（个人感觉好像没这么危险，基本没见指定extprefix的）。
**\-hostprefix**, 如果指定，那么 qmake，rcc，uic 等二进制文件会放置到此路径，而不是 extprefix 路径。

#### platform
 从 **Qt5.0** 开始，Qt不再包含自己的窗口系统 QWS 。嵌入式linux系统有多种platfrom可用：EGLFS（EGL FullScreen），LinuxFB，DirectFB，Wayland。依赖于Qt的配置，很多板子上，EGLFS是默认的选项。如果要用其他platfrom，可以修改QT_QPA_PLATFORM 环境变量或者 -platfrom 参数。

 EGL是OpenGL和原生窗口系统之间的一个接口。EGL并不包含平台相关性，窗口的创建仍然由平台相关的方式来实现，所以EGL仍然需要GPU相关代码，或者EGL设备集成插件。

EGLFS是QT与EGL，OpenGL ES2.0 之间的一个插件，用来实现不依赖确定的原生窗口系统（比如X11，Wayland）的Qt应用。在带有GPU的嵌入式系统上，这是推荐的方案。

EGLFS强制top-level窗口全屏，并且，后续窗口都是其子窗口。因此EGLFS只有一个原生窗口，和EGL surface。因此EGLFS也带了进一步限制，就是不能再打开额外的OpenGL windows或者混合类似 windows的content。

LinuxFB，使用fbdev进行绘图，只能软件渲染，没法使用硬件加速。

XCB，是X11的一个插件，主要用于桌面Linux platform，**在嵌入式系统中，XCB不推荐**。

Wayland， X11的潜在替代品。很多嵌入式ubuntu使用Wayland，并且有更好的性能，主要是为了解决 X11 冗余复杂和低性能的问题。

另，应该还有minimal，offscreen等，也就是用于测试qt程序，应该是没有显示的。

**[Mali SDK](http://forum.odroid.com/viewtopic.php?f=95&t=6615)**<br>
**[Link OpenGL ES3.0 on ubuntu](http://forum.odroid.com/viewtopic.php?f=95&t=23010)**<br>
**[Qt5.6 Mali](http://forum.odroid.com/viewtopic.php?f=95&t=21315)**<br>
**[Qt5 for xu3/xu4](http://forum.odroid.com/viewtopic.php?f=95&t=16305)**<br>
**[OpenGL ES demo](http://forum.odroid.com/viewtopic.php?f=95&t=15528)**<br>
**[EGL problem](http://forum.odroid.com/viewtopic.php?f=95&t=15524)**<br>
**http://forum.odroid.com/viewtopic.php?f=95&t=12484**<br>
**[can't run OpenGL es](http://forum.odroid.com/viewtopic.php?f=95&t=12115)**<br>
**[ES3.0 game load fail](http://forum.odroid.com/viewtopic.php?f=95&t=11919)**<br>
**[glmark2-es2 score](http://forum.odroid.com/viewtopic.php?f=95&t=8679)**<br>
**[OpenGL speed up](http://forum.odroid.com/viewtopic.php?f=95&t=8114)**<br>
**[odroid forum](http://forum.odroid.com/viewtopic.php?f=95&t=6020)**<br>
**[wayland](http://forum.odroid.com/viewtopic.php?p=140416)**<br>
**[egl error](https://devtalk.nvidia.com/default/topic/803737/qt-opengl-eglfs-egl-xcb-x11-wayland-jetson-board-tk1/)**<br>

### <font color="#bb6688">ISSUES </font>
#### 不能启动
中间碰到过一次问题，系统启动到  ***Freeing unused kernel memory：\*\*\*K*** 时就卡死了， 网上搜索都是说什么文件系统 page size 不对或者其他问题，但是本身又没有对文件系统进行操作，只好重新烧写 image 文件，还是卡在这个位置，再次重新写入 image，但是写入之间将 tf 卡分区完全删除，格式化一下，再次写入后可以正常启动。

**[ubuntu16.04 wifi设置](http://blog.csdn.net/u010162887/article/details/52081369)**<br>
**[xu3wifi和远程的例子](http://blog.csdn.net/hzkdac1007/article/details/46813673)**<br>
**[wpa_supplicant安装](http://blog.csdn.net/l_backkom/article/details/39780935)**<br>
**[vi上下左右变ABCD](http://blog.sina.com.cn/s/blog_65aee6e801018ida.html)**<br>
