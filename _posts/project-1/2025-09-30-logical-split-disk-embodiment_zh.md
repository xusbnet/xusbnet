---
layout: post
toc: true
title: "逻辑分盘技术分享与演示"
description: "本文深入解析 Project 1 的核心技术——逻辑分盘（Logical Split Disk，LSDisk）的实现细节，阐述其如何通过底层扇区地址重映射实现单一物理存储的真正隔离与多盘虚拟化，并展示其在多操作系统环境下的原生兼容性与应用价值。我打造了一个USB控制器，通过收回对存储设备的控制权，将1个U盘变为4个。"
author: xusb_team
date: "2025-9-30"
permalink: /project-1/logical-split-disk-embodiment-zh/
project_id: project_1
categories:
  - project-1
lang: zh
rss_exclude: true


tags:
  - "Project-1"
  - "逻辑分盘"
  - "智能存储"
  - "扇区映射"
  - "数据安全"
summary: 通过底层扇区映射与容量回报改写，将单一存储虚拟为多独立磁盘，实现硬件级隔离与OS无感兼容。
---
<div class="lang-switch lang-switch--right" markdown="1">
[English](/project-1/logical-split-disk-embodiment/) | [中文](/project-1/logical-split-disk-embodiment-zh/)
</div>

## **A.前言**
今天，我想分享一个源自我多年来执念的成果：一个我亲手打造的USB控制器。它只做一件事，但这件事我认为至关重要——将存储设备的控制权（包括设备的基本信息、每一个扇区的读写权限和数据）**从主机手中优雅地收回，并交还给设备的主人** 。
而这个过程最直观的结果，就是能将一个普通的U盘或SD卡，变为按照需要切割为多个在主机视角上彼此独立的磁盘。在我的原型机上，实现了4个独立的磁盘。


<iframe src="//player.bilibili.com/player.html?aid=115314419308382&bvid=BV13ZxpzAEiJ&cid=32830916522&page=1&high_quality=1&danmaku=0&autoplay=0" allowfullscreen="allowfullscreen" width="100%" height="500" scrolling="no" frameborder="0" sandbox="allow-top-navigation allow-same-origin allow-forms allow-scripts"></iframe>
<p style="text-align:center">(简要演示视频：基本介绍和在Windows、macOS、Linux、iOS/iPadOS、Android平台的免驱动兼容性测试，完整演示视频请看文章最后)</p>

## **B.一切始于一个问题:我们应该具有对存储设备的完全控制权**
我们已经习惯了这样的流程：将U盘或SD卡等存储设备插入电脑，然后由操作系统完全接管。我们成了“第三方”，如果需要对存储设备进行管理，只能使用操作系统提供的上层工具去请求分区或在系统上安装安全软件。但这种模式意味着我们放弃了对硬件底层的控制权，不得不依赖日益复杂的主机端软件去解决那些本应由硬件来完成的任务——诸如数据隔离、多系统启动、安全防护等。这些软件方案既无法实现真正的控制，也称不上优雅，在某些场景下更是难以实施（例如无法在他人电脑上安装安全软件）。因此，我觉得有必要夺回对存储设备的真正控制权，便着手设计一个 **‘硬件代理’** ——USB控制器，用于管控主机（Host）发送给存储设备的指令。
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-7.png){: width="80%" style="display:block;margin:0 auto;"}
<p style="text-align:center">(失去控制权的一种场景:存储空间被主机整体访问，存储设备的所有人无法按需控制存储空间向计算机连接)</p>
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-7-1.png){: width="80%" style="display:block;margin:0 auto;" }
<p style="text-align:center">(失去控制权的直接结果是：即使可使用存储空间未满的情况下，为了应对不同场景的需求，仍需要使用多个不同的U盘)</p>

## **C.技术核心:控制每一个扇区，实施绝对的硬件主权**
在研究如何控制主机向存储设备发送指令的过程中，我发现了许多有趣的应用场景，其中一种我称为 **‘逻辑分盘’（Logical Split Disk，LSDisk）**。它与传统分区完全不同，因为它工作在更底层的层面，可以将原有的存储介质按照需要切割为N个独立的存储介质，不同逻辑硬盘是完全独立的设备，每块盘都有自己独立的LBA0（即主引导记录MBR；若使用GPT（GUID分区表），则为保护性MBR），并按照需要选择性地接入到主机（Host，指电脑等设备）。
其原理可概括为在USB设备枚举（识别）阶段发生的 **两个关键底层动作** ：
### 1. **定义边界**
当主机通过底层总线发出“你是谁？你有多大？”的查询指令时（例如SCSI（小型计算机系统接口） READ CAPACITY (10)命令），我的控制器会将其拦截。它不会传递NAND闪存（U盘里的存储芯片）的真实物理容量，而是根据我通过物理拨码开关预设的逻辑边界，返回一个精确自定义的容量值。对于主机而言，它完全不知道‘边界’之外还存在更大的物理存储空间。
### 2. **扇区控制**
当主机发出任何读写（I/O）指令（例如SCSI READ(10)/WRITE(10)命令）时，控制器会再次拦截。它像一个严格的硬件守门人，会校验每一个指令请求访问的逻辑块地址（LBA，可简单理解为扇区的编号），并将其映射并限定在当前激活的‘逻辑分盘’于物理闪存上的扇区范围内。
通过这种方式，在主机看来，每一个逻辑分盘都变成了一个完全独立、边界清晰的物理设备。那些未被选中的扇区并非被软件刻意隐藏，而是 **从始至终对主机不可见**。

