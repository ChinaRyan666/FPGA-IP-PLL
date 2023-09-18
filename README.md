# FPGA IP核使用教程——以PLL实验为例
# 0 致读者


此篇为专栏 **《FPGA零基础教程》** 的第一讲，区别与市面上先讲解“FPGA是什么、点灯”等教程，此专栏力求最快速让读者学会如何开发FPGA，即**使用FPGA做可以落地的项目**，大家可以先去此专栏置顶 **《FPGA零基础入门学习路线》** 来做最基础的扫盲。

本专栏内容基于正点原子资料和笔者实际开发经验撰写，将会详细讲解每一个FPGA相关实验的全流程，**诚挚**地欢迎各位读者在评论区或者私信我交流！

**PLL 的英文全称是 Phase Locked Loop，即锁相环，是一种时钟反馈电路，具有时钟倍频、分频、相位偏移、 可编程占空比和优化抖动等功能。** 为了方便我们使用这些功能，Xilinx 提供了 PLL IP 核。本文我将通过一个简单的例程来向大家介绍一下 PLL IP 核的使用方法。

本文的工程文件**开源地址**如下（基于ZYNQ7020，大家 **clone** 到本地就可以直接跑仿真，如果要上板请根据自己的开发板更改约束即可）：

> [https://github.com/ChinaRyan666/FPGA-IP-PLL](https://github.com/ChinaRyan666/FPGA-IP-PLL)


# 1 实验任务

本文实验的任务是使用 FPGA 开发板输出 **4 个不同频率或相位的时钟**，四个时钟分别为一个**倍频时钟（100MHz）**，一个**倍频后相位偏移 180 度的时钟（100MHz）**，一个**与系统时钟相同的时钟（50MHz）** 和 **一个分频时钟（25MHz）**，并在 Vivado 中进行仿真以验证结果是否正确。
# 2 PLL IP核原理讲解

锁相环（PLL）作为一种反馈控制电路，其特点是利用**外部输入的参考信号**来**控制环路内部震荡信号的频率和相位**。因为锁相环可以实现输出信号频率对输入信号频率的自动跟踪，所以锁相环通常用于闭环跟踪电路。

锁相环在工作的过程中，当输出信号的频率与输入信号的频率相等时，输出电压与输入电压保持固定的相位差值，即**输出电压与输入电压的相位被锁住**，这就是锁相环名称的由来。

锁相环拥有强大的性能，可以对输入到 FPGA 的时钟信号进行任意**分频、倍频、相位调整、占空比调整**，从而输出一个期望时钟；除此之外，在一些复杂的工程中，哪怕我们不需要修改任何时钟参数，也常常会使用 PLL 来优化时钟抖动，以此得到一个更为稳定的时钟信号。正是因为 PLL 的这些性能都是我们在实际设计中所需要的，并且是通过编写代码无法实现的，所以 PLL IP 核才会成为程序设计中最常用 IP 核之一。

需要注意的是 Xilinx 中的 PLL 是模拟锁相环，其**优点**是输出的稳定度高、锁定时间较短，相位连续可调；**缺点**是在极端环境（例如极端高/低温环境，高磁场强度等）下容易失锁。

Xilinx7 系列器件中的时钟资源包含了时钟管理单元 CMT（全称 Clock Management Tile，即时钟管理单元），每个 CMT 由一个 MMCM（全称 Mixed-Mode Clock Manager，即混合模式时钟管理）和一个 PLL（全称 Phase Locked Loop，即锁相环）组成，xc7z020 芯片内部有 4 个 CMT， xc7z010 芯片内部有 2 个 CMT，为设备提供强大的系统时钟管理以及高速 I/O 通信的能力。 接下来我们讲解一下 **MMCM 和 PLL 各自的含义以及两者的区别**。


+ **PLL：** 为锁相回路或锁相环，用来统一整合时钟信号，使高频器件正常工作，如内存的存取数据等。 PLL 用于振荡器中的反馈技术。

+ **MMCM（混合模式时钟管理）：** 是基于 PLL 的新型混合模式时钟管理器，实现了最低的抖动和抖动滤波，为高性能的 FPGA 设计提供更高性能的时钟管理功能。

MMCM 是一个 PLL 上加入 DCM 的一部分以进行精细的相移，也就是说 **MMCM 在 PLL 的基础上加上了相位动态调整功能**，又因为 PLL 是模拟电路，而动态调相是数字电路，所以 **MMCM 被称为混合模式**， MMCM 相对 PLL 的优势就是相位可以动态调整，但 PLL 占用的面积更小，而在大部分的设计当中大家使用 MMCM 或者 PLL 来对系统时钟进行分频、倍频和相位偏移都是完全可以的。 

接着我们讲一下 PLL 的工作原理，首先我们画出一个大致的结构模型示意图，如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/fc5cd301e5be4f7f80e730f3719a353a.png)

