# systemverilog

## 1. 数据类型

SV中新引入了一个数据类型logic，便验证人员驱动和连接硬件模块而省去考虑使用reg还是使用wire的精力。

* logic为四值逻辑，即可以表示0、1、X、Z。（integer、reg、logic、net-type）
* bit为二值逻辑，只可以表示0和1。byte、shortint、int、longint、bit）

## 2. 参数使用

例如：

```
module ctrl_regs5
#（parameter int addr_width=8,
parameter int data_width=32）
（
input[addr_width-1:0]cmd_addr_i,
input[data_width-1:0]　cmd_data_i,
output[data_width:0]cmd_data_o,
）;
endmodule
```

对此，可以在模块例化时再决定端口的宽度，例如：

```
trl_regs5 #（.addr_width（16））regs5_inst（...）;
```

代码编译运行的过程分为三个部分：

* 编译阶段（compilation）：工具通过阅读目标代码，进行语法和语义分析，将每个模块分别编入库中（library）。
* 建模阶段（elaboration）：工具将各模块按照设计集成关系最终组成顶层模块。这一过程包括各模块（module）的例化、接口（interface）例化、程序（program）例化、层次集成、计算参数、解决层次信号引用、建立模块连接等。这一过程发生在编译阶段之后、仿真阶段之前，类似于软件编译的link阶段。
* 仿真阶段（simulation）：通过读取建模阶段的对象文件，建立硬件RTL模型和验证环境，以周期驱动（cycle-driven）或事件驱动（event-driven）的方式进行仿真。

VCS可以在独立的elaboration阶段修改参数。

## 3. 宏定义

公共使用的宏存放在公共空间作为头文件（header file）

```
'define ADDR_WIDTH 6
'define DATA_WIDTH 32
module ctrl_regs6
（
input['ADDR_WIDTH-1:0]cmd_addr_i,
input['DATA_WIDTH-1:0]cmd_data_i,
output['DATA_WIDTH:0]cmd_data_o,
）;
endmodule
```

## 4. 接口

![1743084273821](image/SV/1743084273821.png)

我们采用接口（interface）进行stimulator与DUT的连接。interface的基本作用是对各个模块做清晰有序的连接,可以看做一捆智能的集束线（collector）。能将DUT同testbench的连接隔离开。

### 接口连接方式

该方式则定义了三个接口：regs_cf_if、regs_ini_if和regs_rsp_if。

![1743084454484](image/SV/1743084454484.png)


```
interface regs_cr_if;
logic clk;
logic rstn;
endinterface: regs_cr_if

interface regs_ini_if
#（parameter int addr_width=8,
parameter int data_width=32）;
logic　　　　　　　　　　clk;
logic　　　　　　　　　　rstn;
logic[1:0]　　　　　　cmd;
logic[addr_width-1:0]　cmd_addr;
logic[data_width-1:0]　cmd_data_w;
logic[data_width-1:0]　cmd_data_r;
endinterface: regs_ini_if

interface regs_rsp_if;
logic　　　　clk;
logic　　　　rstn;
logic[7:0]slv0_avail;
logic[7:0]slv1_avail;
logic[7:0]slv2_avail;
logic[2:0]slv0_len;
logic[2:0]slv1_len;
logic[2:0]slv2_len;
logic[1:0]slv0_prio;
logic[1:0]slv1_prio;
logic[1:0]slv2_prio;logic　　　　slv0_en;
logic　　　　slv1_en;
logic　　　　slv2_en;
endinterface:　 regs_rsp_if
```

唯一不能使用logic变量的是含有多驱动（multi-drive）的场景，这时必须使用连线类型（如wire）。

### 接口应用

regs_cr_if的功能是提供时钟和复位信号，这里我们可以在regs_cr_if中产生时钟和复位信号。

```
interface regs_cr_if;
logic clk;
logic rstn;
initial begin
clk <=0;
forever begin
#5ns clk <=!clk;
end
end
initial begin
#20ns;
rstn <=1;
#40ns;
rstn <=0;#40ns;
rstn <=1;
end
endinterface: regs_cr_if
```

