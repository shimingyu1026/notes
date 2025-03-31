## 验证平台组成

* driver：施加激励
* scoreboard：checker
* monitor：收集输出
* reference model

## 搭建测试平台

所有组件应该派生与uvm类

* uvm_driver :new()两个参数：mane和uvm_component。

实现main_phase=实现driver

* factory 机制： 集成在 `uvm_component_utils(my_driver) `宏中，将my_driver加入内部表中。使用 `run_test("my_driver");`运行

**所有派生自 `uvm_component`的类都应该使用 `uvm_component_utils`注册**

* objection机制：在每一个phase，UVM检查objection是否被提起（raise_objection），等到drop_objection停止仿真
* virtual interface：在class中使用类似 `virtualmy_ifvif;`语句就行声明。使用config_db机制进行层次通信：使用set与get，由四个参数，相互之间第三个参数必须一致。在build_phase里写，会自动执行。

## 加入各个组件

**transaction**概念：继承自uvm_sequence_item。function void post_randomize();是SV的函数，类的randomize调用之后post_randomize()会自动调用。

基于 uvm_object_utils(my_transaction) 实现；首先使用randomize()方法初始化，然后使用一个自定义task传输数据

* **env**：继承自uvm_env，实例化各种组件。使用drv = my_driver::type_id::create("drv", this);形式实例化class。
* **monitor**：monitor把DUT数据转换为transaction交给后续组件。继承自uvm_monitor。在env中实例化。

**agent封装 ：** 所有agent继承自uvm_agent类，把driver和monitor封装。形成树状结构。

**ref model:** 继承自uvm_component。使用uvm_analysis_port传送数据，参数就是这个analysis_port需要传输的数据类型，在monitor中当收集完一个transaction，使用write方法写入。使用uvm_blocking_get_port接收数据，使用get方法接收数据。需要在env中使用fifo链接（uvm_tlm_analysis_fifo），在内建connect_phase里进行链接

**scoreboard：**  继承自uvm_scoreboard，用于比较数据。

**field_automation机制**：使用类似：

```
`uvm_object_utils_begin(my_transaction)  
	`uvm_field_int(dmac, UVM_ALL_ON)  
	`uvm_field_int(smac, UVM_ALL_ON)  
	`uvm_field_int(ether_type, UVM_ALL_ON)  
	`uvm_field_array_int(pload, UVM_ALL_ON)  
	`uvm_field_int(crc, UVM_ALL_ON)   
`uvm_object_utils_end
```

注册所有字段，可以直接调用transaction的copy，compare，print函数，以及driver中的pack_bytes方法，monitor的unpack_bytes方法。