由上图可以看出**PLL 的工作流程**如下： 

**1.** 通过 PFD(全称： Phase-Frequency Detector，即鉴频鉴相器)对**参考时钟（ref_clk）频率**和**需要比较的时钟频率（即上图中的输出时钟：pll_out）** 进行 **对比**。

**2.**  PFD 的输出连接到 LF（全称： Loop Filter，即环路滤波器） 上，用于控制噪声的带宽，滤掉高频噪声，使之趋于一个稳定的值，**起到将带有噪声的波形变平滑的作用**。如果 PFD 之前的波形抖动比较大，经过环路滤波器后抖动就会变小，趋近于信号的平均值。

**3.** 经过 LF 的输出连接到 VCO（全称： Voltage Controlled Oscillator，即压控振荡器） 上， **LF 输出的电压可以控制 VCO 输出频率的大小**， LF 输出的电压越大 VCO 输出的频率越高，然后将这个频率信号连接到PFD 作为需要比较的频率。

**如果参考时钟输入的频率和需要比较的时钟频率不相等，该系统最终实现的就是让它们逐渐相等并稳定下来。** 例如参考时钟的频率50MHz，经过整个闭环反馈系统后，锁相环对外输出的时钟频率也是50MHz。

这里我们用一个简单的生活行为来带大家理解一下**锁相环的工作流程**，以我们开车为例：

![在这里插入图片描述](https://img-blog.csdnimg.cn/6e7831322c394f518bf91c1d11a78455.png)

**参考时钟**可以类比为**道路的方向**，**需要比较的时钟**可以类比为**汽车的行驶方向**，**人眼**就是**鉴频鉴相器**，**大脑**就相当于**滤波器**，**方向盘**就相当于**VCO**。人眼检测道路方向和行驶方向是否有偏差，大脑对方向偏差做出判断（偏左、偏右或无偏差），并将判断结果转换为方向盘的动作（左打、右打或不变），然后将调整后的行驶方向与道路方向再次通过人眼进行对比，经过反复的对比和调整后，最终使得行驶方向与道路方向一致。

接下来我们讲解一下 **PLL 分频和倍频的工作原理**， **PLL 分频原理图**如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/73a5060ae07b48308f89957da1acfaa5.png)

**分频**是在参考时钟与 PFD 之间加入一级分频器（可称为前置分频器），通过前置分频器 N (N 表示数字) 分频后得到一个新的参考时钟，因此需要比较的时钟频率（即 pll_out） 就始终是和新的参考时钟频率进行对比的， pll_out 的输出结果也会逐渐与新的参考时钟 (ref_clk/N) 相等，从而实现了分频的功能。

**PLL 倍频原理图**如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/b2293730d6dd401fb6589bfb15c69795.png)


**倍频**是在 VCO 与 PFD 之间加入一级分频器（可称为后置分频器），通过后置分频器 M(M 表示数字)分频后得到一个新的需要比较的时钟频率（即 pll_out 分频后的时钟），因为此时与参考时钟频率进行对比的是分频后的输出时钟（pll_out/M），所以此时的输出时钟是参考时钟的 M 倍（pll_out = ref_clk * M），从而实现了倍频的功能。

需要注意的是，一个 PLL IP 核输出的时钟路数是有限的，且输入/输出的时钟频率也是有限制的，我们不能无限制的输入无穷大/小的时钟频率，也不可能通过倍频或分频输出无穷大/小的时钟频率。这里我总结了一下**几款常用芯片的相关信息**，如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/a86dc17cec774e15b3c6db146b427f41.png)

