## 验证平台组成

* driver：施加激励
* scoreboard：checker
* monitor：收集输出
* reference model

## 搭建测试平台

所有组件应该派生与uvm类

* uvm_driver :new()两个参数：mane和uvm_component。

实现main_phase=实现driver

* factory 机制： 集成在 `uvm_component_utils(my_driver) `宏中，将my_driver加入内部表中。使用`run_test("my_driver");`运行

**所有派生自`uvm_component`的类都应该使用`uvm_component_utils`注册**
