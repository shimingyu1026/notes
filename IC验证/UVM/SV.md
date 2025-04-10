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

### 6.1 激励发生器的驱动

Stimulator（激励发生器）是生成激励的源。

#### 6.1.1 激励驱动的方法

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

#### 6.1.2 任务和函数

ask与function的参数列表中均可以声明多个input（输入）、output（输出）、inout（输入输出）和ref（引用）类型。ref 类似于软件中的指针，在调用方式时不会有任何复制行为，而是直接引用或修改外部传入的数据对象。

#### 6.1.3 数据生命周期

在SV中，我们将数据的生命周期分为两类：

* automatic（动态）；
* static（静态）。

static 在仿真过程中的任何时刻都可以被共享，且不会被销毁。而automatic变量则与软件的局部变量一样，在它的作用域生命结束时被销毁回收存储空间：

```
function automatic int auto_cnt（input a）;
int cnt=0;
cnt+=a;
return cnt;
endfunction

function static int static_cnt（input a）;
static int cnt=0;
cnt+=a;
return cnt;
endfunction

function int def_cnt（input a）;
static int cnt=0;
cnt+=a;
return cnt;
endfunction

initial begin
$display（＂@1 auto_cnt=%0d＂,auto_cnt（1））;
$display（＂@2 auto_cnt=%0d＂,auto_cnt（1））;
$display（＂@1 static_cnt=%0d＂,static_cnt（1））;
$display（＂@2 static_cnt=%0d＂,static_cnt（1））;
$display（＂@1 def_cnt=%0d＂,def_cnt（1））;
$display（＂@2 def_cnt=%0d＂,def_cnt（1））;
end
```

输出结果为：

```
# @1 auto_cnt=1
# @2 auto_cnt=1
# @1 static_cnt=1
# @2 static_cnt=2
# @1 def_cnt=1
# @2 def_cnt=2
```

#### 6.1.4 通过接口直接驱动

可以通过virtual interface（虚接口）在stimulator内部直接做采样或者驱动。

```
module stm_ini;
virtual interface regs_ini_if vif;
trans ts[]; //trans 动态数组声明

initial begin: stmgen //指令分发即产生激励
wait（vif !=null）;
@（posedge vif.rstn）;
foreach（ts[i]）begin
op_parse（ts[i]）;
end
repeat（5）@（posedge vif.clk）;
$finish（）;
end
endmodule

module tb;
//例化接口
regs_cr_if　crif（）;
regs_ini_if iniif（）;
ctrl_regs dut（...）;
stm_ini ini（）;
initial begin: arrini
ini.ts=new[3];
ini.ts[0].cmd　　　　=WR;
ini.ts[0].cmd_addr　　=0;
ini.ts[0].cmd_data_w　=	32＇hFFFF_FFFF;
ini.ts[1].cmd　　　　=RD;
ini.ts[1].cmd_addr　　=0;
ini.ts[2].cmd　　　　=IDLE;
end
initial begin: setif //传递接口
ini.vif=iniif;
end
endmodule
```

动态数组的使用和外部初始化使得TB将stimulator的驱动功能和test vector（测试向量）生成这两个任务清晰地剥离开，尽量保证 stm_ini 只完成驱动功能。

### 6.2 激励发生器的封装（Class）

将之前定义的module stm_ini和struct trans改造为class stm_ini和class trans

```
class trans;
bit[1:0]cmd;
bit[7:0]cmd_addr;
bit[31:0]cmd_data_w;
bit[31:0]cmd_data_r;
endclass

class stm_ini;
virtual interface regs_ini_if vif;
trans ts[];
task op_wr（trans t）;
task op_rd（trans t）;
task op_idle（）;
task op_parse（trans t）;

task stmgen（）;
wait（vif !=null）;
@（posedge vif.rstn）;
foreach（ts[i]）begin
op_parse（ts[i]）;
end
endtask
endclass
```

class trans定义了内部的成员，而class stm_ini则定义了成员变量和成员方法。类内部的方法必须是task或function，不能使用module中的硬件过程块always或initial。