# 3 程序设计
## 3.1 PLL IP核配置（基于Vivado）

1. 首先我们创建一个名为 **“ip_clk_wiz”** 的空工程， 然后点击 Vivado 软件左侧 **“Flow Navigator”** 栏中的 **“IP Catalog”** ，如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/46c32ff7f96e4c02b9cd0973604e7a60.png)


2. 点击 **“IP Catalog”** 后会弹出 **“IP Catalog”** 窗口，在搜索栏中输入 **“clock”** 关键字，可以看到 Vivado 已经自动查找出了与关键字匹配的 IP 核名称，如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/a15cddc7cac344de895d9655c2b5cdc3.png)

3. 我们双击 **“FPGA Features and Design”** → **“Clocking”** 下的 **“Clocking Wizard”** ，弹出 **“Customize IP”** 窗口，如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/b0750284896d4e17ac564e97da95b736.png)

4. 接下来就是配置 IP 核的时钟参数了， 第一个 **“Clocking Options”** 选项卡的参数配置如下所示：



![在这里插入图片描述](https://img-blog.csdnimg.cn/0c72b644ee594a99994d1ee79ce0f8bc.png)


5. 接下来切换至 **“Output Clocks”** 选项卡，该选项卡中的参数配置如下：


![在这里插入图片描述](https://img-blog.csdnimg.cn/1cb12d769f2f486b8eb2347d5bb6aaae.png)

我们重点关注红框中的内容，其中：

* 第一列 **“Output Clock”** 为设置输出时钟的路数，因为我们需要输出四路时钟，所以勾选前 4 个时钟。
* 第二列 **“Port Name”** 为设置时钟的名字，这里我们可以保持默认的命名。
* 第三列 **“Output Freq(MHz)”** 为设置输出时钟的频率，这里我们要对 **“Requested（即理想值）”** 进行设置，我们将四路时钟的输出频率分别设为 100、 100、 50 和 25，设置完理想值后，我们就可以在 **“Actual”** 下看到其对应的实际输出频率。需要注意的是 PLL IP 核的时钟输出范围为 6.25MHz~800MHz，但这个范围是一个整体范围，根据驱动器类型的选择不同，其所支持的最大输出频率也会有所差异。
* 第四列 **“Phase (degrees)”** 为设置时钟的相位偏移，同样的我们只需要设置理想值，这里我们将第二路
100MHz 的时钟输出信号的相位偏移设置为 180，其余三路信号不做相位偏移处理。


6. 接着我们来到 **“Port Renaming”** 选项卡， 如下图所示：



![在这里插入图片描述](https://img-blog.csdnimg.cn/aec2bb83c81e4f19b2386f1c9b7719ea.png)

**“Port Renaming”** 选项卡主要是对一些控制信号（复位信号以外的信号） 的重命名。 在上一个选项卡中我们启用了锁定信号 locked，因此这里我们只看到了 locked 这一个可以重命名的信号， 因为默认的名称已经可以让我们一眼看出该信号的含义，所以无需重命名，保持默认即可。

7. **“PLLE2 Setting”** 选项卡展示了对整个 PLL 的最终配置参数，这些参数都是由 Vivado 根据之前用户输入的时钟需求来自动配置的， Vivado 已经对参数进行了最优的配置，在绝大多数情况下都不需要用户对它们进行更改，也不建议更改，所以这一步保持默认即可，如下图所示：


![在这里插入图片描述](https://img-blog.csdnimg.cn/2f333b8c9bad416398205f2f058b3924.png)

8. 最后的“Summary” 选项卡是对前面所有配置的一个总结，在检查没问题后我们点击“OK” 按钮，如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/e59c64e594024f178847bfd46daeef09.png)





9. 接着就弹出了 **“Generate Output Products”** 窗口，我们直接点击 **“Generate”** 即可，如下图所示：



![在这里插入图片描述](https://img-blog.csdnimg.cn/f0241b1df9654a50b34546e12e9b21b4.png)


10. 之后我们就可以在 **“Design Runs”** 窗口的 **“Out-of-Context Module Runs”** 一栏中看到该 IP 核对应的**run “clk_wiz_0_synth_1”** ，其综合过程独立于顶层设计的综合，所以此时我们可以看到其正在综合，在其 Out-of-Context 综合的过程中，我们就可以开始编写代码来调用我们设置好的 IP 核了，如下图所示：




![在这里插入图片描述](https://img-blog.csdnimg.cn/9bc82f7df7364f04ad6a653d98d24111.png)

## 3.2 模块设计

本次实验的目的是**通过 PLL IP 核输出四路不同频率或相位的时钟**，因此给模块命名为 **ip_clk_wiz** 。首先想要输出四路不同频率或相位的时钟，就需要输入一个基准时钟，因此实验需要用到系统时钟.

其次为了使程序能恢复至默认状态，系统复位在 FPGA 系统中也是必不可少的；由此可以分析出，本次实验需要系统时钟和系统复位这两个输入端口，以及四个时钟输出端口，经分析画出模块框图如下所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/73a44064d1a743c8a37b88e8ac4b99cc.png)

**模块端口与功能描述**如下图所示：


![在这里插入图片描述](https://img-blog.csdnimg.cn/1a9c0b007e664e52a33d560e630caf69.png)


## 3.3 绘制波形图


我们开发板的系统时钟为 **50MHz**，即一个时钟周期为 **20ns**。 **100MHz** 的时钟一个时钟周期为 **10ns**；100MHz 相位偏移 180°的时钟就相当于将 100MHz 的时钟的高低电平变化做了一个**反相处理**； **50MHz** 的时钟和系统时钟相同； **25MHz** 的时钟一个时钟周期为 **40ns**。

需要注意的是， PLL IP 核会输出一个时钟锁定信号**（locked）**，所有通过 PLL IP 核产生的时钟都是在 **locked** 为高电平（即时钟已锁定）时才会输出稳定的时钟信号。由此我们可以绘制出如下所示的 **ip_clk_wiz** 模块波形图。


![在这里插入图片描述](https://img-blog.csdnimg.cn/724d0acfbfe0496f8bb8d964f6841ec6.png)


这里需要说明一点， 在时钟锁定信号 **（locked）** 拉高之前，时钟信号是不稳定，既可能是高电平，也可能是低电平，还可能是不稳定的时钟信号 **（用示波器或频谱仪可以看到是飘忽不定的）**，这里绘制时就以低电平来涵盖了。


## 3.4 编写代码

首先打开 IP 核的例化模板，在 **“Source”** 窗口中的 **“IP Sources”** 选项卡中，依次用鼠标单击展开 **“IP”** - **“clk_wiz_0”** -      **“Instantitation Template”** 后，我们可以看到 **“clk_wiz.veo”** 文件，它是由 IP 核自动生成的只读的 verilog 例化模板文件，双击就可以打开它，在例化时钟 IP 核模块的时候，可以直接从这里拷贝，如下图所示：


![在这里插入图片描述](https://img-blog.csdnimg.cn/ddd761fdd68443d4870cadb9dd24b4ba.png)

接下来我们创建一个 verilog 源文件，其名称为 ip_clk_wiz.v，代码如下：

```
module  ip_clk_wiz(
    input               sys_clk        ,  //系统时钟
    input               sys_rst_n      ,  //系统复位，低电平有效
    //输出时钟
    output              clk_100m       ,  //100Mhz时钟频率
    output              clk_100m_180deg,  //100Mhz时钟频率,相位偏移180度
    output              clk_50m        ,  //50Mhz时钟频率
    output              clk_25m           //25Mhz时钟频率
    );

//wire define
wire        locked;

//*****************************************************
//**                    main code
//*****************************************************

//PLL IP核的例化
clk_wiz_0  clk_wiz_0
(
	// Clock out ports
	.clk_out1          (clk_100m       ),  // output clk_out1
	.clk_out2          (clk_100m_180deg),  // output clk_out2
	.clk_out3          (clk_50m        ),  // output clk_out3
	.clk_out4          (clk_25m        ),  // output clk_out4
	// Status and control signals
	.reset             (~sys_rst_n     ),  // input reset
	.locked            (locked         ),  // output locked
	// Clock in ports
	.clk_in1           (sys_clk        )   // input clk_in1
);      

endmodule
```

程序中例化了 **clk_wiz_0**，把 FPGA 的系统时钟 50Mhz 连接到 **clk_wiz_0** 的 **clk_in1**，系统复位信号连接到 **clk_wiz_0** 的 **reset**，由于配置时钟 IP 核时我们保持了默认的高电平复位，而输入的系统复位信号 **sys_rst_n**是低电平复位，因此要对系统复位信号进行取反。 **clk_wiz_0** 输出的 4 个时钟信号直接连接到顶层端口的四个时钟输出信号。

# 4 仿真验证

## 4.1 编写 TestBench

我们接下来对代码进行仿真，因为本实验我们只有系统时钟和系统复位这两个输入信号，所以仿真文件也只需要编写这两个信号的激励即可， TestBench 代码如下：

```
`timescale 1ns / 1ps        //仿真单位/仿真精度

module tb_ip_clk_wiz();

//parameter define
parameter  CLK_PERIOD = 20; //时钟周期 20ns

//reg define
reg     sys_clk;
reg     sys_rst_n;

//wire define
wire    clk_100m;      
wire    clk_100m_180deg;
wire    clk_50m;     
wire    clk_25m;        

//信号初始化
initial begin
    sys_clk = 1'b0;
    sys_rst_n = 1'b0;
    #200
    sys_rst_n = 1'b1;
end

//产生时钟
always #(CLK_PERIOD/2) sys_clk = ~sys_clk;

ip_clk_wiz u_ip_clk_wiz(
    .sys_clk          (sys_clk        ),
    .sys_rst_n        (sys_rst_n      ),

    .clk_100m         (clk_100m       ),
    .clk_100m_180deg  (clk_100m_180deg),
    .clk_50m          (clk_50m        ),
    .clk_25m          (clk_25m        )  
    );

endmodule
```


## 4.2 代码仿真

在 **“Flow Navigator”** 窗口中点击 **“Run Simulation”** 并选择 **“Run Behavioral Simulation”**，如下图所示：

![在这里插入图片描述](https://img-blog.csdnimg.cn/d45f913c5fdb4ae49e5476d66985e97c.png)


经过稍许时间的等待，我们就进入了 Vivado 自带的仿真器界面，如下图所示：


![在这里插入图片描述](https://img-blog.csdnimg.cn/132e4bc1d7b642f58068c36bf02f8109.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/d3fc373d82c0410da5a0489cea6a3a2d.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/eb0c54e746114260b07a0be12f2b29b5.png)

由上述图可知， **locked** 信号拉高之后，锁相环开始输出 4 个稳定的时钟。 **clk_100m** 和 **clk_100m_180deg** 周期都为 10ns，即时钟频率都为 100Mhz，但其中 **clk_100m_180deg** 相对于系统时钟相位偏移了 180 度；**clk_50m** 周期为 20ns， 即时钟频率为 50Mhz； **clk_25m** 周期为 40ns， 即时钟频率为 25Mhz。也就是说，我们创建的锁相环从仿真结果上来看是正确的。

# 5 总结
本文讲解了如何基于Vivado软件来调用IP核，并详细讲解了PLL实验，最终仿真测试成功。读者如果有兴趣，可以进一步做验证实验，也就是进行**引脚约束**和**上板验证**，最终使用**示波器**观察时钟的波形图。因为考虑到每个人手里板卡不一样，本文未作此部分验证，本文重点在于讲解IP核的调用以及PLL的原理，希望对您有所帮助，有兴趣的朋友可以进一步联系我交流。


微博：沂舟Ryan ([@沂舟Ryan 的个人主页 - 微博 ](https://weibo.com/u/7619968945))

GitHub：[ChinaRyan666](https://github.com/ChinaRyan666)

微信公众号：沂舟无限进步

如果对您有帮助的话请点赞支持下吧！



**集中一点，登峰造极。**