## **D.可视化技术细节**
### **1.通常的情况**
以一张256GB容量的SD卡为例。当主机连接到一块标准256GB的存储设备时，首先会查询其总容量。主机从存储设备收到的回答是：256GB。从此主机便认定有权读写从地址0到256GB空间最后一个扇区的任意位置，对设备的访问完全不受限制。
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-1.png){: width="100%" }

### **2.逻辑分盘的情况**
逻辑分盘技术从根本上改变了这种交互方式。通过在主机和存储介质之间插入一层控制设备（即硬件代理），同一个256GB驱动器呈现给主机时不再是单一整体，而是变成了四个大小不同、彼此独立的磁盘——例如一个128GB、一个64GB和两个32GB的驱动器。此时主机交互的是 **虚拟化的硬件层**，而非物理介质本身。
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-2.png){: width="100%" }

### **3.选择逻辑分盘1**
假设用户选择了磁盘1。当主机向设备查询容量时，控制器会拦截该请求，不报告完整的256GB，而是仅回应“128GB”。主机因此只认为存在128GB空间，并将所有读写操作严格限定在这一边界内，对于剩余的存储空间则一无所知。
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-3.png){: width="100%" }
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-3-1.jpg){: width="100%" }

### **4.选择逻辑分盘2**
当用户选择磁盘2后，主机再次查询容量，这一次控制器报告“64GB”。随后主机尝试写入它认为的新磁盘的起始扇区（逻辑地址0），控制器会拦截该命令并在后台透明地重映射地址——通过在地址上添加偏移量，将写操作指向64GB分区实际起始的物理扇区（本例中即紧接第一个128GB分区之后的位置）。主机仍然停留在自己的逻辑视角中，完全不知晓幕后发生了物理地址的转换。
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-4.png){: width="100%" }
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-4-1.png){: width="100%" }

### **5.选择逻辑分盘3**
这个原理对每个逻辑磁盘都同样适用。如果选择了磁盘3，控制器会报告32GB的容量。当主机写入其逻辑0号扇区时，控制器会应用必要的偏移量——在这个例子中是128GB加上64GB——以确保数据被写入介质上正确的物理位置。从主机的角度来看，它只是在与一个标准的32GB驱动器进行交互。
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-5.png){: width="100%" }
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-5-1.jpg){: width="100%" }

### **6.选择逻辑分盘4**
最后，同样的过程也适用于磁盘4。控制器呈现一个32GB的驱动器，并管理所有的I/O，将其重定向到物理介质的最后一个分区。通过这种优雅的底层SCSI命令拦截与重定向技术，一个物理设备能够呈现为多个彼此完全隔离的独立硬件驱动器。
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-6.png){: width="100%" }
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-6-1.jpg){: width="100%" }

## **E.从理念到现实:一个通用、轻量且强大的控制器**
为将这一想法付诸现实，我将其实现为一个通用的USB控制器适配器。它不仅注重功能实现，也尽量保持设计的简洁与优雅。
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-8.png){: width="100%" }
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-9.png){: width="100%" }