我们将之前的module tests拆分为三个类，即class basic_test、class test_wr和class_rd。

![1743132107774](image/SV/1743132107774.png)

#### 6.2.1 虚方法

![1743134410929](image/SV/1743134410929.png)

虚方法使用 virtual 定义：

```
class basic_test;
virtual task test（stm_ini ini）;
$display（＂basic_test::test＂）;
endtask
endclass
```

会被继承的class覆盖

* 在为父类定义方法时，如果该方法日后可能会被覆盖或继承，那么应该声明为虚方法。

#### 6.2.2 句柄

有一些类成员无法通过虚方法来解决索引：

* 父类没有定义，只在子类中定义了的方法；
* 父类没有声明，只在子类中声明了的变量；
* 父类和子类同时声明了的变量。

需要注意句柄悬空的问题：

```
initial begin
wr=new（）;
t=wr;
if（t==null）
$error（＂invalid handle t and wr＂）;$display（＂wr.def=%0d＂,wr.def）;
$display（＂t.def=%0d＂,t.def）;
end
```

使用$cast（）函数检查句柄赋值是否合法：

```
initial begin
wr=new（）;
t=wr;
if（!$cast（hwr,t））
$error（＂cannot assign t to hwr＂）;
if（!$cast（hrd,t））
$error（＂cannot assign t to hrd＂）;end
```

### 6.3 激励发生器的随机化

#### 6.3.1 约束求解器

#### 6.3.2 随机变量

```
class trans;  // 定义名为trans的类
  rand bit[1:0] cmd = WR;       // 2位随机命令，默认值WR（需预先定义WR常量）
  rand logic[7:0] cmd_addr;     // 8位随机命令地址
  rand bit[31:0] cmd_data_w;    // 32位随机写入数据
  bit[31:0] cmd_data_r;        // 32位非随机读取数据（存储用）  // 约束块1：cmd只能取IDLE/RD/WR（需预定义这些枚举值）
  constraint c1 { cmd inside {IDLE, RD, WR}; };  // 约束块2：地址只能是0x0/4/8/10/14/18（修正了中文单引号）
  constraint c2 { cmd_addr inside {'h0, 'h4, 'h8, 'h10, 'h14, 'h18}; };  // 约束块3：写入数据高26位必须为0（限制数据范围）
  constraint c3 { cmd_data_w[31:6] == 0; };  // 打印对象内容的函数
  function void print();
    $display("cmd='h%0x", cmd);           // 显示命令（16进制）
    $display("cmd_addr='h%0x", cmd_addr); // 显示地址（16进制）
    $display("cmd_data_w='h%0x", cmd_data_w); // 显示写入数据
    $display("cmd_data_r='h%0x", cmd_data_r); // 显示读取数据
  endfunction
endclass
```

使用 randomize（）方法生成随机值

RNG（随机数生成器，random number generator）。通过相同的初始RNG，结合相同的种子，就产生了稳定一致的随机数序列。

#### 6.3.3 随机化的流程控制

我们就可以在对象的随机前（pre_randomize（））函数和随机后（post_randomize（））函数中做一些处理。使得后面随机化的生成可以从之前的结果中得到“启示”

简单例子：

```
class c1;
  rand int randnum;     // 可随机化的整数变量，范围待约束
  int hist[$];         // 队列（历史记录），存储已生成的随机数  constraint cstr1 { randnum inside {[0:10]}; };  // 约束1：randnum ∈ [0,10]
  constraint cstr2 { !(randnum inside {hist}); }; // 约束2：randnum 不在 hist 中  function void post_randomize();
    hist.push_back(randnum);  // 随机化后，将 randnum 存入 hist
  endfunction
endclass
```

#### 6.3.4 随机化的系统函数

* $random（int seed）：返回32位有符号随机数，参数种子值是可选的
* $urandom（int seed）：返回32位无符号随机数，参数种子值是可选的。
* $urandom_range（int unsigned MAX,int unsigned MIN=0）：在指定范围内产生无符号随机数。

