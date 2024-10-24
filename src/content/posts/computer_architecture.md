---
title: 计算机体系结构学习
date: 2024-10-24
summary: 学习体系结构相关知识
category: 计算机体系结构
tags: [体系结构, 处理器, 数字IC]
comments: true
---

> 参考书籍："Computer Architecture A Quantitative Approach 6th Edition"
>
> "Modern Processor Design Fundamentals of Superscalar Processors"
>
> 《现代处理器设计——超标量处理器基础》

# Chapter 1 Memory Hierarchy System

# Chapter 2 Instruction Level Parallelism

## 2.1、Branch Prediction

为了避免分支执行错误从而有损流水线性能，循环展开是减少分支冒险的一种方法。此外，还可以通过分支预测的方法来降低分支造成的性能损失。

对于一个常见的分支指令来说，需要注意的要素有两点，分别是分支是否跳转以及分支跳转的地址。因此，对一条分支指令进行预测就需要对其是否跳转与跳转地址进行预测。常见的分支预测包括静态分支预测与动态分支预测。对于一个普通的处理器来说，其流水线深度并不深，因此一般采用静态分支预测，预测分支总是不执行，在流水线较浅的情况下，即使分支预测失败，也只会冲刷掉仅仅几条指令。

以MIPS五级流水线为例，在Decode阶段便可以得到Branch指令的结果，因此，即使预测错误，也只会冲刷一条指令。

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729494322466-ff336b75-2ccc-4b9d-8215-1c2cd2c1e505.png" alt="" referrerpolicy="no-referrer"> </div>

## 2.2、Before Branch Prediction

在对指令进行分支预测之前，需要注意的是如何判断从I-Cache中取出来的指令是否是分支指令。这里有两种方法，一种是类似于蜂鸟E203采用的：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1728109650966-0a541b16-dcae-42f5-a506-e390c1c4a49e.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

在取出指令后进行快速译码（Mini-Decode），接着将相对应的分支指令对应的PC值送到分支预测器中，就可以进行分支预测了，扩展到超标量处理器中如下所示：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729496168264-e73df00e-588f-4a4f-9593-d869917ab77a.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

如果采用这种方式进行分支预测，从开始取指令到分支预测取得结果，需要间隔许多个周期才能完成，从而降低了分支预测的准确度，造成了处理器性能的降低。为了解决这个问题，可以将分支预测的阶段尽可能的提前，让其在**得到PC地址时**，就可以进行分支预测，因此，下一周期也可以根据当前得到的预测结果进行取指：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729498952777-36d7b3ac-404d-4862-9d12-bec96bb7dc73.png" alt="" referrerpolicy="no-referrer"> </div>

## 2.3、分支的方向预测

正如上面介绍的，分支预测共分为方向预测与PC值预测两个大方向，这里首先从方向预测，即是否跳转介绍起。

如果在运行的过程中，不依赖历史信息与运行环境对分支进行预测，那么这种预测即**静态分支预测**。静态分支预测通常是在编译器中就已经做好了，不需要CPU的额外硬件支持了。常见的静态分支预测通常有**预测不跳转**、**预测跳转**以及使用最多的**<font style="color:rgb(25, 27, 31);">Backward Taken Forward Not Taken</font>**，这种预测的依据是在循环中，backward跳转占据了绝大多数的情况，因此，BTFNT的预测的准确率相较于前两种分支预测较高。

如果在运行的过程中，需要依赖历史信息以及运行环境对分支跳转进行预测，那么这种预测的准确率相较于静态分支预测将会大幅上升，这种预测被称为**动态分支预测**。最简单的动态分支预测即直接使用上次分支的结果，也被称为Last-outcome Prediction，方法如下所示：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729501121565-d7813e10-6b56-4ea8-92f8-1176655e0b0a.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

如果上次分支预测的结果是跳转，那么本次预测的结果也将是跳转。这样的方法相比于静态分支预测，一般都能取得更优的结果，但是当分支指令的方向发生变化时，依旧会引起分支预测失败。在更为糟糕的情况下，即分支变化较为频繁，跳转与不跳转相互交替的情况下，这种预测方法的失败率将变为100%，预测精度反而不如静态分支预测，为了防止这种情况出现，引入了基于两位饱和计数器的分支预测。

### 2.3.1、两位分支预测器

