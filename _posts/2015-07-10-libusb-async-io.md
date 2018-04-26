---
layout: post
title:  Libusb Async IO
date:   2015-07-10 13:08:09
categories: Hardware
tags:  USB3.0 Hardware
---
<!--more-->
前段时间已经调通了 cypress例程跟slavefifo--fpga的通信，但是cypress的例程使用的是syncIO方式，对cpu有一些占用，因此打算使用asyncIO的方式。

网上使劲找资料，没有任何例子，只有 www.libusb.info 网站上有一个asyncIO的说明，已经网上能找到几篇这个说明的中文翻译，就再也找不到其他例子或者demo了。

官方的资料没有给出例子，只是有一个说明，就是如果使用asyncIO方式，有5个步骤。

1. 完全没有头绪，查看文档资料后，又查看libusb的源代码，接合aysncIO的官方说明，发现了一个问题，其实syncIO方式就是把几个函数组合到一起了，如果把那几个函数拆开，自己组织一下代码结构，就是aysncIO方式。

2. 讲libusb源码中的头文件和sync.c完全集成到cypress的演示代码中，然后把 cyusb_bulk_transfer 函数的代码找出来，直接按照代码中的代码执行顺序放到main（）中，然后屏蔽到 reader进程。再按照官方资料的说明，添加了 slusb_bulkin_cb（）函数，运行程序，不进中断，也没有反应，libusb_submit_transfer() 之后就停住了。

3. 再看cyusb_bulk_transfer() 函数的源码，确定还需要一个sync_transfer_wait_for_completion（）函数，这个函数里的关键程序是libusb_handle_events_completed(ctx, completed);正是这个函数实现了阻塞。于是在 libusb_submit_transfer()之后执行这个函数，并且在slusb_bulkin_cb()中的libusb_submit_transfer（） 之后都运行这个函数。运行程序，可以收到一个usb包，然后就卡住了，通过打印信息，发现程序执行的逻辑有点问题，cb函数冒失是在 libusb_handle_events_completed（）中间执行的，所以逻辑自然就卡住了，但是想想也不对，如果使用libusb_handle_events_completed（）这样的阻塞函数，那syncIO跟asyncIO就一点区别都没有了。只不过是 传输函数阻塞 哪一个进程的问题，总之会阻塞一个。

4. 再次看官方文档，发现文档里介绍的，处理上下文的程序是libusb_handle_events（），根据命名，可以判断，它跟libusb_handle_events_completed（）是一个功能，只是一个是非阻塞，一个是阻塞的，但是非阻塞的也得在死循环中一直运行，才能保证，后台所有的usb传输操作都能被处理到。按照官方文档的说法。然后将 libusb_handle_events_completed（）换为 libusb_handle_events（），并从程序流程中删掉，把libusb_handle_events（）单独放到一个线程中死循环执行。

5. 修改之后asyncIO模式跑通了，但是初步测试了一下，发现syncIO，asyncIO两种方法，cpu资源的消耗是差不多的，并没有按文档所说那样，more efficient，more powerful。想想也是，都是同样的几个函数，只不过是捣了一下代码执行的流程和逻辑而已，本质上，都需要一个死循环处理后台动作，而不是想象那样，在cb函数中处理后台动作。在T430笔记本上，测试，全速跑起来，都是接近300MB的带宽，cpu占用都是11%左右。aysncIO模式稍微低一点。


搭了一个开发环境，想通过USB3.0采集aptina sensor的数据到PC上，使用的是huanor的xilinx usb3.0开发板。aptina的sensor上电之后默认是不工作的，必须要i2c配置。手头i2c配置程序都齐全，并且测试过，问题是usb3.0的板子上i2c暂时没有调通，比较郁闷。如果通过imx6上的i2c来配置sensor，需要把两个板子搁到一起，imx6还需要跑虚拟机，还有有控制台，实在是繁琐。所以在网上采购了一块usb2iic的小板子。之所以从3,4个备选中挑了这个，是因为淘宝上，这是唯一明确说明支持linux的，但是老板很明确的说，linux下面的demo有代码，但是不管技术支持，不管能不能用。

把老板提供的代码拷贝下来，再看了一下程序结构，写了个makefile，编译，基本没啥问题，唯一有个错误是 DWORD没用定义，讲DWORD define为int，提示type不兼容，按照报错提示，讲int改为long unsigned int，报错解决。

运行程序，提示 open usb device错误。看了一下程序代码，并不是一上来就出错，是调用了几个libusb函数，到claim interface时才出错的。这时用lsusb命令看了一下，usb2iic板子已经连接到pc上了，列表里边有设备（无名称，有跟板子一致的VID，PID）。用cypress的测试程序跑了一下，可以显示对应VID，PID的descriptor。

网上搜索“libusb_claim_interface busy”,结果出来一个帖子 http://bbs.chinaunix.net/thread-1917355-1-1.html
提示在claim之前，先调用 libusb_detach_kernel_driver() 函数。按照提示增加了这个函数，果然，可以正常open并close了。
