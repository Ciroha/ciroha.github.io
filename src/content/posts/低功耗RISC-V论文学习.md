---
title: 低功耗RISC-V论文学习
date: 2024-10-15
summary: 介绍低功耗处理器的相关论文
category: 低功耗处理器
tags: [RISC-V, 处理器, 低功耗, 数字IC]
comments: true
---

# 介绍

常见的用于嵌入式的处理器主要有面向极低功耗场景和面向兼具性能与功耗的场景。当面向极低功耗进行设计的时候，就需要尽量降低处理器的频率以减少功耗，所以这种情况一般会采用较低级数的流水线，如两级流水线。另一方面，如果需要兼具性能与较低功耗，则大部分会采用四级的流水线。

# RV16: An Ultra-Low-Cost Embedded RISC-V Processor Core

![](../../assets/images/低功耗论文/1.png)

两级流水线、datapath用的是16位的，面积小且功耗小。可配置支持指令集。

# HAMSA-DI: A Low-Power Dual-Issue RISC-V Core Targeting Energy-Efficient Embedded Systems

![](../../assets/images/低功耗论文/2.png)

本文基于开源的RI5CY处理器核进行拓展，通过改进寄存器文件、预取指和加入L0cache，将单发射改为了双发射。提高了性能的同时，功耗也没有提高太多。

Issue1为主发射队列，Issue2为副发射队列。主发射队列与原先的RI5CY处理器基本上相同，副发射队列为了节约面积与功耗删除了一些功能，如LSU、ALU中的MAC功能等。此外，还需要遵守一些规则：

1. PC指向的指令发射给主队列，下一条指令经过IDU，检测该指令是否为Load/Store指令、是否与上一条指令有RAW依赖等。
2. 如果检测到RAW依赖，就会阻塞当前队列直到解除。如果没有依赖且可以在Issue2中正常运行，则会发射到IF/ID寄存器中。
3. 多口寄存器文件支持3W5R为两个发射队列同时提供数据。
4. 如果Issue1中的指令是分支指令且被执行，那么会向Issue2队列发送KILL信号阻止Issue2写回。

## RI5CY处理器核

[OpenHW Group CV32E40P User Manual — CORE-V CV32E40P User Manual v1.8.3 documentation](https://docs.openhwgroup.org/projects/cv32e40p-user-manual/en/latest/index.html)

RI5CY的微架构如下：

![](../../assets/images/低功耗论文/3.png)

指令预取单元主要有两个作用，一是在多处理器核同时从指令缓存中取指时，提高取指性能；二是在Icache边缘进行非对齐访问时，如果没有Prefetch，则会需要两个时钟周期才可取得指令，有preftch buffer的话，则可以在一个周期内取出，该情况见下图：

![](../../assets/images/低功耗论文/4.png)

## L0-cache的设计逻辑

L0 cache的适用于双发射的取指判断策略，Issue1作为主队列必须发射，Issue2可为空：

![](../../assets/images/低功耗论文/5.gif)

## Register file的选择

![](../../assets/images/低功耗论文/6.png)

由于如果直接将寄存器文件复制到第二个队列，会导致有两个寄存器文件，会出现存储一致性的相关问题。因此，为了简化设计，直接将寄存器设计为一个5R3W的寄存器文件。其中，Issue1队列用到的是3R2W，Issue2队列用到的是2R1W。Cross-Forwarding Logic是用于处理RAW冲突等的前递逻辑。

## Issue Decision Unit

IDU负责将对第二条指令进行检测、取指、检查依赖，如果没有相关问题，则发送给Issue2队列。

需要注意的是：

1. IDU需要负责取出两条指令，第一条是当前PC值指向的指令，第二条则根据当前指令的类型进行相应的判断是PC+2还是PC+4。
2. 如果当前的PC值所对应的Cache的offset值（PC[3:1]）小于5，则意味着即使当前指令与下一条指令都为32bit指令，也只会在当前Cacheline取指。同样，如果大于等于5，指令可能在同一个Cacheline取指、可能跨Cacheline取指。

## 双发射的执行模块

以一个点乘为例对执行模块进行介绍：

![](../../assets/images/低功耗论文/7.png)

红色框内的是四个展开的循环迭代操作，展开的操作基于双发射的硬件特性，将仅在Issue1中支持的访存操作和Issue2所支持的操作进行“交叉”发射以尽最大可能利用双发射。通过将点乘指令所需要的两个操作数提前两个周期取出，最终可以达到每个周期执行2个8bit的MAC操作。

## 板级实现和性能分析

为了对处理器核的性能、功耗、面积等进行测算，使用了CoreMark、Embench中的Nettle-AES和Dot Product这三个程序作为基准程序，分别对比了各种情况下（L0-Cache、Dual Issue）的功耗与性能。

![](../../assets/images/低功耗论文/8.png)

# A High-Performance Core Micro-Architecture Based on RISC-V ISA for Low Power Applications

## 微架构总览

![](../../assets/images/低功耗论文/9.png)

该处理器的微架构总体上被分为了四个部分以兼顾IPC和频率的平衡，这四个阶段分别为IF、ID、IE和RW。大多数的指令可以再这四个阶段内完成，在设计中也针对具体的指令进行了执行过程的优化，例如JAL指令是无条件跳转指令，不需要对寄存器进行写回操作，所以不需要最后一个阶段；Load/Store指令则需要在寄存器写回（RW）阶段前额外加入访存阶段。在IF阶段采用了带有历史表的2bit动态分支预测，可以满足精度和硬件复杂度之间的平衡。

## 硬件乘法器

常见的硬件乘法方法有`Baugh and Wooley`、`Booth`、`Vedic`、`Dadda`等，本文对以上常见的硬件乘法器算法进行了综合实现并分别计算了它们的功耗、面积等结果。

### Baugh and Wooley乘法器