普通的1-bit分支预测器不能很好的处理分支连续变化的情况，为了弥补这一弱点，经常使用2-bit分支预测机制，也被称为双峰分支预测器（bimodal predictor）。在2-bit分支预测机制中，根据一条指令前两次执行的结果来预测本次的方向。当分支指令的方向总是朝着一个方向时，例如总是跳转/不跳转时，就会处于饱和状态，此时分支预测的正确率比较高，且只有发生两次预测失败才会改变预测的结果；但是当分支预测的方向总是变化时，状态机就无法保持饱和状态，这样正确率就会比较低。

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729580390873-0908bc54-0532-4160-98dd-56a71ffcc7cf.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

上图所示只是基于2-bit饱和计数器的众多预测方法中的一种，该状态机还有其他的实现方法，但效果基本差不多，因此不在此赘述。对于饱和计数器bit数的提升所带来的性能提升也微乎其微，使用3-bit饱和计数器时，预测的正确率的提升就很小了，仅为88.3%到97.0%。因此，对于基于历史信息的分支预测策略，采用2-bit饱和计数器是一种很好的选择。

由于分支预测是以PC值为基础进行的，正常来说，每一个PC值都应该对应一个两位分支预测器，因此，对于32位的PC值需要$ 2^{30}\times 2b $大小的存储器来存储这些值，显然在芯片中无法使用这么大的存储器。由于并不是所有的指令都是分支指令，所以一般以PC值的一部分来作为索引存储两位分支预测器的值：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729564681659-706c431e-2108-42e3-8954-0fe93d36793f.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

在上图中，PHT(Pattern History Table)是一个模式历史信息表，在其中存放着PC值一部分所对应的两位饱和计数器的值，k值越大，所能记录PC值对应的两位饱和计数器就越多，预测准确率也就也高，PHT所占用的存储空间也就越大。由于只采用了PC值的部分进行索引，当两条分支指令部分PC值相同时，就会互相产生干扰，这种情况称为别名（aliasing）。当两条别名所对应的分支方向相同时，此时不会产生负面的影响，两位饱和预测器仍然处于饱和状态，这种情况称为中立的别名（neutral aliasing）；如果两条别名对应的分支方向不同时，会导致两位饱和预测器一直无法处于饱和状态，降低分支预测的准确度，这种情况称为破坏性的别名（destructive aliasing）。因此，饱和计数器的个数直接影响着分支预测的结果。

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729567024925-969f7b67-add4-46f9-87e6-371f5e6bf910.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

在上图中，可以看到，在PHT的大小为2KB时，分支预测的准确率已经达到了93%，因此，在一般处理器的设计中，将PHT的大小设置为1KB~4KB即可。

此外，为了尽量避免别名问题，可以使用Hash算法对PC值进行压缩运算从而固定成一个长度较小的值，称为哈希值。利用这个Hash值进行索引，可以尽量避免别名的发生。

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729567857931-cfa47108-8e7a-4ce5-8550-f4949325e07e.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

两位饱和计数器需要根据分支指令的结果对计数器进行更新，为了防止更新错误，一般将更新时间放在流水线的提交阶段，保证更新一定是正确的。

这种基于两位饱和计数器的分支预测方法的正确率极限在98%，因此现代处理器不会直接使用这种方法。

### 2.3.2、基于局部历史的分支预测

上述两位饱和计数器尽管相较于单bit的分支预测器可以对连续变化的分支指令进行预测，但是结果往往不是很好。假设一条分支指令的执行顺序是taken、not taken、taken、not taken、taken……

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729579125608-90fc432c-586c-44ee-8e38-6d6994c3a595.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

如果分支的初始状态处于Weakly not taken，那么该分支预测器的状态将一直在两个不饱和状态之间进行切换，始终无法到达饱和状态，此时的分支预测正确率是0。如果分支初始状态位于Strongly not taken，那个分支预测器的状态将会在Strongly not taken和Weakly not taken之间不停转换，此时分支预测的准确率是50%。

如此低的分支预测准确率是不能被接受的，因此，为了对规律执行的指令进行预测，可以使用一个寄存器对该指令的历史信息进行记录，当历史信息很有规律的时候，就可以作为分支预测的一个条件，这样的寄存器称之为分支历史寄存器（Branch History Register,BHR），这种预测方法称为基于局部历史的预测方法。

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729581281697-358ef801-df2d-4803-8810-8e1f9d9c840d.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

这种方法也被称为自适应的两级分支预测（Adaptive Two-level Predictor），这种方法一开始由Yeh和Patt在1992年提出。

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729581707135-bdbf7f7a-b1af-45aa-a6b4-b98e9d41f342.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