### 6.4 监测器的采样 monitor

monitor的功能核心就是从interface做数据采样（sampling）和打包（packaging）送给checker。

#### 6.4.1 Interface clocking

一个典型的clocking定义：

```
clocking bus @（posedge clock1）;
default input #10ns output #2ns;
input data,ready,enable=top.mem1.enable;
output negedge ack;
input #1step addr;//上一个时钟周期的数据
endclocking
```

monitor 采样

event用法：

```
class car;
event e_start;

task launch（）;
-> e_start;
$display（＂car is launched＂）;
endtask

task move（）;
wait（e_start.triggered）;
$display（＂car is moving＂）;
endtasktask drive（）;
fork
this.launch（）;
this.move（）;
join
endtask

endclass
```

#### 资源共享/通信

可以通过semaphore（旗语）解决资源共享这个问题。

使用mailbox多线程间通过邮箱传递数据：

```
// 定义事务类型
class transaction;
  int id;
  int data;
  function new(int id, int data);
    this.id = id;
    this.data = data;
  endfunction
endclass
module mailbox_example;
  // 声明一个无容量限制的mailbox（可存储任意数量事务）
  mailbox #(transaction) mbx = new();  // 生产者任务：生成事务并发送到mailbox
  task producer();
    transaction tr;
    for (int i = 0; i < 5; i++) begin
      tr = new(i, $urandom_range(100));  // 创建事务对象
      display("[Producer] @%0t: Sent transaction(id=%0d, data=%0d)", display("[Producer]@time, tr.id, tr.data);
      mbx.put(tr);                       // 将事务放入mailbox
      #10;                               // 模拟处理延迟
    end
    display("[Producer] @%0t: All transactions sent.", display("[Producer]@time);
  endtask  // 消费者任务：从mailbox接收事务并处理
  task consumer();
    transaction tr;
    forever begin
      mbx.get(tr);  // 阻塞等待直到mailbox中有数据
      display("[Consumer] @%0t: Received transaction(id=%0d, data=%0d)", display("[Consumer]@time, tr.id, tr.data);
      #20;          // 模拟处理延迟
    end
  endtask  // 主测试逻辑
  initial begin
    fork
      producer();  // 启动生产者线程
      consumer();  // 启动消费者线程
    join_none
    #200;          // 等待所有操作完成
    $finish;
  end
endmodule
```

仿真输出为：

```shell
[Producer] @0: Sent transaction(id=0, data=83)
[Consumer] @0: Received transaction(id=0, data=83)
[Producer] @10: Sent transaction(id=1, data=11)
[Producer] @20: Sent transaction(id=2, data=97)
[Consumer] @20: Received transaction(id=1, data=11)
[Producer] @30: Sent transaction(id=3, data=42)
[Producer] @40: Sent transaction(id=4, data=75)
[Consumer] @40: Received transaction(id=2, data=97)
[Producer] @40: All transactions sent.
[Consumer] @60: Received transaction(id=3, data=42)
[Consumer] @80: Received transaction(id=4, data=75)
```

* event：最小信息量的触发，即单一的通知功能。可以用来做事件的触发，也可以组合多个event做线程之间的同步。
* semaphore：共享资源的安全卫士。多个线程访问某一公共资源时可以使用这个要素。
* mailbox：精小的SV原生FIFO。在线程之间做数据通信或内部数据缓存时可以考虑使用此元素

### 6.5 比较器和参考模型

#### 6.5.1 异常检查

主要检查复位

#### 6.5.2 常规检查

要求观察DUT的三个方面：配置情况、输入数据、输出数据。

1. **拆分检查：**

   将DUT检查的功能点有机剥离，并对每个功能点做独立的检查。
2. **整体检查：**

   将DUT的配置和接口数据作为参考模型的输入，参考模型会模拟设计功能并输出期望数据，再将其与监测的DUT输出数据做比较。

TODO
