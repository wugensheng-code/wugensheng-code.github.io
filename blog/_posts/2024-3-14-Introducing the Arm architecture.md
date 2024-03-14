---
layout: post
title: Introducing the Arm architecture
date: 2023-12-18 10:16 -0500
image:
  path: /assets/img/blog/Introduce-the-Arm-architecture/micro-architecture.png
description: >
  Introducing the Arm architecture
---
# Introducing the Arm architecture

## 目录

-   [架构（architecture）是指什么](#架构architecture是指什么)
-   [系统架构（System architecture）](#系统架构System-architecture)
-   [架构（Architecture）和微架构（micro-architecture）](#架构Architecture和微架构micro-architecture)
-   [其他 Arm 架构](#其他-Arm-架构)
-   [常见架构术语](#常见架构术语)
    -   [PE Processing Element](#PE-Processing-Element)
    -   [IMPLEMENTATION DEFINED](#IMPLEMENTATION-DEFINED)
    -   [UNPREDICTABLE  和 CONSTRAINED UNPREDICTABLE](#UNPREDICTABLE--和-CONSTRAINED-UNPREDICTABLE)


## 架构（architecture）是指什么

当使用术语架构（architecture）时，指的是功能规范。在Arm架构中，指的是处理器的功能规范。一个架构指明了处理器的行为方式，例如有什么指令，指令的功能。架构是软件和硬件之间的契约。描述了软件可以依赖的硬件提供的功能，有些功能是可选的。

ARM架构规范了如下内容：

| -        | 描述                                                     |
| -------- | ------------------------------------------------------ |
| 指令集      | 指令的功能&#xA;&#xA;该指令在内存中的表示方式（其编码）                       |
| 寄存器集     | 寄存器数量&#xA;&#xA;寄存器的大小&#xA;&#xA;寄存器的功能&#xA;&#xA;寄存器初始状态 |
| 异常模型     | 不同级别的特权&#xA;&#xA;异常的类型&#xA;&#xA;从异常中获取或返回时会发生什么情况      |
| 内存模型     | 如何对内存访问进行排序&#xA;&#xA;缓存的行为方式、软件必须执行显式维护的时间和方式          |
| 调试、跟踪和分析 | 如何设置和触发断点&#xA;&#xA;跟踪工具可以捕获哪些信息以及以何种格式捕获               |

## 系统架构（System architecture）

系统不仅包括一个处理器内核。Arm 还提供了描述包含处理器的系统的要求的规范，如下图所示：

![](/assets/img/blog/Introduce-the-Arm-architecture/System-architecture.png)

规范是软件兼容性的基础。根据规范构建硬件意味着可以编写与之匹配的软件。根据规范编写软件意味着它可以在兼容的硬件上运行。Arm 架构是第一层，通过指令集架构 （ISA） 兼容性为软件提供通用的编程器模型。

基本系统体系结构 （BSA） 规范描述了系统软件可以依赖的硬件系统体系结构。BSA 涵盖了处理器和系统架构的各个方面，例如中断控制器、定时器和操作系统所需的其他常见设备。这为标准操作系统、虚拟机管理程序和固件提供了一个可靠的平台。

BSA 广泛适用于不同的市场和用例。其他标准可以建立在 BSA 的基础上，以提供针对特定市场的标准化。例如，服务器基本系统体系结构 （SBSA） 是针对服务器的 BSA 的补充。SBSA 描述了服务器操作系统的硬件和功能要求。该规范包含一组级别，这些级别随着 CPU 架构的发展而越来越详细地记录硬件功能。

基本引导要求 （BBR） 规范涵盖了基于 Arm 架构以及操作系统和虚拟机管理程序可以依赖的系统的要求。此规范建立了固件接口要求，如 PSCI、SMCCC、UEFI、ACPI 和 SMBIOS。

BBR 还提供了针对特定用例的配方，例如：

-   SBBR：指定 UEFI、ACPI 和 SMBIOS 要求，以启动通用、现成的操作系统和虚拟机管理程序，如 Windows、VMware、RHEL、Oracle Linux 和 Amazon Linux。SBBR 还支持其他操作系统，如 Debian、Fedora、CentOS、SLES、Ubuntu、openSUSE、FreeBSD 和 NetBSD。
-   EBBR：与 EBBR 规范一起指定 UEFI 要求来引导通用的、现成的操作系统，如 Debian、Fedora、Ubuntu、openSUSE，并为垂直集成的操作系统平台提供好处。
-   LBBR：指定 LinuxBoot 固件的潜在要求，以启动某些超大规模提供商使用的操作系统。

## 架构（Architecture）和微架构（micro-architecture）

架构不会表明特定处理器是如何构建或工作的，处理器的构建和设计称为微架构，微架构表明了特定处理器的工作方式。

微架构包括了一下内容：

-   Pipeline的长度和布局
-   cache的大小和数量
-   单个指令的指令周期
-   实现了哪些可选功能

例如，Cortex-A53 和 Cortex-A72 都是 Armv8-A 架构的实现。这意味着它们具有相同的架构，但它们具有非常不同的微架构，如下图和表所示：

![](/assets/img/blog/Introduce-the-Arm-architecture/micro-architecture.png)

| Architecture | Cortex-A53                                                                                    | Cortex-A72                                                                                     |
| ------------ | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Target       | Optimized for power efficiency                                                                | Optimized for performance                                                                      |
| Pipeline     | 8 stages&#xA;&#xA;In-order                                                                    | 15+ stages&#xA;&#xA;Out-of-order                                                               |
| Caches       | L1 I cache: 8KB - 64KB&#xA;&#xA;L1 D cache: 8KB - 64KB&#xA;&#xA;L2 cache: optional, up to 2MB | L1 I cache: 48KB fixed&#xA;&#xA;L1 D cache: 32KB fixed&#xA;&#xA;L2 cache: mandatory, up to 2MB |

架构兼容的软件无需修改即可在 Cortex-A53 或 Cortex-A72 上运行。

### 其他 Arm 架构

Arm 架构是最著名的 Arm 规范，但它并不是唯一的规范。Arm 对构成现代片上系统 （SoC） 的许多组件具有类似的规格。下图提供了一些示例：

![](/assets/img/blog/Introduce-the-Arm-architecture/other-Arm-architecture.png)

-   通用中断控制器：通用中断控制器 （GIC） 规范是用于 Armv7-A/R 和 Armv8-A/R 的标准化中断控制器。
-   系统内存管理单元：系统内存管理单元（SMMU 或有时是 IOMMU）为非处理器主服务器提供转换服务。
-   通用计时器：通用计时器为系统中的所有处理器提供通用参考系统计数。这些计时器提供用于操作系统调度程序滴答等功能。通用计时器是 Arm 体系结构的一部分，但系统计数器是系统组件。
-   服务器基础系统体系结构和可信基础系统体系结构：服务器基础系统体系结构 （SBSA） 和可信基础系统体系结构 （TBSA） 为 SoC 开发人员提供系统设计指南。
-   高级微控制器总线架构：高级微控制器总线架构 （AMBA） 系列总线协议控制基于 Arm 的系统中组件的连接方式以及这些连接上的协议。

## 常见架构术语

### PE Processing Element

PE在Arm架构中指具有自己的程序计数器并可以执行程序的任何东西。例如：

-   Cortex-A8 是一款单核、单线程处理器。整个处理器都是 PE。
-   Cortex-A53 是一款多核处理器，每个内核都是一个单线程。每个内核都是一个 PE。
-   Cortex-A65AE是一个多核处理器，每个内核有两个线程。每个线程都是一个 PE。

### IMPLEMENTATION DEFINED

实现定义（IMPLEMENTATION DEFINED）的功能由特定的微架构来定义。如cache的大小等。

### UNPREDICTABLE  和 CONSTRAINED UNPREDICTABLE

UNPREDICTABLE 和 CONSTRAINED UNPREDICTABLE用于描述软件不应做的事情，当某些事情不可预测时，处理器的行为是不确定的。

***

参考文档：

-   [https://developer.arm.com/documentation/102404/0201/?lang=en](https://developer.arm.com/documentation/102404/0201/?lang=en "https://developer.arm.com/documentation/102404/0201/?lang=en")