如开始的视频所示，这个控制器已经实现了我最初的一些设想，并带来了一些独特的能力：
### **1.广泛的兼容性与通用性** 
它不仅与操作系统无关（ **无需额外驱动即可在Windows、macOS、Linux、iOS/iPadOS、Android上原生运行**），而且也不依赖特定的存储设备类型。无论是传统的U盘、SD读卡器，还是移动固态硬盘（SSD），只要是能够被控制器接口读写的存储设备，都可以通过它获得 **扇区级的控制权**。 进一步来说，由于它真正运行在块设备这一层，对上层文件系统完全不感知。正如演示视频所示，你可以在这些独立磁盘上任意使用任何文件系统——从FAT32到APFS、EXT4，甚至可以在其中安装完整的操作系统（注：这也是控制指令所带来的有趣效果之一，在很低复杂度的条件下实现了极高的兼容性，而且并未对不同操作系统做特殊适配）。
### **2.从主机视角的多个‘物理’级的多个磁盘** 
你可以将一块物理存储设备变成4个（或更多）在逻辑上完全独立的硬盘。在演示视频中，我把一张SD卡划分出了4个不同的逻辑分盘。这4个‘硬盘’在同一物理存储卡上完美共存，但主机却无法以任何方式察觉它们彼此的存在。对U盘的分盘演示也是同理。
### **3.透明的硬件加密** 
USB控制器能够识别主机发送给存储设备的所有读写（I/O）指令（例如SCSI READ(10)/WRITE(10)）。它将所有写入存储介质的数据实时加密后再写入介质，并将从介质读出的数据实时解密后再返回给主机。整个过程对用户和主机而言是完全透明的。如视频所示，将从该控制器上取出的SD卡插入普通读卡器时，其中的数据无法被读取——显示的只是不可识别的乱码。这意味着控制设备本身相当于密钥，可以与SD卡、U盘等存储介质 **分开保管**。即使介质丢失或被盗，只要密钥（控制设备）不落入他人之手，数据的安全性仍能大幅提升。这一点有别于操作系统层的软件加密方案——后者的密钥容易在计算机内存中泄露，而硬件密钥则不存在此隐患。（注：这也是控制指令带来的另一项有趣效果：以极低的复杂度实现了存储数据的透明硬件加解密，全程无需主机或用户干预）
### **4.对普通U盘进行写保护** 
当启用写保护功能后，USB控制器会向操作系统声明设备处于只读模式，操作系统随即在驱动层将其标记为只读设备。例如，当主机发送SCSI MODE SENSE(10)查询设备特性时，控制器返回只读标志，此时该磁盘在操作系统的右键菜单中将不再出现‘新建’‘删除’等选项，以及无法拖拽文件写入磁盘等操作。此时应用层软件将无法对该设备发出任何写入指令；即使主机上的程序强行尝试写入数据，USB控制器也会直接拒绝执行所有写入类I/O命令（例如拦截并拒绝SCSI WRITE(10)）。这种保护由 **硬件底层直接实现**，而非依赖操作系统的文件权限设置。例如，你可以随时对一块原本没有物理写锁的普通U盘启用硬件级写保护，也可以对某个逻辑分盘启用只读保护。（注：这也是控制指令带来的另一有趣效果，以很低的复杂度实现了某些昂贵的USB只读锁设备的功能）
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-10.jpg){: width="100%" }

### **5.独特的隐藏逻辑分盘** 
通过特定的拨码开关组合，可以唤醒在其他任意组合下都完全不可见的“隐藏磁盘”，以演示逻辑分盘的多用途场景。
### **6.实时掌控读写状态** 
通过LED指示灯，可以实时监控主机对存储设备的底层I/O指令：蓝灯表示空闲，绿灯闪烁表示正在读取（例如正在执行SCSI READ(10)命令）,红灯闪烁表示正在写入（例如正在执行SCSI WRITE(10)命令）。这个物理层的“监视器”让你能立刻判断主机的行为是否符合预期，增加了一层透明的信任。
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-11.jpg){: width="100%" }
![](/assets/img/project_1/logical-split-disk-embodiment/logical-split-disk-embodiment-12.jpg){: width="100%" }

### **7.面向 BadUSB 的额外防护**
该控制器的USB接口仅允许标准的大容量存储设备连接。任何试图将U盘伪装成非大容量存储设备的行为（例如将U盘伪装成键盘，通过HID接口注入恶意指令的典型 **BadUSB攻击**）都会被视为异常并被拒绝连接，同时设备上的LED会蓝红灯交替闪烁发出警示。（注：这也是控制指令带来的另一效果，使USB控制器天然地充当BadUSB攻击的检测与拦截器）
### **8.轻量化与效率**
逻辑分盘方案的核心算法对CPU、存储资源的占用极小，可以高效地运行在微控制器（MCU，微控制单元，一种小型计算机芯片）上。（本控制器采用一颗120MHz的MCU实现USB 2.0 High-Speed传输。）。这意味着该技术不仅可以做成外置适配器，还可以 **作为IP核直接集成**到任何需要高级存储管理的嵌入式系统中——例如具备逻辑分盘功能的U盘、硬盘或存储芯片等。对于汽车或工业设备的设计者而言，可以利用单颗大容量存储芯片，安全地虚拟出彼此独立的固件区、日志区、用户数据区和黑匣子区，从而显著降低硬件BOM成本和设计复杂度。设计者甚至可以通过OTA等方式，远程动态配置额外的独立存储分区，却几乎不增加主控芯片的负担。

<iframe src="//player.bilibili.com/player.html?aid=115288666283393&bvid=BV1pFnzzEExM&cid=32728416313&page=1&high_quality=1&danmaku=0&autoplay=0" allowfullscreen="allowfullscreen" width="100%" height="500" scrolling="no" frameborder="0" sandbox="allow-top-navigation allow-same-origin allow-forms allow-scripts"></iframe>
<p style="text-align:center">(完整演示视频：基本介绍和在Windows、macOS、Linux、iOS/iPadOS、Android平台的免驱动兼容性测试)</p>