在该方案中，每个分支地址同时索引BHSR与PHT，也就意味着每个BHSR唯一对应一张PHT。根据选出的BHSR的值（即一个pattern），再去PHT中进行索引寻找对应的两位饱和计数器，其内容作为预测算法的状态信息并生成一个预测结果。在分支执行完毕后，会根据执行的结果同时更新BHSR和PHT中的内容。

如果一个for循环的循环次数很小，称为loop closing，假设for循环的循环次数为4，则这个for循环被编译为汇编指令后，一定会有一条分支指令，这条分支指令就会有类似于TTTNTTTN...这样的行为。由于这个序列的循环周期是3，因此，任何宽度不小于3的BHR都可以对其进行预测，以一个四位的BHR为例：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729584724339-290f706b-4023-45a5-bf81-239f164a4a71.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

使用四位的BHR记录这个序列后，这个序列将会重复的出现1110、1101、1011和0111这四个状态且每个状态的计数器最后都会停在饱和状态。但是，这种方法所带来的浪费也是极大的，在PHT中，只有4个计数器被使用，其他12个计数器都是被浪费的。

上述情况都是在每条分支指令唯一对应一个BHR和PHT的条件下发生的，实际在应用中，不可能为每条指令都分配一个BHR和PHT。因此，与一般的两位饱和计数器相同，也可以采用使用PC值的一部分来寻址BHT和PHT。

Yeh和Patt提出BHR有两种实现方式，分别是统一(G)方式和单独(P)方式，统一方式只使用一个k位的BHR，记录最近k条分支指令的执行方向，分支指令可能不是特定的一条（可以是1到k条）；单独方式(perbranch)的BHR是一组k位的寄存器，由分支地址来索引。统一方式不是局部历史预测将在下一节全局预测中介绍。

PHT的实现有三种方式，分别是统一(g)方式、单独(p)方式和共享(s)方式。统一PHT使用一个表支持所有的分支预测，而多个单独的PHT可以为每条分支指令(p)或每个分支指令组(s)提供预测信息。使用较多的是**PAs**，即使用单独的BHR和共享的PHT的自适应(A)分支预测。

在实际中，BHT不可能照顾到每个PC，因此也可以使用PC的一部分来寻址BHT：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729586254601-96b142a8-0938-48a9-bf2a-af2f7a84e5a8.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

由于只使用了PC的一部分来寻址BHT和PHTs，因此，可能会出现重名的情况，使得分支预测的准确率有所下降。

由于使用PAs会使得PHTs占据的存储空间过大，为了更大限度的节约存储空间，会使用**PAg预测器**，即只有一个PHT。由于只有一个PHT，不需要PC对PHT进行寻址了，这样一定会有冲突情况的发生，如两条不同的分支指令对应两个BHR，但是分支情况是一样的，这样会对应到同一个PHT表项，导致冲突的发生。

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729586664174-85037d2b-956f-4ae8-8ef0-598e1695385f.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

但是这种预测方法最大程度的利用了PHT中的表项，可以尽量减少之前所说的PHT表项的浪费。

为了尽量避免上述冲突，可以对PC值和对应的BHR值进行一定的处理，并使用处理之后的值来寻址PHT：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729587411727-003903a9-bfcb-480c-8ee0-2f30612a3602.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

将PC值进行Hash处理后，得到了一个较短的值，再用此值来寻址BHT，从BHT中得到对应的BHR值后将其与PC值的一部分拼接，用得到的新值来寻址PHT，从而得到饱和计数器的值，也就是分支预测的结果，这样，可以解决上面所提到的冲突。

此外，还有多种方法将PC值和BHR的值进行处理，上述的位拼接是最简单的方法，但是效果不是很理想，有时会引入新的冲突。实际情况中，将PC值的一部分与BHR值进行异或运算会获得得到相对好一些的效果：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729587795754-ed65e9cb-b784-44c1-b01a-6b7d71249415.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

实际设计中会运用更为复杂的一些算法，以尽量避免PHT冲突情况的发生。

### 2.3.3、全局历史分支预测

之前我们说到，对一条分支指令的过去执行情况进行记录可以发现其自身的规律，这种一般适用于循环语句如下：

```c
for(i=0; i<m; i++)
{循环体}
```

那么什么时候一条指令的结果会与其之前的语句有关呢？我们观察如下代码：

```c
if(aa == 2)		/b1/
    aa = 0;
if(bb == 2)		/b2/
    bb = 0;
if(aa != bb)	/b3/
    {...}
```

