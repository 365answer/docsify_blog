# 从 51 到 I.MX6ULL，为什么芯片GPIO操作变得这么复杂

最近在给 I.MX6ULL 写代码的时候，我脑子里突然冒出几个问题：以前的51单片机操作 GPIO 那么简单，为什么现在变得这么复杂？作为芯片的设计人员，他们将这种流程复杂化的原因是什么？今天，我们就尝试着开开脑洞，为这些问题想一个答案。需要说明的是，本篇文章只是个人的一些不成熟想法，并不是严谨的科学论断，有错误的地方，欢迎大家留言讨论。


## 51单片机 GPIO 操作步骤

51 单片机的 GPIO 控制，只需要配置一个 GPIO 寄存器。



### GPIO 输出

如果要在 51 单片机的某个引脚输出高电平或低电平，直接将 GPIO 寄存器对应的位设置为 1 或者 0 即可。



### GPIO 输入

如果要读取引脚的输入值，需要 2 步：

- 将 GPIO 寄存器对应的位设置为 1
- 读取 GPIO 寄存器，提取出对应位的值



## I.MX6ULL GPIO 操作步骤



### GPIO 输出

- 配置时钟控制模块（Clock Control Module），确保 GPIO 的时钟处于使能状态。

- 配置 IOMUX 控制器（IOMUX Controller），使引脚工作于 GPIO 模式，并配置其上/下拉电阻、驱动能力等参数。

- 配置 GPIO 方向寄存器，使其处于输入模式。

- 读取 GPIO 数据寄存器或者 GPIO 引脚状态寄存器，获取引脚的值。



### GPIO 输入

- 配置时钟控制模块（Clock Control Module），确保 GPIO 的时钟处于使能状态。
- 配置 IOMUX 控制器（IOMUX Controller），使引脚工作于 GPIO 模式，并配置其上/下拉电阻、驱动能力等参数。
- 配置 GPIO 方向寄存器，使其处于输出模式。
- 将 GPIO 数据寄存器对应的位配置为0或者1，使GPIO输出低电平或者高电平。
- 如果需要，可以配置 GPIO Loopback，这样就可以从 GPIO 引脚状态寄存器获取到当前引脚的实际电平状态。



## 差异对比

通过比较发现，51单片机 与 I.MX6ULL 的 GPIO 操作主要有如下几个差别：

- 多了时钟控制模块。
- 多了 IOMUX 控制器。
- 有单独的寄存器控制 GPIO 方向。
- 多了 GPIO 引脚状态寄存器，输出模式下可配置 Loopback，监控引脚的实际电平状态。



### 时钟控制模块（Clock Control Module）
对于芯片来说，引入时钟控制模块的一个重要目标就是控制芯片的功耗。

与51单片机相比，现在的芯片内部集成的模块要多了N倍，而用户却要求芯片的功耗越低越好。

在这种需求之下，我们的应对方案也很直接：某些当前不用的模块，直接关闭它的电源或者时钟。因此，时钟控制模块应运而生。有了它，我们可以根据需求开启或者关闭芯片内部各个模块的时钟，从而降低芯片整体功耗。



### IOMUX控制器（IOMUX Controller）
IOMUX 是芯片功能越来越丰富的必然结果。试想一下，如果芯片为每个 UART、IIC、SPI、LCDC、PWM、GPIO、CAN、DDRC 等模块都单独分配独自使用的引脚，那这颗芯片估计需要上千个引脚了。庞大的引脚数量不仅会导致芯片面积增大、PCB布线困难等问题，还会造成很大程度的浪费。毕竟，这些模块并不一定会同时使用。

芯片设计者提供的解决方案就是：多个模块共用一些引脚，然后通过 IOMUX 的配置决定这些引脚当前为哪个模块工作。对于使用者，我们在芯片选型时只要确保我们使用的这些模块之间引脚没有相互冲突即可。

处理引脚复用功能之外，IOMUX 一般还会集成引脚特性配置功能。通过这个功能，我们可以配置引脚的上/下拉电阻、驱动能力等参数特性。



### GPIO 方向寄存器（GPIO Direction Register）
GPIO 方向寄存器的出现比较好理解。独立的功能对应独立的寄存器，降低耦合。

此外，从软件设计角度来说，GPIO 方向寄存器的引入也提高了读取 GPIO 引脚电平状态的效率。51单片机读取输入引脚的电平状态时，需要两次寄存器操作（先写一次 GPIO 数据寄存器，再读 GPIO 数据寄存器获取引脚电平），而 I.MX6ULL 只需要在初始化时将 GPIO 方向寄存器配置为输入模式，之后每次获取引脚电平状态都只需要一次寄存器读操作。



### GPIO 引脚状态寄存器（GPIO Pad Status Register）
GPIO 处于输入模式时，GPIO 引脚状态寄存器几乎没啥作用，因为它和 GPIO 数据寄存器的值是完全一样的。但是，GPIO 处于输出模式时，GPIO 引脚状态寄存器却能够帮我们检测引脚的实际电平状态。

举个栗子，我们想让 GPIO 引脚输出高电平，但由于 PCB 焊接问题，该 GPIO 引脚对地短路了。这个时候，我们从软件角度就可以发现 GPIO 数据寄存器的值是 1，但是 GPIO 引脚状态寄存器的值是 0，从而推断出这个 GPIO 引脚电路有问题。