接口本身既要与DUT连接，也要与stimulator和monitor连接，不同对象的连接信号方向也是不同的。为了限制不同对象对其信号的访问权限和方向，接口通过modport做进一步的声明，以确立信号连接的方向。

```
interface regs_ini_if;
...//省略了接口信号声明
modport dut（
input　cmd,cmd_addr,cmd_data_w,
output cmd_data_r
）;
modport stim（
input　cmd_data_r,
output cmd,cmd_addr,cmd_data_w
）;
modport mon（input cmd,cmd_addr,cmd_data_w,cmd_data_r
）;
endinterface: regs_ini_if
```

在regs_ini_if实体中，进一步定义了3个modport:dut,stim和mon。它们分别用来作为DUT,stimulator和mon的“插座”，这样一方面澄清了各个对象可以连接的interface信号，一方面通过限制方向避免了反向驱动的问题。

![1743085270654](image/SV/1743085270654.png)

## 5. SV仿真调度机制

SV的仿真调度完全支持Verilog的仿真调度，同时又扩展出来支持新的SV结构体，如程序（program）和断言（assertion）

![1743085913655](image/SV/1743085913655.png)

建议将设计部分放置在module中，而将测试采样部分放置在program中。program中的initial块（类软件的执行方式）会在reactive区域被执行，而program之外的initial块（module内部）则在active区域被执行。

## 6. SV组件实现

### 激励发生器的驱动

Stimulator（激励发生器）是生成激励的源。

#### 激励驱动的方法

以regs_ini_if相连接的stimulator ini_stim为例：

```
module stm_ini（
input clk,
input rstn,output[1:0]cmd,
output[7:0]cmd_addr,
output[31:0]cmd_data_w,
input [31:0]cmd_data_r
）;
localparam IDLE=2＇b00;
localparam RD =2＇b01;
localparam WR =2＇b10;
logic[1:0]v_cmd;
logic[7:0]v_cmd_addr;
logic[31:0]v_cmd_data_w;
assign cmd=v_cmd;
assign cmd_addr=v_cmd_addr;
assign cmd_data_w=v_cmd_data_w;
typedef struct{ //trans 数据类型定义
bit[1:0]cmd;
bit[7:0]cmd_addr;
bit[31:0]cmd_data_w;bit[31:0]cmd_data_r;
} trans;
trans ts[3]; //trans固定数组声明和初始化
task op_wr（trans t）; //写指令定义
task op_rd（trans t）; //读指令定义
task op_idle（）; //空闲指令定义
task op_parse（trans t）; //指令类型解析
...//指令分发即产生激励
endmodule
```

stm_ini在内部声明了多个方法（methods），即op_wr、op_rd、op_idle和op_parse，且它们驱动硬件信号。需要先声明几个变量 v_cmd、v_cmd_addr、v_cmd_data_w，这是因为方法内部的非阻塞赋值只能引用logic类型或者reg类型，而无法直接对 stm_ini 的端口（wire 类型）赋值。

方法op_wr有一个参数trans t，若未标明传递方向，则默认为输入端（input trans t）。在时钟的上升沿将变量t中的t.cmd、t.cmd_addr和t.cmd_data_w分别写入硬件信号中，最终触发一次写操作。

```
task op_wr（trans t）;
@（posedge clk）;
v_cmd <=t.cmd;
v_cmd_addr <=t.cmd_addr;
v_cmd_data_w <=t.cmd_data_w;
endtask
```

这里要再声明一个方法 op_parse（），它可以根据参数的命令类型决断调用哪种指令操作方法。

```
task op_parse（trans t）;
case（t.cmd）
WR: op_wr（t）;
RD: op_rd（t）;
IDLE: op_idle（）;
default: $error（＂Invalid CMD!＂）;
endcase
endtask
```

在stm_ini模块的最后，又声明了一个initial块来产生最终的激励：

```
initial begin: stmgen
@（posedge rstn）; //等待复位释放
foreach（ts[i]）begin //解析ts数组中每个成员，从ts[0]至ts[2]
op_parse（ts[i]）; //调用解析方法
end
repeat（5）@（posedge clk）; //连续等待5个时钟上升沿
$finish（）; //主动结束仿真
end
```

#### 任务和函数