通过观察上述代码，很容易发现，如果b1与b2都执行了，那么b3分支一定不执行，这种分支规律是不会在局部历史中体现的。因此，需要在分支预测的时候将b1与b2都考虑进去，这就是基于**全局历史的分支预测**。

如2.3.2节所述，每一条/组分支指令都会有其对应的BHR寄存器，而在基于全局历史的分支预测方法中，只需要一个寄存器来记录程序中所有的分支指令在过去的执行情况，这个寄存器被称为**全局历史寄存器(Global History Register,GHR)**。在现实中，一般不能对所有的分支指令进行记录，所以一般会使用一个k位的GHR来记录最近k条分支指令的运行结果。

在使用基于全局历史进行分支预测的方法时，每当要对一条指令进行分支预测时，就会使用当前GHR寄存器的值来预测，此时仍然需要借助于PHT，通过两位饱和计数器来捕捉GHR寄存器的规律。

基于全局预测的分支预测，最理想的情况是对每条分支指令都使用一个PHT，这样每条指令都会使用当前的GHR来寻址自身对应的PHT，也就是**GAs预测器**，预测方法如下所示：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729597872663-514ed4e5-fb89-44b3-894b-5ec3505f2a52.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

虽然这种情况会保证每个分支指令对应唯一的PHT，但是也会占据非常大的存储空间，因此，一般都会采用Hash算法将PC进行处理，得到位宽较小的值：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729598126510-653e1f60-cd5f-4c17-929e-50cc07c9cbcb.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

其实这种方法实际是基于局部历史预测的一种特殊情况，即将BHRs减少到了一个GHR，对所有分支指令的结果进行记录。同样，也可以将PHTs缩减为1个，即**GAg预测器**：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729598344829-c9b687d4-2a47-4dac-a9d9-4f41754c5b3e.png" alt="" referrerpolicy="no-referrer"> </div>

### 2.3.4、GSHARE预测器

全局分支预测也会像局部分支预测一样，因重名产生冲突。为了尽量避免冲突，1993年，Scott McFarling提出了一个非常高效的关联分支预测器，称为**gshare预测器**。通过将分支地址的j位和全局GHR的k位进行Hash计算，得出的结果位数为$ \max\{k,j\} $、索引$ 2^{\max\{k,j\}}\times 2 $位的PHT表，并选中一个分支预测符进行预测，如下所示：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729587313171-7195d3fd-de71-44c3-84e8-629ea03fae98.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

Gshare只使用了一个统一的k位BHSR和一个很小的PHT，就可以获得和其他关联分支预测器相当的预测精度。在DEC Alpha 21264中就使用了这种技术。

但是这种分支预测算法仍然不能保证对2.3.2节中所述的如TNTNTN的分支指令进行完美的预测，因为可能会受到其他分支指令的干扰，而且每次遇到这种情况，都需要对PHT进行重新训练，这样也降低了分支预测的精确度。如果能够根据实际情况来选用全局与局部分支预测，便可以很好的解决这种问题。

### 2.3.5、竞争预测器

由于之前介绍的两种分支预测：基于局部历史的分支预测与基于全局历史的分支预测各有优缺点，都有着局限性，因此需要设计一种自适应的分支预测器，根据不同的分支指令自动地选择这两种分支预测方法，Alpha 21264就使用了这种方法，称为竞争的分支预测(Tournament Predictor)。

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729609653990-a92d11ae-82eb-4c53-8679-e836ecefab77.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

如图所示，P1表示基于全局历史的分支预测方法，P2表示基于局部历史的分支预测方法。CPHT(Choice PHT)是由分支指令的PC值与GHR运算后来寻址的一个表格，类似于全局历史分支预测寻址的PHT。当其中一种分支预测方法两次预测失败，而同时另外一种分支预测方法两次预测成功时，会使状态机转移到使用另一个分支预测方法的状态：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729610269385-dccf540f-498f-4708-a8cc-6d810764336e.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

该状态机机制如上所述：

1. 当P1预测正确，P2预测错误时(1/0)，计数器减1；
2. 当P1预测错误，P2预测正确时(1/0)，计数器加1；
3. 当P1与P2预测结果一样时(1/1或0/0)，不管预测正确与否，计数器保持不变。

当计数器处于00状态时，会使用P1进行分支预测，当计数器处于11状态时，会使用P2进行预测，此时的预测准确率是比较高的。对于这种竞争的分支预测，经过一段时间的训练后，会根据两种分支预测方法的执行情况，选择P1或者P2作为自身的分支预测的结果。

