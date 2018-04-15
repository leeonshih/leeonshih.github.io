---
layout: post
title:  MCIM驱动CVI输出
date:   2017-10-15 
categories: Hardware
tags: Hardware
---

## DH9801

碰到一个需求，希望将4个摄像头视频集成到一路CVI输出（我也没有办法，人家的设备上只有一个CVI输入接口，想要连接额外的摄像头，只能通过CVI信号输入），于是就上手开始调试DH9801，本来打算是三周时间连硬件带软件全部调试完成，但是中间碰到一个糟心的地方，前后花了两个月时间才调通。 

<div align="center">
<img src="/images/mcmm-to-cvi-1.jpg" width="600" />
</div>