```c
if(aa == 0)		/b1/
    a = 0;
if(bb == 0)		/b2/
    b = 1;
if(aa == bb)	/b3/
    c = 3;
```

当分支b1与b2都执行时，分支b3肯定是执行的，此时使用基于全局历史的预测是较为合适的。当分支b1与b2都不执行时，此时分支b3无法从全局上判断是否执行，对其使用基于局部历史的分支预测方法能取得更好的效果。

因此，对于同一条分支指令来说，即使是同一条分支指令，在GHR不同时也会使用不同的分支预测方式，此时将PC值与GHR寄存器进行Hash运算并将结果作为寻址CPHT的地址。

Alpha 21264处理器只使用了GHR作为CPHT寻址的地址，失去了每条分支指令地址信息，降低了分支预测的准确度：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729651214384-516b8cf5-90d1-472e-b425-a33c1b9f6ba5.png" alt="" referrerpolicy="no-referrer"> </div>

### 2.3.6、BI-MODE分支预测器

## 2.4、现代分支预测器

上面介绍的分支预测器都是基于两位分支预测器的不同变种，性能一般，难以胜任现代高性能处理器的任务。因此现代分支处理器采用的都是基于PPM和<font style="color:rgb(55, 58, 64);">Perceptron的分支预测算法。</font>

### 2.4.1、PPM分支预测

> " Analysis of Branch Prediction via Data Compression"

在介绍基于Partial Matching的分支预测器前，先介绍PPM预测器中使用到的Markov预测器。一个j阶的Markov预测器基于j位bit来预测下一个位，即一个简单的马尔可夫链。

j阶Markov预测器共有$ 2^j $个可能的模式，转移的概率与预测器处于该状态下之前跳转到1或0所观察到的频率一一对应。预测器通过记录在j位模式后的第j+1位跳转的次数来衡量发生跳转的频率。该链是在预测的同时建立的，因此，通常该链的某些模式不是完整的。

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729682292539-89741cf6-8861-4447-8582-61a26d7025e7.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

为了预测分支结果，只需要使用当前需要预测的位前面的j位作为索引来索引模式表中的一个模式，并找出对应跳转最为频繁的一个状态，即计数器最大的状态。如上图所示，如果当前序列为010101101时，下一个bit是0的概率为2/3，为1的概率为1/3，因此下一个bit预测为0。

注意0阶Markov预测器只是以输入序列的相对频率来预测下一个bit。

Prediction by Partial Matching（PPM）是一种通用的压缩/预测算法，在理论上被证明是最优的算法并已经应用于数据压缩和预取。PPM算法包括了一个估算字符概率的预测器和一个算术编码器，在分支预测应用中，暂时只用到预测器的功能。通过将分支结果的跳转与不跳转编码为1或0，可以预测下一位的值。

m阶PPM预测算法是基于m+1个Markov预测器的一种算法，具体实现如下所示：

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729683684134-3f646168-255a-4f57-b966-14fb4a358439.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

首先PPM使用需要预测的位的前m位在m阶Markov预测器中进行预测，如果能找到匹配的pattern，就将对应的结果作为输出；如果找不到pattern，会将转而使用前m-1位来搜索下一个m-1阶的Markov预测器。每次搜索失败都会导致pattern减少一位并向着下一阶的Markov预测器进行搜索，知道找到匹配的预测器并作出相应的预测。

预测器的更新将采用update exclusion策略，这代表我们只需要更新预测使用的预测器以及比其更高阶的预测器，不需要更新低阶预测器。

不难发现，Markov预测器与我们之前所学习的两级自适应分支预测器非常相似，它们的不同之处仅在于用于做历史寄存的信息不同以及选取的用于索引预测器的子集不同。

<div align="center"> <img src="https://cdn.nlark.com/yuque/0/2024/png/29081281/1729692355324-a72b8c45-f21a-482e-afb6-75a25bc2a535.png" alt="" referrerpolicy="no-referrer"> </div>
<br/>

如果都选取分支结果的最后m位，则这两种预测方法的索引地址相同，即第一级相同。在预测器的第二级，两级自适应分支预测使用了一个2bit饱和计数器用来判断当前预测状态，而Markov预测器则对每个结果使用一个频率计数器，因此需要两个计数器。这两个预测器实际上都是通过选取占多数的结果来进行分支预测，2bit饱和预测器中taken占多数则会保持在00状态，而在Markov中，如果接下来的结果是taken比较多，则1's的计数值比较多。

### 2.4.2、O-GEHL

### 2.4.3、TAGE

### 2.4.4、Perceptrons
